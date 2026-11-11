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
| Điểm tổng | 95.0 / 100 | Sẽ cập nhật sau lần chạy 2 |
| Điểm case đã chọn | 50 / 100 | Sẽ cập nhật sau lần chạy 2 |
| Số model calls | 47 calls | Sẽ cập nhật sau lần chạy 2 |
| Token, chi phí nếu có | 0 VNĐ (Gemini Free Tier) | 0 VNĐ (Gemini Free Tier) |

- Lỗi đã giải quyết: Đang chờ đo lường ở lần chạy 2.
- Lỗi còn lại hoặc mới xuất hiện: Đang chờ đo lường ở lần chạy 2.

## 5. Giới hạn

Nêu giới hạn của điểm judge, của snapshot và của việc chỉ chạy một lần mỗi phiên bản.
