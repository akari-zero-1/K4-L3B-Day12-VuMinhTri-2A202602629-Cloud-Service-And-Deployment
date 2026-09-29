# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng mẫu câu trả lời bên dưới bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Vũ Minh Trí  Mã học viên: 2A202602629

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu để mặc định "changeme", khi deploy lên Cloud mà quên cấu hình biến môi trường, app vẫn âm thầm chạy. Kẻ xấu có thể dùng khóa mặc định gọi API /ask miễn phí và đốt sạch tiền hóa đơn LLM. "Chết sớm" (Fail Fast) giúp app dừng ngay lúc deploy, cảnh báo người vận hành bổ sung secret trước khi mở public ra ngoài Internet.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thu được:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:13:04.123456+00:00", "user_id": "sv01", "tokens_in": 48, "tokens_out": 52, "cost_usd": 3.84e-05}`
> Hai việc làm được:
> 1. Truy vấn, lọc và phân tích dữ liệu tự động bằng máy (ví dụ: thống kê user nào tiêu tốn nhiều chi phí nhất hoặc đếm số lượng token tiêu thụ theo thời gian).
> 2. Kết nối với hệ thống giám sát tập trung (Datadog, CloudWatch, Grafana Loki) để tự động kích hoạt cảnh báo (alerting) theo ngưỡng thời gian thực.

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
| 1 stage (bản đầu) | 1020 MB |
| Multi-stage | 272 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (~750 MB) là toàn bộ bộ công cụ biên dịch (compilers, build-essential, gcc), thư viện header C, pip cache, và các dependencies của hệ điều hành chỉ phục vụ quá trình build gói mà không cần thiết khi chạy ứng dụng trong môi trường runtime.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> - Với Dockerfile hiện tại: Các layer cài đặt thư viện (`COPY requirements.txt` và `RUN pip install`) được tận dụng lại từ cache (CACHED); chỉ layer `COPY . .` và các bước sau đó phải chạy lại.
> - Nếu đặt `COPY . .` trước `RUN pip install`: Mỗi lần sửa mã nguồn, cache của layer copy bị hủy, buộc Docker phải cài lại toàn bộ thư viện từ đầu, làm thời gian build kéo dài thêm vài phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: App bị khai thác lỗ hổng thực thi mã (RCE) -> Kẻ tấn công chiếm shell trong container với quyền root (UID 0) -> Lợi dụng lỗ hổng container breakout hoặc socket Docker để can thiệp trực tiếp vào tài nguyên máy host dưới quyền root.
> Lệnh `USER appuser` cắt đứt chuỗi ngay từ bước chiếm quyền: App chạy với user thường không đặc quyền (UID 10001), kẻ tấn công bị cô lập hoàn toàn, không thể ghi đè file hệ thống hay leo thang đặc quyền để thoát ra máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request trong 2 giây.
> Giải thích: Người dùng gửi 10 request vào giây cuối của phút trước (10:00:59), ngay sau đó sang phút mới (10:01:00) bộ đếm reset về 0, người dùng gửi tiếp 10 request vào giây 10:01:01. Về mặt phút đồng hồ cả hai đều đúng luật (10 req/phút), nhưng thực tế server phải chịu 20 request chỉ trong vòng 2 giây.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Khác biệt: Rate limit kiểm soát tần suất/số lượng request trong khoảng thời gian ngắn để chống nghẽn hệ thống. Cost guard kiểm soát ngân sách tài chính và lượng token tiêu thụ theo chu kỳ dài (tháng).
> - Rate limit cho qua nhưng Cost guard chặn: User chỉ gửi 1 request/phút (rất ít), nhưng câu hỏi kèm tài liệu 100k token làm chi phí vượt hạn mức tháng ($10) -> Cost guard chặn bằng mã lỗi 402.
> - Cost guard cho qua nhưng Rate limit chặn: User còn đầy ngân sách tháng, nhưng gửi dồn dập 20 câu hỏi ngắn trong 5 giây -> Rate limit chặn bằng mã lỗi 429 vì spam quá nhanh.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện:
> 1. Redis mất kết nối trong 30 giây.
> 2. Liveness probe gọi vào endpoint kiểm tra Redis thất bại.
> 3. Orchestrator (Docker/Kubernetes) kết luận cả 3 container agent đều bị hỏng và liên tục kill/restart cả 3 container.
> 4. Cụm service rơi vào vòng lặp crash (crash loop), ngắt quãng toàn bộ các request đang xử lý dở thay vì chỉ tạm dừng điều phối traffic như cơ chế tách biệt readiness.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Nếu lưu trong dict Python (trong bộ nhớ RAM của từng container), các request tiếp theo được load balancer phân phối luân phiên vào 3 container khác nhau. Giá trị `history_length` sẽ bị nhảy loạn xạ (lúc tăng, lúc trở về 0), khiến agent bị "mất trí nhớ" và không thể theo dõi liên tục ngữ cảnh hội thoại của user.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> - Thông báo lỗi: Container bị crash lúc startup với lỗi `NotImplementedError: TODO (CP4): cài đặt install` tại `lifecycle.install()`.
> - Cách tìm nguyên nhân: Kiểm tra runtime log trên dashboard platform (`docker compose logs agent`), phát hiện hook `lifespan` tự động gọi `lifecycle.install()` khi ứng dụng khởi động.
> - Cách sửa: Cài đặt đầy đủ `install()` và `request_shutdown()` trong `app/lifecycle.py` để đăng ký signal handler cho SIGTERM/SIGINT, sau đó rebuild và deploy lại container.
