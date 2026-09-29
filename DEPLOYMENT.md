# Thông Tin Deploy — Checkpoint 5

> Tài liệu tổng kết quá trình triển khai service và kiểm thử.
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
| Public URL | https://k4-l3b-day12-thanthikimchi-agent.onrender.com |
| Platform | Render (hỗ trợ Blueprint render.yaml & Redis instance) |
| Ngày deploy | 2026-09-29 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán (mặc định 8000 khi local) |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard Settings → Secrets, không nằm trong repo |
| `REDIS_URL` | ✅ | kết nối tới Redis instance nội bộ của Render / Docker Compose network |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Các lệnh curl kiểm tra theo kịch bản:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i http://localhost:8000/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i http://localhost:8000/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST http://localhost:8000/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output thực tế từ terminal khi chạy kiểm thử stack:

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
{"answer":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud. (Mình đang nhớ 6 lượt trao đổi trước đó.)","user_id":"sv-test","history_length":6,"cost_usd":0.00004785,"tokens":{"in":139,"out":45}}

5. Sliding-window rate limit test (15 requests liên tiếp):
429 429 429 429 429 429 429 429 429 429 429 429 200 200 200
```

## Ảnh Chụp Màn Hình

Đã lưu ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trạng thái quản lý container và stack qua docker compose ps & service monitor
- `screenshots/health.png` — kết quả kiểm tra endpoint /health, /ready và xác thực an toàn

---

## Phương Án Dự Phòng (Local Fallback)

Trong trường hợp cần kiểm tra nhanh không phụ thuộc mạng ngoài:
1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. Chạy `docker compose up -d` và kiểm tra container bằng `docker compose ps`
3. Bộ test `pytest tests/test_cp5.py -v` tự động kiểm tra stack local qua `http://localhost:8000`
4. Lý do ghi nhận: Sử dụng stack Docker Compose chuẩn hóa môi trường local kết hợp pipeline CI/CD GitHub Actions tự động hóa kiểm thử để đảm bảo tính toàn vẹn hệ thống và dự phòng khi mạng hạn chế.
