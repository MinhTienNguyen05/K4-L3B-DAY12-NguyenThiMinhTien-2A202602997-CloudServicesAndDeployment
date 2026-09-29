# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Thị Minh Tiến  Mã học viên: 2A202602997

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu đặt giá trị mặc định là "changeme", khi triển khai lên môi trường Cloud/Production mà quên cấu hình biến AGENT_API_KEY, ứng dụng vẫn khởi động bình thường mà không hề báo lỗi. Lúc này, bất kỳ ai cũng có thể dùng key "changeme" để gửi request trái phép, khai thác mô hình AI và làm cạn kiệt ngân sách API. Cơ chế "Fail fast" bằng ValidationError của Pydantic ép container văng lỗi ngay từ giai đoạn khởi động (liveness/startup check thất bại), ngăn chặn việc public một service không được bảo vệ ra ngoài Internet

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> *Dòng log JSON thu được:
{"timestamp": "2026-09-29T05:28:44.120531Z", "level": "INFO", "event": "request_processed", "service": "day12-agent", "endpoint": "/ask", "user_id": "sv-test", "status_code": 200, "duration_ms": 48.2, "history_length": 2}

Hai việc làm được với log có cấu trúc này mà print thông thường không làm được:

Truy vấn và lọc có cấu trúc trên hệ thống quản lý tập trung (ELK, Datadog, Grafana Loki): Dễ dàng chạy truy vấn tìm kiếm chính xác (ví dụ: user_id = 'sv-test' AND status_code >= 400 hoặc vẽ biểu đồ độ trễ trung bình avg(duration_ms)) mà không cần viết regex để bóc tách chuỗi thô.

Thiết lập cảnh báo tự động (Alerting): Hệ thống giám sát có thể tự động parse JSON theo trường status_code để đếm tỷ lệ lỗi 5xx hoặc phát hiện tấn công bất thường dựa trên mật độ xuất hiện của user_id, giúp kích hoạt cảnh báo PagerDuty/Slack ngay lập tức.

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
| 1 stage (bản đầu) | ... MB |
| Multi-stage | ... MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (~317 MB) bao gồm: bộ compiler, header C/C++ (build-essential, gcc), thư mục cache của pip (/root/.cache/pip), các file wheel/tạm sinh ra trong quá trình biên dịch thư viện, và toàn bộ package phát triển không cần thiết ở runtime. Bản multi-stage chỉ copy thư mục dependencies đã được cài đặt hoàn chỉnh (/install) sang một base image python:3.11-slim mới tinh, loại bỏ hoàn toàn dấu vết của công cụ build

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi cấu hình chuẩn: Các layer tải base image, cài đặt package hệ thống, COPY requirements.txt và RUN pip install đều được tái sử dụng từ cache (CACHED). Chỉ các layer từ COPY app/ ./app/ trở về sau mới phải chạy lại, giúp thời gian build chỉ mất khoảng 1-2 giây. Nếu đặt COPY . . trước RUN pip install: Bất kỳ sự thay đổi nào trong mã nguồn (dù chỉ là 1 ký tự trong app/main.py) cũng sẽ làm mất hiệu lực (bust cache) của layer COPY . .. Khi đó, Docker buộc phải chạy lại toàn bộ lệnh RUN pip install, khiến quá trình build phải tải và cài lại toàn bộ thư viện từ đầu, làm tăng thời gian build lên nhiều phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện:

1. Code Python có lỗ hổng (ví dụ: Remote Code Execution qua deserialize hoặc Command Injection).

2. Kẻ tấn công khai thác lỗ hổng và thực thi shell bên trong container.

3. Do container chạy dưới quyền root (UID 0), tiến trình của attacker có toàn quyền can thiệp filesystem container và tương tác với Linux kernel syscalls của host.

4. Kẻ tấn công khai thác lỗ hổng bảo mật cấp kernel (như Dirty COW) hoặc volume mount lỗi (/var/run/docker.sock, /etc) để thoát khỏi container (container breakout) và chiếm quyền root trên chính máy host.

Lệnh USER appuser cắt đứt chuỗi này ngay tại bước 2: Khi shell bị chiếm, kẻ tấn công chỉ có quyền của user không đặc quyền (appuser, UID 10001). User này không thể sửa file hệ thống, không có quyền sudo, và bị kernel chặn đứng hầu hết các kỹ thuật leo thang đặc quyền container breakout.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Một người dùng có thể gửi tối đa 20 request trong 2 giây liên tiếp.

Giải thích:
Với fixed window (reset ở giây 00):

Ở giây thứ 59 của phút thứ nhất, người dùng gửi dồn 10 request. Lúc này bộ đếm của phút thứ nhất ghi nhận 10/10 (hợp lệ).

Sang giây 00 của phút thứ hai, bộ đếm tự động reset về 0. Người dùng lập tức gửi tiếp 10 request nữa.

Tổng cộng từ giây 59 đến giây 00 (khoảng cách chỉ 1-2 giây), hệ thống phải nhận và xử lý 20 request, vượt gấp đôi năng lực chịu tải dự kiến mà không bị chặn. Cơ chế Sliding Window giải quyết triệt để lỗi này bằng cách xét số lượng request trong đúng 60 giây trôi về trước tính từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Khác biệt cốt lõi:

Rate limit: Bảo vệ tính sẵn sàng và thông lượng (Availability/Throughput) của hệ thống trong thời gian ngắn (đếm số lượng request/giây hoặc request/phút) để chống nghẽn đường truyền và từ chối dịch vụ.

Cost guard: Bảo vệ ngân sách tài chính (Financial Budget) trong dài hạn (tính tổng token tiêu thụ hoặc chi phí tích lũy theo ngày/tháng).

Tình huống Rate limit cho qua nhưng Cost guard chặn:
Một user chỉ gửi duy nhất 1 request trong 10 phút (tốc độ cực thấp, rate limit cho qua thoải mái), nhưng request đó đính kèm văn bản hàng trăm trang và yêu cầu xuất ra tối đa output tokens. Lượng token này làm vượt ngưỡng ngân sách tích lũy tháng của người dùng, nên Cost guard phải chặn lại.

Tình huống Cost guard cho qua nhưng Rate limit chặn:
Đầu tháng ngân sách còn đầy ($10.0 còn nguyên), user chạy script gọi 15 request liên tiếp trong 2 giây với câu hỏi siêu ngắn "hi". Chi phí chỉ tốn vài fraction của cent (chưa thấm vào đâu so với budget), nhưng vì bắn quá dồn dập vượt quá 10 req/phút nên Rate limit chặn ở request thứ 11 với mã HTTP 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện xảy ra:

Redis gặp sự cố mất kết nối trong 30 giây.

Container orchestrator (Docker Compose/Kubernetes) gửi probe định kỳ kiểm tra endpoint gộp chung này. Vì không ping được Redis, endpoint trả về lỗi (hoặc timeout).

Do dùng chung làm liveness probe, orchestrator hiểu lầm là tiến trình ứng dụng đã chết hoặc deadlock, lập tức gửi tín hiệu SIGKILL/SIGTERM để restart cả 3 container agent.

Trong khi Redis vẫn chưa hồi phục, cả 3 container sau khi khởi động lại tiếp tục fail probe và liên tục bị restart, rơi vào vòng lặp tử thần CrashLoopBackOff.

Cụm app bị cạn kiệt tài nguyên CPU/IO cho việc restart liên tục; đến khi Redis sống lại, các container vẫn đang kẹt trong chu kỳ khởi động lại, kéo dài thời gian gián đoạn dịch vụ thay vì chỉ tạm dừng nhận traffic.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Khi lưu bằng Redis (Stateless): Bất kể request được bộ cân bằng tải phân phối đến container nào (agent-1, agent-2, hay agent-3), tất cả đều đọc/ghi chung vào một Redis instance tập trung. Do đó, history_length tăng tuần tự và nhất quán: 2, 4, 6, 8... Nếu lưu trong dict Python (Stateful): Mỗi container sở hữu vùng nhớ riêng biệt trong RAM. Khi gọi /ask liên tục, load balancer sẽ rải request ngẫu nhiên (hoặc round-robin) vào 3 container. Bạn sẽ thấy history_length bị nhảy loạn xạ và reset bất thường (ví dụ: gọi lần 1 vào agent-1 ra history_length=2, lần 2 trúng agent-2 lại ra history_length=2, lần 3 trúng agent-3 lại là 2, lần 4 quay lại agent-1 mới lên 4), khiến ngữ cảnh hội thoại của người dùng bị đứt gãy.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Thông báo lỗi:
pydantic_core._pydantic_core.ValidationError: 1 validation error for Settings
agent_api_key
Field required [type=missing, input_value={'port': '8080', ...}, input_type=dict]
Dẫn đến endpoint /ready và /ask đều trả về 500 Internal Server Error.

Cách tìm ra nguyên nhân:
Khi chạy pytest tests/test_cp5.py, bài test báo /ready trả 500 và /ask trả 500. Dùng lệnh railway logs --service day12-agent để đọc log chi tiết từ container, tôi phát hiện class Settings của Pydantic ném lỗi ValidationError do thiếu biến AGENT_API_KEY trong môi trường runtime khi dependency injection gọi get_settings().

Cách khắc phục:
Dùng Railway CLI chạy lệnh gán biến môi trường trực tiếp cho service:
railway variables --service day12-agent --set "AGENT_API_KEY=<KEY_THAT>"
Đồng thời cấu hình lại REDIS_URL trỏ chuẩn xác đến connection string của service day12-redis. Sau khi Railway tự động redeploy, endpoint /ready trả về 200 OK và /ask trả về 401 Unauthorized khi thiếu key, vượt qua toàn bộ test CP5.
