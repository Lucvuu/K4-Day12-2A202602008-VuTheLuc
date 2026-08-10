# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Vũ Thế Lực  Mã học viên: 2A202602008

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ khi deploy lên production nhưng tôi quên cấu hình biến môi trường
> API_TOKEN. Nếu `api_token` có mặc định là `"changeme"` thì app vẫn khởi
> động bình thường và có thể bị truy cập bằng token mặc định mà tôi không
> nhận ra. Nếu không có giá trị mặc định thì app sẽ lỗi ngay lúc startup,
> giúp tôi phát hiện cấu hình thiếu trước khi service nhận request. Như vậy
> fail fast biến một lỗi bảo mật âm thầm thành một lỗi triển khai dễ phát
> hiện.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log JSON tôi thu được:
>
> ```json
> {"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T09:47:15.921326+00:00", "client_id": "exercise-demo", "prompt_tokens": 7, "completion_tokens": 39, "usd_cost": 2.445e-05}
> ```
>
> Với log JSON này, tôi có thể lọc và thống kê theo từng trường, ví dụ tìm
> toàn bộ request của `client_id="exercise-demo"` hoặc tính tổng `usd_cost`
> của các request. Ngoài ra, tôi có thể đưa log vào các hệ thống giám sát
> như ELK, Loki hoặc Grafana để tạo dashboard và cảnh báo tự động.
> `print("đã trả lời xong")` chỉ cho biết một hành động đã xảy ra, nhưng
> không có timestamp, client, token usage hay chi phí để máy phân tích.

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
| 1 stage (bản đầu) | 1730 MB |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Sau khi chuyển sang multi-stage build, image giảm từ khoảng 1.73 GB xuống
> 270 MB, tức giảm khoảng 1.46 GB. Phần dung lượng chênh lệch chủ yếu là
> compiler, build tools, các dependency chỉ cần trong lúc build, pip cache
> và các file trung gian. Ở multi-stage build, những thành phần này nằm
> trong stage `builder`; sau khi build xong chỉ những thành phần cần để
> chạy ứng dụng được đưa sang runtime stage, còn `builder` bị loại khỏi
> image cuối. Vì vậy image nhỏ hơn và cũng giảm bề mặt tấn công.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Sau khi tôi chỉ sửa một dòng trong `app/main.py`, các layer `WORKDIR`,
> `COPY requirements.txt`, `RUN pip install`, `COPY --from=builder` và
> `RUN useradd` vẫn được lấy từ cache vì dữ liệu đầu vào của chúng không
> thay đổi.
>
> Các layer `COPY app ./app` và `COPY utils ./utils` phải chạy lại do
> source code thuộc phần được copy vào image đã thay đổi.
>
> Nếu tôi đặt `COPY . .` trước `RUN pip install`, chỉ cần sửa một file như
> `app/main.py` thì layer `COPY . .` sẽ thay đổi và làm mất cache của các
> layer phía sau. Khi đó `RUN pip install` cũng phải chạy lại dù
> `requirements.txt` không thay đổi. Vì vậy việc copy `requirements.txt` và
> cài dependency trước khi copy source code giúp build lại nhanh hơn nhờ
> tận dụng Docker layer cache.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Giả sử code Python của tôi có lỗ hổng cho phép kẻ tấn công thực thi lệnh
> từ xa. Nếu container đang chạy bằng root, khi khai thác thành công thì
> attacker cũng có quyền root bên trong container. Từ đó họ có thể đọc
> hoặc sửa các file nhạy cảm, cài thêm công cụ và nếu container có cấu
> hình nguy hiểm hoặc tồn tại lỗ hổng container escape thì có thể tiếp tục
> tấn công sang máy host với quyền cao.
>
> Lệnh `USER` giúp process của ứng dụng chạy bằng một user thường thay vì
> root. Như vậy khi code bị khai thác, attacker ban đầu chỉ nhận được
> quyền hạn chế của user đó. `USER` không loại bỏ hoàn toàn khả năng
> container escape nhưng làm giảm quyền và mức thiệt hại nếu ứng dụng bị
> chiếm quyền.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> Khi trả về 401, response cần có header `WWW-Authenticate: Bearer` để cho
> client biết API này sử dụng cơ chế xác thực Bearer token và request cần
> cung cấp token theo đúng cách.
>
> Tôi trả cùng một thông báo lỗi cho cả ba trường hợp thiếu header, dùng
> sai scheme hoặc token không đúng vì không muốn tiết lộ chi tiết cơ chế
> xác thực cho attacker. Nếu API nói rõ "token sai" hoặc "scheme đúng
> nhưng token không hợp lệ" thì attacker có thêm thông tin để thử từng
> bước. Với người dùng hợp lệ, chỉ cần biết authentication thất bại là đủ
> để kiểm tra lại toàn bộ header Authorization.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Với `capacity=10` và `refill_per_minute=10`, client im lặng 10 phút thì
> khi có `min(capacity, ...)`, số token vẫn bị giới hạn tối đa ở 10. Vì
> vậy client gửi được 10 request liên tiếp, sau đó request tiếp theo bị
> 429.
>
> Nếu bỏ `min(capacity, ...)`, sau 600 giây bucket có thể tích lũy:
>
> `600 × 10 / 60 = 100` token
>
> nên client có thể gửi khoảng 100 request liên tiếp trước khi bị 429.
> Nguyên nhân là không còn giới hạn capacity, nên thời gian client không
> gửi request lại biến thành lượng token tích lũy ngày càng lớn. Điều này
> phá vỡ mục đích của token bucket vì một client im lặng lâu có thể tạo ra
> burst rất lớn.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Với hạn mức $30/tháng, nếu đầu tháng client chưa sử dụng ngân sách thì
> một sự cố bắt đầu từ 2h sáng có thể tiêu gần hết $30 chỉ trong một
> khoảng thời gian ngắn. Sau khi chạm giới hạn, client sẽ bị chặn và phải
> chờ sang kỳ ngân sách tháng tiếp theo mới tự có quota trở lại.
>
> Với hạn mức $1/ngày, cùng sự cố đó chỉ có thể tiêu tối đa khoảng $1
> trong ngày đó trước khi bị chặn. Sang ngày mới, ngân sách ngày được
> reset nên service có thể hoạt động lại.
>
> Vì vậy giới hạn theo ngày làm giảm blast radius của sự cố. Thay vì một
> lỗi có thể đốt hết ngân sách cả tháng trong vài giờ, thiệt hại bị giới
> hạn ở mức nhỏ hơn cho từng ngày.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp `/healthz` và `/readyz` thành một endpoint cùng kiểm tra Redis
> thì khi Redis mất kết nối 30 giây, đầu tiên cả 3 container đều không
> truy cập được Redis và endpoint health bắt đầu trả 503.
>
> Tiếp theo, nếu endpoint đó được dùng làm readiness probe thì cả 3
> container sẽ bị loại khỏi danh sách nhận traffic. Phần này hợp lý vì lúc
> đó chúng chưa thể phục vụ request đầy đủ.
>
> Nhưng nếu cùng endpoint cũng được dùng làm liveness probe thì hệ thống
> orchestration sẽ hiểu nhầm rằng cả 3 process đã chết và bắt đầu restart
> container. Trong thực tế FastAPI vẫn đang chạy, chỉ có Redis tạm thời
> mất kết nối. Vì cả 3 container cùng restart nên hệ thống có thể rơi vào
> restart loop và làm sự cố nghiêm trọng hơn.
>
> Nếu tách riêng hai endpoint thì `/healthz` vẫn trả 200 vì process còn
> sống, còn `/readyz` trả 503 để tạm ngừng nhận traffic. Khi Redis hoạt
> động lại sau 30 giây, `/readyz` trở lại 200 và cả 3 container tiếp tục
> phục vụ mà không cần restart.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> *Câu trả lời của bạn*
