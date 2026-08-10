# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder ở mỗi câu bằng câu trả lời thật.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đoàn Ngọc Linh  Mã học viên: 2A202601762

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi tôi tạo Blueprint trên Render, biến `API_TOKEN` được khai `sync: false`
> trong `render.yaml`, nghĩa là Render bắt tôi phải tự gõ giá trị lúc deploy —
> nếu tôi bỏ trống hoặc gõ nhầm, `Settings()` sẽ raise `ValidationError` ngay
> lúc container khởi động, và Render báo build/deploy fail rõ ràng. Nếu
> `api_token` có giá trị mặc định kiểu `"changeme"`, container vẫn khởi động
> bình thường, `/chat` vẫn trả 200 cho bất kỳ ai gõ đúng `"changeme"` — tôi sẽ
> không biết có chuyện gì sai cho tới khi nhìn dashboard chi phí hoặc bị dò ra
> token mặc định. "Chết sớm" biến một lỗ hổng bảo mật âm thầm thành một lỗi
> build hiện ngay trên màn hình lúc tôi đang theo dõi deploy.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật khi tôi gọi `/chat` ở máy mình:
>
> ```json
> {"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T09:51:53.798756+00:00", "client_id": "sv01", "prompt_tokens": 3, "completion_tokens": 41, "usd_cost": 2.505e-05}
> ```
>
> Hai việc tôi làm được mà `print` không làm được:
> 1. **Lọc/đếm theo trường**: vì log là JSON có cấu trúc, tôi có thể chạy
>    `jq 'select(.client_id=="sv01") | .usd_cost' log.txt` để cộng dồn chi
>    phí của riêng `sv01`, hoặc đếm số dòng có `severity: "ERROR"` trong 5
>    phút gần nhất. `print("đã trả lời xong")` là văn bản tự do — không có
>    trường nào để lọc, muốn biết chi phí của một client phải đọc bằng mắt.
> 2. **Cảnh báo tự động**: nền tảng log (Render, Google Cloud Logging...)
>    đọc đúng khóa `severity` viết hoa để tô màu và cho phép đặt alert kiểu
>    "báo tôi khi có >10 dòng `ERROR` trong 1 phút". `print` không có khóa
>    nào để nền tảng nhận diện mức độ nghiêm trọng, nên không thể đặt alert
>    dựa trên nó.

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

> Tôi build lại đúng bản Dockerfile gốc (1 stage, `FROM python:3.11`, không
> multi-stage) từ commit đầu tiên và đo bằng `docker images`: 1.73GB. Bản
> multi-stage tôi viết ở CP2 chỉ 270MB — chênh khoảng 1.46GB. Phần chênh lệch
> đó chủ yếu là:
> 1. **Base image đầy đủ vs slim**: `python:3.11` (không phải `-slim`) cài kèm
>    rất nhiều gói hệ thống (biên dịch C, thư viện đồ họa, tài liệu...) mà
>    một web service như thế này không bao giờ dùng tới.
> 2. **Toolchain build còn sót lại trong image cuối**: bản 1 stage `pip
>    install` ngay trong stage duy nhất, nên mọi thứ `pip` cần để biên dịch
>    (nếu có package cần compile) và cache của `pip` đều nằm luôn trong image
>    xuất ra. Bản multi-stage cài dependency ở stage `builder` riêng, rồi chỉ
>    `COPY --from=builder /install /usr/local` — chỉ mang phần cài đặt xong
>    sang, vứt bỏ toàn bộ builder (base image, cache pip, mọi thứ trung gian).

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Tôi sửa `SERVICE_VERSION = "1.0.0"` thêm một comment rồi build lại, xem log
> build thật:
>
> ```
> [builder 2/4] WORKDIR /app                         CACHED
> [builder 3/4] COPY requirements.txt .               CACHED
> [builder 4/4] RUN pip install ...                   CACHED
> [runtime 3/6] COPY --from=builder /install /usr/local  CACHED
> [runtime 4/6] RUN useradd ...                       CACHED
> [runtime 5/6] COPY app ./app                        chạy lại (không cache)
> [runtime 6/6] COPY utils ./utils                    chạy lại (do layer trước đổi)
> ```
>
> Toàn bộ layer cài dependency (`pip install`) và cả stage `builder` dùng
> lại cache — chỉ 2 layer cuối (copy code) phải chạy lại, nên build gần như
> tức thì. Lý do: tôi `COPY requirements.txt` và `pip install` **trước** khi
> `COPY app ./app`, nên Docker chỉ so sánh hash của `requirements.txt` (không
> đổi) để quyết định cache, chưa đụng tới code.
>
> Nếu đặt `COPY . .` lên trước `RUN pip install` (tức copy toàn bộ source
> code, gồm cả `app/`, trước khi cài thư viện) thì mọi lần sửa dù chỉ một
> dòng trong `app/main.py` cũng làm layer `COPY . .` đổi hash → Docker huỷ
> cache từ layer đó **và mọi layer sau nó**, kể cả `pip install`. Kết quả là
> mỗi lần sửa code, kể cả sửa 1 ký tự, Docker phải tải và cài lại toàn bộ
> `requirements.txt` từ đầu — build chậm hẳn dù chẳng có thư viện nào đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) một lỗ hổng trong code Python — ví dụ một dependency
> nào đó (hoặc chính `generate_reply`) bị khai thác để chạy lệnh hệ thống tuỳ
> ý (remote code execution); (2) tiến trình Python trong container đang chạy
> bằng root, nên lệnh mà kẻ tấn công chèn vào cũng chạy với quyền root **bên
> trong container**; (3) nếu container engine có lỗ hổng escape (container
> breakout — không hiếm, nhất là khi cấu hình mount volume hoặc capabilities
> không chuẩn), root trong container có thể leo thành root trên chính máy
> host — lúc đó kẻ tấn công kiểm soát toàn bộ server, không chỉ riêng app.
>
> `USER appuser` (trong `Dockerfile` của tôi, uid 10001) cắt đứt chuỗi này ở
> bước (2): dù lỗ hổng ở bước (1) vẫn xảy ra, lệnh kẻ tấn công chèn vào chỉ
> chạy được với quyền của `appuser` — một user thường, không có quyền ghi
> vào hệ thống file ngoài `/home/appuser`, không cài được package, không có
> quyền root ngay cả khi container escape thành công. Thiệt hại bị giới hạn
> lại ở bước (1)-(2) thay vì lan tới toàn bộ host.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> `WWW-Authenticate` là header chuẩn của HTTP (RFC 7235/6750): mọi response
> `401` phải nói cho client biết **cách** xác thực, không chỉ nói "bạn chưa
> xác thực". Không có header này, client (đặc biệt thư viện tự động, không
> phải người đọc lỗi) không biết phải gửi Basic auth, API key hay Bearer
> token — `WWW-Authenticate: Bearer` trả lời thẳng câu đó.
>
> Trả cùng một thông báo cho cả ba trường hợp (thiếu header / sai scheme /
> sai token) là để không "tặng" thông tin cho kẻ đang dò token. Nếu tôi trả
> "sai scheme, phải dùng Bearer" riêng và "token không đúng" riêng, kẻ tấn
> công dò mù sẽ biết ngay khi nào họ đã đúng *hình thức* header và chỉ còn
> phải brute-force đúng *giá trị* token — thu hẹp không gian tấn công. Trả
> cùng một câu "invalid or missing bearer token" cho mọi trường hợp buộc kẻ
> tấn công phải đoán mù cả hai thứ cùng lúc, không có tín hiệu nào phản hồi
> lại việc họ đang đi đúng hướng hay sai hướng.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Với `min(capacity, ...)` như code tôi viết: xô chưa bao giờ chứa quá 10
> token dù im lặng bao lâu, nên client này gửi được đúng **10 request** liên
> tiếp trước khi request thứ 11 bị 429.
>
> Nếu bỏ `min(capacity, ...)`: xô bắt đầu đầy ở mức `capacity=10`, rồi trong
> 10 phút im lặng nó cứ nạp thêm `10 token/phút × 10 phút = 100 token` mà
> không bị chặn trên, tổng thành `10 + 100 = 110` token tích luỹ. Client sẽ
> gửi được **110 request** liên tiếp trước khi bị chặn — gấp 11 lần mức lẽ
> ra được phép, vì thuật toán đang thưởng cho việc im lặng thay vì chỉ cho
> phép "bùng" trong giới hạn sức chứa của xô.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Với hạn mức **$30/tháng**: nếu sự cố xảy ra lúc 2h sáng ngày đầu tháng và
> không ai để ý, client có thể tiêu hết $30 chỉ trong vài giờ (rate limit
> không chặn được vì đây là vấn đề *số tiền mỗi request*, không phải *số
> request*), rồi mất luôn phần còn lại của tháng vì hạn mức đã cạn — dịch vụ
> bị khoá tới tận đầu tháng sau. Thiệt hại tối đa: **$30**, phục hồi sau tối
> đa gần 30 ngày.
>
> Với hạn mức **$1/ngày** (như tôi cài trong `cost_guard.py`, key theo
> `spend:<client>:<YYYY-MM-DD>`): cùng sự cố lúc 2h sáng chỉ có thể tiêu tối
> đa $1 trước khi `check()` chặn bằng 402. Đến 0h UTC ngày hôm sau, key mới
> với ngày mới tự bắt đầu từ 0 — service tự hồi phục **mà không cần ai can
> thiệp gì**, kể cả khi không ai thức dậy lúc 2h sáng để xử lý. Thiệt hại tối
> đa: **$1**, tức 1/30 so với hạn mức tháng, và tự phục hồi trong vài giờ
> thay vì gần một tháng.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Theo đúng thứ tự sự kiện: (1) Redis mất kết nối; (2) endpoint gộp (đóng vai
> cả liveness lẫn readiness) gọi `store.ping()`, nhận `False`, trả 503 cho cả
> 3 container cùng lúc vì cả 3 đều mất kết nối tới cùng một Redis; (3)
> orchestrator đọc endpoint này như **liveness probe** — thấy 503 nghĩa là
> "process này hỏng, cần restart", nên nó **restart cả 3 container cùng
> lúc**, không phải chỉ rút chúng khỏi load balancer; (4) trong lúc cả 3
> container đang khởi động lại (kéo image, chạy lại app), không còn container
> nào phục vụ request — service downtime hoàn toàn, dù bản thân code app
> chẳng có lỗi gì; (5) khi Redis sống lại sau 30 giây, các container mới khởi
> động xong mới bắt đầu nhận traffic trở lại — nhưng khoảng downtime đã xảy
> ra rồi, có thể dài hơn 30 giây vì còn cộng thêm thời gian khởi động lại.
>
> Nếu tách riêng như tôi đã làm (`/healthz` không đụng Redis, `/readyz` có
> kiểm tra), Redis chết 30 giây chỉ khiến `/readyz` trả 503 → load balancer
> rút container khỏi vòng xoay (không restart) → khi Redis sống lại,
> `/readyz` tự trả 200 trở lại → LB tự đẩy traffic vào lại, không cần
> restart bất cứ container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Thành thật thì lần deploy Blueprint lên Render của tôi chạy xuôn sẻ ngay
> từ lần đầu — không có build fail hay health check timeout. Tôi cho rằng lý
> do là những vấn đề kể trên (app không đọc `$PORT`, healthcheck sai đường
> dẫn, `REDIS_URL` trỏ sai) đã bị CP1/CP2/CP4 bắt từ trước khi lên tới bước
> deploy: `Dockerfile` của tôi đã đọc cổng qua `${PORT:-8000}`, `render.yaml`
> tự nối `REDIS_URL` vào Redis add-on qua `fromService`, và `/healthz` đã
> được test kỹ ở CP1 nên không phụ thuộc gì ngoài chính process.
>
> Vấn đề thật tôi gặp lại nằm ở bước **kiểm tra sau khi deploy**, không phải
> lúc deploy: khi gọi `/chat` bằng `curl` trong PowerShell, JSON bị lỗi
> `"json_invalid": "Expecting property name enclosed in double quotes"` liên
> tục. Nguyên nhân là PowerShell không xử lý `\"` như ký tự escape giống
> bash/cmd, nên chuỗi JSON `-d "{\"message\":\"Hello\"}"` bị cắt cụt trước
> khi tới `curl.exe`. Tôi tìm ra bằng cách so sánh với kết quả `Invoke-
> WebRequest` (chạy đúng) và nhận ra khác biệt duy nhất là cách quote. Sửa
> bằng cách đổi sang nháy đơn bao ngoài: `curl.exe -d '{"message":"Hello"}'`
> — PowerShell truyền chuỗi đó nguyên vẹn, không tự ý diễn giải `\"`.
