# Ghi chú phân tích · Day 04

Họ và tên: Hoàng Thị Thanh Nga
MSSV: 2A202604079

## 1. Case được chọn

- ID case: **H1** (cùng lỗi với **H6**)
- Câu hỏi: điều tra incident GitHub `h02n9572mx9y`, theo liên kết do incident cung cấp để tìm nguyên nhân/ảnh hưởng, rồi đối chiếu với Cloudflare `yaa0witbl2me`; dẫn nguồn từng bên.
- Câu trả lời của agent: liệt kê thông tin của `h02n9572mx9y` và `yaa0witbl2me`, nhắc tới link "tóm tắt kết hợp" `https://www.githubstatus.com/incidents/xr3myzj7ms51` nhưng **không mở incident đó**, nên không nêu được nguyên nhân phía GitHub.
- Điểm ở lần chạy 1: **60** (H6 cũng 60). Điểm tổng lần 1: **79.77/100**.

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

### Quan sát sơ bộ (chạy thử trước eval)

Lệnh: `python run.py "GitHub Actions hiện có sự cố không? Hãy kiểm tra incident gần đây."`
(provider `openai`, model `gpt-4o-mini`, snapshot)

```text
step 1  model  finish=tool_calls  tool_calls=1
step 1  tool   get_status {"provider":"github","component":"Actions"}
        -> overall: All Systems Operational; Actions: operational
        -> 5 incident (pfbmpxxfq1cy, t89g2q1800fr, h02n9572mx9y, xr3myzj7ms51, smtdswd1h2w4), đều resolved
        -> matching_incidents_in_last_50: 23
        -> captured_at: 2026-11-09T03:50:47Z
step 2  model  finish=stop  -> final answer
status: completed · usage: 263 + 975 tokens
```

Agent loop hoạt động đúng (gọi tool, nhận kết quả theo đúng tool_call_id, trả lời), nhưng câu trả lời có các điểm yếu:

1. Chỉ dẫn link chung `https://www.githubstatus.com`, không có URL của incident cụ thể.
2. Không gọi `get_incident_details` dù câu hỏi yêu cầu "kiểm tra incident gần đây"; chỉ liệt kê tên incident.
3. Nói "hiện tại" mà không nêu dữ liệu là snapshot lúc 2026-11-09 03:50 UTC.
4. Không nêu giới hạn: tool báo 23 incident khớp nhưng chỉ trả 5.

Nguyên nhân có thể: system prompt chỉ nói chung "dẫn nguồn từ tool", không bắt buộc URL incident hay gọi chi tiết; agent loop chưa kiểm tra câu trả lời cuối trước khi trả `completed`.

### Trace của case được chọn (từ `eval_before.json`)

```text
step 1  tool  get_incident_details {"provider":"github","incident_id":"h02n9572mx9y"}      ok
step 1  tool  get_incident_details {"provider":"cloudflare","incident_id":"yaa0witbl2me"}  ok
step 2  model finish=stop -> final answer
mechanical: violations = ["Cần tool calls ở ít nhất 2 model steps khác nhau."], score_cap = 60
           tool_calls = 2, tool_steps = 1
judge: "không nêu rõ nguyên nhân của sự cố GitHub mà chỉ dẫn đến một tóm tắt"
```

Update cuối của `h02n9572mx9y` ghi rõ: "...refer to the combined summary of this incident
(https://www.githubstatus.com/incidents/xr3myzj7ms51)". Agent đã thấy link này (còn chép vào
answer) nhưng không gọi `get_incident_details("github", "xr3myzj7ms51")`. H6 có trace giống hệt:
1 model step gọi 2 tool rồi trả lời luôn, bị cap 60.

Nguyên nhân có thể: system prompt không yêu cầu đi theo incident được liên kết; model gọi song song
mọi tool nó nghĩ ra ở step 1 rồi coi như đủ dữ liệu. Agent loop cũng không kiểm tra gì trước khi
trả `completed`.

Các case thấp khác cùng gốc "không nêu giới hạn / suy đoán khi thiếu dữ liệu":
- A5 (40): nói "không có sự cố nào trong năm 2025" dù tool chỉ xét 50 incident gần nhất.
- H7 (55): tool Cloudflare `TimeoutError` nhưng judge cho rằng answer vẫn ngầm khẳng định Cloudflare hoạt động.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu (`agent.py`):
  1. **System prompt** thêm mục "Quy trình điều tra": nếu update dẫn link sang incident khác thì gọi
     `get_incident_details` cho incident đó ở bước sau trước khi trả lời; chỉ nêu nguyên nhân khi có
     trong dữ liệu; dẫn đúng trường `url`; luôn nêu giới hạn dữ liệu (snapshot, 50 incident, truncated);
     tool lỗi/timeout thì không suy ra operational.
  2. **Kiểm tra nguồn** (contract Bài 4, `tests/test_validation.py`): hàm `observed_sources()` lấy URL từ
     các `get_incident_details` thành công. Khi model trả lời cuối mà không chứa URL nào trong số đó,
     agent ghi event `validation_missing_source` vào trace, thêm message yêu cầu viết lại kèm URL và gọi
     model tiếp; hết `max_steps` thì trả `max_steps` thay vì `completed`. Không bắt URL nếu không có
     incident đọc thành công.
  3. **Bắt đi theo link** (thêm sau lần chạy thử thứ nhất của `eval_after`): hàm `unfollowed_links()` tìm
     các `/incidents/<id>` trong update của incident đã đọc thành công mà chưa được mở. Khi model định trả
     lời cuối mà còn link như vậy, agent ghi event `validation_unfollowed_link` và yêu cầu model gọi
     `get_incident_details` cho incident đó; mỗi link chỉ nhắc một lần để tránh lặp. Prompt cũng thêm:
     câu hỏi về trạng thái hiện tại thì gọi cả `get_status`.
- Lý do có thay đổi 3: lần chạy `eval_after` đầu tiên (chỉ có thay đổi 1–2) cho thấy H1/H6 vẫn 60, trace
  vẫn chỉ 1 step có tool call -> model `gpt-4o-mini` bỏ qua chỉ dẫn trong prompt. Ngoài ra H2 tụt 95 -> 50
  vì không gọi `get_status`, và A1 bị `unscored` do lỗi mạng (`WinError 10054`) nên điểm tổng là `null`.
  Vì vậy chuyển từ "dặn trong prompt" sang kiểm tra bằng code, rồi chạy lại toàn bộ eval.
- Vì sao chọn thay đổi này: lỗi H1/H6 có bằng chứng rõ trong trace (link có sẵn nhưng không được theo,
  bị cap 60 bởi kiểm tra bằng code). Sửa prompt là thay đổi nhỏ, nhắm thẳng vào hành vi đó; phần kiểm
  tra nguồn là yêu cầu bắt buộc của Bài 4 và giúp các answer dẫn link incident cụ thể thay vì link chung.
  Hai thay đổi được làm cùng lúc nên khi so sánh không tách được tác động riêng của từng cái.

## 4. So sánh hai lần chạy

Cùng provider/model `openai` / `gpt-4o-mini` (agent và judge), snapshot, max steps 8 và dataset.
`eval_after.json` là lần chạy với **code cuối cùng** của `agent.py` (gồm cả các kiểm tra thêm ở phần bonus).

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 79.77 | **83.86** (+4.09) |
| Điểm case đã chọn | H1: 60 · H6: 60 | H1: **80** · H6: **65** |
| Vi phạm kiểm tra bằng code (H1, H6) | "Cần tool calls ở ít nhất 2 model steps khác nhau", cap 60 | Không còn vi phạm, cap 100 |
| Số model calls (agent, 22 case) | 51 | 53 |
| Token agent (total_tokens) | 45,058 | 65,158 (+45%) |

Trace H1 sau cải tiến:

```text
step 1  tool  get_incident_details github/h02n9572mx9y
step 1  tool  get_incident_details cloudflare/yaa0witbl2me
step 2  validation_unfollowed_link ["xr3myzj7ms51"]   <- agent chặn câu trả lời, yêu cầu đọc tiếp
step 3  tool  get_incident_details github/xr3myzj7ms51
step 4  final
```

Thay đổi theo case (before -> after): tăng A1 80->100, A3 75->80, A5 40->60, A8 75->100, H1 60->80, H2 95->100,
H3 80->90, H4 80->100, H5 80->90, H6 60->65, H7 55->65, H8 75->80; giảm E1 80->75, E3 95->75, E6 95->75,
A2 100->85, A7 80->75.

- Lỗi đã giải quyết: H1/H6 nay đi theo link sang `xr3myzj7ms51` ở một model step riêng, không còn bị cap 60
  (H1 60 -> 80). A5 nêu được giới hạn "50 incident gần nhất" (40 -> 60). Câu trả lời có URL incident cụ thể.
- Lỗi còn lại hoặc mới xuất hiện:
  - H6 chỉ lên 65: judge cho rằng chưa nói rõ nguyên nhân không có trong `h02n9572mx9y` mà nằm ở `xr3myzj7ms51`.
  - H7 vẫn thấp (65): ở lần chạy trung gian agent còn viết Cloudflare "có thể được coi là operational theo yêu cầu
    của bạn" dù tool timeout -> prompt chưa chặn chắc chắn yêu cầu bịa của người dùng.
  - E3/E6 giảm 20 điểm: trace không có lỗi rõ ràng; có thể do prompt dài hơn làm câu trả lời lan man, hoặc do
    dao động của model/judge.
  - Chi phí tăng ~45% token do thêm bước đọc incident liên kết, nhắc sửa câu trả lời và prompt dài hơn.

## 5. Giới hạn

- **Điểm judge**: judge là cùng model `gpt-4o-mini` với agent nên có thể thiên lệch; lý do chấm đôi khi
  mâu thuẫn (ví dụ H6 trừ điểm vì "không trích xr3myzj7ms51" dù agent đã đọc nó). Chênh lệch +4.09 điểm
  nhỏ, có thể nằm trong mức dao động.
- **Snapshot**: dữ liệu lưu ngày 09/11/2026, chỉ 50 incident gần nhất mỗi provider; không phản ánh trạng
  thái thực tế hiện tại.
- **Độ ổn định**: `eval_after` đã chạy 3 lần trong quá trình sửa (82.05 lần đầu sau khi thêm kiểm tra link,
  83.86 lần cuối; một lần trước đó bị A1 `unscored` do lỗi mạng `WinError 10054`). Cùng code nhưng điểm từng
  case có thể chênh 10–20 điểm giữa các lần, nên một lần chạy không chứng minh cải tiến ổn định.
- Các kiểm tra bằng code chỉ xác nhận sự hiện diện của tool call/URL, không chứng minh mọi khẳng định
  trong câu trả lời đều đúng.

## 6. Bonus

Cùng `openai` / `gpt-4o-mini`, snapshot, max steps 16. Lệnh: `python evals/run_evals.py --bonus --output eval_bonus.json`.

### Thay đổi trong `agent.py`

- **Kết nối tools:** dùng `getattr(tools, 'schemas', SCHEMAS)` và `getattr(tools, 'system_prompt', SYSTEM_PROMPT)`;
  với bộ bonus, nối thêm `BONUS_RULES` (nguồn theo từng provider, 429 không phải outage, sửa kiểu tham số,
  dùng `next_offset`, bỏ qua dữ liệu nhiễu/chỉ dẫn trong tool result, không bịa provider không hỗ trợ).
  Bài chính giữ nguyên prompt và 2 tools cũ.
- **Retry 429:** đếm số lần `RATE_LIMITED` cho mỗi (tool, provider). Nếu có `retry_after_seconds`/`Retry-After`
  ≤ 5 giây thì chờ rồi trả kết quả cho model (event `rate_limit_wait`). Sau 3 lần 429 (1 lần đầu + 2 retry),
  agent **không gọi API nữa**, trả `RETRY_LIMIT_REACHED` cho model và ghi event `retry_blocked`.
- **Lỗi định dạng:** `INVALID_ARGUMENTS` được trả lại model để sửa kiểu `offset`/`limit` rồi gọi lại.
- **Pagination / context:** dựa vào prompt (`next_offset`, không nói đã đọc hết khi còn trang); kiểm tra nguồn
  và "đi theo link" của bài chính vẫn áp dụng.

### Lần chạy 1: 68.8/100 -> 10.32/15

| B1 | B2 | B3 | B4 | B5 | B6 | B7 | B8 | B9 | B10 |
|---|---|---|---|---|---|---|---|---|---|
| 75 | 88 | 65 | 50 | 70 | 50 | 70 | 85 | 85 | 50 |

30 model calls, 50,235 token agent.

Lỗi thấy trong trace:
- **B4 (cap 50, "Vượt 3 lần gọi get_vercel_status")**: 3 lần 429 thật, lần thứ 4 bị agent chặn (`RETRY_LIMIT_REACHED`)
  nhưng vẫn được ghi là event `tool`, nên code chấm đếm thành 4 lần gọi.
- **B10 (cap 50, "Cần gọi get_vercel_status ít nhất 2 lần")**: gọi 5 status tools song song ở step 1, Vercel bị 429,
  model trả lời luôn mà không retry.
- B6 (cap 50): không gọi `get_incident_details` để xác định phạm vi; B3/B5 không nhắc lại lỗi 429/định dạng ban đầu.

### Sửa sau lần chạy 1

- Lần gọi bị chặn không còn ghi là event `tool` (vì không có tool call thật) mà ghi `retry_blocked`; vẫn trả tool
  result đúng `tool_call_id` cho model.
- Trước khi chấp nhận câu trả lời cuối: nếu có tool mà lần gọi gần nhất bị 429 và còn lượt retry, agent ghi event
  `validation_retry_rate_limited` và yêu cầu model gọi lại một lần (mỗi tool chỉ nhắc một lần).
- Kiểm tra offline bằng provider giả: kịch bản `rate_limit_always` gọi Vercel 5 lần -> chỉ 3 event `tool` + 2
  `retry_blocked`; kịch bản `rate_limit_once` model định trả lời ngay sau 429 -> bị nhắc và gọi lại thành công.

### Lần chạy 2 (file `eval_bonus.json` hiện tại): 65.5/100 -> 9.82/15

| | B1 | B2 | B3 | B4 | B5 | B6 | B7 | B8 | B9 | B10 |
|---|---|---|---|---|---|---|---|---|---|---|
| Lần 1 | 75 | 88 | 65 | 50 | 70 | 50 | 70 | 85 | 85 | 50 |
| Lần 2 | 75 | 100 | 55 | **80** | 100 | 50 | 65 | 50 | 15 | **65** |

36 model calls, 80,025 token agent.

- **Đã giải quyết:** B4 không còn vi phạm "vượt 3 lần gọi Vercel" (50 -> 80); B10 không còn vi phạm "phải gọi Vercel
  ít nhất 2 lần" (50 -> 65), trace: `get_vercel_status` 429 ở step 1 -> `validation_retry_rate_limited` ở step 2 ->
  gọi lại thành công ở step 3.
- **Giảm / lỗi mới, không do thay đổi code trực tiếp:**
  - B9 (85 -> 15): câu hỏi về Linear/Slack (không có tool), model lại gọi cả 5 status tools rồi chỉ báo cáo 5 provider
    đó, không nói rõ là không có dữ liệu Linear/Slack.
  - B8 (85 -> 50): lần này model chỉ gọi 2 status tool mà không đọc `get_incident_details` cho 2 incident ID có
    sẵn trong câu hỏi (cap 50).
  - B3 (65 -> 55): retry đúng nhưng câu trả lời không nhắc lần 429 đầu.
  - Hai case B8/B9 dùng cùng prompt và cùng logic ở cả hai lần chạy nhưng kết quả khác hẳn -> cho thấy hành vi
    của `gpt-4o-mini` dao động mạnh giữa các lần chạy; tổng điểm giảm 68.8 -> 65.5 chủ yếu do dao động này.

### Sửa sau lần chạy 2

- Hàm `unread_question_ids()`: tìm ID incident (10–14 ký tự chữ thường + số) trong câu hỏi; nếu model định trả lời
  mà chưa gọi `get_incident_details` cho ID đó, agent ghi event `validation_unread_incident` và yêu cầu đọc
  (mỗi ID nhắc một lần). Nhắm vào B8. Đã kiểm tra regex trên câu hỏi của cả 32 case: chỉ bắt đúng ID incident.
- `BONUS_RULES` thêm: câu trả lời phải nêu lỗi 429/định dạng đã gặp và số lần retry (B3); dịch vụ không có tool
  thì nói thẳng ngay đầu câu trả lời, không gọi status tool của provider khác để thay thế (B9).

### Lần chạy 3 (file `eval_bonus.json` cuối cùng): 79.0/100 -> 11.85/15

| | B1 | B2 | B3 | B4 | B5 | B6 | B7 | B8 | B9 | B10 | Tổng | Bonus |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Lần 1 | 75 | 88 | 65 | 50 | 70 | 50 | 70 | 85 | 85 | 50 | 68.8 | 10.32 |
| Lần 2 | 75 | 100 | 55 | 80 | 100 | 50 | 65 | 50 | 15 | 65 | 65.5 | 9.82 |
| Lần 3 | 75 | 75 | 65 | 85 | 100 | 50 | 80 | 85 | 100 | 75 | **79.0** | **11.85** |

31 model calls, 62,645 token agent.

- B9 (15 -> 100): không gọi tool nào, nói thẳng không có tool/dữ liệu cho Linear/Slack, không bịa link.
- B8 (50 -> 85): lần này model tự đọc cả hai incident ở step 1 nên `validation_unread_incident` không cần kích hoạt;
  chưa có bằng chứng trực tiếp là kiểm tra mới đã cứu case này (có thể do dao động).
- B2 (100 -> 75): đọc timeline Vercel qua 4 trang (offset 0, 2, 4, 6) đúng pagination; sau đó kiểm tra "đi theo link"
  bắt thêm `93njd414rpbx` và mở nó với provider `vercel`. Hàm `unfollowed_links()` giả định link cùng provider với
  incident gốc nên có thể đọc sai nguồn khi link trỏ sang status page khác.
  **Đã sửa sau lần chạy 3:** link `https://www.githubstatus.com/incidents/93njd414rpbx` trong incident Vercel trước đây
  bị nhắc mở bằng provider `vercel`; nay `link_provider()` xác định provider theo tên miền của link (githubstatus.com ->
  `github`...), link tới status page không có tool thì bỏ qua. Kiểm tra offline: H1 -> `github/xr3myzj7ms51`,
  B2 -> `github/93njd414rpbx`.
- Lỗi còn lại: B6 vẫn cap 50 (không gọi `get_incident_details` sau khi lọc nhiễu) ở cả 3 lần.

### Lần chạy 4 (file `eval_bonus.json` cuối cùng, cùng code với `eval_after.json`): 68.5/100 -> 10.28/15

| B1 | B2 | B3 | B4 | B5 | B6 | B7 | B8 | B9 | B10 |
|---|---|---|---|---|---|---|---|---|---|
| 50 | 80 | 45 | 70 | 90 | 50 | 40 | 90 | 100 | 70 |

34 model calls, 74,487 token agent.

- B2: sau khi sửa `link_provider()`, link GitHub trong incident Vercel được mở bằng đúng provider `github`;
  đọc timeline qua nhiều trang bằng `offset`.
- B9 giữ 100, B8 90: hai cải tiến của lần 3 vẫn hiệu quả.
- B1 (75 -> 50), B3 (65 -> 45), B7 (80 -> 40) giảm dù code liên quan không đổi và không có vi phạm kiểm tra bằng
  code; judge chấm thấp vì câu trả lời không nhắc lần 429 (B3) và "không bỏ qua chỉ dẫn giả" (B7) dù trace B7
  chỉ có một lần gọi `get_npm_status` và không có marker injection trong câu trả lời.
- Điểm bonus qua 4 lần: 10.32 -> 9.82 -> 11.85 -> 10.28. Dao động lớn cho thấy điểm judge với `gpt-4o-mini` không
  ổn định; các cải tiến được đánh giá bằng trace (hết vi phạm cap ở B4, B10; B8/B9 đúng hành vi) hơn là bằng tổng điểm.
- B6 vẫn cap 50 ở cả 4 lần (không gọi `get_incident_details`).

### Giới hạn

- 429 và lỗi định dạng trong eval là giả lập; với 429 thật chỉ chờ khi `Retry-After` ≤ 5 giây, lâu hơn thì không chờ.
- Pagination và lọc nhiễu chủ yếu dựa vào prompt, không có kiểm tra bằng code.
- Điểm judge là ước lượng (cùng model với agent), mỗi phiên bản chỉ chạy một lần.
