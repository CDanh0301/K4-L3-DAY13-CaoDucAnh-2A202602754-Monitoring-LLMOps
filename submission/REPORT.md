# Báo cáo cá nhân — K4-L3B Day 13 Monitoring & LLMOps

> Mỗi học viên hoàn thiện một file duy nhất này. Khi dẫn evidence, dùng đường dẫn tương đối, ví dụ `evidence/07-trace-waterfall.png`.

## 1. Thông tin học viên

- **Họ và tên:** Cao Đức Anh
- **MSSV:** 2A202602754
- **Lớp:** K4-L3B
- **Repository URL:** https://github.com/CDanh0301/K4-L3-DAY13-CaoDucAnh-2A202602754-Monitoring-LLMOps
- **Commit SHA cuối:** c0d4b8de5bb8e9c3aa95a843c6a0206d6da2de16
- **Challenge ID:** day13-k4-l3b-monitoring-llmops-v1
- **Tên project Langfuse cá nhân:** `day13-k4-l3b-2A202602754`

## 2. Evidence index

Điền đúng đường dẫn tới evidence thực tế. Có thể đổi tên hoặc dùng nhiều ảnh nếu cần.

| Evidence | Đường dẫn |
|---|---|
| Pytest cuối | `evidence/01-pytest.png` |
| Log validator | `evidence/02-log-validator.png` |
| Dashboard validator | `evidence/03-dashboard-validator.png` |
| Structured log | `evidence/04-structured-log.png` |
| PII redaction | `evidence/05-pii-redaction.png` |
| Trace list | `evidence/06-trace-list.png` |
| Trace waterfall | `evidence/07-trace-waterfall.png` |
| Trace metadata | `evidence/08-trace-metadata.png` |
| Prompt versions | `evidence/09-prompt-versions.png` |
| Prompt rollback | `evidence/10-prompt-rollback.png` |
| Dashboard runtime | `evidence/11-dashboard-overview.png` |
| Incident metric | `evidence/12-incident-metric.png` |
| Incident log | `evidence/13-incident-log.png` |
| Incident trace | `evidence/14-incident-trace.png` |

## 3. Kết quả kỹ thuật

| Nội dung | Baseline | Kết quả cuối | Nhận xét |
|---|---|---|---|
| `validate_logs.py` | 30/100 | 100/100 | Đạt toàn bộ các tiêu chí schema, context enrichment, correlation ID và scrub PII |
| `validate_dashboard.py` | 6/6 panel có trong dashboard contract | 6/6 panel hợp lệ | Đạt chuẩn contract của cả 6 panel theo cấu hình YAML |
| `pytest` | 22 passed | 24 passed | Toàn bộ 24 unit tests pass, bổ sung đầy đủ test cho PII CCCD & Credit Card |
| Số traces hợp lệ | 0 | ≥ 10 | Ghi nhận đầy đủ trace trên project Langfuse cá nhân |
| Số PII leak | > 0 | 0 | Không còn rò rỉ email, phone, CCCD, credit card |
| Latency P95 / TTFT P95 | ~151ms / 50ms | 2653ms / 50ms (lúc incident) | Latency P95 tăng vọt do incident `rag_slow`, TTFT giữ ổn định ở 50ms |
| Retrieval success rate | 100% | 100% | Retrieval thành công, không phát sinh lỗi ngoại lệ ngoại trừ độ trễ tăng cao |

## 4. Logging và PII

- **Cách tạo/nhận và truyền correlation ID:** Trong [CorrelationIdMiddleware], middleware gọi `clear_contextvars()` trước mỗi request để tránh rò rỉ context giữa các request. Sau đó kiểm tra header `x-request-id`: nếu client truyền lên thì nhận giá trị đó, nếu không có thì tự động sinh mới theo định dạng `req-<8-char-hex>`. Middleware bind `correlation_id` vào contextvars qua `bind_contextvars(correlation_id=correlation_id)` và gán vào `request.state.correlation_id`. Cuối cùng, gán lại vào response header `x-request-id` và tính thời gian xử lý trả về trong `x-response-time-ms`.
- **Các metadata được ghi vào structured log:** Tại endpoint [/chat], các trường metadata gồm `user_id_hash`, `session_id`, `feature`, `model` (`agent.model`), và `env` (`APP_ENV`) được bind vào contextvars qua `bind_contextvars(...)` ngay trước khi log `request_received`. Nhờ đó, tất cả log records liên quan đến request đều mang đầy đủ enrichment fields (`ts`, `level`, `service`, `event`, `correlation_id`, `user_id_hash`, `session_id`, `feature`, `model`, `env`).
- **Cách bảo đảm PII được scrub trước khi ghi:** Trong [app/logging_config.py], processor `scrub_event` được đăng ký vào pipeline của `structlog` trước `JsonlFileProcessor` và `JSONRenderer`. Khi mỗi log event đi qua pipeline, `scrub_event` duyệt qua trường `event` và các giá trị chuỗi trong `payload` bao gồm `message_preview`, `answer_preview`, `detail`, áp dụng regex thay thế email, số điện thoại VN, CCCD và thẻ tín dụng thành `[REDACTED_<TYPE>]` trước khi render ra JSON hoặc ghi vào `data/logs.jsonl`.
- **Cách kiểm chứng kết quả:** Xóa/lưu trữ `data/logs.jsonl` cũ, khởi động lại server uvicorn, chạy script `python scripts/load_test.py` để tạo workload, sau đó chạy `python scripts/validate_logs.py` để kiểm tra không còn PII leaks, đầy đủ required fields và context enrichment fields cùng `pytest` để kiểm tra test cases cho PII.

## 5. Tracing và prompt versioning

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:** Toàn bộ request được gửi đến API sử dụng key pair của project `day13-k4-l3b-2A202602754` cấu hình trong `.env`. Trên Langfuse Cloud, trace metadata hiển thị `user_id_hash`, `session_id`, `environment="dev"`, và `correlation_id` khớp với từng dòng log trong `data/logs.jsonl`.
- **Cấu trúc root/retrieval/generation observations:**
  ```text
  day13-agent-request (trace)
  └── lab-agent-run (agent root span)
      ├── retrieval (retriever child span - tìm tài liệu, đo thời gian truy xuất)
      └── generation (generation child span - model claude-sonnet-4-5, input/output tokens, cost, prompt)
  ```
- **Cách nối trace với log:** Sử dụng chung trường `correlation_id` (định dạng `req-<8-hex>`). Middleware tạo correlation ID được ghi vào cả log (`structlog.contextvars`) và truyền vào Langfuse context metadata thông qua `propagate_attributes(metadata={"correlation_id": correlation_id})`.
- **Prompt name:** `day13-chat`
- **Version/label baseline:** Version 1 (labels: `baseline`, `production`)
- **Version/label candidate:** Version 2 (label: `candidate`)
- **Trace ID của mỗi version:** Version 1: 46dade7834f81b9593de32d7a2c84e04, Version 2: e414d8fa2b2b825f4af4a7e0ae0e2764
- **Cách promote và rollback `production`:**
  - Promote: Trên Langfuse UI vào Prompt Management -> `day13-chat` -> Version 2, gán nhãn `production` trỏ sang v2.
  - Rollback: Trên Langfuse UI chuyển nhãn `production` từ Version 2 quay trở lại Version 1 mà không cần sửa hay redeploy source code.

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:** Tuân thủ contract cấu hình trong [config/dashboard.yaml], bao gồm 6 panel:
  1. `latency`: Theo dõi P50, P95, P99 và TTFT P95 từ event `response_sent`.
  2. `traffic`: Tần suất request nhận vào (`count() by 1m`) từ event `request_received`.
  3. `errors`: Tỉ lệ lỗi tổng thể (`error_rate_pct`) và tỉ lệ retrieval thành công (`tool_success_rate_pct`).
  4. `cost`: Chi phí USD tích lũy và theo từng phút (`cost_usd`).
  5. `tokens`: Tổng lượng token in và token out (`tokens_in`, `tokens_out`).
  6. `quality`: Điểm chất lượng trung bình của câu trả lời (`quality_score`).
- **SLO và lý do chọn:** Chọn SLO `fast_successful_requests`: 99.5% request hoàn thành thành công và có `latency_ms <= 3000ms` trong chu kỳ 28 ngày. Lý do: người dùng ứng dụng trợ lý hội thoại yêu cầu phản hồi nhanh (< 3 giây) và không gặp lỗi gián đoạn để duy trì trải nghiệm liền mạch.
- **Cách tính error budget:** SLO 99.5% tương ứng với Error Budget là 0.5% (tức $100\% - 99.5\%$). Ví dụ: nếu trong 28 ngày có 10,000 requests được gửi đến hệ thống, error budget cho phép tối đa $10,000 \times 0.5\% = 50$ requests bị lỗi hoặc có độ trễ vượt quá 3000ms.
- **Ba alert và runbook tương ứng:** Cấu hình trong [config/alert_rules.yaml] và chi tiết tại [docs/alerts.md]:
  1. `HighLatencyP95` (Warning, >3000ms trong 5m, Slack `#k4-l3b-alerts`): Tra cứu dashboard và trace để tìm span bị chậm; thực hiện rollback prompt hoặc tối ưu retrieval.
  2. `HighErrorRate` (Critical, error rate >2% trong 5m, Slack `#k4-l3b-alerts`): Tra cứu log `request_failed` và trace để xác định nguyên nhân ngoại lệ (500); cô lập hoặc tắt module gây lỗi.
  3. `RetrievalFailureSpike` (Warning, tool success rate <90% trong 5m, Slack `#k4-l3b-alerts`): Kiểm tra kết nối dịch vụ vector store; chuyển hướng fallback sang local corpus.

> Ví dụ cách viết error budget: "SLO 99.5% trong 28 ngày nghĩa là error budget 0.5%. Nếu workload có 10,000 request thì tối đa 50 request được phép lỗi hoặc chậm hơn ngưỡng SLO."

## 7. Điều tra challenge

- **Challenge ID:** `day13-k4-l3b-monitoring-llmops-v1`
- **Khoảng thời gian điều tra:** `2026-09-30T05:16:17Z` đến `2026-09-30T05:16:42Z`
- **Triệu chứng từ metrics:** Dashboard và log cho thấy độ trễ trung bình/P95 tăng đột biến từ mức baseline ~151ms lên trên 2650ms (vượt ngưỡng 2000ms), trong khi TTFT vẫn giữ nguyên ở 50ms và số lượng tokens/cost không có biến động bất thường.
- **Log line và correlation ID liên quan:** `correlation_id=req-3cf1b401`. Log line đại diện trong `data/logs.jsonl`:
  ```json
  {"service": "api", "latency_ms": 2653, "ttft_ms": 50, "tokens_in": 35, "tokens_out": 98, "cost_usd": 0.001575, "quality_score": 0.8, "tool_name": "retrieval", "tool_success": true, "payload": {"answer_preview": "Starter answer. You should improve this output logic and add better quality chec..."}, "event": "response_sent", "correlation_id": "req-3cf1b401", "model": "claude-sonnet-4-5", "env": "dev", "session_id": "k4-l3b-challenge-s03", "feature": "monitoring", "user_id_hash": "189d0a182d4e", "level": "info", "ts": "2026-09-30T05:16:31.974464Z"}
  ```
- **Trace ID và span gây ảnh hưởng:** Trace ID `58f793b0d6d7f5a240ae051065402e7c`. Trên Langfuse waterfall, span `retrieval` (retriever observation) mất đến 2.50s trong tổng số 2.65s của request, còn span `generation` chỉ mất ~0.15s.
- **Root cause:** Module truy xuất dữ liệu RAG gặp sự cố độ trễ cao (`rag_slow` được kích hoạt), mô phỏng trường hợp vector database hoặc embedding service bị nghẽn dẫn đến thời gian chờ tài liệu kéo dài thêm 2.5s.
- **Fix action:** Tắt sự cố qua endpoint `/incidents/rag_slow/disable`, kiểm tra tải và tối ưu hóa kết nối đến Vector Store.
- **Preventive measure:** Bổ sung alert riêng cho latency của retriever span (`retrieval_latency > 1000ms`), cài đặt timeout và circuit-breaker cho vector search với cơ chế fallback về local cache hoặc phản hồi trực tiếp khi timeout.

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:** Đặt processor `scrub_event` trong `structlog` trước bước serialize JSON và ghi file (`JsonlFileProcessor`). Quyết định này đảm bảo 100% dữ liệu nhạy cảm (email, SĐT, CCCD, thẻ tín dụng) được che giấu triệt để ngay tại nguồn trước khi bất kỳ byte nào được ghi xuống đĩa hoặc truyền qua mạng.
- **Một lỗi/blocker đã gặp:** Quên xóa file `data/logs.jsonl` cũ trước khi chạy lại `validate_logs.py`, dẫn đến validator đọc lại các dòng log trước khi sửa PII scrubber và báo lỗi.
- **Cách tìm nguyên nhân và xử lý:** Đọc kỹ tài liệu [docs/CHECKPOINTS.md] và code của validator; nhận ra validator quét toàn bộ log file từ đầu đến cuối nên cần xóa file cũ để tái tạo dữ liệu sạch sau khi hoàn thiện code.
- **Cách hiểu luồng Metrics → Logs → Traces:** 
  1. *Metrics* phát hiện triệu chứng và thời điểm xảy ra sự cố (hệ thống có vấn đề gì và khi nào).
  2. *Logs* cung cấp ngữ cảnh chi tiết và trích xuất `correlation_id` của request bị ảnh hưởng.
  3. *Traces* mổ xẻ request đó để xác định chính xác span/thao tác cụ thể gây ra lỗi hoặc chậm (bước nào là nguyên nhân gốc rễ).
- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:** Quản lý prompt theo version và label cho phép đội ngũ kỹ sư linh hoạt thử nghiệm cải tiến mà vẫn đảm bảo an toàn vận hành có thể rollback tức thì về version ổn định mà không cần sửa code. Giám sát token và cost liên tục giúp tránh sự cố bùng nổ chi phí (cost spike), trong khi SLO và error budget định lượng rõ ràng ranh giới giữa độ tin cậy và tốc độ phát triển tính năng.
- **Điều quan trọng nhất đã học:** Kỹ năng quan sát toàn diện cho các hệ thống ứng dụng LLM: không chỉ xem LLM như một hộp đen mà có thể bóc tách đo lường từng mắt xích từ tiền xử lý, RAG retrieval đến sinh token và chi phí.
- **Hạn chế hoặc phần chưa hoàn thành, nếu có:** Các mô hình LLM và RAG trong bài đang chạy ở dạng mô phỏng (fake/mock); trong môi trường production thực tế sẽ cần tích hợp thêm semantic cache và các bộ đánh giá chất lượng tự động phức tạp hơn.

## 9. Checklist trước khi nộp

- [x] Kết quả và evidence thuộc commit SHA cuối.
- [x] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [x] Incident evidence nối đúng metric → log → trace.
- [x] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [x] Repository chạy lại được theo README.
- [x] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [x] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
