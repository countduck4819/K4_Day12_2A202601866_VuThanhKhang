# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Vũ Thành Khang  Mã học viên: 2A202601866

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn,so với việc để mặc định `"changeme"`.

> Ví dụ: khi deploy production mà quên set API_TOKEN, nếu có mặc định "changeme" thì app vẫn chạy và người khác có thể dùng token đó để gọi API, gây phát sinh chi phí. Nếu không có giá trị mặc định, app sẽ dừng ngay khi khởi động, giúp phát hiện lỗi cấu hình trước khi service được public.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log JSON thu được:

{"event":"chat_completed","severity":"INFO","ts":"2026-08-10T10:35:20+00:00","client_id":"sv01","prompt_tokens":120,"completion_tokens":45,"usd_cost":0.0002}

Từ dòng log này, tôi có thể:

Lọc và thống kê theo từng trường, ví dụ đếm số request của một client hoặc tính tổng chi phí usd_cost.
Thiết lập cảnh báo tự động khi chi phí tăng cao, số request bất thường hoặc xuất hiện lỗi.

Trong khi đó, print("đã trả lời xong") chỉ là một chuỗi văn bản, không có cấu trúc để hệ thống log tự động lọc, thống kê hoặc cảnh báo

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ... MB |
| Multi-stage | ... MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản	Dung lượng
1 stage (bản đầu)	... MB
Multi-stage	... MB

Phần dung lượng chênh lệch chủ yếu đến từ việc bản 1-stage giữ lại toàn bộ môi trường build trong image cuối, gồm compiler, build-essential, cache và các package trung gian dùng để cài dependency. Ở bản multi-stage, các thành phần đó chỉ tồn tại trong stage builder rồi bị loại bỏ; image runtime chỉ giữ Python slim, dependency đã cài và source code cần để chạy service. Vì vậy image multi-stage nhỏ hơn và cũng giảm bề mặt tấn công.

---
### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Khi chỉ sửa một ký tự trong `app/main.py`, các layer liên quan đến base image, `COPY requirements.txt` và `RUN pip install` vẫn được dùng lại từ cache vì `requirements.txt` không thay đổi. Chỉ layer `COPY app ./app` và các layer phía sau nó phải chạy lại.

Nếu đặt `COPY . .` trước `RUN pip install`, chỉ cần sửa một ký tự trong source code thì layer `COPY . .` thay đổi, làm mất cache của tất cả layer phía sau. Khi đó Docker phải chạy lại `pip install`, dù dependency không hề thay đổi. Điều này làm thời gian build lâu hơn đáng kể.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Nếu code Python có một lỗ hổng cho phép kẻ tấn công thực thi lệnh trong container, họ sẽ có quyền của user đang chạy ứng dụng. Nếu container chạy bằng root, kẻ tấn công có quyền root bên trong container. Nếu kết hợp thêm lỗi cấu hình hoặc lỗ hổng container runtime, họ có thể truy cập tài nguyên của host với quyền cao.

Lệnh `USER appuser` cắt chuỗi này bằng cách chạy ứng dụng với user thường. Vì vậy nếu app bị chiếm quyền điều khiển, kẻ tấn công trước tiên chỉ có quyền hạn chế của `appuser`, giảm đáng kể khả năng gây ảnh hưởng tới container và host.

---

### Câu 6 — Bearer token (CP3)

Response `401` cần có header `WWW-Authenticate: Bearer` để client biết API yêu cầu xác thực bằng Bearer token và có thể xử lý đúng theo chuẩn HTTP.

Ta dùng cùng một thông báo lỗi cho trường hợp thiếu header, sai scheme và sai token để không tiết lộ thêm thông tin cho kẻ tấn công. Nếu trả lời cụ thể lỗi nằm ở đâu, người đang dò token có thể dùng thông tin đó để thu hẹp phạm vi thử và tấn công dễ hơn.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10` và `refill_per_minute=10`, sau khi client im lặng 10 phút, xô vẫn chỉ có tối đa 10 token. Vì vậy client gửi được 10 request liên tiếp trước khi request tiếp theo bị `429`.

Nếu bỏ `min(capacity, ...)`, sau 10 phút client có thể tích thêm 100 token. Nếu trước đó xô đang đầy 10 token thì tổng có thể thành khoảng 110 token, nên client có thể gửi khoảng 110 request liên tiếp. Đây là lý do phải chặn số token tối đa bằng `capacity`.

---

### Câu 8 — Ngân sách theo ngày (CP3)

Với hạn mức `$30/tháng`, nếu sự cố bắt đầu lúc 2 giờ sáng và client gọi liên tục, thiệt hại có thể lên tới gần `$30` trước khi bị chặn. Sau khi hết ngân sách, service của client đó có thể tiếp tục bị chặn cho tới khi sang tháng mới hoặc có người can thiệp.

Với hạn mức `$1/ngày`, thiệt hại tối đa trong ngày chỉ khoảng `$1`. Khi sang ngày UTC mới, key chi tiêu đổi sang ngày mới nên ngân sách tự được reset và service tự hoạt động trở lại mà không cần can thiệp thủ công.

---

### Câu 9 — `/healthz` khác `/readyz` (CP4)

Nếu gộp hai endpoint và để health check kiểm tra Redis, khi Redis mất kết nối 30 giây sẽ xảy ra:

1. Cả 3 container gọi health check và đều thấy Redis lỗi.
2. Cả 3 cùng trả `503`.
3. Orchestrator hiểu rằng cả 3 process không khỏe mạnh.
4. Orchestrator có thể restart cả 3 container.
5. Trong thời gian restart, không còn instance nào phục vụ request.
6. Redis có thể đã hoạt động trở lại nhưng các container vẫn đang khởi động lại.

Như vậy một sự cố Redis ngắn đã bị biến thành sự cố toàn service. Nếu tách `/healthz` và `/readyz`, Redis lỗi chỉ làm `/readyz` trả `503`, load balancer ngừng gửi traffic vào instance nhưng không restart process.

---

### Câu 10 — Deploy thật (CP5)

Một lỗi tôi gặp khi deploy lên Render là `/readyz` trả:

```json
{"status":"not ready","redis":false}
```

Trong khi `/healthz` vẫn trả `200`. Tôi kiểm tra kết quả hai endpoint và nhận ra process vẫn chạy bình thường nhưng dependency Redis chưa kết nối được. Sau đó tôi kiểm tra biến môi trường và thấy `REDIS_URL` đang trỏ tới `redis://localhost:6379/0`.

Nguyên nhân là trên cloud, `localhost` chỉ chính container chạy ứng dụng, không phải Redis. Tôi sửa `REDIS_URL` sang địa chỉ Redis cloud mà service có thể truy cập. Sau khi deploy lại, `/readyz` trả:

```json
{"status":"ready","redis":true}
```

cho thấy ứng dụng đã kết nối Redis thành công.
