#  Delivery Checklist — Day 12 Lab Submission

> **Student Name:** Dương Trường Giang
> **Student ID:** 2A202600990 
> **Date:** 6/13/2026

---

# Day 12 Lab - Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found
1. API Key/Database URL Hardcoded: API Key (`OPENAI_API_KEY`) và Database URL (`DATABASE_URL`) bị ghi cứng trực tiếp trong file code `app.py`.
2. Thiếu Config Management: Các cài đặt (`DEBUG = True`, `MAX_TOKENS = 500`) được định nghĩa cố định
3. Sử dụng print() thông thường & rò rỉ log nhạy cảm: Ghi log bằng hàm `print()` không có cấu trúc (unstructured log), khó lọc và quản lý, đồng thời in cả API Key ra log.
4. Không có Health Check endpoint: Thiếu các endpoint `/health` và `/ready` khiến Cloud platform không giám sát được tình trạng ứng dụng để tự động khởi động lại khi gặp sự cố.
5. Cố định Host và Port: Thiết lập `host="localhost"` và `port=8000`. Khi đóng gói Docker Container, `localhost` sẽ ngăn chặn kết nối từ bên ngoài; đồng thời cổng `8000` cố định sẽ gây lỗi khi Cloud platform tự động gán cổng qua biến môi trường `PORT`.
6. reload=True ở môi trường Production: Việc bật tự động tải lại code làm hao tổn tài nguyên và giảm hiệu suất đáng kể của máy chủ thực tế.
7. Không xử lý tắt an toàn (Graceful Shutdown): Thiếu cơ chế xử lý tín hiệu `SIGTERM` dẫn đến các request đang xử lý bị ngắt đột ngột khi container bị tắt hoặc scale-down.

### Exercise 1.3: Comparison table
| Feature | Develop | Production | Why Important? |
|---------|---------|------------|----------------|
| Config  | Ghi cứng (hardcode) trong code. | Sử dụng biến môi trường (Environment Variables) qua class `Settings`. | Bảo vệ thông tin nhạy cảm (secrets); dễ dàng thay đổi cấu hình mà không cần sửa code khi deploy sang các môi trường khác nhau. |
| Health Check | Không có endpoint kiểm tra sức khỏe. | Hỗ trợ endpoint `/health` và `/ready`. | Giúp Cloud platform (Railway, Render, K8s) tự động theo dõi, restart container nếu lỗi, hoặc điều phối traffic thông minh. |
| Logging | Sử dụng `print()` thông thường, log cả API key bí mật. | Dùng thư viện `logging` tiêu chuẩn, định dạng JSON có cấu trúc (Structured JSON Logging). | Dễ dàng parse và quản lý log tự động ở production; ngăn chặn lộ thông tin bí mật qua log. |
| Shutdown | Dừng đột ngột (`uvicorn` ngắt ngang các request). | Xử lý sự kiện `lifespan` và tín hiệu `SIGTERM` để tắt an toàn (Graceful Shutdown). | Đảm bảo các request hiện tại được xử lý hoàn tất trước khi tắt server, tăng độ tin cậy và tránh làm mất mát dữ liệu của người dùng. |
| Host & Port binding | Cố định `localhost` và port `8000`. | Bind vào `0.0.0.0` và cổng động được lấy từ biến môi trường `PORT`. | Cho phép container nhận kết nối từ bên ngoài và tự động tương thích với cơ chế gán Port của Cloud platform. |
| reload (Debug Mode) | Luôn bật `reload=True`. | Chỉ bật khi `DEBUG=True`, mặc định tắt ở Production. | Tránh việc reload liên tục làm giảm hiệu năng của ứng dụng trên máy chủ thực tế. |

## Part 2: Docker

### Exercise 2.1: Dockerfile questions
1. Base image: `python:3.11` cho bản develop và `python:3.11-slim` cho bản production (giúp tối ưu dung lượng).
2. Working directory: `/app`
3. Tại sao COPY requirements.txt trước?: Để tận dụng cơ chế Layer Cache của Docker. Nếu file `requirements.txt` không đổi, Docker sẽ lấy các thư viện đã cài từ cache ra dùng lại thay vì phải tải và cài đặt lại toàn bộ mỗi khi ta sửa một dòng code nhỏ.
4. CMD vs ENTRYPOINT khác nhau thế nào?: `CMD` định nghĩa lệnh chạy mặc định nhưng có thể dễ dàng bị ghi đè (override) khi truyền thêm tham số vào lệnh `docker run`. `ENTRYPOINT` quy định file thực thi cốt lõi luôn luôn chạy, các tham số truyền thêm ở `docker run` sẽ được tự động nối vào sau `ENTRYPOINT`.

### Exercise 2.3: Image size comparison
- Develop: 1.66 GB (~1700 MB)
- Production: 236 MB
- Difference: ~85% (giảm khoảng 85% dung lượng)

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment
- URL: https://upbeat-hope-production-27a0.up.railway.app
- Screenshot: `screenshots\dashboard.png`
- Screenshot: `screenshots\running.png`

## Part 4: API Security

### Exercise 4.1-4.3: Test results
- **Câu hỏi 4.1 (API Key):**
  - API key được check trong middleware/dependency của FastAPI (thường nằm ở `auth.py`). Hệ thống sẽ đọc header `X-API-Key` và so sánh với biến môi trường `AGENT_API_KEY`.
  - Nếu sai key, server sẽ trả về lỗi `401 Unauthorized`.
  - Để rotate (đổi) key, ta chỉ cần đổi giá trị biến môi trường `AGENT_API_KEY` trên dashboard của Cloud platform (Railway/Render) rồi restart lại container mà không cần sửa code.
- **Câu hỏi 4.3 (Rate Limiting):**
  - Thuật toán được dùng thường là **Sliding Window** hoặc **Token Bucket** (sử dụng Redis để đếm số request).
  - Giới hạn (Limit) thường được thiết lập ở mức 10 requests / phút.
  - Để bypass cho admin, ta có thể thêm một câu lệnh điều kiện: nếu `user_id == "admin"` thì bỏ qua bước check giới hạn (bỏ qua hàm gọi Redis).

*[Kết quả test API thực tế]*
**1. Test thành công (Mã 200 OK):**
```json
{
  "question": "Hello",
  "answer": "This is a mock response from the LLM.",
  "platform": "Railway"
}
```

**2. Test thất bại do thiếu API Key (Mã 401 Unauthorized):**
```json
{
  "detail": "Invalid API Key"
}
```

**3. Test thất bại do gọi quá nhanh (Mã 429 Rate Limit):**
```json
{
  "detail": "Rate limit exceeded. Try again later."
}
```

### Exercise 4.4: Cost guard implementation
**Cách tiếp cận (Cost guard):**
Tôi đã sử dụng Redis để lưu trữ mức chi tiêu của từng user theo tháng. Key trong Redis sẽ có dạng `budget:{user_id}:{YYYY-MM}`.
Mỗi khi có request, hệ thống sẽ dùng `r.get(key)` để lấy tổng số tiền đã tiêu trong tháng này. Nếu `tổng + chi phí dự tính > 10$`, hệ thống sẽ trả về lỗi từ chối phục vụ (budget exceeded). Nếu còn ngân sách, hệ thống dùng `r.incrbyfloat` để cộng dồn tiền và set thời gian hết hạn (expire) cho key là khoảng 32 ngày để tự động reset vào tháng sau.

## Part 5: Scaling & Reliability

### Exercise 5.1-5.5: Implementation notes
- **Health & Readiness Checks (5.1):** Tôi đã thêm endpoint `/health` (Liveness) luôn trả về `200 OK` để platform biết container không bị crash, và endpoint `/ready` (Readiness) để test kết nối tới Redis/Database, đảm bảo app chỉ nhận traffic khi đã kết nối đủ các service phụ trợ.
- **Graceful Shutdown (5.2):** Tôi sử dụng `lifespan` trong FastAPI (hoặc cấu hình bắt tín hiệu `SIGTERM`) để khi có lệnh tắt, server sẽ ngừng nhận request mới nhưng vẫn chờ và hoàn thành nốt các request hiện tại trước khi tắt hẳn, tránh làm mất kết nối ngắt ngang của user.
- **Stateless Design (5.3):** Thay vì lưu lịch sử chat vào một biến `dictionary` trên RAM của code (stateful), tôi chuyển việc lưu trữ lịch sử sang Redis. Như vậy khi scale (nhân bản) ra 3 server, server nào cũng có thể xử lý request của user vì dữ liệu đều lấy từ 1 nguồn chung là Redis.
- **Load Balancing (5.4):** Chạy nhiều instance của Agent bằng Docker Compose (`--scale agent=3`) kết hợp Nginx làm Reverse Proxy. Nginx sẽ đứng ra nhận request từ ngoài vào và chia đều (round-robin) đến 3 instance Agent, giúp phân tải và tăng khả năng chịu lỗi.
