# Day 12 Lab - Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found
Có ít nhất 5 anti-patterns chính được phát hiện trong file `develop/app.py`:
1. **Hardcoded Secrets**: API Key (`sk-hardcoded-fake-key-never-do-this`) và Database URL (`postgresql://admin:password123@localhost:5432/mydb`) bị ghi cứng trực tiếp vào code. Nếu push lên GitHub sẽ bị lộ ngay lập tức.
2. **Thiếu Config Management**: Các cài đặt cấu hình như `DEBUG` hay `MAX_TOKENS` bị gán trực tiếp bằng biến thay vì sử dụng biến môi trường (Environment Variables) hoặc công cụ quản lý cấu hình.
3. **Print-based Logging**: Dùng hàm `print()` thay vì thư viện logging tiêu chuẩn. Điều này gây khó khăn khi thu thập log tập trung (Loki, Datadog), không thể kiểm soát log level, và có nguy cơ in thẳng các secret (API key) ra stdout/stderr.
4. **Thiếu Health Check Endpoints**: Không định nghĩa endpoint `/health` hoặc `/ready`. Nếu ứng dụng bị treo hoặc crash, các platform container orchestration (K8s, ECS, Cloud Run) không thể phát hiện để tự khởi động lại (restart) container.
5. **Port & Host cố định**: Gán cứng `host="localhost"` và `port=8000` trong code. Trên các môi trường Cloud (Railway/Render), PORT được inject động qua biến môi trường. Ngoài ra, việc lắng nghe trên `localhost` thay vì `0.0.0.0` khiến container không thể tiếp nhận request gửi từ bên ngoài.
6. **Bật Debug Mode ở Production**: Thiết lập `reload=True` khi khởi chạy uvicorn, điều này tiêu tốn nhiều tài nguyên hệ thống và mở ra lỗ hổng bảo mật khi chạy trên môi trường production.

---

### Exercise 1.3: Comparison table

| Feature | Develop | Production | Tại sao quan trọng? |
|---------|---------|------------|---------------------|
| **Config** | Gán cứng (Hardcoded) | Biến môi trường (Environment variables) | Cho phép ứng dụng hoạt động linh hoạt trên nhiều môi trường khác nhau (dev, staging, prod) mà không cần thay đổi source code; tránh lộ lọt secrets lên hệ thống kiểm soát phiên bản (VCS). |
| **Health check** | Không có (❌) | Hỗ trợ (/health & /ready) (✅) | Giúp các công cụ giám sát (monitoring) và điều phối container (orchestrator) xác định trạng thái của ứng dụng (liveness/readiness) để tự động khởi động lại khi crash hoặc dừng định tuyến traffic khi đang quá tải. |
| **Logging** | Dùng hàm `print()` | Định dạng JSON structured | Cho phép thu thập, phân tích và lọc logs tự động trên các hệ thống giám sát log tập trung. Đồng thời có thể kiểm soát log levels linh hoạt (DEBUG, INFO, ERROR). |
| **Shutdown** | Đột ngột (Graceful ❌) | Graceful shutdown (SIGTERM) (✅) | Đảm bảo hoàn thành các request đang xử lý dở dang (in-flight requests) và ngắt các kết nối cơ sở dữ liệu/Redis một cách an toàn trước khi tắt ứng dụng, tránh mất mát dữ liệu. |

---

## Part 2: Docker

### Exercise 2.1: Dockerfile questions
1. **Base image**: `python:3.11` (Chứa hệ điều hành Debian đầy đủ + Python runtime, dung lượng lớn khoảng ~1 GB).
2. **Working directory**: `/app` (Thư mục làm việc mặc định trong container, nơi tất cả các file code và cấu hình của ứng dụng được copy vào).
3. **Tại sao COPY requirements.txt trước?**: Để tận dụng tối đa cơ chế Docker Layer Caching. Docker sẽ chỉ chạy lại bước cài đặt thư viện (`pip install`) khi file `requirements.txt` thay đổi. Nếu copy toàn bộ code trước, mọi thay đổi nhỏ trong code đều khiến Docker phải cài lại toàn bộ thư viện từ đầu, làm tăng đáng kể thời gian build.
4. **CMD vs ENTRYPOINT**:
   - `ENTRYPOINT` định nghĩa câu lệnh chính thức sẽ luôn chạy khi container khởi động và khó bị ghi đè hơn.
   - `CMD` định nghĩa các tham số mặc định truyền vào cho `ENTRYPOINT` (hoặc là câu lệnh mặc định nếu không khai báo `ENTRYPOINT`). `CMD` có thể dễ dàng bị ghi đè khi ta chạy lệnh `docker run <image> <command_overridden>`.

---

### Exercise 2.3: Image size comparison
- **Develop (Single-stage, base `python:3.11`)**: Khoảng **1.01 GB**
- **Production (Multi-stage, base `python:3.11-slim`)**: Khoảng **143 MB**
- **Difference (Sự chênh lệch)**: Giảm khoảng **85.8%** dung lượng.
- **Giải thích**: Multi-stage build cho phép ta chia việc đóng gói container thành hai bước. Stage 1 (Builder) sử dụng đầy đủ công cụ build (gcc, build-essential) để tải và biên dịch dependencies. Stage 2 (Runtime) chỉ sử dụng một bản phân phối Python slim rút gọn và copy các thư viện đã được cài đặt từ Stage 1 sang. Việc này giúp loại bỏ hoàn toàn các build tools và file rác trung gian khỏi image cuối cùng.

---

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment
- **URL**: `https://day12ha-tang-cloudvadeployment-production-c103.up.railway.app`
- **Screenshot**: Đã đính kèm hình ảnh kiểm tra `/health` và `/ask` trong thư mục `screenshots/test-results.png`.

---

## Part 4: API Security

### Exercise 4.1-4.3: Test results
Dưới đây là các kết quả thử nghiệm bảo mật (local test):
1. **Test API Key Auth**:
   - Gửi request không có key: Nhận mã lỗi `401 Unauthorized` kèm thông điệp `"Invalid or missing API key"`.
   - Gửi request với API key chính xác: Nhận mã lỗi `200 OK` và dữ liệu phản hồi từ AI agent thành công.
2. **Test public deployment**:
   - Public endpoint `GET /health` trên Railway trả về `status: ok`, `environment: production`, và `checks.llm: mock`.
   - Public endpoint `POST /ask` với header `X-API-Key: test-key` trả về `200 OK` và response từ mock AI agent.
3. **Test Rate Limiting**:
   - Với cấu hình `RATE_LIMIT_PER_MINUTE=20`, khi gửi vượt quá 20 requests trong vòng dưới 1 phút, hệ thống trả về mã lỗi `429 Too Many Requests` kèm theo header `Retry-After`.

---

### Exercise 4.4: Cost guard implementation
**Giải pháp tiếp cận**:
Chúng ta sử dụng Redis để lưu trữ thông tin chi phí sử dụng hàng ngày của từng user (`cost:user_id:YYYY-MM-DD`). 
- Trước khi gọi LLM API, ứng dụng sẽ lấy giá trị chi phí hiện tại từ Redis. Nếu chi phí vượt quá `daily_budget_usd` (mặc định là $5.0), ứng dụng sẽ trả về lỗi `402 Payment Required` lập tức mà không gọi LLM để bảo vệ tài khoản.
- Sau khi nhận được câu trả lời từ LLM, chúng ta tính toán số lượng tokens (ước tính dựa trên độ dài câu hỏi và câu trả lời) nhân với đơn giá của model (ví dụ GPT-4o-mini: $0.15/1M input tokens, $0.60/1M output tokens), sau đó cộng dồn giá trị này vào Redis bằng lệnh `setex` với thời gian hết hạn (TTL) là 48 giờ để tự động dọn dẹp dữ liệu cũ.

---

## Part 5: Scaling & Reliability

### Exercise 5.1-5.5: Implementation notes
- **Health Checks**: Endpoint `/health` trả về liveness status `ok` cùng thời gian chạy (uptime) của service. Endpoint `/ready` thực hiện kiểm tra thực tế kết nối ping tới Redis cluster, nếu Redis offline sẽ trả về lỗi `503 Service Unavailable`, báo cho Load Balancer biết để tạm thời rút container này khỏi cụm định tuyến.
- **Graceful Shutdown**: Đăng ký lắng nghe signal `SIGTERM` (được gửi từ Docker/Railway khi scale down hoặc cập nhật phiên bản mới). Khi nhận signal, biến `_is_ready` chuyển sang `False` để ứng dụng ngừng tiếp nhận request mới qua endpoint `/ready`. Đồng thời, uvicorn chờ 30 giây (timeout) để toàn bộ các request in-flight hoàn thành xử lý rồi mới đóng kết nối Redis và thoát tiến trình.
- **Stateless Design**: Chúng ta hoàn toàn loại bỏ biến lưu trữ trạng thái dạng in-memory trong code python. Lịch sử chat được lưu tại Redis key `history:user_id` với cấu trúc JSON list. Khi ứng dụng scale lên 3 instances thông qua Nginx Load Balancer, các request tiếp theo của một user có thể được phân phối đến bất cứ instance nào nhưng vẫn đọc và ghi nhận chung một lịch sử trò chuyện được đồng bộ ở Redis.
