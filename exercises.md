# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder `*Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đặng Đỉnh Đoàn  Mã học viên: 2A202602927

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi tạo service `agent` trên Railway, mình phải tự thêm biến `AGENT_API_KEY` vào
tab Variables. Nếu quên bước đó mà code có mặc định `"changeme"`, app vẫn khởi
động bình thường, health check xanh, Railway báo deploy thành công. Nhưng
`/ask` lúc đó được bảo vệ bằng một khóa ai cũng đoán được (nó nằm công khai
trong repo), nên bất kỳ ai tìm ra URL đều gọi LLM bằng tiền của mình. Mình chỉ
biết khi nhìn hóa đơn.

Không có mặc định thì `Settings()` ném `ValidationError: agent_api_key Field
required` ngay lúc container khởi động. Container crash, health check fail,
deploy đỏ, và lỗi hiện ngay trong Deploy Logs lúc mình còn đang theo dõi. Sửa
thì chỉ cần thêm biến rồi deploy lại, chưa có request lạ nào lọt vào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log thu được từ `docker compose logs agent`:

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T04:55:42.812334+00:00", "user_id": "sv01", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}
```

1. **Lọc và tổng hợp theo trường.** Vì mỗi dòng là một JSON, mình có thể lọc
   `event == "ask_completed"` rồi cộng `cost_usd` theo `user_id` để trả lời
   "user nào tiêu nhiều tiền nhất hôm nay". Với `print("đã trả lời xong")` thì
   không có user, không có chi phí, chẳng có gì để cộng.
2. **Đặt cảnh báo tự động.** Có `level` và `timestamp` chuẩn ISO nên hệ thống
   log có thể đếm số dòng `level == "error"` trong 5 phút gần nhất và báo động
   khi vượt ngưỡng, hoặc sắp xếp đúng thứ tự sự kiện giữa nhiều container.
   Chuỗi văn bản tự do thì phải viết regex riêng cho từng kiểu câu, đổi câu
   chữ là regex hỏng.

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
| 1 stage (bản đầu) | 1730 MB (1.73GB) |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Chênh lệch khoảng 1.46GB, gần như toàn bộ đến từ **base image**. Bản đầu dùng
`python:3.11` đầy đủ, dựa trên Debian đầy đủ, mang theo trình biên dịch
(gcc, make), header phát triển (`libc-dev`, `libssl-dev`...), git, curl và rất
nhiều thư viện hệ thống mà app chạy không cần. Bản multi-stage dùng
`python:3.11-slim` cho stage runtime, chỉ có Python và vài thư viện tối thiểu.

Theo `docker history agent:single`, bản thân layer `pip install` chỉ khoảng
95MB và layer `COPY . .` chỉ 156kB (vì `.dockerignore` đã loại `.git`, `.venv`,
`tests`). Tức là thư viện Python không phải nguyên nhân. Nếu có gói cần biên
dịch, stage `builder` được phép cài compiler rồi bị vứt đi; stage cuối chỉ
nhận thư mục `/install` đã cài xong qua `COPY --from=builder`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Mình thêm một dòng vào cuối `app/main.py` rồi build lại với `--progress=plain`:

- **Dùng lại cache (`CACHED`):** `WORKDIR /build`, `COPY requirements.txt`,
  `RUN pip install`, `COPY --from=builder /install`, `RUN useradd`,
  `WORKDIR /app`, `COPY utils`.
- **Chạy lại:** chỉ layer cuối `COPY app ./app`, mất chưa tới 1 giây.

Lý do: Docker so checksum từng layer và hủy cache từ layer đầu tiên có thay
đổi trở đi. `requirements.txt` không đổi nên layer `pip install` giữ nguyên.
`app/` được copy ở bước cuối nên chỉ bước đó bị ảnh hưởng.

Mình build thử lại bản 1 stage gốc (có `COPY . .` đứng trước `pip install`)
với cùng thay đổi đó: `COPY . .` chạy lại vì một file trong context đã đổi,
kéo theo `RUN pip install -r requirements.txt` chạy lại toàn bộ, mất **66
giây** chỉ vì sửa một dòng code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện khi container chạy root:

1. Code có lỗ hổng cho phép thực thi lệnh tùy ý, ví dụ một thư viện bị lỗi
   deserialize hoặc một chỗ đưa input người dùng vào `subprocess`.
2. Kẻ tấn công chạy lệnh trong container với quyền của process uvicorn, tức
   **root (uid 0)**.
3. Là root trong container, họ đọc/sửa được mọi file trong image, cài thêm
   công cụ, đọc biến môi trường chứa secret như `AGENT_API_KEY`, `REDIS_URL`.
4. Container dùng chung kernel với host, và uid 0 trong container chính là
   uid 0 trên host (khi không bật user namespace). Nếu có volume mount từ host,
   Docker socket bị mount vào, hay một lỗ hổng kernel/runtime để thoát
   container, thì họ ra ngoài với **quyền root trên host**.

Lệnh `USER appuser` cắt chuỗi ở **bước 2**: process chạy bằng uid 10001, một
user thường. Mình kiểm tra bằng `docker compose exec agent id`, ra
`uid=10001(appuser)`. Kẻ tấn công vẫn vào được app nhưng không ghi được vào
thư mục hệ thống, không cài được gói, và nếu thoát ra được host thì cũng chỉ là
uid 10001 không có đặc quyền gì, thay vì root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa **20 request trong 2 giây**, gấp đôi hạn mức.

Cách làm: gửi 10 request lúc 10:00:59. Chúng thuộc bộ đếm của phút 10:00, vừa
đủ 10/10 nên đều được qua. Sang 10:01:00 bộ đếm reset về 0, gửi tiếp 10 request
lúc 10:01:00, lại được qua hết vì phút 10:01 mới có 10/10. Tổng cộng 20
request trong khoảng 10:00:59 đến 10:01:00.

Với sliding window, lúc 10:01:00 mình đếm các request trong 60 giây gần nhất
(từ 10:00:00 đến 10:01:00), thấy đã có 10 nên chặn ngay. Test thật trên bản
deploy Railway: 15 request liên tiếp cùng một user cho ra
`200 ×10` rồi `429 ×5`.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn **tần suất** (bao nhiêu request trong 60 giây, trả 429),
bảo vệ hệ thống khỏi bị dồn dập. Cost guard giới hạn **tổng tiền** trong một
tháng (trả 402), bảo vệ ngân sách. Một cái đếm số lần, một cái đếm số tiền,
và mỗi request có giá rất khác nhau.

- **Rate limit cho qua, cost guard chặn:** một user gửi đều đặn 5 request mỗi
  phút, không bao giờ chạm mức 10/phút. Nhưng mỗi request gửi prompt dài kèm
  lịch sử hội thoại, tốn nhiều token. Liên tục cả ngày trong vài tuần thì tổng
  chi phí vượt `MONTHLY_BUDGET_USD = 10`. Rate limit vẫn thấy hợp lệ, nhưng cost
  guard trả 402.
- **Cost guard cho qua, rate limit chặn:** một user mới, chi phí tháng gần như
  0, chạy script gửi 15 câu hỏi ngắn trong vài giây. Mỗi câu chỉ tốn khoảng
  $0.00002 nên ngân sách còn nguyên, nhưng từ request thứ 11 rate limit trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Redis mất kết nối. Cả 3 container cùng lúc gọi `ping()` thất bại, endpoint
   gộp trả 503.
2. Orchestrator coi 503 ở liveness probe là "process hỏng, cần restart". Sau
   vài lần thử (ví dụ `retries: 3`, `interval: 15s`) nó **restart cả 3
   container**, vì cả 3 đều báo lỗi giống nhau.
3. Trong lúc restart, không còn container nào nhận request. User nhận 502/503
   cho **mọi** endpoint, kể cả những thứ không cần Redis.
4. Redis quay lại sau 30 giây, nhưng các container vẫn đang khởi động lại, có
   khi còn rơi vào vòng crash-restart nếu Redis chưa ổn định hẳn.
5. Kết quả: 30 giây Redis chập chờn biến thành sự cố sập toàn hệ thống kéo dài
   hơn nhiều.

Khi tách riêng: `/health` không đụng Redis nên vẫn 200, không container nào bị
restart. `/ready` trả 503 nên load balancer tạm ngừng đẩy traffic vào. Redis
quay lại thì `/ready` về 200 và traffic chảy lại ngay, container không phải
khởi động lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Mình chạy `docker compose up -d --scale agent=3`, gọi qua nginx ở
`localhost:8000` 5 lần với cùng `X-User-Id`. `history_length` trả về lần lượt
là **`0 2 4 6 8`**, tăng đều 2 mỗi lượt (1 message user + 1 message assistant).
Xem `docker compose logs agent` thì thấy request được chia cho cả `agent-1`,
`agent-2` và `agent-3`, vậy mà lịch sử vẫn liền mạch vì cả 3 cùng đọc/ghi một
key `history:<user>` trong Redis.

Nếu lịch sử nằm trong dict Python, mỗi container có dict riêng trong RAM của
nó. Nginx chia round-robin nên con số sẽ nhảy lung tung, kiểu `0 0 0 2 2 2 4
4 4`: mỗi container chỉ thấy những lượt đã rơi vào chính nó. Agent "quên" câu
trả lời vừa nói nếu request tiếp theo rơi vào container khác. Restart một
container thì lịch sử của phần đó mất hẳn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

**Lỗi:** job `Deploy to Railway` trong GitHub Actions đỏ sau 12 giây, trong khi
`test` và `build` đều xanh:

```
Run railway up --ci --service "$RAILWAY_SERVICE"
Indexing...
Uploading...
Failed to upload code with status code 404 Not Found
```

**Tìm nguyên nhân:**
1. Nghi tên service sai, nhưng log in ra `RAILWAY_SERVICE: agent`, khớp với tên
   mình đặt.
2. Nghi token sai loại, nhưng đúng là Project Token tạo trong Project Settings.
3. Cài Railway CLI ở máy rồi chạy `railway status` với đúng token đó. Output
   cho thấy token trỏ đúng project và environment `production`, nhưng mục
   "All resources" **chỉ có Redis**, không có service `agent` nào.
4. Mở lại dashboard mới thấy thanh "Apply 6 changes": service `agent` và 5 biến
   môi trường mới chỉ là thay đổi **đang chờ (staged)** trên canvas, chưa được
   tạo thật. Vì vậy API trả 404 khi CLI upload code vào một service không tồn
   tại.

**Sửa:** bấm **Deploy** để apply các thay đổi đang chờ. Service `agent` được
tạo thật, `railway status` đã thấy cả `agent` lẫn `Redis`. Re-run job deploy
thì `railway up` upload và build thành công. Sau đó generate domain, đặt
biến `PUBLIC_URL` và re-run; bước smoke test gọi `/health`, `/ready`, `/ask`
đều pass.

**Rút ra:** khi log báo lỗi mơ hồ kiểu 404, hãy kiểm tra trực tiếp trạng thái
thật bằng CLI thay vì chỉ nhìn giao diện. Giao diện hiển thị cả những thứ chưa
thực sự tồn tại.
