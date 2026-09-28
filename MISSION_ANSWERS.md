# Day 12 Lab - Mission Answers

> **AICB-P1 · VinUniversity 2026**  
> **Course:** Day 12 - Cloud Deployment & Infrastructure

---

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found in `01-localhost-vs-production/develop/app.py`
1. **Hardcoded Secrets:** `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"` và `DATABASE_URL` bị đặt trực tiếp trong mã nguồn. Khi commit lên GitHub, secrets sẽ bị lộ.
2. **Missing Configuration Management:** Cấu hình ứng dụng (`DEBUG = True`, `MAX_TOKENS = 500`) không đọc từ biến môi trường (vi phạm nguyên tắc Factor III của The Twelve-Factor App).
3. **Insecure & Unstructured Logging:** Sử dụng `print()` thay vì logging framework chuẩn. Đồng thời log trực tiếp secret key ra stdout (`print(f"[DEBUG] Using key: {OPENAI_API_KEY}")`).
4. **Missing Health Check / Readiness Endpoints:** Không có route `/health` hay `/ready`, khiến các platform như Railway, Render, Kubernetes không thể giám sát trạng thái container để tự động restart hoặc điều phối traffic.
5. **Hardcoded Binding & Debug Reload:** Thiết lập `host="localhost"` (khiến container không nhận được kết nối từ ngoài), `port=8000` cố định (không tương thích với biến môi trường `PORT` của cloud), và bật `reload=True` trong môi trường production.

---

### Exercise 1.3: Comparison Table (Basic vs Advanced)

| Feature | Develop (`basic`) | Production (`advanced`) | Tại sao quan trọng? |
| :--- | :--- | :--- | :--- |
| **Config** | Hardcoded trong source code | Environment variables (`.env` / Pydantic Settings) | Bảo mật secrets, dễ dàng cấu hình cho từng môi trường (dev, staging, prod) mà không cần build lại code. |
| **Health Check** | Không có | `GET /health` (liveness) & `GET /ready` (readiness) | Giúp container orchestrator / load balancer phát hiện sự cố và tự động phục hồi instance. |
| **Logging** | `print()` plain text | Structured JSON Logging (`{"ts", "lvl", "msg"}`) | Giúp log aggregator (Datadog, Loki, CloudWatch) dễ dàng parse, filter, và cảnh báo real-time. |
| **Shutdown** | Dừng đột ngột (`SIGKILL`) | Graceful shutdown (`lifespan`, bắt `SIGTERM`) | Đảm bảo hoàn thành các request đang dở dang và đóng các kết nối database/Redis an toàn. |
| **Binding** | `localhost:8000` | `0.0.0.0:${PORT}` | Container bắt buộc bind `0.0.0.0` để tiếp nhận kết nối qua Docker bridge network và ingress proxy. |
| **CORS** | Không kiểm soát | `CORSMiddleware` với whitelist origins | Ngăn chặn các website giả mạo thực hiện request trái phép từ trình duyệt người dùng. |

---

## Part 2: Docker Containerization

### Exercise 2.1: Dockerfile Questions
1. **Base image là gì?**  
   Base image trong `02-docker/develop/Dockerfile` là `python:3.11`. Đây là image Debian đầy đủ đi kèm runtime Python 3.11 cùng nhiều tiện ích hệ thống.
2. **Working directory là gì?**  
   Working directory là `WORKDIR /app`. Đây là thư mục làm việc mặc định bên trong container cho các lệnh tiếp theo (`COPY`, `RUN`, `CMD`).
3. **Tại sao `COPY requirements.txt` trước?**  
   Để tận dụng **Docker layer caching**. Dependencies ít khi thay đổi hơn mã nguồn. Bằng cách copy và cài `requirements.txt` trước, Docker tái sử dụng cache của layer này khi ta sửa đổi code, giúp tốc độ build nhanh hơn đáng kể.
4. **`CMD` vs `ENTRYPOINT` khác nhau thế nào?**  
   - `ENTRYPOINT`: Xác định tiến trình cố định chạy khi container khởi động (không bị ghi đè bởi command line argument thông thường).
   - `CMD`: Xác định tham số mặc định cho `ENTRYPOINT` (hoặc lệnh mặc định nếu không có entrypoint). Tham số này dễ dàng bị ghi đè khi chạy `docker run <image> <override_command>`.

---

### Exercise 2.3: Image Size Comparison
- **Develop (`python:3.11` single-stage):** ~1020 MB
- **Production (`python:3.11-slim` multi-stage, non-root):** ~248 MB (Content size: 58.6 MB)
- **Difference:** Giảm hơn **75%** dung lượng image.

**Multi-stage Build Analysis:**
- **Stage 1 (Builder):** Sử dụng `python:3.11-slim`, cài đặt `gcc`, `libpq-dev` và build dependencies vào thư mục `/root/.local`.
- **Stage 2 (Runtime):** Tạo container sạch chỉ chứa runtime Python, copy dependencies từ builder sang non-root user (`appuser` hoặc `agent`), loại bỏ hoàn toàn compiler và cache apt.

---

### Exercise 2.4: Docker Compose Stack Architecture
Hệ thống gồm 4 services:
1. `nginx`: Reverse proxy & Load balancer ở cổng 80/443.
2. `agent`: FastAPI AI Agent (hỗ trợ scale nhiều replicas).
3. `redis`: Lưu trữ session state, token bucket cho rate limiting, và token metrics.
4. `qdrant`: Vector database phục vụ RAG.

---

## Part 3: Cloud Deployment

### So Sánh Nền Tảng Triển Khai
| Platform | Độ phức tạp | Ưu điểm | Phù hợp nhất |
| :--- | :--- | :--- | :--- |
| **Railway** | Rất thấp (1-click / CLI) | Hỗ trợ Dockerfile trực tiếp, cấp domain HTTPS tự động | Demo, MVP, Prototype |
| **Render** | Thấp | Cấu hình Blueprint `render.yaml`, quản lý service + database tập trung | Side projects, SME |
| **GCP Cloud Run** | Trung bình - Cao | Serverless container tự động scale-to-zero, chuẩn enterprise | Production chịu tải lớn |

---

## Part 4: API Security

### Exercise 4.1 - 4.3: Test Results

#### 1. Kiểm tra API Key Authentication:
- Không gửi header `X-API-Key`:
  ```
  POST /ask -> 401 Unauthorized
  {"detail": "Invalid or missing API key. Include header: X-API-Key: <key>"}
  ```
- Gửi `X-API-Key: dev-key-change-me-in-production`:
  ```
  POST /ask -> 200 OK
  {"question": "What is Docker?", "answer": "Container là cách đóng gói app để chạy ở mọi nơi...", "model": "gpt-4o-mini"}
  ```

#### 2. Kiểm tra Rate Limiting (Sliding Window):
- Cấu hình: `RATE_LIMIT_PER_MINUTE=20`
- Gửi 21 requests liên tiếp:
  - Requests 1 → 20: HTTP `200 OK`
  - Request 21: HTTP `429 Too Many Requests`
  - Detail: `{"detail": "Rate limit exceeded: 20 req/min"}` kèm header `Retry-After: 60`.

---

### Exercise 4.4: Cost Guard Implementation
**Mục tiêu:** Tránh bill bất ngờ từ OpenAI/LLM provider.  
**Cơ chế hoạt động:**
1. Mỗi user được cấp một ngân sách chi tiêu hàng ngày (mặc định $5.0 - $10.0/ngày).
2. Trước khi gọi LLM: Tính toán độ dài câu hỏi (input tokens) và kiểm tra xem tổng chi tiêu trong ngày cộng với chi phí ước tính có vượt ngân sách hay không.
3. Nếu vượt ngân sách: Trả về HTTP `402 Payment Required` hoặc `503 Service Unavailable`.
4. Sau khi có phản hồi: Cập nhật số token thực tế vào Redis/Memory và tích lũy chi phí. Tự động reset bộ đếm khi sang ngày mới (UTC).

---

## Part 5: Scaling & Reliability

### Exercise 5.1: Health & Readiness Checks
- `/health`: Liveness probe kiểm tra process còn sống và mock/real LLM available.
- `/ready`: Readiness probe trả về `200 OK` khi container đã khởi tạo xong và Redis sẵn sàng, trả về `503` khi đang khởi động hoặc shutdown.

### Exercise 5.2: Graceful Shutdown
- Ứng dụng đăng ký signal handler cho `SIGTERM` và `SIGINT`.
- Khi nhận tín hiệu dừng: đặt cờ `_is_ready = False` (ngừng nhận request mới), chờ các request đang xử lý (`_in_flight_requests`) hoàn tất (timeout 30s) trước khi thoát.

### Exercise 5.3 & 5.5: Stateless Design với Redis
- Session state và conversation history được lưu trữ trong Redis theo key `session:{session_id}`.
- Kiểm tra scale multi-turn: Client gửi liên tiếp các câu hỏi cùng `session_id`, bất kỳ instance nào trong cụm load balancer đều có thể tải ngữ cảnh và trả lời chính xác, bảo toàn trọn vẹn 4 messages trong history.
