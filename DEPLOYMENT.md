# Thông Tin Deploy — Checkpoint 5

> Tài liệu tổng kết quá trình triển khai service và kiểm thử trên Cloud.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Thân Thị Kim Chi |
| Mã học viên | 2A202602797 |
| Repo | https://github.com/kchi24/K4-L3B-ThanThiKimChi-2A202602797-Cloud-Service-And-Deployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-0a72.onrender.com |
| Platform | Render (Blueprint qua render.yaml & Key Value Redis) |
| Ngày deploy | 2026-09-29 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform Render tự gán khi khởi chạy container |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard Blueprint Settings (sync: false), không lưu trong repo |
| `REDIS_URL` | ✅ | Render tự động liên kết từ service day12-redis (property: connectionString) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Các lệnh curl kiểm tra theo kịch bản:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://day12-agent-0a72.onrender.com/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-agent-0a72.onrender.com/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://day12-agent-0a72.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://day12-agent-0a72.onrender.com/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://day12-agent-0a72.onrender.com/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output thực tế từ terminal khi gọi vào Public URL trên Render:

```text
1. Liveness check:
HTTP/1.1 200 OK
content-type: application/json
{"status":"ok","service":"day12-agent","version":"1.0.0"}

2. Readiness check:
HTTP/1.1 200 OK
content-type: application/json
{"status":"ready","redis":true}

3. Unauthorized ask check:
HTTP/1.1 401 Unauthorized
content-type: application/json
{"detail":"invalid or missing API key"}

4. Authorized ask check:
HTTP/1.1 200 OK
content-type: application/json
{"answer":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud.","user_id":"sv-test","history_length":0,"cost_usd":0.00002145,"tokens":{"in":3,"out":35}}

5. Sliding-window rate limit test (15 requests liên tiếp):
429 429 429 429 429 429 429 429 429 429 429 429 200 200 200
```

## Ảnh Chụp Màn Hình

Đã lưu ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service day12-agent ở trạng thái Deploy succeeded / Live và log runtime thực tế trên Render
- `screenshots/redis_log.png` — log service day12-redis ở trạng thái Ready to accept connections trên Render
- `screenshots/health.png` — kết quả kiểm tra endpoint /health, /ready và xác thực an toàn qua Public URL https://day12-agent-0a72.onrender.com
