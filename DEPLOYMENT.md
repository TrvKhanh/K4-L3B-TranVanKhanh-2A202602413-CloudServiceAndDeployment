# Thông Tin Deploy — Checkpoint 5

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Trần Văn Khánh |
| Mã học viên | 2A202602413 |
| Repo | https://github.com/TrvKhanh/K4-L3B-TranVanKhanh-2A202602413-CloudServiceAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-prod.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-09-29 |

## Biến Môi Trường Đã Set Trên Cloud

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis add-on của platform (upstash/railway redis) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://day12-agent-prod.up.railway.app/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-agent-prod.up.railway.app/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://day12-agent-prod.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://day12-agent-prod.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://day12-agent-prod.up.railway.app/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

```json
HTTP/1.1 200 OK
content-type: application/json
{"status":"ok","service":"day12-agent","version":"1.0.0"}

HTTP/1.1 200 OK
content-type: application/json
{"status":"ready","redis":true}

HTTP/1.1 401 Unauthorized
content-type: application/json
{"detail":"invalid or missing API key"}

HTTP/1.1 200 OK
content-type: application/json
{"answer":"Deploy là quá trình đưa ứng dụng lên hạ tầng thực tế...","user_id":"sv-test","history_length":0,"cost_usd":0.0001,"tokens":{"in":12,"out":25}}

200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Đã lưu ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl
