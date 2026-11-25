# Day 14 Lab - Mission Answers

**Họ tên:** Phạm Nguyễn Tất Đạt  
**MSSV:** 2A202603401  

---

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found (`01-localhost-vs-production/develop/app.py`)
1. **Hardcode secrets trong source code** (dòng 17–18): `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"` và `DATABASE_URL = "postgresql://admin:password123@localhost:5432/mydb"`. Nếu push lên GitHub sẽ lộ ngay lập tức.
2. **Không có config management qua biến môi trường** (dòng 21–22): `DEBUG = True`, `MAX_TOKENS = 500` bị hardcode, không thể đổi giữa dev/staging/prod mà không sửa code.
3. **Dùng `print()` thay cho structured logging và log cả secret** (dòng 33–34, 38): `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")` làm lộ API key ra log và khó parse/filter trên production.
4. **Thiếu health check / readiness endpoint** (dòng 42–43): Không có `GET /health` hay `GET /ready` nên cloud platform/load balancer không biết khi nào app chết hoặc chưa sẵn sàng.
5. **Hardcode host/port và bật `reload=True`** (dòng 49–54): `host="localhost"` không nhận được traffic từ bên ngoài container, `port=8000` không đọc `PORT` của cloud platform, và `reload=True` gây tốn tài nguyên/không an toàn ở production.

### Exercise 1.2: Kết quả chạy bản develop
Lệnh gọi `POST http://127.0.0.1:8010/ask?question=Hello` (cổng 8010 do cổng 8000 đang bận trên máy; bản develop nhận `question` qua query param):
```text
Status: 200
{"answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}
```

### Exercise 1.3: So sánh Develop vs Production
| Feature | Develop (`develop/app.py`) | Production (`production/app.py`) | Tại sao quan trọng? |
|---|---|---|---|
| Config | Hardcode trong code (`OPENAI_API_KEY`, `DEBUG = True`, `port=8000`) | Đọc từ biến môi trường qua `config.settings` (`PORT`, `ENVIRONMENT`, `ALLOWED_ORIGINS`...) | Tuân thủ 12-Factor App; đổi cấu hình không cần sửa code, không lộ secret lên Git |
| Host binding | `host="localhost"` | `host=settings.host` (`0.0.0.0`) | `localhost` chỉ nghe loopback; trong Docker/cloud bắt buộc `0.0.0.0` để nhận traffic ngoài |
| Health check | Không có | Có `GET /health` (liveness) và `GET /ready` (readiness, trả 503 nếu chưa ready) | Platform tự restart container khi lỗi và load balancer chỉ route traffic khi sẵn sàng |
| Logging | `print(...)`, in cả secret | `logging` format JSON (`{"time":...,"level":...,"msg":...}`), không log secret | Dễ thu thập và truy vấn trên Datadog/Loki/Cloud Logging, đảm bảo an toàn dữ liệu |
| Shutdown | Tắt đột ngột, không `lifespan` | `lifespan` + `signal.signal(signal.SIGTERM, handle_sigterm)` | Hoàn thành các request đang xử lý (in-flight) trước khi thoát, tránh lỗi 502 cho user |
| CORS & Reload | Không CORS, `reload=True` | `CORSMiddleware` giới hạn origins, `reload=settings.debug` | Bảo vệ API khỏi cross-origin trái phép và tránh overhead của file watcher ở production |

---

## Part 2: Docker

### Exercise 2.1: Dockerfile questions (`02-docker/develop/Dockerfile`)
1. **Base image:** `python:3.11` (bản đầy đủ ~1 GB).
2. **Working directory:** `/app` (`WORKDIR /app`).
3. **Tại sao `COPY requirements.txt` đứng trước `COPY . .`:** Để tận dụng **Docker layer cache**. `requirements.txt` ít thay đổi hơn source code; nếu đặt trước, khi sửa `app.py` Docker chỉ build lại từ bước copy code mà không phải chạy lại `pip install`.
4. **Khác nhau giữa `CMD` và `ENTRYPOINT`:** `ENTRYPOINT` định nghĩa executable cố định của container (khó bị ghi đè khi `docker run ... <args>`, các args truyền thêm sẽ nối vào sau), còn `CMD` cung cấp lệnh hoặc tham số mặc định và dễ bị ghi đè hoàn toàn nếu truyền command lúc `docker run`.

### Exercise 2.2 & 2.3: Build, đo kích thước và so sánh image
Chạy build từ thư mục gốc repo:
```bash
docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .
docker build -f 02-docker/production/Dockerfile -t my-agent:advanced .
docker images my-agent
```
Kết quả đo thực tế bằng `docker images my-agent`:
- **Develop (`my-agent:develop`):** `1.67GB` (1670 MB)
- **Production (`my-agent:advanced`):** `240MB`
- **Chênh lệch:** Giảm **1430 MB (~85.6%)** nhờ dùng multi-stage build: stage `builder` (`python:3.11-slim` + `gcc`, `libpq-dev`) chỉ dùng để compile/cài dependencies vào `/root/.local`, sau đó stage `runtime` (`python:3.11-slim`) chỉ copy `/root/.local` và source code sang, bỏ lại toàn bộ compiler, cache apt/pip và chạy bằng user non-root `appuser`.

### Exercise 2.4: Docker Compose stack (`02-docker/production/docker-compose.yml`)
**Kiến trúc các service trong stack:**
```text
Client (HTTP :80 / HTTPS :443)
   │
   ▼
[ nginx ] (nginx:alpine, reverse proxy)
   │  (network: internal)
   ▼
[ agent ] (multi-stage runtime, port 8000 nội bộ)
   ├──▶ [ redis ]  (redis:7-alpine, cache & rate limit)
   └──▶ [ qdrant ] (qdrant/qdrant:v1.9.0, vector DB cho RAG)
```
`nginx` nhận request từ bên ngoài ở cổng 80/443 và forward vào `agent` qua mạng nội bộ `internal`. `agent` chỉ khởi động sau khi `redis` và `qdrant` vượt qua `healthcheck` (`service_healthy`).

---

## Part 3: Cloud Deployment

### Exercise 3.1 / 3.2: Deploy Agent Part 6 lên Cloud
- **Public URL:** *(Chờ user đăng nhập Railway/Render và deploy thư mục `06-lab-complete/` — xem hướng dẫn trong `DEPLOYMENT.md`)*
- **Screenshots:** `screenshots/dashboard.png`, `screenshots/running.png`, `screenshots/test.png`

### So sánh `railway.toml` và `render.yaml`
- **`railway.toml`** (định dạng TOML): Cấu hình tối giản cho Railway, chỉ khai báo `[build] builder = "DOCKERFILE"` và `[deploy] healthcheckPath = "/health"`, chính sách restart (`ON_FAILURE`). Biến môi trường và Redis service được cấu hình qua Dashboard/CLI (`railway variables set`), không khai báo trực tiếp trong `railway.toml`.
- **`render.yaml`** (định dạng YAML — Blueprint / Infrastructure-as-Code): Mô tả đầy đủ các service (`type: web`, `runtime: docker`, `healthCheckPath: /health`) lẫn danh sách `envVars` ngay trong file. Với secret như `AGENT_API_KEY`, Render cho phép dùng `generateValue: true` (tự sinh key ngẫu nhiên) hoặc `sync: false` (nhập tay trên dashboard khi deploy) để không lộ secret trong Git.

### Exercise 3.3: GCP Cloud Run CI/CD (`03-cloud-deployment/production-cloud-run/`)
Pipeline trong `cloudbuild.yaml` tự động kích hoạt khi push lên nhánh `main` và đi qua 4 bước tuần tự (`waitFor`): (1) **test** dùng `python:3.11-slim` cài dependencies và chạy `pytest`; (2) **build** đóng gói Docker image gắn tag `$COMMIT_SHA` và `latest` (tận dụng `--cache-from`); (3) **push** đẩy image lên Container Registry (`gcr.io/$PROJECT_ID/ai-agent`); (4) **deploy** dùng `gcloud run deploy` triển khai lên Cloud Run (`asia-southeast1`) với `min-instances=1`, `max-instances=10`, giới hạn `512Mi` RAM / `1` CPU và nạp secret từ Secret Manager. File `service.yaml` định nghĩa cấu hình Knative/Cloud Run dạng IaC gồm autoscaling (concurrency 80), `livenessProbe` (`/health`), `startupProbe` (`/ready`) và tham chiếu `secretKeyRef`.

---

## Part 4: API Security

### Exercise 4.1–4.3: Kết quả test thực tế
- **Exercise 4.1 (`04-api-gateway/develop/app.py`, `AGENT_API_KEY=secret-key-123`):**
  - Không có `X-API-Key`:
    ```text
    HTTP/1.1 401 Unauthorized
    {"detail":"Missing API key. Include header: X-API-Key: <your-key>"}
    ```
  - Có `X-API-Key: secret-key-123`:
    ```text
    HTTP/1.1 200 OK
    {"question":"Hello","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}
    ```
- **Exercise 4.2 & 4.3 (`04-api-gateway/production/app.py`, tài khoản `student` / `demo123`, giới hạn 10 req/phút):**
  - Lấy token từ `POST /auth/token` → `200 OK` (token đã che: `Bearer eyJ...REDACTED`).
  - Gọi vòng lặp `POST /ask`:
    - Request 1..10: `200 OK` (`requests_remaining` giảm từ `9` xuống `0`).
    - Request 11..14: `429 Too Many Requests`:
      ```text
      HTTP/1.1 429 Too Many Requests
      {"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":59}}
      ```
  - **Thuật toán & bypass cho admin:** `rate_limiter.py` dùng thuật toán **Sliding Window Counter** (`deque` lưu timestamp trong 60 giây). Tài khoản `admin` dùng instance `rate_limiter_admin` (100 req/phút) thay cho `rate_limiter_user` (10 req/phút).

### Exercise 4.4: Cost guard
- Rate limit chỉ giới hạn **tần suất** (số request/phút), không kiểm soát được **chi phí** (một câu hỏi dài hoặc model đắt vẫn có thể đốt hết tiền dù dưới ngưỡng rate limit). Do đó cần cả hai.
- **Cách triển khai:** Trước khi gọi LLM kiểm tra tổng chi phí hiện tại trong Redis (`cost:daily:<date>`) so với `DAILY_BUDGET_USD`; nếu vượt thì trả `HTTP 402`. Sau khi gọi LLM xong, tính `cost = (input_tokens/1000)*PRICE_INPUT_PER_1K + (output_tokens/1000)*PRICE_OUTPUT_PER_1K` rồi cộng dồn bằng `INCRBYFLOAT` kèm `EXPIRE`.

---

## Part 5: Scaling & Reliability

### Exercise 5.1–5.5: Ghi chú & kết quả test stateless
- **Liveness (`/health`) vs Readiness (`/ready`):** `/health` kiểm tra process còn sống (luôn trả `200`, không phụ thuộc Redis) — fail thì platform restart container. `/ready` kiểm tra phụ thuộc như Redis (`r.ping()`) — trả `503` nếu mất kết nối để load balancer tạm ngừng route traffic vào instance đó mà không restart oan.
- **Graceful shutdown:** Đăng ký `SIGTERM` handler để ghi log sự kiện shutdown và gọi lại handler gốc của `uvicorn`, giúp hoàn thành các request đang chạy trước khi thoát với exit code `0`.
- **Stateless với Redis + Nginx Load Balancer (`05-scaling-reliability/production/`):**
  Chạy `docker compose -p k5-day14-scaling up -d --build --scale agent=3` và `python test_stateless.py`:
  ```text
  Session ID: 1496c6aa-c748-459b-9b72-cc1eeab35b13
  Request 1: [instance-54ed11]
  Request 2: [instance-38b311]
  Request 3: [instance-22732e]
  Request 4: [instance-54ed11]
  Request 5: [instance-38b311]
  Total requests: 5
  Instances used: {'instance-22732e', 'instance-38b311', 'instance-54ed11'}
  ✅ All requests served despite different instances!
  Total messages: 10
  ✅ Session history preserved across all instances via Redis!
  ```
  **Giải thích:** Nếu lưu `conversation_history` trong RAM, khi Nginx round-robin sang instance khác, instance mới không có lịch sử cũ. Lưu state trong Redis giúp cả 3 instance dùng chung dữ liệu phiên làm việc.

---

## Part 6: Final Project (`06-lab-complete/`)

Kết quả chạy `python check_production_ready.py` đầy đủ trong môi trường Linux + Docker (`lab14-checker` với `/var/run/docker.sock`):
- Docker image build: `69 MB` (`< 500 MB`)
- `docker stop` graceful shutdown: exit code `0`, `1.1s`

```text
  📋 Checklist Deliverable
     ✅ Dockerfile (multi-stage, < 500 MB)
     ✅ docker-compose.yml (agent + redis)
     ✅ .dockerignore
     ✅ Health check endpoint (GET /health)
     ✅ Readiness endpoint (GET /ready)
     ✅ API Key authentication
     ✅ Rate limiting
     ✅ Cost guard
     ✅ Config từ environment variables
     ✅ Structured logging
     ✅ Graceful shutdown
     ✅ Public URL ready (Railway / Render config)
----------------------------------------------------------------
  Result: 12/12 tiêu chí đạt · 43/43 checks passed
  🎉 PASS 12/12 — PRODUCTION READY! Deploy nào!
```

---

## AI log
- **Công cụ sử dụng:** Antigravity AI Assistant.
- **Nội dung đã hỏi / hỗ trợ:**
  - Tổng hợp hướng dẫn lab và các lưu ý riêng của lớp K5 từ `README.md`, `CHECKPOINTS.md`, `CODE_LAB.md`, `RUBRIC.md`.
  - Hỗ trợ hoàn thiện các module `TODO` trong `06-lab-complete/` (`config.py`, `redis_client.py`, `auth.py`, `rate_limiter.py`, `cost_guard.py`, `main.py`, `Dockerfile`, `docker-compose.yml`, `.dockerignore`, `.env.example`, `railway.toml`, `render.yaml`) và kiểm tra bằng `check_production_ready.py` đạt `PASS 12/12`.
  - Chạy thực nghiệm các bài tập Part 1, 2, 4, 5, 7 và kiểm chứng số liệu thực tế từ Docker, curl, `load_test.py`, `run_eval.py` và Locust.
