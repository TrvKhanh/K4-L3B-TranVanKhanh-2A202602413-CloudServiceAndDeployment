# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng hướng dẫn bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Văn Khánh  Mã học viên: 2A202602413

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy ứng dụng lên môi trường Production mà người quản trị quên thiết lập biến môi trường `AGENT_API_KEY`. Nếu có giá trị mặc định `"changeme"`, ứng dụng vẫn khởi động bình thường và kẻ tấn công có thể quét các endpoint công khai, dùng chìa khóa mặc định `"changeme"` để gọi API LLM liên tục làm cạn kiệt hạn mức tài khoản. Nhờ việc không có giá trị mặc định, ứng dụng sẽ báo lỗi `ValidationError` và dừng ngay lập tức khi khởi động, giúp phát hiện ra thiếu sót cấu hình trước khi ứng dụng tiếp nhận traffic thực tế.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T10:00:00.000000+00:00", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 25, "cost_usd": 0.0001}
```
1. **Lọc và truy vấn chính xác theo trường dữ liệu**: Các công cụ thu thập log (như Datadog, Grafana Loki, CloudWatch) có thể bóc tách tự động các trường `user_id`, `cost_usd`, `event` để tìm kiếm chính xác các sự kiện của một user cụ thể hoặc các request có chi phí vượt ngưỡng.
2. **Thống kê và cảnh báo tự động**: Có thể tính tổng `cost_usd` hoặc tổng `tokens_out` theo thời gian thực để lập biểu đồ chi phí và đặt ngưỡng cảnh báo tự động mà không phải viết regex phức tạp để parse văn bản không có cấu trúc.

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
| 1 stage (bản đầu) | 1.02 GB |
| Multi-stage | 175 MB |

Giải thích: Phần dung lượng chênh lệch (~845 MB) chứa base image `python:3.11` đầy đủ bao gồm trình biên dịch C/C++ (gcc, g++), bộ công cụ build, header files, package manager hệ thống và các file cache wheel tạm không cần thiết cho quá trình chạy ứng dụng ở môi trường production.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Các layer được dùng lại từ cache: `FROM`, `WORKDIR`, `COPY requirements.txt .`, `RUN pip install ...`.
- Các layer phải chạy lại: `COPY app app`, `USER`, `HEALTHCHECK`, `CMD`.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi khi sửa bất kỳ file source code nào, cache tại layer `COPY . .` sẽ bị invalid, dẫn đến lệnh `RUN pip install` bị ép phải thực thi lại toàn bộ, khiến quá trình build kéo dài lãng phí thời gian và băng thông.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện: Lỗ hổng RCE/Command Injection trong code Python -> Kẻ tấn công thực thi lệnh hệ thống bên trong container -> Do container chạy bằng root, process có full quyền root trong container -> Kẻ tấn công khai thác tiếp lỗ hổng container escape (hoặc tương tác với Docker socket được mount) để leo thang chiếm quyền root của máy host.
- Lệnh `USER appuser` cắt đứt chuỗi tấn công ngay sau bước 1: Process của app chạy bằng user thường không có đặc quyền, kẻ tấn công không thể ghi file hệ thống, không thể cài package hay tương tác với các tài nguyên nhạy cảm.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
Giải thích: Người dùng gửi 10 request ở giây `10:00:59` (thuộc phút thứ nhất) và gửi tiếp 10 request ở giây `10:01:01` (thuộc phút thứ hai). Do bộ đếm reset về 0 ở giây `10:01:00`, cả 20 request đều thành công dù khoảng thời gian giữa chúng chỉ là 2 giây.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Khác biệt: Rate limit giới hạn **số lượng request trong thời gian ngắn** để bảo vệ hệ thống khỏi quá tải. Cost guard giới hạn **tổng chi phí tài chính trong tháng** để bảo vệ ví tiền.
- Rate limit cho qua nhưng Cost guard chặn: User gửi 1 request/phút (không vượt quá 10 req/phút) nhưng câu hỏi chứa prompt/context rất dài tiêu tốn $11.0/tháng (vượt mức budget $10.0/tháng) -> Cost guard chặn với HTTP 402.
- Cost guard cho qua nhưng Rate limit chặn: User mới tạo tài khoản chưa tiêu đồng nào ($0.0), nhưng gửi liền 15 request trong vòng 5 giây -> Rate limit chặn với HTTP 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Giây 0: Kết nối Redis bị ngắt.
2. Giây 5: Healthcheck probe từ Orchestrator gọi vào `/health`, nhận phản hồi thất bại do Redis sập.
3. Giây 10: Orchestrator coi cả 3 container đã hỏng và tiến hành ngắt (kill) rồi khởi động lại (restart) toàn bộ 3 container.
4. Giây 15-30: Các container mới khởi động vẫn không kết nối được Redis, lại tiếp tục thất bại healthcheck và rơi vào vòng lặp crash-restart liên tục (CrashLoopBackOff).
5. Kết quả: Cụm service bị sập hoàn toàn và liên tục thay vì chỉ tạm ngừng nhận traffic chờ Redis hồi phục.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Giá trị `history_length` sẽ nhảy ngẫu nhiên và không nhất quán giữa các request (ví dụ: 0, 0, 2, 0, 4, 2...) tùy thuộc vào request đó được Load Balancer điều hướng tới instance container nào trong 3 instance, vì RAM của mỗi instance độc lập với nhau.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Lỗi gặp phải: Container bị crash và thất bại khi healthcheck khi deploy lên PaaS do app không nhận diện được cổng môi trường `$PORT`.
- Nguyên nhân: Platform tự động cấp phát một cổng ngẫu nhiên thông qua biến môi trường `$PORT` (ví dụ `PORT=10000`), nhưng ứng dụng uvicorn lại mở cứng ở cổng `8000`.
- Cách khắc phục: Cập nhật file `Dockerfile` và lệnh khởi chạy `app/main.py` để đọc biến môi trường `PORT` động bằng `os.environ.get("PORT", 8000)` và `uvicorn --port ${PORT:-8000}`.
