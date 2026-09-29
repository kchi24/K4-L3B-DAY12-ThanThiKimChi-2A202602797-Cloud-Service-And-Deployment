# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
> Cách trả lời: điền câu trả lời chi tiết cho từng câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Thân Thị Kim Chi  Mã học viên: 2A202602797

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống cụ thể: Khi deploy ứng dụng lên nền tảng đám mây (Railway/Render), nếu lập trình viên quên thêm biến môi trường `AGENT_API_KEY` trong dashboard cấu hình.
> Nếu có giá trị mặc định (như `"changeme"` hoặc `"sk-default"`), ứng dụng vẫn khởi động thành công và mở cổng ra Internet. Các bot quét mạng công khai sẽ dò ra key mặc định này trong vài giờ và gọi liên tục vào `/ask`, đốt sạch tài nguyên hoặc làm phát sinh chi phí lớn mà chủ sở hữu không hề biết cho đến khi nhận hóa đơn.
> Ngược lại, nhờ không có mặc định, Pydantic ném `ValidationError` ngay lúc nạp cấu hình làm container dừng ngay lập tức (fail fast). Lỗi hiện ra rõ ràng trên build/runtime log giúp ta phát hiện và bổ sung secret ngay khi đang theo dõi màn hình deploy.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thu được:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T02:46:27.123456+00:00", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 28, "cost_usd": 0.00014}`
> 
> Hai việc làm được với log JSON mà `print` thông thường không làm được:
> 1. Phân tích và tổng hợp số liệu tự động (Aggregation/Metrics): Hệ thống gom log tập trung (Datadog, Loki, CloudWatch) có thể tự động parse các trường số để tính tổng chi phí (`sum(cost_usd)`), vẽ biểu đồ tiêu thụ token theo thời gian hoặc nhóm theo từng `user_id` để biết user nào tiêu tốn nhiều ngân sách nhất.
> 2. Thiết lập bộ lọc và cảnh báo tự động (Alerting & Monitoring): Ta có thể cấu hình rule cảnh báo khi `cost_usd` vượt ngưỡng bất thường hoặc truy vấn nhanh tất cả log có `level == "error"` hay sự cố của một `user_id` cụ thể một cách có cấu trúc mà không cần viết regex bóc tách chuỗi phức tạp.

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
| 1 stage (bản đầu) | ~1.02 GB |
| Multi-stage | 272 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (~750 MB) bao gồm:
> 1. Trình biên dịch và bộ công cụ build hệ thống (gcc, g++, make, build-essential) và các file header libc/kernel dùng để biên dịch thư viện C/C++.
> 2. File cache của trình quản lý gói hệ thống (apt cache) và cache tải về của pip trong quá trình cài đặt dependencies.
> 3. Các package hệ điều hành đầy đủ không cần thiết cho môi trường chạy ứng dụng Python (stage runtime chuyển sang `python:3.11-slim` chỉ giữ lại tối thiểu các gói thư viện cần thiết để thực thi).

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi sửa một ký tự trong `app/main.py` và build lại:
> - Các layer được dùng lại từ cache (CACHED): `FROM ...`, `WORKDIR /app`, `COPY requirements.txt .`, `RUN pip install --prefix=/install ...`, `COPY --from=builder /install /usr/local`, và `RUN useradd ...`.
> - Layer phải chạy lại: Chỉ từ layer `COPY --chown=appuser:appuser . .` trở về sau.
> 
> Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi lần sửa code (dù chỉ một ký tự hay một dấu phẩy), nội dung build context thay đổi làm layer `COPY . .` bị mất cache. Docker sẽ buộc phải chạy lại toàn bộ lệnh `RUN pip install` ở phía sau, tải và cài đặt lại toàn bộ dependencies, khiến thời gian build tăng từ vài giây lên vài phút trong mỗi lần deploy.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện leo thang:
> 1. Ứng dụng Python tồn tại lỗ hổng (ví dụ RCE - Remote Code Execution qua insecure deserialization hoặc injection).
> 2. Kẻ tấn công gửi payload khai thác thành công và mở được shell bên trong container.
> 3. Nếu container chạy với user root (UID 0), tiến trình của kẻ tấn công có quyền root (UID 0) tương đương với host nếu không bật user namespace remap. Kẻ tấn công có thể truy cập các volume mount nhạy cảm từ host (ví dụ docker socket `/var/run/docker.sock` hoặc file hệ thống), khai thác lỗ hổng kernel để thoát container (container breakout) và nắm toàn quyền điều khiển máy host.
> 
> Lệnh `USER appuser` cắt đứt chuỗi ngay tại bước 2: Payload bị ép thực thi dưới tài khoản thường không đặc quyền (UID 10001). Kẻ tấn công không có quyền root, không thể can thiệp vào các tiến trình hay file hệ thống, và bị chặn đứng phần lớn các kỹ thuật leo thang đặc quyền để breakout ra ngoài host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
> 
> Cách đạt được:
> - Người dùng gửi 10 request ở giây thứ `10:00:59` (giây cuối cùng của phút thứ nhất). Vì hạn mức là 10/phút nên hệ thống cho qua cả 10 request.
> - Ngay ở giây tiếp theo `10:01:00` (giây đầu tiên của phút thứ hai), bộ đếm cố định theo phút đồng hồ tự động reset về 0. Người dùng lập tức gửi tiếp 10 request nữa và tiếp tục được cho qua.
> - Kết quả: Trong khoảng thời gian chỉ 2 giây (từ 10:00:59 đến 10:01:00), người dùng đã gửi thành công 20 request, gấp đôi hạn mức mong muốn. Thuật toán cửa sổ trượt (sliding window) khắc phục triệt để lỗi này bằng cách luôn tính chính xác tổng số request trong 60 giây gần nhất tính từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Điểm khác nhau:
> - Rate Limit kiểm soát **tốc độ / tần suất** request (số lượng request / đơn vị thời gian) nhằm chống quá tải server và ngăn chặn tấn công từ chối dịch vụ (DoS).
> - Cost Guard kiểm soát **chi phí tài chính** tích lũy (USD / tháng) phát sinh từ số token mô hình ngôn ngữ tiêu thụ.
> 
> Hai tình huống cụ thể:
> 1. Rate Limit cho qua nhưng Cost Guard chặn: User chỉ gửi 1 request trong phút (hoàn toàn hợp lệ theo rate limit 10 req/phút), nhưng request chứa một tài liệu văn bản khổng lồ dài 100.000 token, chi phí ước tính vượt quá ngân sách tháng còn lại của user -> Cost Guard chặn ngay với mã `402 Payment Required`.
> 2. Cost Guard cho qua nhưng Rate Limit chặn: User mới đầu tháng, còn nguyên $10.0 ngân sách, nhưng viết script gửi dồn dập 15 câu hỏi ngắn liên tiếp chỉ trong 3 giây (mỗi câu chỉ tốn $0.0001) -> Cost Guard vẫn đủ tiền, nhưng Rate Limit sẽ chặn từ request thứ 11 với mã `429 Too Many Requests` do vi phạm tốc độ gọi.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện xảy ra khi gộp chung kiểm tra Redis vào `/health`:
> 1. Giây 0: Redis gặp sự cố tạm thời hoặc mạng chập chờn kéo dài 30 giây.
> 2. Giây 10: Orchestrator (Docker/Kubernetes/Cloud) thực hiện liveness probe định kỳ bằng cách gọi `/health`. Do Redis chết, endpoint trả về lỗi 503 cho cả 3 container.
> 3. Giây 15 - 20: Orchestrator coi liveness probe thất bại nên tự động kill và restart đồng loạt cả 3 container để cố gắng "tự chữa lành".
> 4. Giây 25: Cả 3 container khởi động lại, lại thăm dò `/health`, nhưng Redis vẫn chưa xong sự cố -> tiếp tục fail và lại bị restart tiếp (vào vòng lặp CrashLoopBackOff).
> 5. Giây 30+: Khi Redis vừa kết nối lại được, toàn bộ 3 container của service vẫn đang trong trạng thái khởi động lại/chết, không còn instance nào phục vụ được người dùng.
> -> Tách `/ready` giúp giải quyết vấn đề: khi Redis lỗi, `/ready` báo 503 để load balancer tạm thời ngừng đẩy request mới vào, trong khi `/health` vẫn báo 200 giúp 3 container sống bình thường mà không bị restart vô ích.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Nếu lưu lịch sử trong một dict Python (RAM nội bộ của container):
> - Khi gọi lần 1, request vào container A: A lưu vào RAM của mình, trả về `history_length = 0`.
> - Khi gọi lần 2, load balancer phân phối sang container B: Vì RAM của B không có dữ liệu của A, B coi đây là hội thoại mới và lại trả về `history_length = 0` (agent bị "mất trí nhớ").
> - Khi gọi lần 3, request quay lại container A: A thấy lịch sử lần 1 của mình nên trả về `history_length = 2`.
> - Khi gọi lần 4, request rơi vào container C: C lại thấy trống và trả về `history_length = 0`.
> -> Kết quả: `history_length` sẽ nhảy lộn xộn không thể đoán trước (0, 0, 2, 0, 4...) tùy theo request được route vào container nào. Khi lưu tập trung tại Redis, mọi container đều đọc chung một state, `history_length` luôn tăng đều đặn (0, 2, 4, 6, 8...).

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi ghi nhận: Health check timeout / Connection refused khi deploy lên platform do cố định cổng chạy và bind sai địa chỉ host.
> - Thông báo lỗi: Platform báo `Deploy failed: container failed health check on port $PORT` hoặc `Connection refused` khi platform gọi health probe.
> - Cách tìm ra nguyên nhân: Mở tab Runtime Logs của platform, nhận thấy nền tảng tự động cấp một cổng ngẫu nhiên thông qua biến môi trường `$PORT` (ví dụ `PORT=10000` trên Render), nhưng lệnh CMD ban đầu lại hardcode chạy trên cổng cố định `8000` hoặc bind vào `127.0.0.1` (chỉ cho phép gọi nội bộ container).
> - Cách sửa: Cập nhật lệnh CMD trong Dockerfile sử dụng shell expansion để đọc cổng linh hoạt: `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`. Tham số `0.0.0.0` cho phép nhận traffic từ bên ngoài container và `${PORT:-8000}` tự động lấy đúng cổng do cloud cấp phát.
