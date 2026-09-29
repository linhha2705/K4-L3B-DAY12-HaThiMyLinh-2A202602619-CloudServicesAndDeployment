# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời ngay dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Hà Thị Mỹ Linh  Mã học viên: 2A202602619

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu cấu hình giá trị mặc định là `"changeme"`, khi deploy lên nền tảng cloud (như Railway/Render) mà người phát triển quên khai báo biến `AGENT_API_KEY` trong phần Variables, dịch vụ vẫn âm thầm khởi động thành công và mở cổng nhận traffic. Khi đó, các bot quét bảo mật trên Internet có thể dò ra endpoint `/ask` và sử dụng khóa mặc định `"changeme"` để gọi API, làm cạn kiệt ngân sách token LLM và rò rỉ tài nguyên. Việc không gán mặc định giúp kích hoạt cơ chế Fail-Fast của Pydantic: ứng dụng văng `ValidationError` và crash ngay lúc khởi động, báo đỏ lập tức trên dashboard để cảnh báo người quản trị phải cung cấp secret trước khi phục vụ bất kỳ ai.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thực tế:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T04:09:53.123456+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 36, "cost_usd": 0.00002205}`

Hai việc làm được với dòng log có cấu trúc trên:
1. **Truy vấn, lọc và tổng hợp số liệu tự động**: Các hệ thống quản lý log (Datadog, Loki, CloudWatch) có thể parse các trường JSON để tính tổng số token đã dùng, đo độ trễ hoặc cộng dồn chi phí theo từng `user_id` cụ thể theo thời gian thực (ví dụ: `sum(cost_usd) by user_id`). Log dạng text tự do `print(...)` không thể phân tách trường hay tính toán được.
2. **Cấu hình cảnh báo tự động (Alerting)**: Có thể thiết lập quy tắc tự động gửi tin nhắn cảnh báo khi phát hiện request có chi phí vượt ngưỡng (ví dụ `cost_usd > 0.05`) hoặc log có `level == "error"`.

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
| 1 stage (bản đầu) | ~1020 MB |
| Multi-stage | 329 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~700 MB) bao gồm:
1. Base image đầy đủ (`python:3.11`) chứa toàn bộ các công cụ phát triển của Debian như trình biên dịch C/C++ (`gcc`, `g++`, `make`), git, các gói header thư viện hệ thống khổng lồ vốn chỉ cần lúc build. Ngược lại, bản `python:3.11-slim` đã lược bỏ sạch các gói này.
2. Trong multi-stage build, stage `builder` thực hiện cài đặt các thư viện vào virtualenv `/opt/venv`, sau đó stage runtime chỉ copy thư mục virtualenv thành phẩm sang.
3. Toàn bộ file rác, bộ nhớ đệm khi build package của pip (`~/.cache/pip`) và package manager apt không bị lưu vào image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- **Với Dockerfile hiện tại**:
  - Khi sửa code trong `app/main.py`, các layer ở stage builder (`COPY requirements.txt .`, `RUN pip install ...`) và các layer đầu của stage runtime (`RUN apt-get install curl`, `COPY --from=builder /opt/venv`) đều giữ nguyên mã hash nên Docker dùng lại từ bộ nhớ cache (`CACHED`).
  - Chỉ có layer `COPY . .` và các layer sau nó (`RUN chown ...`) bị invalid cache và chạy lại. Thời gian build lại chỉ mất 1-2 giây.
- **Nếu đặt `COPY . .` lên trước `RUN pip install`**:
  - Bất cứ khi nào sửa đổi source code, layer `COPY . .` bị thay đổi kéo theo layer `RUN pip install` phía sau bị phá vỡ cache. Docker sẽ phải tải lại và biên dịch lại toàn bộ dependencies trong `requirements.txt` từ đầu, làm tăng thời gian build lên rất nhiều lần.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- **Chuỗi sự kiện khi chạy bằng root**:
  1. Ứng dụng Python tồn tại lỗ hổng (ví dụ RCE qua deserialization, command injection, hay buffer overflow trong thư viện C).
  2. Kẻ tấn công kích hoạt lỗ hổng để thực thi shellcode trong container. Vì tiến trình đang chạy dưới quyền root (UID 0), kẻ tấn công chiếm toàn quyền kiểm soát bên trong container (đọc/sửa mọi file, cài package).
  3. Kẻ tấn công khai thác tiếp các lỗ hổng container escape (như lỗ hổng runc, mount nhầm docker.sock hoặc volume host) để thoát khỏi ranh giới container.
  4. Vì UID của tiến trình là 0 (root), khi thoát ra máy host nó nghiễm nhiên thừa hưởng quyền root của máy host, chiếm đoạt hoàn toàn máy chủ.
- **Lệnh `USER appuser` cắt đứt chuỗi**:
  Lệnh này cắt đứt ngay tại bước 2. Tiến trình chạy dưới quyền một user không đặc quyền (`appuser`). Dù kẻ tấn công có chiếm được shell trong container, họ cũng không có quyền can thiệp hệ thống hay khai thác các lỗ hổng breakout để leo thang đặc quyền ra máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.

Cách đạt được:
- Giả sử đồng hồ reset về 0 ở đầu mỗi phút (ví dụ lúc `10:01:00`).
- Người dùng gửi dồn dập 10 request vào giây `10:00:59` (giây cuối cùng của phút trước). Lúc này hạn mức của phút trước là 10/10 nên vẫn hợp lệ.
- Ngay khi bước sang giây `10:01:00`, bộ đếm được reset về 0. Người dùng lập tức gửi tiếp 10 request nữa trong giây này.
- Kết quả: Trong khoảng thời gian chỉ vỏn vẹn 2 giây (từ `10:00:59` đến `10:01:00`), hệ thống đã phải xử lý tới 20 request, gấp đôi hạn mức quy định. Cửa sổ trượt (sliding window 60s) giải quyết triệt để vấn đề này vì luôn đếm số request trong đúng 60 giây gần nhất tính từ thời điểm gọi.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Khác biệt cốt lõi**:
  - Rate limit bảo vệ **tính sẵn sàng của hạ tầng (hạn chế nghẽn mạng/server)** trong khoảng thời gian ngắn (tính theo phút), dựa trên **số lượng request**.
  - Cost guard bảo vệ **ngân sách tài chính** trong chu kỳ dài (tính theo tháng), dựa trên **tổng chi phí tiền tệ phát sinh** từ việc tiêu thụ token LLM.
- **Rate limit cho qua nhưng Cost guard chặn**:
  User chỉ gửi 1 request trong ngày (hoàn toàn dưới hạn mức 10 req/phút). Tuy nhiên request này khiến tổng tiền tích lũy trong tháng vượt quá ngân sách 10.0$ USD (hoặc trước đó user đã tiêu hết tiền). Cost guard sẽ chặn với mã lỗi `402 Payment Required`.
- **Cost guard cho qua nhưng Rate limit chặn**:
  Vào ngày đầu tháng, user mới dùng 0.01$ trên tổng 10.0$ ngân sách (tiền còn rất nhiều). Nhưng user dùng script spam 15 request liên tục trong 3 giây. Cost guard kiểm tra thấy còn ngân sách, nhưng Rate limit sẽ chặn từ request thứ 11 trở đi với mã `429 Too Many Requests` để tránh làm tê liệt server.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra:
1. Redis mất kết nối mạng trong 30 giây.
2. Endpoint gộp kiểm tra thấy Redis lỗi nên trả về 500/503.
3. Bộ điều phối (orchestrator) xem đây là probe Liveness bị fail (nghĩ rằng process đã chết) nên lập tức **tiến hành kill và restart lại toàn bộ 3 container**.
4. Khi 3 container vừa khởi động lại, Redis vẫn chưa có mạng, endpoint tiếp tục trả về lỗi.
5. Cụm container rơi vào thảm họa **CrashLoopBackOff / restart storm**: liên tục bị restart tuần hoàn, ngốn kiệt tài nguyên CPU/RAM của host và làm gián đoạn toàn bộ dịch vụ.
*(Khi tách riêng, `/health` vẫn trả 200 để container không bị khởi động lại oan, còn `/ready` trả 503 để load balancer tạm thời không chuyển traffic vào cho đến khi Redis kết nối lại).*

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- **Khi dùng Redis (Stateless)**:
  Cả 3 container đều truy xuất chung vào một nguồn dữ liệu tập trung ở Redis. Dù request được load balancer đẩy ngẫu nhiên vào bất kỳ container nào, `history_length` vẫn tăng đều đặn và tuyến tính: `0 -> 2 -> 4 -> 6...` (mỗi lượt hỏi-đáp thêm 2 message).
- **Nếu lưu trong dict Python (Stateful)**:
  Mỗi container sở hữu một bộ nhớ RAM riêng biệt. Khi gửi nhiều request liên tiếp:
  - Request 1 vào container A: `history_length = 0` (lưu vào RAM của A).
  - Request 2 bị chia tải sang container B: `history_length` vẫn là `0` vì RAM của B chưa có dữ liệu.
  - Request 3 sang container C: tiếp tục là `0`.
  - Request 4 quay lại container A: `history_length` nhảy lên `2`.
  Giá trị `history_length` sẽ nhảy lộn xộn, ngắt quãng và người dùng sẽ thấy mô hình bị "mất trí nhớ" tùy thuộc vào việc request rơi trúng container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi**:
  Khi deploy service lên Railway, endpoint `/ready` trả về `503 Service Unavailable` và log báo không thể kết nối tới cơ sở dữ liệu Redis.
- **Cách tìm ra nguyên nhân**:
  Vào tab **Deployments** -> chọn mục **Logs** của service trên Railway Dashboard để theo dõi log thực tế lúc container khởi chạy. Phát hiện biến môi trường `REDIS_URL` ban đầu được cấu hình là `redis://localhost:6379/0`. Trong môi trường container hóa trên cloud, `localhost` trỏ về chính container của Agent chứ không phải container Redis.
- **Cách sửa**:
  1. Tạo Redis database add-on trong cùng project Railway.
  2. Tại tab **Variables** của service Agent, cấu hình lại biến `REDIS_URL` dưới dạng reference variable: `${{Redis.REDIS_URL}}`.
  3. Railway tự động redeploy service với URL kết nối chính xác và endpoint `/ready` trả về `200 {"status":"ready","redis":true}`.
