# Benchmark & Observability Report — Day 14

> Copy file này thành `BENCHMARK_REPORT.md` trong repo của bạn và điền đầy đủ.
> Mọi con số phải lấy từ report JSON / CSV / screenshot thật, không ước lượng.

**Họ tên:** Phạm Nguyễn Tất Đạt (MSSV: 2A202603401)  
**Ngày chạy:** 2026-11-25  
**Môi trường:** Windows + Docker Desktop (Python 3.12.2 local, container Linux `python:3.11-slim`)

---

## 1. SLO của agent

| SLI | SLO | Lý do chọn ngưỡng này |
|---|---|---|
| p95 latency `/ask` | < 500 ms | Đảm bảo trải nghiệm người dùng phản hồi nhanh dưới tải bình thường (base LLM ~120-180ms + retrieval ~20ms) |
| Error rate `/ask` | < 2.0 % | Chấp nhận lỗi upstream ngẫu nhiên nhỏ (1% mock) nhưng cảnh báo khi lỗi tăng bất thường |
| Accuracy trên golden dataset | ≥ 80 % (mục tiêu 100%) | Đảm bảo agent v2 có RAG trả lời đúng kiến thức nền, không regression khi cập nhật |

---

## 2. Benchmark hiệu năng (develop, `load_test.py`)

### 2.1 Concurrency vs latency (agent v2)

| Concurrency | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Error rate | SLO |
|---|---|---|---|---|---|---|
| 1 | 4.24 | 202.5 | 237.3 | 1700.8 | 0.33% | PASS |
| 10 | 42.47 | 193.3 | 234.6 | 1690.2 | 0.67% | PASS |
| 50 | 101.10 | 216.5 | 279.4 | 1721.4 | 0.33% | PASS |
| 100 | 46.53 | 416.2 | 4700.8 | 5246.3 | 0.00% | FAIL |

**Nhận xét:** RPS có tăng tuyến tính theo concurrency không? Vì sao?

Không tăng tuyến tính ở mức cao. Từ concurrency 1 → 10 → 50, RPS tăng mạnh (4.24 → 42.47 → 101.10 req/s) nhờ xử lý bất đồng bộ (`async/await`). Tuy nhiên ở concurrency 100, event loop / kết nối trên 1 process uvicorn bị quá tải (queueing), khiến p95 vọt lên 4700.8 ms và RPS giảm xuống còn 46.53 req/s (vi phạm SLO p95 ≤ 500 ms).

### 2.2 Tail latency

| `MOCK_SLOW_RATE` | avg (ms) | p95 (ms) | p99 (ms) |
|---|---|---|---|
| 0 | 191.7 | 229.8 | 247.2 |
| 0.03 (mặc định) | 259.4 | 243.0 | 1710.6 |
| 0.1 | 334.4 | 1703.4 | 1730.7 |

**Nhận xét:** Chỉ số nào nhạy nhất với request chậm? Vì sao SLO không nên đặt theo average?

`p99` (và `p95` khi tỉ lệ chậm đạt 10%) nhạy nhất với request chậm: khi `MOCK_SLOW_RATE` tăng từ 0 lên 0.03, `avg` chỉ tăng từ 191.7 ms lên 259.4 ms (+35%), trong khi `p99` tăng vọt từ 247.2 ms lên 1710.6 ms (gấp ~7 lần). Nếu đặt SLO theo `average`, phần lớn request nhanh sẽ che lấp trải nghiệm rất tệ của 1–5% người dùng bị chậm.

---

## 3. Benchmark chất lượng (develop, `eval/run_eval.py`)

| Chỉ số | Agent v1 | Agent v2 | Agent v2 (sau khi sửa KB) |
|---|---|---|---|
| Accuracy | 4.35% (1/23) | 91.3% (21/23) | 100.0% (23/23) |
| Avg tokens / câu | 57.1 | 108.8 | 113.7 |
| Tổng cost (USD) | $0.000404 | $0.000758 | $0.000790 |
| Cost / câu đúng (USD) | $0.00040395 | $0.00003607 | $0.00003436 |
| Quality gate | FAIL | PASS (≥ 80%) | PASS (≥ 100%) |

**Những câu v2 trả lời sai ban đầu và cách bạn sửa:**

| ID | Keyword bị thiếu | Bạn đã sửa gì |
|---|---|---|
| `security-02` | `token`, `hết hạn`, `chữ ký` | Thêm mục `(["jwt"], ...)` vào `KNOWLEDGE_BASE` trong `utils/mock_llm.py` |
| `scaling-03` | `cpu`, `memory`, `custom metrics` | Thêm mục `(["hpa", "kubernetes", "autoscal"], ...)` vào `KNOWLEDGE_BASE` trong `utils/mock_llm.py` |

**Nhận xét về trade-off chất lượng / chi phí / latency giữa v1 và v2:**

Agent v2 thêm bước retrieval (RAG) nên prompt dài hơn (avg tokens tăng từ 57.1 lên 113.7, latency p95 tăng nhẹ từ 180.5 ms lên 226.4 ms, tổng cost tăng từ $0.000404 lên $0.000790). Tuy nhiên accuracy tăng từ 4.35% lên 100%, giúp **cost / câu đúng** giảm hơn 11 lần ($0.00040395 xuống $0.00003436).

---

## 4. Observability stack (production)

*(Phần này chạy manual bằng Docker Compose trong `06-lab-complete/benchmark-observability/production/` và điền số liệu + chụp 4 ảnh vào `screenshots/`)*

### 4.1 Screenshot

- [ ] `screenshots/grafana-dashboard.png`: Dashboard Grafana khi đang có tải (thấy rõ Golden Signals)
- [ ] `screenshots/jaeger-slow-trace.png`: Một trace chậm trong Jaeger (thấy span nào chiếm thời gian)
- [ ] `screenshots/jaeger-error-trace.png`: Một trace lỗi trong Jaeger (có event `exception`)
- [ ] `screenshots/prometheus-alert-firing.png`: Trang Alerts của Prometheus với ít nhất 2 alert đang **firing**

### 4.2 Thí nghiệm "phá hệ thống"

| # | Thay đổi | Alert nào firing | Thời gian từ lúc đổi → firing | Panel nào phản ánh rõ nhất |
|---|---|---|---|---|
| A | `MOCK_ERROR_RATE=0.2` | | | |
| B | `MOCK_SLOW_RATE=0.3` | | | |
| C | `LLM_MODEL=mock-llm-large` | | | |
| D | `docker pause` 1 instance | | | |
| E | `AGENT_VERSION=v1` | | | |

**Thí nghiệm E không làm alert nào firing. Bạn sẽ phát hiện loại sự cố này trong production bằng cách nào?**

...

### 4.3 Debug bằng trace

Chọn 1 trace chậm nhất bạn tìm được:

- `trace_id`: ...
- Tổng thời gian: ... ms
- Span chiếm nhiều thời gian nhất: ... (... ms, ...%)
- Dòng log tương ứng (grep theo `trace_id`):

```json

```

---

## 5. Benchmark scaling (production, Locust headless, 100 users, 2 phút)

| Số instance | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Failures |
|---|---|---|---|---|---|
| 1 | | | | | |
| 3 | | | | | |

**Giải thích kết quả:** Scale có cải thiện p95 không? Nút thắt nằm ở đâu?

...

---

## 6. Kết luận

- 3 điều bạn học được về benchmark & observability:
  1. ...
  2. ...
  3. ...
- Nếu đưa agent này lên production thật, bạn sẽ thêm metric / alert nào nữa?

...
