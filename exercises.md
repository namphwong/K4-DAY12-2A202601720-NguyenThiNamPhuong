# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: Trả lời trực tiếp ngay bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Thị Nam Phương  Mã học viên: 2A202601720

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy lên môi trường Cloud/Production, nếu bạn quên cài đặt biến `API_TOKEN` trên Dashboard. Nếu để giá trị mặc định `"changeme"`, app vẫn khởi động thành công và chấp nhận requests. Kẻ tấn công có thể mò ra token mặc định `"changeme"` để truy cập trái phép hoặc lạm dụng làm tăng hóa đơn tiền Cloud vọt lên. Việc "chết sớm" (Fail fast) khiến ứng dụng crash ngay lập tức ở bước khởi động, buộc developer phải phát hiện và sửa ngay trước khi hệ thống chấp nhận bất kỳ luồng traffic nguy hiểm nào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T13:54:30+00:00", "client_id": "sv-test", "usd_cost": 0.0001}`

Hai việc làm được với log JSON mà `print()` không làm được:
1. **Lọc và tự động phát cảnh báo trên Log Aggregator (GCP Logging/Datadog/ELK):** Các công cụ quản lý log tự động parse cấu trúc JSON để lọc lỗi theo `severity`, đo lường tổng `usd_cost` hoặc nhóm theo `client_id`. Lệnh `print()` chỉ tạo chuỗi không cấu trúc, dễ bị vỡ dòng và không parse tự động được.
2. **Truy vấn phân tích dữ liệu (Metrics & Analytics):** Có thể chạy truy vấn SQL/Elasticsearch để trả lời các câu hỏi như: *"Client nào tiêu nhiều tiền nhất trong ngày?"* hoặc *"Tỷ lệ lỗi trong 5 phút qua là bao nhiêu?"*.

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
| 1 stage (bản đầu) | 1.8 GB |
| Multi-stage | 280 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~1.5 GB) bao gồm hệ điều hành Linux đầy đủ cùng các công cụ biên dịch (gcc, g++, make), thư viện C/C++ phát triển (build-essential, python-dev) và file tạm cache của `pip` sinh ra trong quá trình cài đặt thư viện. Multi-stage build đã bỏ qua stage `builder` nặng nề và chỉ copy kết quả binary/thư viện đã cài sang stage runtime `python:3.11-slim` gọn nhẹ.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại: Layer `COPY requirements.txt .` và `RUN pip install ...` được dùng lại từ Cache (vì `requirements.txt` không đổi). Chỉ có layer `COPY . .` và các layer sau đó là phải chạy lại, giúp thời gian build chỉ mất vài giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Khi sửa code trong `main.py`, layer `COPY . .` bị mất cache. Docker sẽ vô hiệu hóa cache cho tất cả các layer phía sau, buộc lệnh `RUN pip install` phải tải và cài đặt lại toàn bộ thư viện từ đầu, làm thời gian build kéo dài từ vài giây lên nhiều phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Code Python gặp lỗ hổng RCE (Remote Code Execution) hoặc Command Injection.
2. Kẻ tấn công khai thác lỗ hổng để thực thi lệnh shell bên trong container.
3. Vì container chạy với quyền root (UID 0) mặc định, kẻ tấn công chiếm quyền root bên trong container.
4. Kẻ tấn công khai thác lỗ hổng container escape hoặc truy cập volume mount để chiếm quyền root trên máy host.

Lệnh `USER appuser` cắt đứt chuỗi ở **bước 3**: Container được hạ quyền xuống user thường (UID 10001). Ngay cả khi khai thác thành công code Python, kẻ tấn công chỉ có quyền hạn chế trong container, không thể chỉnh sửa file hệ thống hoặc thực hiện các cuộc tấn công leo thang quyền hạn lên máy host.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

1. Header `WWW-Authenticate: Bearer` tuân theo chuẩn HTTP (RFC 6750) để thông báo cho client (như browser hoặc HTTP library) biết cơ chế xác thực được yêu cầu là Bearer Token.
2. Trả cùng một thông báo lỗi chung giúp bảo mật hệ thống theo nguyên tắc phòng thủ chuyên sâu (Defense in Depth). Nếu báo lỗi quá chi tiết (ví dụ: "Token không tồn tại" vs "Token hết hạn" vs "Sai định dạng"), kẻ tấn công có thể lợi dụng để thăm dò (enumeration attack) cấu trúc header và trạng thái các token hợp lệ.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

- Số request gửi được: **10 request** (đến request thứ 11 sẽ bị 429).
- Nếu bỏ `min(capacity, ...)`: Con số sẽ thành **100 request** (vì 10 phút × 10 token/phút = 100 token tích lũy). Khi đó, client im lặng lâu có thể xả dồn dập 100 request liên tiếp, làm mất đi khả năng giới hạn lưu lượng tức thời (burst limit) của Token Bucket và có thể gây nghẽn service.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

- **Hạn mức $30/tháng:** Thiệt hại tối đa là **$30** (client tiêu hết toàn bộ hạn mức tháng chỉ trong vài giờ từ 2h sáng). Client bị khóa trong suốt khoảng thời gian còn lại của tháng và service tự hồi phục vào ngày đầu tiên của tháng sau.
- **Hạn mức $1/ngày:** Thiệt hại tối đa chỉ là **$1**. Sau khi tiêu hết $1, client bị chặn cho đến hết ngày. Service tự động hồi phục và cấp lại hạn mức vào **00:00 UTC ngày hôm sau**.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Redis mất kết nối trong 30 giây.
2. Endpoint gộp trả về 500/503 do không nối được Redis.
3. Orchestrator (Docker/Kubernetes/Cloud Run) lầm tưởng cả 3 container ứng dụng bị hỏng (unhealthy) và ra lệnh **restart toàn bộ 3 container**.
4. Trong lúc 3 container bị restart cùng lúc, ứng dụng không còn container nào phục vụ traffic → Hệ thống bị downtime hoàn toàn.
5. Việc restart container hàng loạt gây ra hiệu ứng tuyết lở (thay vì chỉ tạm thời ngưng nhận request ở `/readyz` và tự động phục hồi êm đẹp ngay khi Redis kết nối lại).

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi:** Khi deploy lên Railway, lệnh `railway up` báo lỗi `/bin/sh: 1: exec: docker-entrypoint.sh: not found` và `/readyz` trả về `{"status":"not ready","redis":false}`.
- **Nguyên nhân:** Khi chạy lệnh `railway add --database redis` ban đầu, Railway tự động link thư mục hiện tại vào service `Redis` thay vì một Web service riêng, khiến `railway up` ghi đè ứng dụng Python lên Redis container image.
- **Cách sửa:** Sử dụng lệnh `railway add --service web` để tạo riêng Web service, thiết lập lại biến `REDIS_URL='${{Redis.REDIS_URL}}'`, chạy `railway redeploy --service Redis` để phục hồi Redis và `railway up` để deploy Web service thành công.
