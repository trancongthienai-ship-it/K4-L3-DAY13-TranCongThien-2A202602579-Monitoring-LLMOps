# Báo cáo cá nhân — K4-L3B Day 13 Monitoring & LLMOps

> Mỗi học viên hoàn thiện một file duy nhất này. Khi dẫn evidence, dùng đường dẫn tương đối, ví dụ `evidence/07-trace-waterfall.png`.

## 1. Thông tin học viên

- **Họ và tên:** Trần Công Thiện
- **MSSV:** 2A202602579
- **Lớp:** K4-L3B
- **Repository URL:** https://github.com/trancongthienai-ship-itK4-L3-DAY13-TranCongThien-2A202602579-Monitoring-LLMOps
- **Commit SHA cuối:** 61a34f827748393ced851ea7c9b412dd53dced23
- **Challenge ID:** day13-k4-l3b-monitoring-llmops-v1
- **Tên project Langfuse cá nhân:** `day13-k4-l3b-2A202602579`

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
| `validate_logs.py` | 30/100 | 100/100 | Đã đầy đủ trường thông tin, correlation_id và scrub PII |
| `validate_dashboard.py` | 6/6 | 6/6 | Dashboard hợp lệ |
| `pytest` | Fail | Pass (22/22) | Đã sửa lại code pass mọi unit tests |
| Số traces hợp lệ | 0 | >10 | Tạo thành công qua load_test |
| Số PII leak | >0 | 0 | Scrubbing hoạt động tốt |
| Latency P95 / TTFT P95 | N/A | ~400ms | Dưới ngưỡng SLO |
| Retrieval success rate | N/A | 100% | RAG hoạt động bình thường |

## 4. Logging và PII

- **Cách tạo/nhận và truyền correlation ID:** Lấy từ header `x-request-id`, nếu không có thì sinh UUID mới. Dùng `structlog.contextvars.bind_contextvars()` để gán vào toàn bộ log sinh ra trong chu kỳ request.
- **Các metadata được ghi vào structured log:** `user_id_hash`, `session_id`, `feature`, `model`, `env`.
- **Cách bảo đảm PII được scrub trước khi ghi:** Đăng ký hàm `scrub_event` vào mảng processors của structlog (nằm trước `JSONRenderer` và File processor) để quét regex và ẩn danh (REDACTED).
- **Cách kiểm chứng kết quả:** Chạy `validate_logs.py` đạt 100/100 điểm, log in ra không còn chứa plaintext email/phone.

## 5. Tracing và prompt versioning

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:** Các trace lưu đúng vào project Langfuse có gắn API key trong file `.env` cá nhân.
- **Cấu trúc root/retrieval/generation observations:** Một root span `lab-agent-run` bọc ngoài, bên trong có 2 child spans là `retrieve` (chạy RAG) và `fake-llm-generate` (gọi LLM).
- **Cách nối trace với log:** Đẩy `correlation_id` (được sinh ra từ middleware) vào tham số `metadata` của Langfuse trace.
- **Prompt name:** `day13-chat`
- **Version/label baseline:** Version 1, label `baseline`
- **Version/label candidate:** Version 2, label `candidate`
- **Trace ID của mỗi version:** Traces được lưu trữ tương ứng với lúc gọi `load_test` cho từng label.
- **Cách promote và rollback `production`:** Vào giao diện Langfuse Prompts, thao tác gắn/gỡ thẻ (label) `production` giữa các version.

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:** 6 panel bao gồm: Latency, Traffic, Error rate, Cost, Tokens, Quality.
- **SLO và lý do chọn:** Chọn SLO Latency P95 <= 3000ms. Lý do: RAG + LLM thường tốn nhiều thời gian xử lý, 3s là ngưỡng trải nghiệm người dùng tối thiểu chấp nhận được.
- **Cách tính error budget:** SLO 99% trong 30 ngày. Nếu có 1,000 request thì tối đa được phép có 10 request trả về trễ hơn 3000ms.
- **Ba alert và runbook tương ứng:** Cảnh báo Latency cao, Error Rate > 2%, và Cảnh báo Chi phí Cost vượt ngưỡng. Runbook: Kiểm tra database, check log `correlation_id`, nâng cấp tier LLM.

## 7. Điều tra challenge

- **Challenge ID:** day13-k4-l3b-monitoring-llmops-v1
- **Khoảng thời gian điều tra:** (Lúc chạy load_test --challenge)
- **Triệu chứng từ metrics:** Chỉ số Latency (độ trễ) nhảy vọt lên quá ngưỡng 2000ms.
- **Log line và correlation ID liên quan:** Dòng log `event: response_sent` ghi nhận `latency_ms: 2267`, với `correlation_id` là `req-beb78b07`.
- **Trace ID và span gây ảnh hưởng:** Trace ID tra theo `req-beb78b07` trên Langfuse cho thấy child span `retrieve` tiêu tốn tới >2.5s.
- **Root cause:** Hệ thống RAG (khâu retrieval) gặp sự cố truy xuất dữ liệu quá chậm (sự cố rag_slow).
- **Fix action:** Tối ưu hóa lại Vector Database hoặc index. Restart cụm RAG.
- **Preventive measure:** Thêm bộ nhớ Cache cho các câu hỏi lặp lại, cài Alert giám sát riêng latency của span `retrieve`.

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:** Quyết định để PII scrubbing chạy trước JSONRenderer trong `logging_config.py` để đảm bảo dữ liệu nhạy cảm bị che giấu trước khi bị ghi xuống ổ đĩa file `logs.jsonl`.
- **Một lỗi/blocker đã gặp:** Gặp lỗi khi Langfuse bị timeout hoặc tìm kiếm correlation_id bị sai trường Metadata.
- **Cách tìm nguyên nhân và xử lý:** Cuộn xuống tìm chuẩn trường Metadata thay vì Full-text search để tìm trace chính xác.
- **Cách hiểu luồng Metrics → Logs → Traces:** Metrics báo động có vấn đề tổng quan (Latency cao) -> Logs giúp tìm ra cụ thể Request nào bị lỗi thông qua ID -> Traces giúp mổ xẻ Request đó ra xem chính xác hàm/span nào chạy chậm.
- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:** Giúp kiểm soát chất lượng đầu ra, quản lý rủi ro chi phí và có thể khôi phục lại (Rollback) bản prompt cũ ngay tức thì mà không cần deploy lại code.
- **Điều quan trọng nhất đã học:** Biết cách kết nối 3 trụ cột của Observability (Metrics, Logs, Traces) bằng Correlation ID để debug.
- **Hạn chế hoặc phần chưa hoàn thành, nếu có:** Hoàn thành tốt.

## 9. Checklist trước khi nộp

- [x] Kết quả và evidence thuộc commit SHA cuối.
- [x] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [x] Incident evidence nối đúng metric → log → trace.
- [x] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [x] Repository chạy lại được theo README.
- [x] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [ ] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
