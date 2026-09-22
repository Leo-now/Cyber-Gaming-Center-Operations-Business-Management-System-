# HƯỚNG DẪN CODEBASE & CẨM NANG PHÁT TRIỂN (DEVELOPER GUIDE)

## HỆ THỐNG QUẢN LÝ VẬN HÀNH PHÒNG MÁY GAMING (CYBER GAMING CENTER)

Chào mừng bạn đến với mã nguồn Backend của dự án **Cyber Gaming Center — Operations & Business Management System**. Tài liệu này cung cấp cái nhìn tổng quan về công nghệ, kiến trúc hệ thống và quy chuẩn phát triển tính năng mới.

---

## 1. CÔNG NGHỆ & KIẾN TRÚC SỬ DỤNG

* **Kiến trúc chính**: Layered Architecture (`Controller → Service → Repository → Entity`), Spring Boot Monolith.
* **Backend**: Java, Spring Boot, Spring Data JPA/Hibernate, Spring Security (JWT).
* **Giao diện**: Spring MVC + Thymeleaf — dành cho Staff/Admin và Customer.
* **Database**: Microsoft SQL Server.
* **Real-time**: Spring WebSocket (STOMP) — sử dụng cho Chat và Notification.
* **AI**: Module gợi ý game, có cơ chế fallback khi dịch vụ AI gặp lỗi.
* **Tài liệu API**: Swagger / springdoc-openapi.

---

## 2. QUY ĐỊNH CẤU TRÚC LAYER & NGUYÊN TẮC TỔ CHỨC CODE (ĐẶC BIỆT QUAN TRỌNG)

Lập trình viên **bắt buộc** phải viết mã nguồn đúng layer và đúng thư mục tính năng, không tổ chức code lộn xộn.

### 2.1. Phân chia Layer & Trách nhiệm (Dev viết gì ở đâu?)

* **Tầng Entity (`entity/`)**:

  * ** Các Entity JPA ánh xạ với bảng trong `Database/schema.sql`.
  * *Quy tắc:* Không chứa business logic.

* **Tầng Repository (`repository/`)**:

  * *Viết gì ở đây:* Các interface kế thừa `JpaRepository`, chịu trách nhiệm truy vấn dữ liệu.
  * *Quy tắc:* Không xử lý business logic.

* **Tầng DTO (`dto/<feature>/`)**:

  * ** Các Request/Response DTO dùng để truyền dữ liệu giữa Controller và Service.
  * *Quy tắc:* DTO phải tách biệt khỏi Entity. Controller không được trả Entity trực tiếp ra ngoài.

* **Tầng Service (`service/<feature>/`)**:

  * ** Toàn bộ business logic của hệ thống như tính tiền phiên chơi, xử lý Order, Wallet và xác nhận thanh toán.
  * *Quy tắc:* Các quy tắc nghiệp vụ phải được kiểm tra và xử lý tại đây.

* **Tầng Controller (`controller/<feature>/`)**:

  * ** Tiếp nhận HTTP request, gọi Service và trả response.
  * *Quy tắc:* Controller phải giữ mỏng, không chứa business logic.

### 2.2. Quy tắc tổ chức Code theo thư mục tính năng (Feature Folders)

Để tránh mã nguồn bị lộn xộn, các DTO, Service và Controller **bắt buộc phải được phân cụm theo từng feature**.

**Cấu trúc đúng:**

```text
src/main/java/.../
├── dto/
│   ├── wallet/
│   │   ├── DepositRequestDto.java
│   │   └── WalletBalanceResponseDto.java
│   ├── session/
│   │   └── StartSessionRequestDto.java
│   └── chat/
│       └── ChatMessageDto.java
│
├── service/
│   ├── wallet/
│   ├── session/
│   └── chat/
│
└── controller/
    ├── wallet/
    ├── session/
    └── chat/
```

**CẤM TUYỆT ĐỐI:** tạo DTO, Service hoặc Controller trực tiếp dưới thư mục cha (`dto/`, `service/`, `controller/`) mà không nằm trong thư mục feature cụ thể.

---

## 3. QUY TẮC NGHIỆP VỤ CỐT LÕI (ĐỌC TRƯỚC KHI CODE)

Các quy tắc dưới đây được ràng buộc tại tầng Database và phải được tuân thủ khi viết Service.

| Quy tắc                     | Chi tiết                                                                                                                             |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| **Snapshot giá**            | `gaming_sessions.price_per_hour` và `order_details.unit_price` được chốt tại thời điểm tạo. Không join với giá hiện tại để tính lại. |
| **Tính tiền theo giây**     | `gaming_amount = ROUND(price_per_hour × duration_seconds / 3600, 0)`. Không làm tròn thời gian, chỉ làm tròn kết quả tiền cuối cùng. |
| **Xác nhận thanh toán**     | Deposit/Payment bắt đầu ở trạng thái `PENDING_CONFIRMATION` và chỉ được xử lý số dư sau khi Staff xác nhận.                          |
| **1 PC = 1 session ACTIVE** | Một PC chỉ được có một session `ACTIVE` tại cùng thời điểm. Database dùng filtered unique index để bảo đảm ràng buộc này.            |
| **Ví không âm**             | `wallets.balance` không được nhỏ hơn `0`. Service phải xử lý trường hợp không đủ số dư.                                              |

Chi tiết đầy đủ được quy định trong **Cyber Gaming Center SRS**, mục 5.

---

## 4. DEVELOPER GUIDE: QUY TRÌNH THÊM MỘT TÍNH NĂNG / API MỚI

Khi tạo một API chức năng mới, lập trình viên phải thực hiện theo đúng 5 bước:

### Bước 1: Entity

Tạo Entity trong:

```text
entity/
```

Entity phải ánh xạ đúng với bảng trong `Database/schema.sql`.

Không tự ý thay đổi tên bảng hoặc tên cột nếu chưa cập nhật Database và SRS.

### Bước 2: Repository

Tạo Repository trong:

```text
repository/
```

Repository kế thừa:

```java
JpaRepository<Entity, Long>
```

Repository chỉ chịu trách nhiệm truy vấn và thao tác dữ liệu.

### Bước 3: DTO

Tạo Request/Response DTO trong:

```text
dto/<feature>/
```

Mỗi DTO phải nằm trong một file riêng.

Ví dụ:

```text
dto/session/
├── StartSessionRequestDto.java
└── SessionResponseDto.java
```

### Bước 4: Service

Tạo Service trong:

```text
service/<feature>/
```

Service chịu trách nhiệm xử lý toàn bộ business logic.

Ví dụ:

* Tính tiền phiên chơi.
* Kiểm tra PC đang có session hay không.
* Kiểm tra số dư Wallet.
* Xử lý Order.
* Xác nhận Deposit/Payment.
* Xử lý trạng thái nghiệp vụ.

Sử dụng exception rõ ràng cho các lỗi nghiệp vụ, ví dụ:

```text
NotFoundException
InsufficientBalanceException
ConflictException
```

### Bước 5: Controller

Tạo Controller trong:

```text
controller/<feature>/
```

Controller chỉ tiếp nhận request → gọi Service → trả response.

Không đưa business logic vào Controller.

### Nếu cần thêm bảng hoặc cột mới

Phải cập nhật đồng bộ:

```text
Database/schema.sql
Database/seed.sql
docs/Cyber-Gaming-Center-SRS.docx
```

Không để Database, Backend và SRS lệch nhau.

---

## 5. CÁC LỆNH CHẠY DỰ ÁN THƯỜNG DÙNG

### Build toàn bộ project

```bash
mvn clean install
```

### Chạy ứng dụng

```bash
mvn spring-boot:run
```

### Khởi tạo Database

```bash
sqlcmd -S localhost -i Database\schema.sql
sqlcmd -S localhost -i Database\seed.sql
```

---

## 6. HƯỚNG DẪN CẤU HÌNH & SỬ DỤNG CÁC DỊCH VỤ TÍCH HỢP

### 6.1. WebSocket — Chat & Notification

* **Endpoint**: `/ws`
* **Protocol**: STOMP.
* Sử dụng cho:

  * Chat giữa Customer ↔ Staff.
  * Notification cho Staff khi có Deposit/Order request.
* Dữ liệu Chat và Notification vẫn được lưu trong Database.
* Nếu WebSocket bị mất kết nối tạm thời, dữ liệu không bị mất và có thể được tải lại khi Client kết nối lại.

### 6.2. AI Gợi ý Game

* Module nhận `input_context` từ Customer.
* AI trả về danh sách game được xếp hạng dựa trên danh mục `games`.
* **Bắt buộc có fallback**:

  * Nếu AI service lỗi hoặc timeout, hệ thống trả về danh sách game phổ biến.
  * Hệ thống không được để Customer không nhận được phản hồi chỉ vì AI service gặp lỗi.

### 6.3. VNPay (Mô phỏng)

* Hiện tại hệ thống chỉ mô phỏng VNPay thông qua channel `VNPAY`.
* Chưa tích hợp cổng thanh toán VNPay thật.
* Các giao dịch VNPAY vẫn phải trải qua bước Staff xác nhận.
* Việc tích hợp cổng thanh toán thật nằm ngoài phạm vi hiện tại.

---

## 7. TÀI LIỆU

* **Software Requirements Specification**: `docs/Cyber-Gaming-Center-SRS.docx`
* **Database Schema**: `Database/schema.sql`
* **Seed Data**: `Database/seed.sql`
