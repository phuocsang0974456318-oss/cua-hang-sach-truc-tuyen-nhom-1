# Thiết kế Chi tiết RESTful API - Hệ thống Cửa hàng Sách Trực tuyến (Nhóm 1)

> Tài liệu kỹ thuật chi tiết dành cho Backend Developer và Frontend Developer.
> Phiên bản API: `v1.0.0` | Base URL: `http://localhost:3000/api/v1`

---

## 1. Quy ước kỹ thuật chung

### 1.1. Protocol & Headers
- **Format**: JSON (`Content-Type: application/json; charset=utf-8`).
- **Authentication**: Session-cookie (`connect.sid`) được cấp tự động qua `Set-Cookie` khi đăng nhập thành công.
- **CORS & Credentials**: Client phải gửi request với `withCredentials: true` (Axios) hoặc `credentials: 'include'` (Fetch API).

### 1.2. Chuẩn hóa Phân trang (Pagination Standard)
Mọi API trả về danh sách có phân trang (`GET /books`, `GET /orders`, `GET /admin/users`, `GET /admin/orders`) **bắt buộc** dùng chung một định dạng Response:

```json
{
  "items": [],
  "pagination": {
    "totalItems": 120,
    "totalPages": 10,
    "currentPage": 1,
    "limit": 12,
    "hasNextPage": true,
    "hasPrevPage": false
  }
}
```

### 1.3. Chuẩn hóa Cấu trúc Lỗi (Error Handling Standard)
Tất cả mã lỗi `4xx` và `5xx` đều tuân theo format chuẩn:

```json
{
  "statusCode": 400,
  "message": "Dữ liệu đầu vào không hợp lệ",
  "error": "Bad Request",
  "errors": [
    {
      "field": "email",
      "message": "Email không đúng định dạng"
    }
  ],
  "timestamp": "2026-11-07T14:30:00.000Z"
}
```

---

## 2. Danh mục API Endpoints

| STT | HTTP Method | Endpoint | Tên chức năng | Phân quyền |
|---|---|---|---|---|
| **AUTH** | | | | |
| 1 | `POST` | `/auth/register` | Đăng ký tài khoản | Công khai |
| 2 | `POST` | `/auth/login` | Đăng nhập (tái tạo session + gộp cart) | Công khai |
| 3 | `POST` | `/auth/logout` | Đăng xuất | Đăng nhập |
| 4 | `GET` | `/auth/me` | Lấy thông tin tài khoản hiện tại | Đăng nhập |
| **BOOKS & REVIEWS** | | | | |
| 5 | `GET` | `/books` | Lấy danh sách sách (lọc/tìm kiếm/sắp xếp/phân trang) | Công khai |
| 6 | `POST` | `/books` | Thêm sách mới | `MANAGER`, `ADMIN` |
| 7 | `GET` | `/books/{id}` | Lấy chi tiết 1 cuốn sách | Công khai |
| 8 | `PUT` | `/books/{id}` | Cập nhật thông tin sách | `MANAGER`, `ADMIN` |
| 9 | `DELETE` | `/books/{id}` | Xóa sách | `ADMIN` |
| 10 | `GET` | `/books/{id}/reviews` | Lấy danh sách đánh giá của sách | Công khai |
| 11 | `POST` | `/books/{id}/reviews` | Tạo đánh giá sách | Đăng nhập (Đã mua thành công) |
| **CART** | | | | |
| 12 | `GET` | `/cart` | Xem chi tiết giỏ hàng hiện tại | Công khai |
| 13 | `DELETE` | `/cart` | Xóa sạch giỏ hàng | Công khai |
| 14 | `POST` | `/cart/items` | Thêm sách vào giỏ hàng | Công khai |
| 15 | `PATCH` | `/cart/items/{bookId}` | Thay đổi số lượng sách trong giỏ | Công khai |
| 16 | `DELETE` | `/cart/items/{bookId}` | Xóa 1 cuốn sách khỏi giỏ | Công khai |
| **ORDERS** | | | | |
| 17 | `GET` | `/orders` | Danh sách đơn hàng cá nhân | Đăng nhập |
| 18 | `POST` | `/orders` | Tạo đơn hàng (Checkout) | Đăng nhập |
| 19 | `GET` | `/orders/{id}` | Chi tiết 1 đơn hàng cá nhân | Đăng nhập |
| 20 | `POST` | `/orders/{id}/cancel` | Hủy đơn hàng | Đăng nhập |
| **ADMIN** | | | | |
| 21 | `GET` | `/admin/users` | Quản lý danh sách người dùng | `ADMIN` |
| 22 | `PATCH` | `/admin/users/{id}` | Cập nhật vai trò / Trạng thái khóa tài khoản | `ADMIN` |
| 23 | `GET` | `/admin/orders` | Quản lý tất cả đơn hàng | `MANAGER`, `ADMIN` |
| 24 | `PATCH` | `/admin/orders/{id}/status` | Cập nhật trạng thái đơn hàng | `MANAGER`, `ADMIN` |

---

## 3. Chi tiết API Endpoints & Request/Response Specification

### 3.1. Phân nhóm Authentication

#### 1. `POST /auth/register` - Đăng ký tài khoản
- **Quyền**: Công khai
- **Request Body**:
```json
{
  "name": "Nguyễn Văn A",
  "email": "nguyenvana@example.com",
  "password": "Password123!"
}
```
- **Ràng buộc dữ liệu**:
  - `name`: string, bắt buộc, length 2 - 100 ký tự.
  - `email`: string, bắt buộc, format email hợp lệ.
  - `password`: string, bắt buộc, độ dài từ 6 - 50 ký tự.
- **Responses**:
  - `201 Created`:
```json
{
  "id": "usr_6581f29b4a12",
  "name": "Nguyễn Văn A",
  "email": "nguyenvana@example.com",
  "role": "USER",
  "active": true,
  "createdAt": "2026-11-07T14:30:00.000Z"
}
```
  - `400 Bad Request`: Validation lỗi.
  - `409 Conflict`: Email đã tồn tại trên hệ thống (`"message": "Email trùng lặp."`).

---

#### 2. `POST /auth/login` - Đăng nhập
- **Quyền**: Công khai
- **Logic Backend bắt buộc**:
  1. Xác thực `email` & `password`.
  2. Kiểm tra `active === true`. Nếu `false` -> Trả về `403 Forbidden`.
  3. **Cart Merge Logic**: Kiểm tra giỏ hàng tạm thời trong session hiện tại (`guestCart`). Nếu có item, cộng dồn vào giỏ hàng của User trong DB/Session.
  4. Thực hiện `req.session.regenerate()` để phòng chống Session Fixation Attack.
  5. Trả cookie `connect.sid` chứa Session ID mới.
- **Request Body**:
```json
{
  "email": "nguyenvana@example.com",
  "password": "Password123!"
}
```
- **Responses**:
  - `200 OK`: Trả thông tin User đăng nhập thành công.
```json
{
  "id": "usr_6581f29b4a12",
  "name": "Nguyễn Văn A",
  "email": "nguyenvana@example.com",
  "role": "USER",
  "active": true,
  "createdAt": "2026-11-07T14:30:00.000Z"
}
```
  - `401 Unauthorized`: Sai email hoặc mật khẩu.
  - `403 Forbidden`: Tài khoản bị khóa (`"message": "Tài khoản của bạn đã bị khóa."`).

---

#### 3. `POST /auth/logout` - Đăng xuất
- **Quyền**: Người dùng đã đăng nhập.
- **Logic Backend**: Hủy Session trên Server (`req.session.destroy()`) và xóa cookie phía Client.
- **Responses**:
  - `204 No Content`: Hủy session thành công.
  - `401 Unauthorized`: Chưa đăng nhập.

---

#### 4. `GET /auth/me` - Lấy thông tin người dùng đang đăng nhập
- **Quyền**: Người dùng đã đăng nhập.
- **Responses**:
  - `200 OK`:
```json
{
  "id": "usr_6581f29b4a12",
  "name": "Nguyễn Văn A",
  "email": "nguyenvana@example.com",
  "role": "USER",
  "active": true,
  "createdAt": "2026-11-07T14:30:00.000Z"
}
```
  - `401 Unauthorized`: Chưa đăng nhập.

---

### 3.2. Phân nhóm Sách và Đánh giá (Books & Reviews)

#### 5. `GET /books` - Lấy danh sách sách
- **Quyền**: Công khai
- **Query Parameters**:
  - `q` *(optional, string)*: Tìm kiếm tương đối theo tiêu đề sách hoặc tên tác giả.
  - `category` *(optional, string)*: Lọc chính xác theo danh mục.
  - `minPrice` *(optional, integer)*: Giá tối thiểu (VND).
  - `maxPrice` *(optional, integer)*: Giá tối đa (VND).
  - `sort` *(optional, string)*: `newest` (mặc định), `price_asc`, `price_desc`, `title`.
  - `page` *(optional, integer)*: Trang hiện tại (min = 1, default = 1).
  - `limit` *(optional, integer)*: Số item/trang (min = 1, max = 50, default = 12).
- **Responses**:
  - `200 OK`:
```json
{
  "items": [
    {
      "id": "bk_101",
      "title": "Lập trình Node.js Chuyên nghiệp",
      "author": "Nguyễn Văn B",
      "price": 180000,
      "category": "Lập trình",
      "image": "https://cdn.bookstore.com/covers/nodejs.jpg",
      "description": "Cuốn sách chi tiết về xây dựng REST API...",
      "stock": 45,
      "ratingAverage": 4.8,
      "reviewCount": 12,
      "active": true,
      "createdAt": "2026-10-01T08:00:00.000Z"
    }
  ],
  "pagination": {
    "totalItems": 1,
    "totalPages": 1,
    "currentPage": 1,
    "limit": 12,
    "hasNextPage": false,
    "hasPrevPage": false
  }
}
```

---

#### 6. `POST /books` - Thêm sách mới
- **Quyền**: `MANAGER`, `ADMIN`
- **Request Body**:
```json
{
  "title": "Clean Code - Mã Sạch",
  "author": "Robert C. Martin",
  "price": 215000,
  "category": "Công nghệ thông tin",
  "image": "https://cdn.bookstore.com/covers/cleancode.jpg",
  "description": "Hướng dẫn viết mã sạch và bảo trì hệ thống.",
  "stock": 100,
  "active": true
}
```
- **Responses**:
  - `201 Created`: Trả về thông tin cuốn sách vừa tạo thành công.
  - `400 Bad Request`: Lỗi thiếu các trường bắt buộc (`title`, `author`, `price`, `stock`).
  - `401 Unauthorized` / `403 Forbidden`: Quyền không hợp lệ.

---

#### 7. `GET /books/{id}` - Chi tiết sách
- **Quyền**: Công khai
- **Path Parameters**: `id` (string) - ID cuốn sách.
- **Responses**:
  - `200 OK`: Trả về thông tin chi tiết cuốn sách kèm điểm đánh giá trung bình.
  - `404 Not Found`: Không tìm thấy sách với ID tương ứng.

---

#### 8. `PUT /books/{id}` - Cập nhật thông tin sách
- **Quyền**: `MANAGER`, `ADMIN`
- **Request Body**: Tương tự `POST /books`.
- **Responses**:
  - `200 OK`: Trả về thông tin cuốn sách đã được cập nhật.
  - `400 Bad Request` / `404 Not Found`.

---

#### 9. `DELETE /books/{id}` - Xóa sách
- **Quyền**: `ADMIN`
- **Logic Backend**: Khuyến nghị dùng **Soft Delete** (chuyển `active = false`) thay vì xóa vật lý khỏi DB để tránh hỏng dữ liệu các đơn hàng lịch sử.
- **Responses**:
  - `204 No Content`: Xóa thành công.
  - `404 Not Found`: Sách không tồn tại.

---

#### 10. `GET /books/{id}/reviews` - Lấy danh sách đánh giá
- **Quyền**: Công khai
- **Responses**:
  - `200 OK`:
```json
[
  {
    "id": "rev_501",
    "bookId": "bk_101",
    "userId": "usr_6581f29b4a12",
    "userName": "Nguyễn Văn A",
    "rating": 5,
    "comment": "Sách trình bày rất dễ hiểu, ví dụ chuẩn xác!",
    "createdAt": "2026-11-05T09:15:00.000Z"
  }
]
```

---

#### 11. `POST /books/{id}/reviews` - Tạo đánh giá cho sách
- **Quyền**: Đăng nhập
- **Logic Ràng buộc Nghiệp vụ**:
  Backend phải query bảng `Orders` kiểm tra: `Order.userId === currentUserId` AND `Order.items` chứa `bookId` AND `Order.status === 'COMPLETED'`. Nếu không đạt, từ chối trả về `403 Forbidden`.
- **Request Body**:
```json
{
  "rating": 5,
  "comment": "Sách rất hay và giao hàng cực nhanh!"
}
```
- **Responses**:
  - `201 Created`: Trả về Review vừa khởi tạo.
  - `403 Forbidden`: Người dùng chưa từng mua sản phẩm này hoặc đơn hàng chưa hoàn thành.
  - `409 Conflict`: Người dùng đã gửi đánh giá cho cuốn sách này trước đó.

---

### 3.3. Phân nhóm Giỏ hàng (Cart)

*Giỏ hàng được quản lý dựa trên Session cookie. Khách vãng lai và User đã đăng nhập đều có thể thao tác.*

#### 12. `GET /cart` - Lấy thông tin giỏ hàng
- **Quyền**: Công khai
- **Responses**:
  - `200 OK`:
```json
{
  "items": [
    {
      "book": {
        "id": "bk_101",
        "title": "Lập trình Node.js Chuyên nghiệp",
        "price": 180000,
        "image": "https://cdn.bookstore.com/covers/nodejs.jpg",
        "stock": 45
      },
      "quantity": 2,
      "subtotal": 360000
    }
  ],
  "totalQuantity": 2,
  "totalAmount": 360000
}
```

---

#### 13. `DELETE /cart` - Xóa toàn bộ giỏ hàng
- **Quyền**: Công khai
- **Responses**:
  - `200 OK`: Trả giỏ hàng về trạng thái rỗng (`items: []`, `totalQuantity: 0`, `totalAmount: 0`).

---

#### 14. `POST /cart/items` - Thêm sản phẩm vào giỏ hàng
- **Quyền**: Công khai
- **Logic Backend**:
  - Kiểm tra `stock` trong DB. Nếu `quantityToAdd + item.quantityInCart > book.stock` -> Trả về lỗi `409 Conflict`.
  - Nếu sản phẩm đã tồn tại trong giỏ hàng -> Cộng dồn `quantity`.
- **Request Body**:
```json
{
  "bookId": "bk_101",
  "quantity": 1
}
```
- **Responses**:
  - `200 OK`: Trả về object `Cart` đã cập nhật.
  - `404 Not Found`: `bookId` không hợp lệ.
  - `409 Conflict`: Khống chế vượt quá tồn kho.

---

#### 15. `PATCH /cart/items/{bookId}` - Cập nhật số lượng item
- **Quyền**: Công khai
- **Request Body**:
```json
{
  "quantity": 3
}
```
*Lưu ý: Nếu `quantity === 0`, tự động loại bỏ sản phẩm khỏi giỏ hàng.*
- **Responses**:
  - `200 OK`: Trả về object `Cart` mới nhất.
  - `409 Conflict`: Vượt tồn kho hiện có.

---

#### 16. `DELETE /cart/items/{bookId}` - Xóa 1 sản phẩm khỏi giỏ
- **Quyền**: Công khai
- **Responses**:
  - `200 OK`: Trả về object `Cart` sau khi xóa item.

---

### 3.4. Phân nhóm Đặt hàng (Orders)

#### 17. `GET /orders` - Lịch sử đơn hàng cá nhân
- **Quyền**: Đăng nhập
- **Query Parameters**: `status` (PENDING, CONFIRMED, SHIPPING, COMPLETED, CANCELLED), `page`, `limit`.
- **Responses**:
  - `200 OK`:
```json
{
  "items": [
    {
      "id": "ord_9901",
      "totalAmount": 360000,
      "status": "COMPLETED",
      "paymentMethod": "COD",
      "createdAt": "2026-11-01T10:00:00.000Z",
      "itemCount": 1
    }
  ],
  "pagination": {
    "totalItems": 1,
    "totalPages": 1,
    "currentPage": 1,
    "limit": 10,
    "hasNextPage": false,
    "hasPrevPage": false
  }
}
```

---

#### 18. `POST /orders` - Tạo đơn hàng (Checkout)
- **Quyền**: Đăng nhập
- **Quy trình xử lý Transaction Backend (BẮT BUỘC)**:
  1. Mở Database Transaction (`BEGIN TRANSACTION`).
  2. Lấy dữ liệu giỏ hàng từ Session (`req.session.cart`). Nếu giỏ rỗng -> Abort, trả lỗi `400`.
  3. Khóa các dòng dữ liệu sách trong DB liên quan bằng `FOR UPDATE` (Row Locking) để chống tranh chấp.
  4. Duyệt qua từng item: So sánh `cartItem.quantity` với `book.stock`.
     - Nếu thiếu stock -> Rollback Transaction, trả lỗi `409 Conflict`.
  5. Trừ số lượng tồn kho `book.stock = book.stock - cartItem.quantity`.
  6. Khởi tạo bản ghi `Order` và các bản ghi `OrderItem` lưu snapshot giá tại thời điểm mua.
  7. Làm trống giỏ hàng Session (`req.session.cart = null`).
  8. Commit Transaction (`COMMIT`).
- **Request Body**:
```json
{
  "fullName": "Nguyễn Văn A",
  "phone": "0901234567",
  "address": "Số 12 Nguyễn Huệ, Phường Bến Nghé, Quận 1, TP. Hồ Chí Minh",
  "paymentMethod": "COD",
  "note": "Giao vào giờ hành chính"
}
```
- **Responses**:
  - `201 Created`:
```json
{
  "id": "ord_9901",
  "user": "usr_6581f29b4a12",
  "items": [
    {
      "bookId": "bk_101",
      "title": "Lập trình Node.js Chuyên nghiệp",
      "price": 180000,
      "quantity": 2,
      "subtotal": 360000
    }
  ],
  "totalAmount": 360000,
  "status": "PENDING",
  "paymentMethod": "COD",
  "fullName": "Nguyễn Văn A",
  "phone": "0901234567",
  "address": "Số 12 Nguyễn Huệ, Phường Bến Nghé, Quận 1, TP. Hồ Chí Minh",
  "note": "Giao vào giờ hành chính",
  "createdAt": "2026-11-07T14:35:00.000Z"
}
```
  - `400 Bad Request`: Giỏ hàng trống hoặc thông tin nhận hàng thiếu.
  - `409 Conflict`: Sách bị hết hàng trong lúc thao tác thanh toán (`"message": "Sách 'Lập trình Node.js' chỉ còn 1 cuốn trong kho."`).

---

#### 19. `GET /orders/{id}` - Xem chi tiết 1 đơn hàng
- **Quyền**: Đăng nhập (Chủ sở hữu đơn hàng hoặc Admin/Manager).
- **Responses**:
  - `200 OK`: Trả về object chi tiết `Order` tương tự như Response của API 18.
  - `403 Forbidden`: Truy cập đơn hàng của người dùng khác.
  - `404 Not Found`: Không tìm thấy mã đơn.

---

#### 20. `POST /orders/{id}/cancel` - Hủy đơn hàng
- **Quyền**: Đăng nhập (Chủ đơn hàng).
- **Logic Backend bắt buộc**:
  1. Kiểm tra đơn hàng thuộc về `currentUserId`.
  2. Trạng thái đơn phải đang ở `PENDING`. Nếu đã chuyển `CONFIRMED`, `SHIPPING`, `COMPLETED` -> Từ chối hủy, trả về `409 Conflict`.
  3. Mở Transaction: Cập nhật `status = 'CANCELLED'`, hoàn trả số lượng kho `book.stock = book.stock + orderItem.quantity`.
- **Responses**:
  - `200 OK`: Hủy đơn thành công, trả về trạng thái đơn mới.
  - `409 Conflict`: Đơn đã được xác nhận/đang giao, không thể hủy.

---

### 3.5. Phân nhóm Quản trị (Admin & Manager)

#### 21. `GET /admin/users` - Quản lý danh sách người dùng
- **Quyền**: `ADMIN`
- **Query Parameters**: `role` (USER, MANAGER, ADMIN), `active` (boolean), `page`, `limit`.
- **Responses**:
  - `200 OK`:
```json
{
  "items": [
    {
      "id": "usr_6581f29b4a12",
      "name": "Nguyễn Văn A",
      "email": "nguyenvana@example.com",
      "role": "USER",
      "active": true,
      "createdAt": "2026-10-15T03:20:00.000Z"
    }
  ],
  "pagination": {
    "totalItems": 450,
    "totalPages": 45,
    "currentPage": 1,
    "limit": 10,
    "hasNextPage": true,
    "hasPrevPage": false
  }
}
```

---

#### 22. `PATCH /admin/users/{id}` - Đổi vai trò hoặc Khóa/Mở tài khoản
- **Quyền**: `ADMIN`
- **Request Body**:
```json
{
  "role": "MANAGER",
  "active": false
}
```
- **Responses**:
  - `200 OK`: Trả về User record đã cập nhật.
  - `400 Bad Request`: Không thể tự khóa tài khoản Admin của chính mình.

---

#### 23. `GET /admin/orders` - Danh sách tất cả đơn hàng toàn hệ thống
- **Quyền**: `MANAGER`, `ADMIN`
- **Query Parameters**: `status`, `page`, `limit`.
- **Responses**: Trả về dữ liệu danh sách đơn hàng toàn hệ thống kèm thông tin người mua (Pagination Standard).

---

#### 24. `PATCH /admin/orders/{id}/status` - Chuyển trạng thái đơn hàng
- **Quyền**: `MANAGER`, `ADMIN`
- **Luồng trạng thái đơn hợp lệ (State Machine)**:
  - `PENDING` ➔ `CONFIRMED` hoặc `CANCELLED`
  - `CONFIRMED` ➔ `SHIPPING` hoặc `CANCELLED`
  - `SHIPPING` ➔ `COMPLETED`
- **Request Body**:
```json
{
  "status": "CONFIRMED"
}
```
- **Responses**:
  - `200 OK`: Trả về object `Order` đã chuyển trạng thái.
  - `400 Bad Request`: Chuyển trạng thái sai quy trình (ví dụ: Chuyển trực tiếp từ `PENDING` sang `COMPLETED`).

---

## 4. Bảng mã trạng thái HTTP (HTTP Status Codes)

| HTTP Code | Ý nghĩa | Ngữ cảnh sử dụng |
|---|---|---|
| **200 OK** | Thành công | Trả về dữ liệu cho các truy vấn `GET`, `PUT`, `PATCH`, `DELETE`. |
| **201 Created** | Tạo thành công | Đã tạo thành công Resource mới (`POST /register`, `POST /orders`, `POST /books`). |
| **204 No Content** | Thành công rỗng | Xóa thành công hoặc Đăng xuất thành công, không cần gửi lại body. |
| **400 Bad Request** | Lỗi dữ liệu | Request body/query vi phạm validation rules (thiếu trường, sai format). |
| **401 Unauthorized** | Chưa xác thực | Client chưa gửi Session Cookie hoặc Session đã hết hạn. |
| **403 Forbidden** | Không đủ quyền | Sai vai trò (User truy cập Admin API) hoặc không thỏa mãn ràng buộc nghiệp vụ. |
| **404 Not Found** | Không tìm thấy | ID tài nguyên truyền trên URL không tồn tại trong DB. |
| **409 Conflict** | Xung đột dữ liệu | Trùng email khi đăng ký, vượt quá số lượng tồn kho khi mua, hoặc hủy đơn không đúng lúc. |
| **500 Internal Error** | Lỗi Server | Lỗi không xác định từ Backend, đứt kết nối Database. |