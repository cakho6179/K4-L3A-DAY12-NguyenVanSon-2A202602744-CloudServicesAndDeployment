# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: 10 câu bên dưới đã được trả lời đầy đủ.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: NGUYỄN VĂN SƠN  Mã học viên: 2A202602744

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tôi deploy lên Railway và quên set `AGENT_API_KEY` trong dashboard (chỉ set ở file `.env` máy local, mà `.env` không được push). Nếu `agent_api_key` có mặc định `"changeme"`, app vẫn start xanh, health check vẫn 200, tôi tưởng deploy thành công. Vài giờ sau bot quét Internet phát hiện `/ask` mở và gọi miễn phí bằng key mặc định `changeme` — tiền LLM bị đốt mà tôi không hay. Với thiết kế hiện tại (không mặc định), uvicorn crash ngay lúc start với `ValidationError: agent_api_key Field required`, log deploy đỏ lè, tôi thấy và fix trong 2 phút trước khi có traffic thật.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log thật khi chạy `python -c "from app.logging_utils import log_event; print(log_event('ask_completed', user_id='sv01', cost_usd=0.0001, tokens_in=12, tokens_out=45))"`:

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T09:06:30.347402+00:00", "user_id": "sv01", "cost_usd": 0.0001, "tokens_in": 12, "tokens_out": 45}
```

Hai việc `print` thường không làm được: (1) Trên Railway/Datadog tôi filter `event="ask_completed"` và `SUM(cost_usd) GROUP BY user_id` để tìm user tiêu nhiều tiền nhất hôm nay — với `print` string tự do thì không parse được. (2) Đặt alert "tỷ lệ `level=error` trên 5% trong 5 phút thì gửi cảnh báo" — log có trường `level` và `timestamp` ISO-8601 chuẩn nên máy đếm được, còn `print("đã trả lời xong")` không có cấu trúc để đếm.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu, `FROM python:3.11` đầy đủ) | ~1 GB (ước lượng, máy tôi Docker daemon chưa chạy nên chưa build được số thật) |
| Multi-stage (`builder` + `runtime python:3.11-slim`) | ~200–250 MB (ước lượng theo doc python slim) |

Giải thích: phần chênh lệch (~700–800 MB) chính là compiler/toolchain (`build-essential`, gcc, header), pip cache (`--no-cache-dir` đã bỏ ở bản cũ), và bản thân base image đầy đủ (Debian full vs slim). Stage `builder` cài và biên dịch xong rồi bị vứt đi; stage `runtime` chỉ `COPY --from=builder /install` nên image cuối không mang theo rác build. Lưu ý trung thực: máy tôi lúc làm bài `docker info` báo `dockerDesktopLinuxEngine: The system cannot find the file specified` (daemon chưa bật) nên chưa đo được số thật — con số trên là ước lượng từ tài liệu, cần bật Docker Desktop và chạy lại 2 lệnh trên để lấy số đo chính xác.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Dockerfile của tôi: `COPY requirements.txt` → `RUN pip install --prefix=/install` → `COPY app ./app` + `COPY utils ./utils`. Sửa 1 ký tự trong `app/main.py` thì 2 layer đầu (COPY requirements + pip install) được dùng lại từ cache vì input của chúng không đổi; chỉ layer `COPY app` trở đi phải chạy lại (vài giây). Nếu đặt `COPY . .` lên trước `RUN pip install` thì mọi lần sửa code đều làm thay đổi layer COPY đầu tiên, Docker hủy toàn bộ cache từ đó trở đi và chạy lại `pip install` (vài phút + tải mạng). Đây là lý do lab bắt copy requirements trước, source sau.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi: (1) Code Python có lỗ hổng (ví dụ `eval()` câu hỏi user hoặc pickle model không tin cậy) cho kẻ tấn công thực thi lệnh tùy ý trong container. (2) Container mặc định chạy UID 0 = root trong container. (3) Kẻ tấn công dùng container-escape (mount `/var/run/docker.sock` bị hở, kernel exploit, hoặc `docker exec` nhầm quyền) để thoát ra host — vì process trong container là root nên sau khi escape nó có quyền cao trên host (đọc file host, tạo container đặc quyền). Lệnh `USER appuser` (UID 10001) trong Dockerfile cắt đứt ở bước 2: code Python chỉ chạy với quyền user thường, dù bị RCE thì kẻ tấn công cũng chỉ là user không đặc quyền, khó mount, khó ghi file hệ thống, và escape (nếu có) cũng chỉ được quyền thấp.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa 20 request trong 2 giây. Cách: gửi 10 request vào 2 giây cuối của phút N (ví dụ 10:00:58–10:00:59, vừa đủ hạn mức phút N), đợi đồng hồ sang 10:01:00 rồi gửi tiếp 10 request trong 1 giây đầu phút N+1 (hạn mức mới). Tổng 20 request trong khoảng 2 giây ôm lấy mốc :00 mà bộ đếm theo phút vẫn thấy "mỗi phút chỉ 10, đúng luật". Sliding window 60 giây của bài lab không có kẽ hở này vì nó đếm 60 giây gần nhất tính từ `now`, không reset theo đồng hồ.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn *số lượng* request/60s (chống spam, chống burst), key `ratelimit:<user>` dạng ZSET, lỗi 429. Cost guard giới hạn *số tiền*/tháng (chống cháy ngân sách), key `cost:<user>:<YYYY-MM>` dạng counter, lỗi 402. Tình huống 1 (rate cho qua, cost chặn): user gửi đúng 5 request/phút (dưới hạn 10) nhưng mỗi request kèm history dài 20 message + câu hỏi 2000 ký tự, mỗi lần tốn ~$0.05; cuối tháng đã tiêu $9.98/10.0 nên request tiếp theo dù thưa vẫn bị 402. Tình huống 2 (cost cho qua, rate chặn): đầu tháng user mới tiêu $0.01 nhưng gửi 15 request trong 10 giây (test script loop) → vượt 10/phút nên bị 429 dù ngân sách còn nguyên.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự: (1) Redis mất kết nối 30s. (2) Cả 3 container gọi endpoint gộp đều fail vì bên trong có `store.ping()` → trả 503. (3) Orchestrator hiểu nhầm 503 liveness = "process chết" nên restart cả 3 container cùng lúc (thay vì chỉ ngừng gửi traffic). (4) Trong lúc restart không còn container nào phục vụ → downtime toàn hệ thống dù app Python vẫn khỏe. (5) Khi Redis quay lại thì container mới đang cold-start, request đầu còn chậm/timeout thêm. Thiết kế đúng của lab: `/health` không chạm Redis (vẫn 200, không restart), `/ready` chạm Redis (trả 503 để load balancer tạm rút instance khỏi vòng xoay, hết 30s tự vào lại).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Với Redis (bản đúng): `history_length` tăng dần đều 0, 2, 4, 6... vì mọi container cùng đọc/ghi một list `history:<user>` trên Redis — dù request 1 vào container A, request 2 vào container B thì B vẫn thấy 2 message A đã lưu. Nếu lưu trong `dict` RAM của từng process: mỗi container có dict riêng, load balancer round-robin đẩy request đi ngẫu nhiên nên `history_length` nhảy loạn (0 rồi lại 0, hoặc 2 rồi về 0) — agent "mất trí nhớ" ngẫu nhiên. Test `test_lich_su_duoc_dung_lai_giua_cac_request` trong `tests/test_cp4.py` kiểm tra đúng điều này (lượt 2 phải thấy `history_length == 2`).

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi tôi gặp ở môi trường local (chưa deploy cloud thật vì đang chờ tài khoản Railway): `docker info` báo `failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine: The system cannot find the file specified`. Tôi tìm ra bằng cách chạy `docker info` và `docker build` trực tiếp — client Docker 29.1.2 đã cài nhưng Docker Desktop daemon chưa start nên pipe không tồn tại. Cách sửa: mở Docker Desktop → đợi engine chuyển `running` → chạy lại `docker compose up -d`. Bài học cho deploy cloud: lỗi tương đương trên Railway là `/ready` 503 do `REDIS_URL` chưa trỏ sang Redis add-on (trong container `localhost` là chính nó, phải dùng hostname `redis` hoặc URL Upstash) — kiểm tra bằng `railway logs` và tab Variables trên dashboard, fix bằng `railway add --database redis` rồi redeploy. (Mục này sẽ cập nhật output thật sau khi deploy Railway xong.)
