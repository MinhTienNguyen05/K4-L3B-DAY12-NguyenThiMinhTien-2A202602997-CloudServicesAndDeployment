# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục           | Nội dung                                                               |
| -------------- | ----------------------------------------------------------------------- |
| Họ và tên   | Nguyễn Thị Minh Tiến                                                   |
| Mã học viên | 2A202602997                                                            |
| Repo           | K4-L3B-DAY12-NguyenThiMinhTien-2A202602997-CloudServicesAndDeployment  |

## Service

| Mục         | Nội dung                                                                            |
| ------------ | ------------------------------------------------------------------------------------ |
| Public URL   | https://day12-agent-production-defb.up.railway.app                                    |
| Platform     | Railway                                                                             |
| Ngày deploy | 2026-09-29                                                                          |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến                     | Đã set | Ghi chú                                             |
| ------------------------- | -------- | ---------------------------------------------------- |
| `PORT`                  | ✅       | platform tự gán                                    |
| `AGENT_API_KEY`         | ✅       | đặt trong dashboard, không nằm trong repo        |
| `REDIS_URL`             | ✅       | Redis add-on của Railway (${{Redis.REDIS_URL}})     |
| `RATE_LIMIT_PER_MINUTE` | ✅       | 10                                                   |
| `MONTHLY_BUDGET_USD`    | ✅       | 10.0                                                 |
| `LOG_LEVEL`             | ✅       | INFO                                                 |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://day12-agent-production-defb.up.railway.app/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-agent-production-defb.up.railway.app/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://day12-agent-production-defb.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://day12-agent-production-defb.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://day12-agent-production-defb.up.railway.app/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo


## Kết Quả Chạy Thật
```text
1.
HTTP/2 200
content-type: application/json
date: Tue, 29 Sep 2026 05:28:32 GMT
server: railway-hikari
x-railway-request-id: VmeT8M5OSYeXzIuixtoGcA
content-length: 57
x-hikari-trace: sin1.hs0s
x-railway-edge: sin1

{"status":"ok","service":"day12-agent","version":"1.0.0"}%

2.
HTTP/2 200
content-type: application/json
date: Tue, 29 Sep 2026 05:28:44 GMT
server: railway-hikari
x-railway-request-id: R49U8h_vS8aG6OE09o6EoQ
content-length: 31
x-hikari-trace: hkg1.hn7d
x-railway-edge: hkg1

{"status":"ready","redis":true}%

3.
HTTP/2 401
content-type: application/json
date: Tue, 29 Sep 2026 05:28:51 GMT
server: railway-hikari
x-railway-request-id: 2dE7bIm7Sb-f1UcGCYBc-A
content-length: 39
x-hikari-trace: hkg1.aebn
x-railway-edge: hkg1

{"detail":"invalid or missing API key"}%

4.
HTTP/2 401
content-type: application/json
date: Tue, 29 Sep 2026 05:28:58 GMT
server: railway-hikari
x-railway-request-id: mGTy7uXXStmCG8sZLPU1MQ
content-length: 39
x-hikari-trace: sin1.tr00
x-railway-edge: sin1

{"detail":"invalid or missing API key"}%


5.
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429 
```