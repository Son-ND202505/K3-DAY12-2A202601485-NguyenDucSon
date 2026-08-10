# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng đánh dấu phần trả lời còn trống bằng câu trả lời thật.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Đức Sơn  Mã học viên: 2A202601485

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Lúc deploy lên Railway ở CP5, tôi từng quên chưa set `AGENT_API_KEY` trong
> dashboard trước khi bấm deploy (thực ra là quên push code, nhưng tình huống
> tương tự đã xảy ra khi test local: chạy `docker compose up` mà `.env` chưa
> có `AGENT_API_KEY`). Nếu `agent_api_key` có mặc định kiểu `"changeme"`, app
> vẫn khởi động bình thường, `/health` vẫn trả 200 — nhìn vào là tưởng mọi thứ
> ổn. Nhưng thực chất endpoint `/ask` công khai đang được bảo vệ bằng một
> khóa ai cũng đoán được, ai gọi cũng qua được `verify_api_key`. Tôi sẽ không
> biết chuyện này cho tới khi nhận hóa đơn LLM bất thường hoặc bị người lạ
> lạm dụng. Vì `agent_api_key: str` không có default, Pydantic ném
> `ValidationError` ngay lúc khởi động — container crash ngay, tôi thấy lỗi
> ngay trong log, sửa ngay lúc đang đứng nhìn màn hình, thay vì để lỗ hổng
> âm thầm chạy trên production.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật lấy từ chạy `uvicorn` local và gọi `/ask`:
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T06:02:17.187736+00:00", "user_id": "sv01", "tokens_in": 3, "tokens_out": 41, "cost_usd": 2.505e-05}
> ```
>
> Hai việc làm được mà `print("đã trả lời xong")` không làm được:
> 1. **Lọc và tổng hợp theo trường**: vì log là JSON có cấu trúc, tôi có thể
>    dùng `jq` hoặc query trên Datadog/Railway logs để hỏi "tổng `cost_usd`
>    của `user_id=sv01` trong 24h qua là bao nhiêu?" — với `print()` tôi phải
>    tự viết regex để bóc tách chuỗi tự do, dễ vỡ khi câu chữ đổi.
> 2. **Đặt cảnh báo tự động (alerting)**: vì có trường `level` chuẩn hóa
>    (info/error...) và `cost_usd` là số, tôi có thể set rule "báo động nếu
>    tổng `cost_usd` theo `user_id` vượt X trong 1 giờ" trên hệ thống giám
>    sát. `print()` không có trường nào để máy đọc và so sánh ngưỡng.

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
| 1 stage (bản đầu) | 1730 MB |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Số đo build thật trên máy tôi: bản 1 stage dùng `python:3.11` (bản đầy đủ,
> có sẵn compiler, dev headers, nhiều gói hệ thống không cần lúc chạy) và
> `COPY . .` trước khi cài — 1.73GB. Bản multi-stage dùng `python:3.11-slim`
> cho cả 2 stage, và stage `builder` (nơi chạy `pip install`, có thể cần
> compiler tạm thời) bị loại bỏ hoàn toàn ở stage `runtime` — chỉ
> `COPY --from=builder /install /usr/local` phần thư viện Python đã cài
> xong, không mang theo compiler hay cache pip — còn 270MB. Chênh lệch
> ~1.46GB chủ yếu là: (1) phần base OS đầy đủ vs slim, (2) build tool
> (gcc, headers...) chỉ dùng lúc cài đặt rồi bị vứt đi, (3) cache pip và
> file trung gian không cần thiết ở runtime.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Dockerfile của tôi: `COPY requirements.txt .` → `RUN pip install ...` →
> `COPY app ./app`. Sửa 1 ký tự trong `app/main.py` chỉ làm thay đổi input
> của layer `COPY app ./app` trở đi — Docker so sánh hash nội dung file, các
> layer trước đó (`FROM`, `WORKDIR`, `COPY requirements.txt`,
> `RUN pip install`) không đổi input nên được lấy thẳng từ cache, build lại
> chỉ mất vài giây thay vì cài lại toàn bộ `requirements.txt`.
>
> Nếu đặt `COPY . .` lên trước `RUN pip install`: bất kỳ thay đổi nào trong
> code (kể cả sửa 1 ký tự) cũng làm layer `COPY . .` bị invalidate, và mọi
> layer đứng SAU nó — bao gồm `RUN pip install` — cũng mất cache theo (cache
> Docker chỉ hợp lệ nếu layer đó VÀ mọi layer trước nó đều không đổi). Kết
> quả là mỗi lần sửa code, dù chỉ 1 dòng, `pip install` phải chạy lại từ đầu,
> tải và cài lại toàn bộ thư viện — build chậm đi rất nhiều so với đặt đúng
> thứ tự.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) một thư viện Python trong `requirements.txt` hoặc chính
> code `app/` có lỗ hổng (VD: deserialization không an toàn, command
> injection qua input người dùng); (2) kẻ tấn công khai thác lỗ hổng đó để
> chạy được lệnh tùy ý bên trong container; (3) nếu process đang chạy bằng
> root (UID 0) bên trong container, lệnh tùy ý đó có toàn quyền trong
> container — đọc/ghi mọi file, cài phần mềm; (4) nếu container runtime có
> lỗ hổng thoát container (container escape, kernel namespace bị khai thác)
> hoặc container được mount volume/socket nhạy cảm (VD: Docker socket), thì
> quyền root TRONG container có thể leo thang thành quyền root TRÊN HOST.
>
> Lệnh `USER appuser` (tôi dùng `useradd --uid 10001 appuser` rồi `USER
> appuser`) cắt đứt chuỗi ngay ở bước (3): dù kẻ tấn công chạy được lệnh tùy
> ý trong container, lệnh đó chỉ có quyền của user thường (UID 10001), không
> đọc/ghi được file hệ thống, không leo thang được lên root ngay trong
> container — nên kể cả nếu có lỗ hổng escape ở tầng dưới, thiệt hại tối đa
> vẫn chỉ là quyền user thường, không phải root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request trong 2 giây. Cách đạt được: gửi đủ 10 request lúc
> 10:00:59 (vẫn còn nằm trong "phút 10:00", chưa vượt hạn mức 10/phút của
> phút đó), rồi gửi tiếp 10 request lúc 10:01:00–10:01:01 (bộ đếm vừa reset
> về 0 vì sang "phút 10:01" mới, nên lại được phép 10 request nữa). Tổng
> cộng 20 request lọt qua trong cửa sổ thời gian thực tế chỉ 2 giây, dù danh
> nghĩa "đúng luật 10/phút" ở cả hai phút. Đây chính là kẽ hở mà sliding
> window (đếm 60 giây gần nhất tính từ thời điểm hiện tại, không neo theo
> mốc giây 00) không có, vì nó luôn nhìn lùi lại đúng 60 giây trước bất kỳ
> request nào, không có ranh giới cố định để "canh giờ" lách qua.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn *tần suất/số lượng* request bất kể request đó tốn bao
> nhiêu tiền; cost guard giới hạn *tổng số tiền* bất kể request đến nhanh hay
> chậm.
>
> Tình huống rate limit cho qua nhưng cost guard phải chặn: user chỉ gửi 3
> request/phút (dưới hạn mức 10/phút nên rate limit không chặn), nhưng mỗi
> câu hỏi rất dài (50.000 token), 3 request đó đã đủ tiêu hết ngân sách tháng
> — cost guard trả 402 dù rate limit hoàn toàn im lặng.
>
> Tình huống ngược lại: user gửi 15 request/phút, mỗi request chỉ hỏi 1 câu
> ngắn vài token, tổng chi phí cả tháng vẫn rất nhỏ so với ngân sách 10 USD
> (cost guard không chặn) — nhưng vì vượt hạn mức 10 request/phút, rate
> limiter vẫn trả 429 cho những request cuối, bảo vệ server khỏi bị spam dù
> chưa tốn tiền đáng kể.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện: (1) Redis mất kết nối; (2) endpoint gộp (giờ vừa đóng vai
> liveness vừa readiness) gọi `store.ping()` thất bại, trả 503 cho CẢ 3
> container cùng lúc vì cả 3 đều phụ thuộc cùng 1 Redis; (3) orchestrator
> (Docker/Railway/K8s) đang dùng chính endpoint này làm liveness probe, thấy
> 503 liên tục trong khoảng thời gian probe timeout thì kết luận "process
> chết" và RESTART cả 3 container cùng lúc — dù thực ra process vẫn sống
> bình thường, chỉ là Redis chưa kết nối được; (4) trong lúc cả 3 container
> đang restart, không còn instance nào phục vụ request — toàn bộ service
> down hoàn toàn; (5) khi Redis quay lại sau 30 giây, cả 3 container có thể
> vẫn đang trong quá trình khởi động lại, service tiếp tục gián đoạn thêm
> một khoảng thời gian nữa. Một sự cố Redis thoáng qua (30 giây) bị khuếch
> đại thành downtime toàn hệ thống — đúng lý do lab tách riêng `/health`
> (không đụng dependency, chỉ dùng cho liveness) và `/ready` (có kiểm tra
> Redis, dùng cho readiness — LB chỉ ngừng gửi traffic chứ không restart).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Tôi chạy `docker compose up -d --scale agent=3`, lấy IP nội bộ của cả 3
> container (172.19.0.4, .5, .3), rồi gọi `/ask` luân phiên đúng thứ tự
> `.4 → .5 → .3 → .4 → .5` với cùng `X-User-Id: sv02`. Kết quả `history_length`
> thật quan sát được: `0 → 2 → 4 → 6 → 8` — tăng đều đặn dù mỗi request rơi
> vào một container vật lý khác nhau. Điều này chứng minh state (lịch sử hội
> thoại) không nằm trong process nào cả mà nằm ở Redis dùng chung.
>
> Nếu lịch sử lưu trong một dict Python trong process (`conversation_history
> = {}`), mỗi container sẽ có một dict riêng trong RAM của chính nó. Gọi
> luân phiên qua 3 container như trên, `history_length` sẽ KHÔNG tăng đều mà
> nhảy lung tung kiểu `0, 0, 0, 1, 1` (mỗi container tự đếm từ 0 dựa trên số
> lần chính nó từng thấy user đó, không biết 2 container kia đã xử lý bao
> nhiêu request rồi) — agent sẽ trông như "mất trí nhớ" ngẫu nhiên tùy request
> rơi vào container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi gặp phải: build và deploy trên Railway đều pass, nhưng bước Healthcheck
> luôn fail. Vào tab "Deploy Logs" thấy lặp lại liên tục:
> `Error: Invalid value for '--port': '$PORT' is not a valid integer.`
>
> Nguyên nhân: file `railway.toml` của tôi có khai báo
> `startCommand = "uvicorn app.main:app --host 0.0.0.0 --port $PORT"`. Railway
> chạy `startCommand` KHÔNG qua shell, nên `$PORT` không được shell thay thế
> thành số cổng thật — nó bị truyền y nguyên chuỗi ký tự `"$PORT"` vào
> `uvicorn --port`, và uvicorn không parse được chuỗi đó thành số nguyên.
> Trong khi đó `Dockerfile` của tôi đã viết đúng
> `CMD ["sh", "-c", "uvicorn ... --port ${PORT:-8000}"]` — có `sh -c` nên
> shell thay thế biến bình thường, nhưng `startCommand` trong `railway.toml`
> lại ĐÈ LÊN `CMD` của Dockerfile.
>
> Cách sửa: xóa hẳn dòng `startCommand` khỏi `railway.toml`, để Railway dùng
> thẳng `CMD` đã đúng sẵn trong Dockerfile. Commit, push, Railway build lại
> — nhưng vẫn tiếp tục fail vì gặp thêm vấn đề thứ hai không liên quan: kết
> nối GitHub App của Railway bị đứt sau khi repo từng đổi tên trước đó
> (`Settings → Source` báo "GitHub Repo not found"), nên webhook không tự
> trigger build mới dù đã push đúng code. Phải vào "Configure GitHub App"
> cấp lại quyền truy cập repo, kết nối lại Source Repo, lúc đó Railway mới
> build đúng commit mới nhất và deploy thành công — `/health` và `/ready`
> đều trả 200.
