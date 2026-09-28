# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | NGUYỄN VĂN SƠN |
| Mã học viên | 2A202602744 |
| Repo | https://github.com/cakho6179/K4-L3A-DAY12-NguyenVanSon-2A202602744-CloudServicesAndDeployment |

> Lưu ý: đổi `cakho6179` thành username GitHub của bạn sau khi fork/đổi tên repo theo đúng cú pháp `K4-L3A-DAY12-NguyenVanSon-2A202602744-CloudServicesAndDeployment`.

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://railway-add-database-redis-production.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-09-28 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard Railway → Variables, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis add-on của Railway (`railway add --database redis`) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây sau khi deploy Railway xong:

```
# 1. Liveness — 200 {"status":"ok"}
{"status":"ok","service":"day12-agent","version":"1.0.0"}

# 2. Readiness — 200 {"status":"ready"} (đã nối Redis add-on qua REDIS_URL private)
{"status":"ready","redis":true}

# 3. Không API key — 401
401

# 4. Có API key — 200 kèm câu trả lời (user sv-test, history_length 0)
{"answer":"Ngắn gọn: Deploy la gi phụ thuộc vào ba yếu tố — cấu hình qua biến môi trường, health check để orchestrator biết trạng thái, và giới hạn tài nguyên.","user_id":"sv-test","history_length":0,...}

# 5. Rate limit — 12 request liên tiếp với user rate-test: 10x200 rồi 2x429
[200, 200, 200, 200, 200, 200, 200, 200, 200, 200, 429, 429]
```

Các bước deploy Railway (làm tiếp sau khi code đã xanh CP1–CP4):

```bash
npm i -g @railway/cli
railway login
railway init
railway add --database redis
railway variables --set AGENT_API_KEY=<khoa-rieng-cua-ban> --set RATE_LIMIT_PER_MINUTE=10 --set MONTHLY_BUDGET_USD=10.0 --set LOG_LEVEL=INFO
railway up
railway domain
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl

---

## Nếu Dùng Phương Án Dự Phòng

Không đăng ký được tài khoản cloud? Vẫn nộp được bài, nhưng CP5 tối đa 60% điểm:

1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. Chạy `docker compose up -d` rồi kiểm tra `docker compose ps`
3. Chụp màn hình vào `screenshots/`
4. Chạy `pytest tests/test_cp5.py -v` — bộ test sẽ tự chuyển sang kiểm tra
   `http://localhost:8000`
5. Ghi rõ lý do không deploy được vào phần dưới đây:

```
Deploy thành công đường Railway cloud thật ngày 2026-09-28, không dùng phương án dự phòng.
```
