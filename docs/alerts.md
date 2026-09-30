# Template Alert và Runbook

Mỗi alert phải dựa trên triệu chứng người dùng hoặc SLO, không dựa trực tiếp vào tên implementation nội bộ.

## Alert mẫu để tham khảo

Ví dụ dưới đây minh họa mức độ cụ thể cần có. Học viên không cần copy nguyên, nhưng ba alert trong bài nộp nên rõ ràng tương tự: điều kiện là gì, kéo dài bao lâu, ảnh hưởng tới user ra sao và người trực cần kiểm tra gì trước.

- Tên: `HighLatencyP95`
- Severity: `warning`
- Duration: `5m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: latency P95 của `response_sent.latency_ms`
- Điều kiện và thời gian duy trì: `p95(latency_ms) > 3000ms` trong 5 phút
- Ảnh hưởng tới người dùng: người dùng phải chờ lâu hơn trước khi nhận câu trả lời
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard latency để xác nhận P95/P99 và khoảng thời gian tăng.
  2. Lọc `data/logs.jsonl` trong khoảng đó, lấy một `correlation_id` có `latency_ms` cao.
  3. Mở trace cùng `correlation_id` trên Langfuse, so sánh các span chính để xác định bước nào bất thường.
- Mitigation tạm thời: dựa trên evidence thực tế để rollback prompt, khôi phục cấu hình liên quan, tắt practice scenario hoặc giảm tải khi demo.
- Owner: `student-<MSSV>`

## Alert 1

- Tên: `HighLatencyP95`
- Severity: `warning`
- Duration: `5m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: Latency P95 của `response_sent.latency_ms` (ngưỡng 3000ms)
- Điều kiện và thời gian duy trì: `p95(latency_ms) > 3000ms` kéo dài liên tục trong 5 phút
- Ảnh hưởng tới người dùng: Người dùng bị chậm trễ rõ rệt, phải chờ lâu mới nhận được câu trả lời từ chatbot.
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard panel `latency` để xác định thời điểm bắt đầu tăng và so sánh giữa P50 và P95/P99.
  2. Lọc `data/logs.jsonl` tìm các log record `response_sent` có `latency_ms > 3000` và trích xuất `correlation_id`.
  3. Tìm kiếm trace tương ứng trên Langfuse theo `correlation_id` để kiểm tra span nào bị nghẽn (`retrieval` hay `generation`).
- Mitigation tạm thời: Nếu do prompt mới làm sinh quá nhiều token hoặc LLM chậm, thực hiện rollback prompt label `production` về version trước đó; nếu do retrieval chậm thì kiểm tra dịch vụ RAG vector store.
- Owner: `student-2A202602754`

## Alert 2

- Tên: `HighErrorRate`
- Severity: `critical`
- Duration: `5m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: Tỉ lệ lỗi tổng thể `count(request_failed) / count(request_received) * 100` (ngưỡng 2%)
- Điều kiện và thời gian duy trì: `error_rate_pct > 2%` kéo dài trong 5 phút
- Ảnh hưởng tới người dùng: Người dùng nhận phản hồi lỗi HTTP 500, dịch vụ gián đoạn không trả lời được câu hỏi.
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard panel `errors` xem tỉ lệ lỗi hiện tại và phân loại lỗi theo `error_type`.
  2. Tra cứu `data/logs.jsonl` với sự kiện `request_failed`, đọc `error_type`, `detail` và lấy `correlation_id` bị lỗi.
  3. Mở trace trên Langfuse theo `correlation_id` để xem chi tiết exception stack trace và trạng thái root/child observation.
- Mitigation tạm thời: Khởi động lại dịch vụ hoặc kích hoạt circuit breaker, kiểm tra kết nối API upstream; nếu do incident inject thử nghiệm thì vô hiệu hóa incident qua endpoint `/incidents/{name}/disable`.
- Owner: `student-2A202602754`

## Alert 3

- Tên: `RetrievalFailureSpike`
- Severity: `warning`
- Duration: `5m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: Tỉ lệ thành công của retrieval tool `tool_success_rate_pct` (ngưỡng 90%)
- Điều kiện và thời gian duy trì: `tool_success_rate_pct < 90%` trong 5 phút
- Ảnh hưởng tới người dùng: Bot không truy xuất được tài liệu liên quan, dẫn đến trả lời fallback thiếu chính xác hoặc bị timeout.
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard panel `errors` kiểm tra đồ thị `retrieval success rate` và số lượng `tool_success == false`.
  2. Lọc log sự kiện `request_failed` có `tool_name == "retrieval"` để xác định thông tin ngoại lệ (ví dụ: `RuntimeError: Vector store timeout`).
  3. Mở trace trên Langfuse, kiểm tra observation span `retrieval` để phân tích latency và thông báo lỗi.
- Mitigation tạm thời: Chuyển sang fallback tài liệu local corpus sẵn có, khởi động lại service kết nối vector database hoặc tắt lỗi tool inject nếu đang test.
- Owner: `student-2A202602754`
