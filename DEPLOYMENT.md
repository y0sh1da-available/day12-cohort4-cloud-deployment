# Deployment Information

## 1. Local Deployment (Docker Compose)
Dịch vụ hiện đang chạy và được kiểm thử thành công trên Docker local:

- **Local Host URL:** `http://localhost:8000`
- **Swagger Docs:** `http://localhost:8000/docs`
- **Health Check:** `http://localhost:8000/health`
- **Readiness Check:** `http://localhost:8000/ready`

## 2. Cloud Platforms Deployment Configs

### Railway (`railway.toml`)
```bash
# Cài đặt Railway CLI & login
npm i -g @railway/cli
railway login

# Khởi tạo & deploy từ thư mục 06-lab-complete
cd 06-lab-complete
railway init
railway variables set AGENT_API_KEY=dev-key-change-me-in-production
railway variables set ENVIRONMENT=production
railway up
railway domain
```

### Render (`render.yaml`)
1. Push repo lên GitHub cá nhân.
2. Tại dashboard Render: Chọn **New** -> **Blueprint**.
3. Kết nối repository, Render tự động đọc file `render.yaml` tạo cụm service gồm:
   - Web service: `FastAPI AI Agent`
   - Database: `Redis Instance`

---

## 3. Test Commands

### Health Check (Liveness Probe)
```bash
curl http://localhost:8000/health
# Expected: {"status": "ok", "version": "1.0.0", "environment": "staging"}
```

### Readiness Check (Readiness Probe)
```bash
curl http://localhost:8000/ready
# Expected: {"ready": true}
```

### Protected Endpoint Test (Without Key -> 401)
```bash
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# Expected: 401 Unauthorized
```

### Protected Endpoint Test (With API Key -> 200 OK)
```bash
curl -X POST http://localhost:8000/ask \
  -H "X-API-Key: dev-key-change-me-in-production" \
  -H "Content-Type: application/json" \
  -d '{"question": "What is Docker?"}'
# Expected: 200 OK with AI answer
```

### Metrics & Usage Check
```bash
curl http://localhost:8000/metrics \
  -H "X-API-Key: dev-key-change-me-in-production"
```

---

## 4. Environment Variables Reference
| Variable | Value | Description |
| :--- | :--- | :--- |
| `PORT` | `8000` | Port lắng nghe kết nối của Web Server |
| `ENVIRONMENT` | `staging` / `production` | Môi trường triển khai |
| `AGENT_API_KEY` | `dev-key-change-me-in-production` | Secret key bảo vệ API |
| `REDIS_URL` | `redis://redis:6379/0` | Chuỗi kết nối Redis cluster |
| `RATE_LIMIT_PER_MINUTE` | `20` | Giới hạn số lượng request trên mỗi API key |
| `DAILY_BUDGET_USD` | `5.0` | Ngân sách chi tiêu tối đa mỗi ngày |
