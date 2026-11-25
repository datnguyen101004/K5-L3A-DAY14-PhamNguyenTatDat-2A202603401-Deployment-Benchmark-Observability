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
| A | `MOCK_ERROR_RATE=0.2` | `AgentHighErrorRate` | ~60s (ngưỡng `for: 1m`) | Error Rate & HTTP 5xx trên Grafana |
| B | `MOCK_SLOW_RATE=0.3` | `AgentHighLatencyP95` | ~120s (ngưỡng `for: 2m`) | Latency p50/p95/p99 `/ask` trên Grafana |
| D | `docker pause` 1 instance (`agent-2`) | `AgentInstanceDown` | ~30s (ngưỡng `for: 30s`) | Requests/s theo instance (`agent-2` về 0) |
| - | Tải liên tục 100 users | `LLMCostBurnRateHigh` | ~120s (ngưỡng `for: 2m`) | LLM Cost Burn Rate ($/h) trên Grafana |
| E | `AGENT_VERSION=v1` | Không có alert nào | — | Không có (cần chạy eval định kỳ) |

**Thí nghiệm E không làm alert nào firing. Bạn sẽ phát hiện loại sự cố này trong production bằng cách nào?**

Alert hạ tầng (HTTP 5xx, latency, CPU, instance down) không phát hiện được suy giảm chất lượng câu trả lời (vì HTTP vẫn trả `200 OK` và nhanh hơn). Cần chạy **eval trên golden dataset** trong CI/CD (quality gate trước khi deploy), theo dõi `agent_info{version=...}`, và kết hợp đánh giá online (LLM-as-judge trên mẫu traffic thật hoặc thumbs-up/down của người dùng).

### 4.3 Debug bằng trace

Chọn 1 trace chậm tìm được (khi chạy tải `scale_1` / `scale_3`):

- `trace_id`: `9a9e4d1c7b44bca84fe31ea9e0e6de7f`
- Tổng thời gian: `1677.7 ms`
- Span chiếm nhiều thời gian nhất: `llm.generate` (chiếm phần lớn thời gian ~1650 ms, > 95% tổng thời gian request do dính tail latency `MOCK_SLOW_MS=1500`)
- Dòng log tương ứng (grep theo `trace_id`):

```json
{"ts": "2026-11-25T16:40:26.446665+00:00", "level": "INFO", "service": "observable-agent", "instance": "98806ebc2cdb", "event": "http_request", "trace_id": "9a9e4d1c7b44bca84fe31ea9e0e6de7f", "span_id": "5cdc55269c6856fe", "request_id": "b09b5ba0ae79", "method": "POST", "route": "/ask", "status": 200, "ms": 1677.7}
```

---

## 5. Benchmark scaling (production, Locust headless, 100 users, 2 phút)

*(Số liệu lấy từ `benchmark-results/scale_1_stats.csv` và `benchmark-results/scale_3_stats.csv` — dòng Aggregated)*

| Số instance | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Failures |
|---|---|---|---|---|---|
| 1 | 65.84 | 190 | 220 | 1700 | 64 (0.82%) |
| 3 | 65.70 | 190 | 220 | 1700 | 79 (1.01%) |

**Giải thích kết quả:** Scale có cải thiện p95 không? Nút thắt nằm ở đâu?

Scale từ 1 lên 3 instance không làm thay đổi RPS (~65.8 req/s) hay p95 (220 ms). Nguyên nhân là kịch bản Locust `AgentUser` có `wait_time` giữa các request nên 100 user ảo chỉ tạo ra ~65 req/s — mức tải này 1 instance FastAPI bất đồng bộ (`async/await`) đã xử lý dư sức mà chưa bị nghẽn CPU hay event loop. Nút thắt nằm ở **tốc độ sinh tải của client** (và latency giả lập `asyncio.sleep` mỗi request), không phải ở số lượng instance.

---

## 6. Kết luận

- 3 điều bạn học được về benchmark & observability:
  1. **Average che giấu tail latency:** `p95`/`p99` phản ánh đúng trải nghiệm của nhóm người dùng bị chậm và là cơ sở đúng đắn để đặt SLO thay cho trung bình.
  2. **Đánh giá chất lượng định lượng bằng Eval & Quality Gate:** Bổ sung RAG (v2) giúp accuracy tăng từ 4.35% lên 100% và giảm cost/câu đúng hơn 11 lần; gắn `run_eval.py` vào CI giúp chặn regression trước khi merge.
  3. **Liên kết Metrics – Traces – Logs:** Prometheus/Grafana cho biết *khi nào* hệ thống bất thường và alert nào firing, Jaeger chỉ ra *span nào* gây chậm/lỗi, còn JSON log chứa `trace_id` giúp đối chiếu trực tiếp tới từng request cụ thể.
- Nếu đưa agent này lên production thật, bạn sẽ thêm metric / alert nào nữa?

Thêm metric theo dõi **TTFT (Time To First Token)** cho streaming, **tỉ lệ cache hit của Redis**, **số request bị chặn bởi Rate Limit (429) / Cost Guard (402)** theo tenant, và **alert khi accuracy trên bộ eval định kỳ tụt dưới ngưỡng**.
