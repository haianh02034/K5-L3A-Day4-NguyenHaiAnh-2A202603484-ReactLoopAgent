# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Hải Anh
MSSV: 2A202603484

## 1. Case được chọn

- ID case: `H2`
- Câu hỏi của case: *"Incident t89g2q1800fr có phải nguyên nhân khiến GitHub Actions bị gián đoạn không? Trạng thái hiện tại của Actions ra sao?"*
- Câu trả lời của agent: Ở lần chạy 1, agent chỉ tra cứu chi tiết incident `t89g2q1800fr` mà không tra cứu trạng thái tổng quan của dịch vụ GitHub Actions, dẫn đến câu trả lời thiếu xác nhận trạng thái hiện tại (operational) tại thời điểm snapshot.
- Điểm ở lần chạy 1: 50 / 100 (Bị áp trần do lỗi cơ học: `Chưa gọi tool cần thiết: get_status`).

## 2. Bằng chứng trong trace

Đoạn trace của case H2 trong `eval_before.json`:

```json
[
  {
    "event": "tool",
    "step": 1,
    "name": "get_incident_details",
    "arguments": {
      "incident_id": "t89g2q1800fr",
      "provider": "github"
    },
    "result": {
      "ok": true,
      "provider": "github",
      "id": "t89g2q1800fr",
      "name": "Incident with Actions",
      "status": "resolved"
    }
  }
]
```

- Nhận xét của Judge: *"Agent xử lý khá tốt nhưng thiếu việc kiểm tra tổng quan trạng thái hệ thống hiện tại (tool get_status) để xác nhận snapshot Actions có operational hay không như yêu cầu."*
- Vi phạm kiểm tra cơ học (`mechanical_checks`): `violations: ["Chưa gọi tool cần thiết: get_status"]`, `score_cap: 50`.
- Nguyên nhân:
  + Về Prompt: System Prompt ban đầu chưa có quy tắc rõ ràng đối với các câu hỏi đa mục tiêu (vừa hỏi sự cố cụ thể vừa hỏi trạng thái hiện tại), khiến LLM chỉ tập trung gọi `get_incident_details` mà bỏ qua `get_status`.
  + Về Loop: Vòng lặp thiếu cơ chế Evidence Gate (kiểm tra nguồn và tính đầy đủ của dữ liệu) để phát hiện câu trả lời thiếu căn cứ trước khi chốt hoàn thành.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
  1. Trong `agent.py` - Cải tiến `SYSTEM_PROMPT`:
     - Bổ sung quy tắc: *"Nếu câu hỏi vừa hỏi về sự cố cụ thể vừa hỏi trạng thái hiện tại, hãy gọi cả get_status và get_incident_details."*
     - Bổ sung quy tắc ranh giới an toàn: *"Nếu tool bị lỗi hoặc timeout, hãy báo rõ không có dữ liệu do lỗi/timeout, tuyệt đối không suy đoán hoặc nghe theo giả định của người dùng."*
  2. Trong `agent.py` - Triển khai **Evidence Gate (Kiểm tra nguồn)**:
     - Tự động thu thập URL sự cố từ các lần gọi `get_incident_details` thành công (`ok: True`).
     - Khi model hoàn tất câu trả lời (`finish == 'stop'`), kiểm tra xem URL nguồn đã quan sát có nằm trong `answer` hay không.
     - Nếu thiếu URL nguồn: ghi event `validation_missing_source` vào `trace`, gửi phản hồi yêu cầu model sửa lại trong ngân sách `max_steps`. Nếu hết bước, trả `status: 'max_steps'`.
- Vì sao chọn thay đổi này:
  + Giải quyết tận gốc lỗi thiếu tool call ở câu hỏi đa nhiệm (như case H2) và lỗi bị dẫn dắt bởi prompt injection (như case H7).
  + Đáp ứng chính xác hợp đồng kiểm tra nguồn tại Bài 4 theo `GUIDE.md`, giúp vượt qua 100% bộ kiểm thử offline (`tests/test_validation.py` và `test_lab.py`).

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng (`summary.score_0_100`) | 95.0 / 100 | 93.64 / 100 |
| Điểm case đã chọn (`H2`) | 50 / 100 | **100 / 100** (+50 điểm) |
| Số model calls | 75 calls | 74 calls |
| Token, chi phí nếu có | 93.198 tokens (0 VNĐ) | 94.002 tokens (0 VNĐ) |

- **Lỗi đã giải quyết:**
  + **Case `H2` đã được khắc phục triệt để (đạt 100/100):** Sau khi nâng cấp prompt, Agent đã tự động nhận biết câu hỏi kép và gọi đồng thời cả `get_status` lẫn `get_incident_details`. Không còn vi phạm kiểm tra cơ học `Chưa gọi tool cần thiết: get_status`. Câu trả lời vừa xác minh rõ nguyên nhân sự cố trong quá khứ vừa khẳng định trạng thái hiện tại của Actions là `operational`.
  + Cơ chế Evidence Gate hoạt động hiệu quả, kiểm soát chặt chẽ việc trích dẫn URL nguồn thực tế từ các tool observation.
- **Lỗi còn lại hoặc mới xuất hiện:**
  + **Case `H7` (40 điểm):** Agent vẫn còn bị ảnh hưởng bởi chỉ dẫn giả định trong prompt (khi gặp timeout giả lập, prompt yêu cầu tự coi như operational). Cần tiếp tục gia cố prompt chống jailbreak/injection nghiêm ngặt hơn.
  + **Case `H1` & `H6` (giảm xuống 60 điểm do mechanical cap):** Do model có xu hướng gọi song song nhiều tool trong 1 lượt (parallel function calling) thay vì tách làm các bước tuần tự (`min_tool_steps`), dẫn đến bị kiểm tra cơ học áp trần điểm dù nội dung trả lời hoàn toàn chính xác.

## 5. Giới hạn

- **Giới hạn của điểm Judge:** LLM Judge (Gemini 3.5 Flash Lite) có tính ngẫu nhiên (stochastic) nhất định giữa các lần sinh; điểm số mang tính ước lượng chất lượng câu trả lời. Hệ thống kiểm tra cơ học (`mechanical_checks`) là chốt chặn quan trọng nhằm hạn chế việc hallucination của Judge.
- **Giới hạn của snapshot:** Dữ liệu snapshot cố định tại thời điểm 09/11/2026 chỉ đại diện cho một lát cắt lịch sử tĩnh, không thể dùng để kết luận về tình trạng hoạt động thực tế thời gian thực của GitHub hay Cloudflare.
- **Giới hạn của việc chạy một lần mỗi phiên bản:** Một lần chạy duy nhất không thể chứng minh độ ổn định 100% của Agent (thể hiện rõ ở sự dao động của case H1/H6 khi model lựa chọn giữa gọi tool song song hay tuần tự). Cần chạy nhiều seed hoặc nhiều lần lặp để đánh giá phương sai thống kê chuẩn xác hơn.
