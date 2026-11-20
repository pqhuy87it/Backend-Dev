Khi chuyển từ Monolithic sang **Microservices**, một thao tác nghiệp vụ đơn giản của người dùng (như *Xem chi tiết sản phẩm* hay *Đặt hàng*) không còn nằm gọn trong một câu lệnh `SQL JOIN` hay một `Database Transaction` (`BEGIN ... COMMIT/ROLLBACK`) duy nhất. Dữ liệu lúc này bị phân mảnh ra nhiều service độc lập.

Bốn khái niệm bạn nêu chính là **bộ khung tư duy sống còn** để trả lời hai câu hỏi lớn nhất của kiến trúc phân tán:

1. **Khi ĐỌC dữ liệu (Read Path):** Làm sao gom dữ liệu từ nhiều nơi mà1 service phụ bị chết không kéo sập toàn bộ màn hình? (*Data Composition, Graceful Degradation, Fail-open vs. Fail-closed*).
2. **Khi GHI dữ liệu (Write Path):** Làm sao đảm bảo tính toàn vẹn dữ liệu qua nhiều services khi mạng chập chờn hoặc một bước ở giữa bị lỗi? (*Async Saga & Idempotency*).

---

## 1. Data Composition & Graceful Degradation (Tổng hợp dữ liệu & Suy giảm êm dịu)

### Data Composition là gì?

Vì mỗi Microservice sở hữu Database riêng, để hiển thị một màn hình **Chi tiết sản phẩm (Product Detail)** trên App, hệ thống phải thực hiện **Data Composition** (Tổng hợp dữ liệu) từ ít nhất 4 services khác nhau thông qua một tầng trung gian (thường là **API Gateway / BFF - Backend For Frontend** hoặc một Aggregator Service):

* **Product Service:** Tên sản phẩm, mô tả, hình ảnh.
* **Pricing/Promotion Service:** Giá gốc, giá flash-sale, mã giảm giá.
* **Inventory Service:** Số lượng tồn kho còn lại.
* **Review/Recommendation Service:** Danh sách bình luận và "Sản phẩm tương tự".

### Vấn đề toán học của sự phụ thuộc

Nếu một API tổng hợp dữ liệu từ 5 services độc lập, và mỗi service có độ sẵn sàng (Uptime SLA) rất cao là **99%**, thì xác suất để cả 5 services cùng sống tại một thời điểm chỉ còn là:


$$0.99^5 \approx 95.1\%$$


Nghĩa là cứ 20 lần người dùng mở màn hình sẽ có **1 lần bị lỗi 500 Internal Server Error** nếu bạn lập trình theo kiểu: *"Cứ 1 trong 5 thằng lỗi là quăng Exception báo lỗi toàn màn hình"*.

### Graceful Degradation (Suy giảm êm dịu)

**Graceful Degradation** là triết lý thiết kế mà ở đó: khi một phần của hệ thống gặp sự cố hoặc quá tải, hệ thống **không sập hoàn toàn (hard crash)** mà chủ động cắt bỏ các tính năng phụ hoặc trả về dữ liệu dự phòng (Fallback), miễn sao luồng nghiệp vụ cốt lõi của người dùng vẫn hoạt động được.

---

## 2. Fail-open vs. Fail-closed: Ra quyết định khi một mắt xích bị đứt

Để triển khai *Graceful Degradation* trong lúc *Data Composition*, Backend Engineer phải phân loại từng lệnh gọi sang service khác thuộc nhóm **Fail-open** hay **Fail-closed**.

### A. Fail-open (Hỏng phần phụ vẫn cho qua / chạy tiếp phần chính)

Khi gọi sang một **Non-critical Service** (dịch vụ phụ trợ) mà bị Timeout hoặc lỗi 5xx, hệ thống sẽ **bắt lỗi (catch exception), ghi log cảnh báo và trả về giá trị mặc định/cache** để luồng chính tiếp tục hoàn tất.

* **Ví dụ thực tế:**
* Đang tải trang Chi tiết sản phẩm mà **Review Service** hoặc **Recommendation Service** bị sập $\rightarrow$ BFF trả về mảng `reviews: []` hoặc ẩn luôn khối "Sản phẩm gợi ý" trên UI. Người dùng vẫn xem được giá và bấm **Mua hàng** bình thường.
* Hệ thống **Rate Limiter** (giới hạn số request bằng Redis) bị mất kết nối tới Redis $\rightarrow$ Thay vì chặn toàn bộ người dùng hợp lệ, API Gateway chọn **Fail-open**: tạm thời cho phép request đi qua vào Backend chính trong lúc chờ Redis hồi phục.


* **Chiến lược Fallback phổ biến cho Fail-open:**
1. **Stale Cache:** Trả về dữ liệu cũ đã lưu trong Redis/Local Memory trước đó.
2. **Default/Empty Value:** Trả về mảng rỗng `[]`, `null`, hoặc trạng thái mặc định.
3. **Silent Drop:** Bỏ qua tác vụ phụ (như ghi log tracking hành vi người dùng).



### B. Fail-closed (Hỏng là chặn đứng ngay lập tức)

Ngược lại, với các **Critical Service** liên quan trực tiếp đến **Bảo mật, Tiền bạc, Quyền truy cập hoặc Dữ liệu cốt lõi**, nếu lệnh gọi bị lỗi hoặc không xác minh được, hệ thống bắt buộc phải **chặn đứng toàn bộ request (ngắt luồng và trả lỗi ngay)**, tuyệt đối không được đoán mò hay dùng dữ liệu tạm.

* **Ví dụ thực tế:**
* **Authentication / Authorization Service** hoặc **Fraud Detection (Chống gian lận)** bị Timeout khi người dùng bấm rút tiền $\rightarrow$ Bắt buộc **Fail-closed** (từ chối giao dịch ngay), không bao giờ được phép "tạm cho qua vì service kiểm duyệt đang bận".
* Khi tạo đơn hàng mà **Pricing Service** hoặc **Inventory Service** không phản hồi $\rightarrow$ Không thể dùng "giá mặc định = 0đ" hay "tồn kho giả định" để tạo đơn. Phải hủy request và báo người dùng thử lại.



### Bảng so sánh và Ví dụ Code (TypeScript/Node.js)

| Tiêu chí | Fail-open (Mở khi lỗi) | Fail-closed (Đóng khi lỗi) |
| --- | --- | --- |
| **Mục tiêu ưu tiên** | **Availability** (Độ sẵn sàng & Trải nghiệm người dùng) | **Consistency & Security** (Tính đúng đắn, An toàn & Tiền bạc) |
| **Hành vi khi lỗi** | Nuốt lỗi (Catch), trả về Fallback/Cache, HTTP 200 OK | Ném lỗi (Throw), dừng luồng xử lý, trả HTTP 4xx/5xx |
| **Áp dụng cho** | Gợi ý sản phẩm, Đánh giá/Review, Banner quảng cáo, Analytics, Rate Limit | Xác thực (Auth), Thanh toán (Payment), Trừ kho (Inventory), Kiểm tra số dư |

```typescript
async function getProductDetailScreen(productId: string) {
  // Chạy song song bằng Promise.allSettled để 1 thằng chết không làm hỏng các thằng khác
  const [productRes, priceRes, reviewRes, recommendRes] = await Promise.allSettled([
    productService.getById(productId),         // CRITICAL (Fail-closed)
    pricingService.getPrice(productId),        // CRITICAL (Fail-closed)
    reviewService.getReviews(productId),       // NON-CRITICAL (Fail-open)
    recommendService.getRelated(productId),    // NON-CRITICAL (Fail-open)
  ]);

  // 1. Xử lý FAIL-CLOSED: Hỏng phần chính là chặn đứng ngay
  if (productRes.status === 'rejected' || priceRes.status === 'rejected') {
    throw new ServiceUnavailableException('Không thể tải thông tin sản phẩm lúc này.');
  }

  // 2. Xử lý FAIL-OPEN: Hỏng phần phụ thì dùng Fallback, phần chính vẫn chạy tiếp
  const reviews = reviewRes.status === 'fulfilled' 
    ? reviewRes.value 
    : { items: [], isDegraded: true }; // Báo cho Client biết để ẩn tab Review khéo léo

  const recommendations = recommendRes.status === 'fulfilled' 
    ? recommendRes.value 
    : [];

  return {
    product: productRes.value,
    price: priceRes.value,
    reviews,
    recommendations,
  };
}

```

---

## 3. Luồng Async Saga & Idempotency (Giao dịch phân tán & Tính lũy đẳng)

Khi người dùng bấm **"Thanh toán đơn hàng"**, hệ thống cần cập nhật dữ liệu ở 3 Database của 3 services khác nhau:

1. `Order Service`: Tạo đơn hàng ở trạng thái `PENDING`.
2. `Inventory Service`: Trừ số lượng tồn kho sản phẩm.
3. `Payment Service`: Trừ tiền trong ví/thẻ của khách.

Trong Microservices, người ta **tránh dùng Two-Phase Commit (2PC)** (khóa đồng thời cả 3 DB chờ nhau) vì nó cực kỳ chậm và tạo điểm nghẽn toàn cục. Thay vào đó, chuẩn mực hiện nay là **Async Saga Pattern** kết hợp chặt chẽ với **Idempotency**.

### A. Async Saga Pattern là gì?

**Saga** chia một giao dịch lớn xuyên nhiều services thành một chuỗi các **Local Transactions** (giao dịch cục bộ trên từng service), nối tiếp nhau qua **Message Broker** (như Kafka hoặc RabbitMQ).

Điểm cốt lõi của Saga: Vì không có lệnh `ROLLBACK` chung cho cả hệ thống, nếu bước thứ 3 (`Payment`) thất bại (do thẻ hết tiền), bạn không thể "quay ngược thời gian" ở tầng DB của bước 2 (`Inventory`). Thay vào đó, Saga bắt buộc bạn phải viết các **Compensating Transactions (Giao dịch bù trừ)** — tức là chạy thêm một hành động ngược lại để hoàn tác về mặt nghiệp vụ:

* **Chiều đi (Forward Flow):** `Tạo đơn (PENDING)` $\rightarrow$ `Giữ hàng (Reserve Stock)` $\rightarrow$ `Trừ tiền (Charge Card)` $\rightarrow$ `Xác nhận đơn (CONFIRMED)`.
* **Chiều lùi bù trừ khi lỗi (Compensation / Undo Flow):** Nếu `Trừ tiền` thất bại $\rightarrow$ Bắn sự kiện `PaymentFailed` $\rightarrow$ `Inventory Service` lắng nghe và chạy hàm **Nhả hàng lại kho (`Release Stock`)** $\rightarrow$ `Order Service` chuyển trạng thái đơn thành **`CANCELLED`**.

*(Lưu ý: Saga có 2 cách điều phối là **Choreography** — các service tự bắn và nghe sự kiện của nhau, hoặc **Orchestration** — có một nhạc trưởng trung tâm chỉ huy từng bước).*

### B. Tại sao Async Saga bắt buộc phải có Idempotency (Tính lũy đẳng)?

Trong môi trường mạng phân tán và Message Queue (Kafka/RabbitMQ), cơ chế truyền tin chuẩn là **At-least-once delivery** (Đảm bảo gửi *ít nhất một lần*).

Hãy tưởng tượng kịch bản sau:

1. `Order Service` gửi message yêu cầu `Payment Service` trừ `500.000đ` của khách.
2. `Payment Service` nhận message, **đã trừ `500.000đ` thành công trong DB**, nhưng đúng lúc gửi phản hồi `ACK` (xác nhận đã xử lý) về cho Kafka/Order Service thì **mạng bị rớt 1 giây**.
3. Vì không nhận được `ACK`, Message Broker tưởng rằng `Payment Service` chưa xử lý xong nên **tự động gửi lại (Retry) chính message đó lần thứ 2**.
4. Nếu `Payment Service` không có cơ chế phòng vệ, khách hàng sẽ bị **trừ tiền 2 lần (1.000.000đ)** cho cùng 1 đơn hàng! Tương tự, ở chiều Rollback, nếu message bù trừ `Release Stock` bị gửi lặp 2 lần, kho hàng sẽ bị **cộng dư thêm sản phẩm**.

**Idempotency (Tính lũy đẳng)** trong toán học và khoa học máy tính có nghĩa là: **Một thao tác dù được thực thi 1 lần hay $N$ lần với cùng một bộ tham số thì kết quả trạng thái hệ thống vẫn giống hệt như chỉ chạy 1 lần duy nhất:**


$$f(x) = f(f(x))$$

### Cách triển khai Idempotency thực tế trong Backend

Để biến một API hoặc Consumer thành Idempotent, ta sử dụng **Idempotency Key** (thường là `UUID` gắn liền với hành động, ví dụ `transaction_id` hoặc `order_id + step_name`):

1. **Tạo bảng `idempotency_records` (hoặc `processed_messages`) trong cùng Database của Service nhận:**
Bảng này có khóa chính hoặc `UNIQUE INDEX` trên cột `idempotency_key`, lưu kèm trạng thái (`PROCESSING`, `COMPLETED`) và kết quả phản hồi (`response_payload`).
2. **Gói chung vào 1 Local ACID Transaction:**
Khi `Payment Service` nhận yêu cầu trừ tiền kèm `idempotency_key = "pay_order_999"`:
* Mở DB Transaction:
* Thử `INSERT` mã `"pay_order_999"` vào bảng `idempotency_records`.
* **Nếu đã tồn tại:** Bỏ qua toàn bộ logic trừ tiền, lấy luôn kết quả đã lưu lần trước trả về `ACK` thành công.
* **Nếu chưa tồn tại:** Thực hiện trừ tiền trong tài khoản $\rightarrow$ Lưu kết quả vào bảng `idempotency_records` $\rightarrow$ `COMMIT` cả hai cùng lúc.



---

## 4. Mô phỏng tương tác: Luồng Async Saga, Bù trừ (Compensation) & Idempotency

Bạn có thể sử dụng trình mô phỏng dưới đây để chuyển đổi giữa các kịch bản (**Thành công**, **Thất bại ở bước Thanh toán cần Rollback**, và **Mạng lỗi gây Retry trùng lặp**) và bấm **Bước tiếp theo** để quan sát cách từng Microservice xử lý trạng thái và kiểm tra `Idempotency Key`:

---

**Tóm lại trong một câu để dễ nhớ khi thiết kế hệ thống:**

* Khi **ĐỌC (Data Composition)**: Phân loại rõ phần phụ (**Fail-open** $\rightarrow$ ẩn đi hoặc dùng cache để **Graceful Degradation**) và phần cốt lõi (**Fail-closed** $\rightarrow$ lỗi là chặn đứng để bảo vệ an toàn).
* Khi **GHI (Distributed Transaction)**: Chia nhỏ thành chuỗi sự kiện (**Async Saga**), viết sẵn hàm hoàn tác (**Compensation**) cho mọi bước có thể lỗi, và luôn khóa đầu vào bằng **Idempotency Key** để miễn nhiễm với việc gửi lặp message.
