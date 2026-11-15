Trong kiến trúc **Monolithic** (nguyên khối), các module như Auth, Order, và Payment nằm chung trong một tiến trình (process) và gọi nhau trực tiếp qua hàm trong bộ nhớ (in-memory function call) với tốc độ tính bằng nano-giây. Khi tách thành **Microservices**, mỗi service chạy trên một container hoặc server riêng biệt. Lúc này, việc gọi hàm biến thành **giao tiếp qua mạng (Network I/O)** — đòi hỏi cơ chế đóng gói dữ liệu (Serialization) và giao thức truyền tải chuẩn hóa.

Hai phương thức giao tiếp đồng bộ (Synchronous) phổ biến nhất hiện nay là **HTTP/REST** và **gRPC**.

---

## 1. Giao tiếp qua HTTP/REST (Representational State Transfer)

HTTP/REST là chuẩn mực giao tiếp dựa trên tài nguyên (Resource-oriented), sử dụng giao thức HTTP/1.1 (hoặc HTTP/2) và định dạng văn bản **JSON**. Nếu bạn đã quen làm việc với API từ phía Client/Mobile, giao tiếp giữa các Microservices bằng REST hoạt động hoàn toàn tương tự: Service A đóng vai trò là HTTP Client gọi sang endpoint của Service B.

**Cơ chế hoạt động**

* **Định dạng dữ liệu (Payload):** Text-based (chủ yếu là JSON). Dữ liệu dễ đọc bằng mắt thường (human-readable) nhưng kích thước gói tin lớn và tốn CPU để parse/serialize chuỗi văn bản.
* **Giao thức nền:** Thường chạy trên HTTP/1.1. Dù có Keep-Alive, HTTP/1.1 vẫn dễ gặp hiện tượng *Head-of-Line Blocking* (các request trên cùng một kết nối TCP phải chờ nhau tuần tự).
* **Hợp đồng giao tiếp (Contract):** Lỏng lẻo (Loose coupling). Thường được tài liệu hóa qua **OpenAPI / Swagger**, nhưng không bắt buộc ở tầng biên dịch (compile-time).

**Khi nào dùng trong Microservices?**

* **API Gateway / BFF (Backend For Frontend):** Giao tiếp từ hệ thống ra bên ngoài cho Mobile App, Web Browser hoặc đối tác thứ ba (Webhook).
* **Service đơn giản, lưu lượng vừa phải:** Khi tốc độ phát triển và khả năng debug nhanh bằng Postman/cURL quan trọng hơn việc tối ưu vài mili-giây độ trễ.

---

## 2. Giao tiếp qua gRPC (Google Remote Procedure Call)

gRPC là framework mã nguồn mở do Google phát triển, thiết kế theo tư tưởng **gọi hàm từ xa (RPC)**. Thay vì tư duy theo kiểu "gửi HTTP POST tới đường dẫn `/orders`", gRPC cho phép bạn gọi trực tiếp `orderService.CreateOrder(request)` như thể hàm đó nằm ngay trong source code của mình.

**3 trụ cột tạo nên tốc độ của gRPC:**

1. **Protocol Buffers (Protobuf) — Định dạng nhị phân (Binary Serialization):**
Thay vì gửi chuỗi JSON lặp lại tên các trường (như `"user_id": 123`), Protobuf mã hóa dữ liệu thành mảng byte nhị phân dựa trên đánh số thứ tự trường (`1: 123`). Kích thước payload nhỏ hơn JSON từ **3 đến 5 lần**, và tốc độ serialize/deserialize nhanh hơn gấp nhiều lần.
2. **Chạy trên nền HTTP/2:**
* **Multiplexing (Đa luồng):** Gửi hàng chục request và nhận response đồng thời trên **duy nhất một kết nối TCP** mà không bị chặn nhau.
* **Header Compression (HPACK):** Nén các HTTP Header lặp đi lặp lại, giảm đáng kể băng thông nội bộ.


3. **Strict Contract & Code Generation (Tạo code tự động):**
Mọi giao tiếp bắt đầu bằng file định nghĩa `.proto`. Từ file này, trình biên dịch `protoc` tự động sinh ra (generate) **Server Skeleton** cho bên nhận và **Client Stub** cho bên gọi ở bất kỳ ngôn ngữ nào (Go, Node.js, Java, Python, Dart, Swift...).

**Ví dụ quy trình với file `order.proto`:**

```protobuf
syntax = "proto3";
package order;

// 1. Định nghĩa Service (như một Interface)
service OrderService {
  rpc CreateOrder (CreateOrderRequest) returns (CreateOrderResponse);
}

// 2. Định nghĩa cấu trúc dữ liệu (kèm số thứ tự Tag dùng để mã hóa binary)
message CreateOrderRequest {
  string user_id = 1;
  repeated string item_ids = 2; // tương đương mảng List<String>
  double total_amount = 3;
}

message CreateOrderResponse {
  string order_id = 1;
  string status = 2;
}

```

Khi chạy lệnh generate code, nếu Service A (viết bằng Go) muốn gọi Service B (viết bằng Java), dev ở Service A chỉ cần import bộ Stub đã sinh ra và gọi hàm với kiểu dữ liệu chặt chẽ (Type-safe) ngay lúc code, loại bỏ hoàn toàn lỗi sai tên trường JSON.

**4 kiểu giao tiếp (Streaming) của gRPC:**
Nhờ HTTP/2, gRPC không chỉ có gửi-nhận 1-1 mà hỗ trợ 4 chế độ:

* **Unary RPC:** Client gửi 1 request, nhận về 1 response (giống hệt gọi REST thông thường).
* **Server Streaming:** Client gửi 1 request, Server mở luồng trả về liên tục nhiều gói dữ liệu (VD: theo dõi trạng thái xử lý lô hàng theo thời gian thực).
* **Client Streaming:** Client đẩy liên tục một luồng dữ liệu lên, kết thúc Server trả về 1 kết quả (VD: upload file log lớn hoặc dữ liệu cảm biến IoT).
* **Bidirectional Streaming:** Hai bên mở kênh truyền gửi/nhận dữ liệu độc lập, song song liên tục (VD: hệ thống Chat nội bộ, đồng bộ trạng thái real-time).

---

## 3. Bảng so sánh trực diện HTTP/REST và gRPC

| Tiêu chí | HTTP / REST | gRPC |
| --- | --- | --- |
| **Định dạng dữ liệu** | JSON / Text (nặng, dễ đọc) | Protobuf / Binary (siêu nhẹ, không đọc được bằng mắt) |
| **Giao thức truyền tải** | HTTP/1.1 (hoặc HTTP/2) | Bắt buộc HTTP/2 |
| **Tốc độ & Băng thông** | Trung bình, tốn CPU parse chuỗi | Rất nhanh (nhanh hơn 5–10 lần ở tải cao), tiết kiệm CPU |
| **Kiểu kết nối** | Request – Response (1-1) | Unary, Server/Client/Bidirectional Streaming |
| **Ràng buộc kiểu (Type Safety)** | Lỏng lẻo, kiểm tra lúc Runtime | Chặt chẽ (Strict), kiểm tra ngay lúc Compile-time |
| **Tạo code (Codegen)** | Cần công cụ ngoài (OpenAPI Generator) | Tích hợp sẵn trong cốt lõi (`protoc`) |
| **Hỗ trợ Trình duyệt (Browser)** | Hỗ trợ tự nhiên 100% | Kém (cần proxy chuyển đổi qua gRPC-Web) |

---

## 4. Kiến trúc thực tế: Kết hợp cả hai như thế nào?

Trong các hệ thống Production hiện đại, người ta **không chọn 1 bỏ 1**, mà kết hợp cả hai theo mô hình **Bắc - Nam (North-South)** và **Đông - Tây (East-West)**:

```
[ Mobile / Web Client ]
          │
          ▼  (Giao tiếp North-South: HTTP/REST hoặc GraphQL - JSON)
┌───────────────────┐
│    API Gateway    │  (Chuyển đổi REST sang gRPC, xác thực JWT, Rate Limit)
└─────────┬─────────┘
          │
          ├──────────────────────────┐ (Giao tiếp East-West nội bộ: gRPC - Protobuf)
          ▼                          ▼
┌───────────────────┐      ┌───────────────────┐
│   Order Service   │─────▶│  Payment Service  │
└───────────────────┘ gRPC └───────────────────┘

```

* **Traffic North-South (Từ Client vào hệ thống):** Dùng **HTTP/REST (hoặc GraphQL)**. Lý do: Mobile app, Web browser, và các công cụ kiểm thử mạng (như Charles Proxy, Postman) làm việc tốt nhất với JSON. Việc đổi version API với Client bên ngoài cũng linh hoạt hơn.
* **Traffic East-West (Giữa các Microservices nội bộ):** Dùng **gRPC**. Một request từ người dùng vào API Gateway có thể kích hoạt chuỗi 5–10 lệnh gọi chéo giữa các service bên trong (Auth -> User -> Inventory -> Order -> Payment). Dùng gRPC ở tầng này giúp giảm tổng độ trễ (latency) xuống mức tối thiểu và đảm bảo tính toàn vẹn kiểu dữ liệu giữa các team backend khác nhau.

---

## 5. Lưu ý quan trọng khi thiết kế giao tiếp HTTP/gRPC

Vì cả HTTP và gRPC đều là **giao tiếp đồng bộ (Synchronous)** — tức là bên gọi phải chờ bên nhận trả lời — khi bắt đầu xây dựng backend, bạn sẽ sớm đối mặt với 3 bài toán cốt lõi sau:

* **Cascading Failure (Lỗi dây chuyền) & Timeout:** Nếu Service A gọi Service B, B gọi C, mà C bị treo, toàn bộ kết nối ở A và B sẽ bị nghẽn theo. **Nguyên tắc sống còn:** Luôn luôn cấu hình `Timeout` (hoặc `Deadline` trong gRPC) cho mọi lệnh gọi mạng, kết hợp với **Circuit Breaker** (cầu chì tự ngắt khi service đích chết).
* **Service Discovery & Load Balancing:** Với gRPC, vì HTTP/2 giữ một kết nối TCP bền vững (long-lived connection), nếu bạn dùng Load Balancer truyền thống ở tầng TCP (L4), mọi request sẽ bị dồn vào đúng 1 instance duy nhất. gRPC đòi hỏi Load Balancing ở tầng Application (L7 - như Envoy Proxy) hoặc Client-side Load Balancing.
* **Khi nào KHÔNG nên dùng cả HTTP lẫn gRPC?** Khi tác vụ không cần phản hồi ngay lập tức (ví dụ: gửi email xác nhận, cộng điểm thưởng sau khi mua hàng) hoặc cần chịu tải đột biến. Lúc đó, hệ thống sẽ chuyển sang **Giao tiếp bất đồng bộ (Asynchronous)** qua Message Broker như **Kafka** hoặc **RabbitMQ**.
