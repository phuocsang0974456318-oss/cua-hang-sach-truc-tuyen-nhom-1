# Thiết kế API - Cửa hàng sách trực tuyến (Nhóm 1)

> Bản thiết kế REST API. Nguồn chính xác nhất là file `openapi.yaml`; tài liệu này được sinh từ file đó.

## 1. Quy ước chung

| Mục | Nội dung |
|---|---|
| Base URL | `http://localhost:3000/api/v1` |
| Định dạng | JSON (`Content-Type: application/json`) |
| Xác thực | Session cookie (`connect.sid`) do `express-session` cấp sau khi đăng nhập |
| Phân quyền | `USER` < `MANAGER` < `ADMIN` |
| Tiền tệ | VND, số nguyên |
| Phân trang | `page` (từ 1), `limit`. Kết quả danh sách phân trang luôn trả về cấu trúc object dạng `{ items, totalItems, totalPages, page, limit }` |
| Giỏ hàng | Lưu trong Session. Khi người dùng đăng nhập (`POST /auth/login`), backend có trách nhiệm gộp các sản phẩm trong giỏ hàng tạm (Guest Session) vào tài khoản trước khi tái tạo Session ID |
| Lỗi | `{ "message": "...", "errors": [{ "field": "...", "message": "..." }] }` |

## 2. Tổng quan endpoint

| # | Method | Đường dẫn | Chức năng | Quyền |
|---|---|---|---|---|
| 1 | `POST` | `/auth/register` | Đăng ký tài khoản | Công khai |
| 2 | `POST` | `/auth/login` | Đăng nhập (tự động gộp giỏ hàng Guest) | Công khai |
| 3 | `POST` | `/auth/logout` | Đăng xuất | Đăng nhập |
| 4 | `GET` | `/auth/me` | Lấy thông tin người dùng đang đăng nhập | Đăng nhập |
| 5 | `GET` | `/books` | Danh sách sách (tìm kiếm, lọc, phân trang) | Công khai |
| 6 | `POST` | `/books` | Thêm sách mới | Đăng nhập (MANAGER/ADMIN) |
| 7 | `GET` | `/books/{id}` | Chi tiết một cuốn sách | Công khai |
| 8 | `PUT` | `/books/{id}` | Cập nhật sách | Đăng nhập (MANAGER/ADMIN) |
| 9 | `DELETE` | `/books/{id}` | Xóa sách | Đăng nhập (ADMIN) |
| 10 | `GET` | `/books/{id}/reviews` | Danh sách đánh giá của một cuốn sách | Công khai |
| 11 | `POST` | `/books/{id}/reviews` | Viết đánh giá (đã mua sách và đơn `COMPLETED`) | Đăng nhập |
| 12 | `GET` | `/cart` | Xem giỏ hàng | Công khai |
| 13 | `DELETE` | `/cart` | Xóa toàn bộ giỏ hàng | Công khai |
| 14 | `POST` | `/cart/items` | Thêm sách vào giỏ | Công khai |
| 15 | `PATCH` | `/cart/items/{bookId}` | Cập nhật số lượng (quantity = 0 thì xóa khỏi giỏ) | Công khai |
| 16 | `DELETE` | `/cart/items/{bookId}` | Xóa một sách khỏi giỏ | Công khai |
| 17 | `GET` | `/orders` | Lịch sử đơn hàng của tôi | Đăng nhập |
| 18 | `POST` | `/orders` | Đặt hàng (checkout từ giỏ hàng, sử dụng Transaction) | Đăng nhập |
| 19 | `GET` | `/orders/{id}` | Chi tiết đơn hàng của tôi | Đăng nhập |
| 20 | `POST` | `/orders/{id}/cancel` | Hủy đơn hàng | Đăng nhập |
| 21 | `GET` | `/admin/users` | Danh sách người dùng (có phân trang) | Đăng nhập (ADMIN) |
| 22 | `PATCH` | `/admin/users/{id}` | Gán vai trò, khóa hoặc mở tài khoản | Đăng nhập (ADMIN) |
| 23 | `GET` | `/admin/orders` | Tất cả đơn hàng (có phân trang) | Đăng nhập (MANAGER/ADMIN) |
| 24 | `PATCH` | `/admin/orders/{id}/status` | Cập nhật trạng thái đơn | Đăng nhập (MANAGER/ADMIN) |

---

## 3. Chi tiết từng nhóm

### 3.1. Auth - Đăng ký, đăng nhập

#### 1. `POST /api/v1/auth/register`
**Đăng ký tài khoản**  
Quyền: Công khai

| Vị trí | Tên | Kiểu | Bắt buộc | Mô tả / Ràng buộc |
|---|---|---|---|---|
| body | name | string | Có | maxLen = 100; VD: `Nguyễn Văn A` |
| body | email | string (email) | Có | VD: `a@example.com` |
| body | password | string (password) | Có | minLen = 6; VD: `matkhau123` |

Ví dụ request body:
```json
{
  "name": "Nguyễn Văn A",
  "email": "a@example.com",
  "password": "matkhau123"
}
```

| Mã | Ý nghĩa |
|---|---|
| 201 | Tạo thành công → `User` |
| 400 | Dữ liệu gửi lên không hợp lệ → `Error` |
| 409 | Email đã tồn tại → `Error` |

#### 2. `POST /api/v1/auth/login`
**Đăng nhập**  
Quyền: Công khai  
*Lưu ý backend*: Đăng nhập thành công sẽ trả header `Set-Cookie` chứa Session ID mới. Cần gộp giỏ hàng trong Guest Session hiện tại vào Session mới của User.

| Vị trí | Tên | Kiểu | Bắt buộc | Mô tả / Ràng buộc |
|---|---|---|---|---|
| body | email | string (email) | Có |  |
| body | password | string (password) | Có |  |

Ví dụ request body:
```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

| Mã | Ý nghĩa |
|---|---|
| 200 | Đăng nhập thành công → `User` |
| 401 | Sai email hoặc mật khẩu → `Error` |
| 403 | Tài khoản đã bị khóa → `Error` |

#### 3. `POST /api/v1/auth/logout`
**Đăng xuất**  
Quyền: Đăng nhập

| Mã | Ý nghĩa |
|---|---|
| 204 | Đã đăng xuất |

#### 4. `GET /api/v1/auth/me`
**Lấy thông tin người dùng đang đăng nhập**  
Quyền: Đăng nhập

| Mã | Ý nghĩa |
|---|---|
| 200 | OK → `User` |
| 401 | Chưa đăng nhập → `Error` |

---

### 3.2. Books - Sách và đánh giá

#### 5. `GET /api/v1/books`
**Danh sách sách (tìm kiếm, lọc, phân trang)**  
Quyền: Công khai

| Vị trí | Tên | Kiểu | Bắt buộc | Mô tả / Ràng buộc |
|---|---|---|---|---|
| query | q | string | Không | Tìm theo tên sách, tác giả, thể loại |
| query | category | string | Không | VD: `Lập trình` |
| query | minPrice | integer | Không | min = 0 |
| query | maxPrice | integer | Không | min = 0 |
| query | sort | string | Không | Giá trị: `newest`, `price_asc`, `price_desc`, `title`; mặc định = `newest` |
| query | page | integer | Không | min = 1; mặc định = 1 |
| query | limit | integer | Không | min = 1; max = 50; mặc định = 12 |

| Mã | Ý nghĩa |
|---|---|
| 200 | OK → `{ items: array<Book>, totalItems: integer, totalPages: integer, page: integer, limit: integer }` |

#### 6. `POST /api/v1/books`
**Thêm sách mới (MANAGER/ADMIN)**  
Quyền: Đăng nhập (MANAGER/ADMIN)

| Vị trí | Tên | Kiểu | Bắt buộc | Mô tả / Ràng buộc |
|---|---|---|---|---|
| body | title | string | Có |  |
| body | author | string | Có |  |
| body | price | integer | Có | min = 0 |
| body | category | string | Không |  |
| body | image | string | Không |  |
| body | description | string | Không |  |
| body | stock | integer | Có | min = 0 |
| body | active | boolean | Không | mặc định = true |

Ví dụ request body:
```json
{
  "title": "Lập trình Node.js Nâng cao",
  "author": "Nguyễn Văn A",
  "price": 150000,
  "category": "Lập trình",
  "image": "https://example.com/cover.jpg",
  "description": "Mô tả cuốn sách...",
  "stock": 50,
  "active": true
}
```

| Mã | Ý nghĩa |
|---|---|
| 201 | Đã tạo → `Book` |
| 400 | Dữ liệu gửi lên không hợp lệ → `Error` |
| 401 | Chưa đăng nhập → `Error` |
| 403 | Không đủ quyền → `Error` |

#### 7. `GET /api/v1/books/{id}`
**Chi tiết một cuốn sách**  
Quyền: Công khai

| Vị trí | Tên | Kiểu | Bắt buộc | Mô tả / Ràng buộc |
|---|---|---|---|---|
| path | id | string | Có | VD: `1` |

| Mã | Ý nghĩa |
|---|---|
| 200 | OK → `Book` |
| 404 | Không tìm thấy → `Error` |

#### 8. `PUT /api/v1/books/{id}`
**Cập nhật sách (MANAGER/ADMIN)**  
Quyền: Đăng nhập (MANAGER/ADMIN)

| Vị trí | Tên | Kiểu | Bắt buộc | Mô tả / Ràng buộc |
|---|---|---|---|---|
| path | id | string | Có | VD: `1` |
| body | title | string | Có |  |
| body | author | string | Có |  |
| body | price | integer | Có | min = 0 |
| body | category | string | Không |  |
| body | image | string | Không |  |
| body | description | string | Không |  |
| body | stock | integer | Có | min = 0 |
| body | active | boolean | Không | mặc định = true |

| Mã | Ý nghĩa |
|---|---|
| 200 | Đã cập nhật → `Book` |
| 400 | Dữ liệu gửi lên không hợp lệ → `Error` |
| 401 | Chưa đăng nhập → `Error` |
| 403 | Không đủ quyền → `Error` |
| 404 | Không tìm thấy → `Error` |

#### 9. `DELETE /api/v1/books/{id}`
**Xóa sách (ADMIN)**  
Quyền: Đăng nhập (ADMIN)

| Vị trí | Tên | Kiểu | Bắt buộc | Mô tả / Ràng buộc |
|---|---|---|---|---|
| path | id | string | Có | VD: `1` |

| Mã | Ý nghĩa |
|---|---|
| 204 | Đã xóa |
| 401 | Chưa đăng nhập → `Error` |
| 403 | Không đủ quyền → `Error` |
| 404 | Không tìm thấy → `Error` |

#### 10. `GET /api/v1/books/{id}/reviews`
**Danh sách đánh giá của một cuốn sách**  
Quyền: Công khai

| Vị trí | Tên | Kiểu | Bắt buộc | Mô tả / Ràng buộc |
|---|---|---|---|---|
| path | id | string | Có | VD: `1` |

| Mã | Ý nghĩa |
|---|---|
| 200 | OK → `array<Review>` |

#### 11. `POST /api/v1/books/{id}/reviews`
**Viết đánh giá sách**  
Quyền: Đăng nhập  
*Ràng buộc nghiệp vụ*: Người dùng phải có ít nhất một đơn hàng chứa cuốn sách này ở trạng thái `COMPLETED`.

| Vị trí | Tên | Kiểu | Bắt buộc | Mô tả / Ràng buộc |
|---|---|---|---|---|
| path | id | string | Có | VD: `1` |
| body | rating | integer | Có | min = 1; max = 5; VD: `5` |
| body | comment | string | Không | maxLen = 1000; VD: `Sách dễ hiểu` |

Ví dụ request body:
```json
{
  "rating": 5,
  "comment": "Sách rất dễ hiểu và thực tế"
}
```

| Mã | Ý nghĩa |
|---|---|
| 201 | Đã tạo đánh giá → `Review` |
| 400 | Dữ liệu gửi lên không hợp lệ → `Error` |
| 401 | Chưa đăng nhập → `Error` |
| 403 | Chưa từng mua sách hoặc đơn chưa ở trạng thái COMPLETED → `Error` |
| 404 | Không tìm thấy sách → `Error` |

---

### 3.3. Cart - Giỏ hàng (lưu trong session)

#### 12. `GET /api/v1/cart`
**Xem giỏ hàng**  
Quyền: Công khai

| Mã | Ý nghĩa |
|---|---|
| 200 | OK → `Cart` |

#### 13. `DELETE /api/v1/cart`
**Xóa toàn bộ giỏ hàng**  
Quyền: Công khai

| Mã | Ý nghĩa |
|---|---|
| 200 | Đã xóa giỏ hàng thành công → `Cart` (trả về giỏ hàng rỗng) |

#### 14. `POST /api/v1/cart/items`
**Thêm sách vào giỏ**  
Quyền: Công khai  
Nếu sách đã có trong giỏ thì cộng dồn số lượng.

| Vị trí | Tên | Kiểu | Bắt buộc | Mô tả / Ràng buộc |
|---|---|---|---|---|
| body | bookId | string | Có | VD: `1` |
| body | quantity | integer | Không | min = 1; mặc định = 1 |

Ví dụ request body:
```json
{
  "bookId": "1",
  "quantity": 1
}
```

| Mã | Ý nghĩa |
|---|---|
| 200 | Giỏ hàng sau khi thêm → `Cart` |
| 404 | Không tìm thấy sách → `Error` |
| 409 | Số lượng vượt quá tồn kho → `Error` |

#### 15. `PATCH /api/v1/cart/items/{bookId}`
**Cập nhật số lượng (quantity = 0 thì xóa khỏi giỏ)**  
Quyền: Công khai

| Vị trí | Tên | Kiểu | Bắt buộc | Mô tả / Ràng buộc |
|---|---|---|---|---|
| path | bookId | string | Có |  |
| body | quantity | integer | Có | min = 0; VD: `2` |

Ví dụ request body:
```json
{
  "quantity": 2
}
```

| Mã | Ý nghĩa |
|---|---|
| 200 | Giỏ hàng sau khi cập nhật → `Cart` |
| 404 | Không tìm thấy sách trong giỏ → `Error` |
| 409 | Số lượng vượt quá tồn kho → `Error` |

#### 16. `DELETE /api/v1/cart/items/{bookId}`
**Xóa một sách khỏi giỏ**  
Quyền: Công khai

| Vị trí | Tên | Kiểu | Bắt buộc | Mô tả / Ràng buộc |
|---|---|---|---|---|
| path | bookId | string | Có |  |

| Mã | Ý nghĩa |
|---|---|
| 200 | Giỏ hàng sau khi xóa sản phẩm → `Cart` |

---

### 3.4. Orders - Đặt hàng, lịch sử đơn

#### 17. `GET /api/v1/orders`
**Lịch sử đơn hàng của tôi**  
Quyền: Đăng nhập

| Vị trí | Tên | Kiểu | Bắt buộc | Mô tả / Ràng buộc |
|---|---|---|---|---|
| query | status | OrderStatus | Không | Giá trị: `PENDING`, `CONFIRMED`, `SHIPPING`, `COMPLETED`, `CANCELLED` |
| query | page | integer | Không | min = 1; mặc định = 1 |
| query | limit | integer | Không | min = 1; max = 50; mặc định = 10 |

| Mã | Ý nghĩa |
|---|---|
| 200 | OK → `{ items: array<Order>, totalItems: integer, totalPages: integer, page: integer, limit: integer }` |
| 401 | Chưa đăng nhập → `Error` |

#### 18. `POST /api/v1/orders`
**Đặt hàng (checkout từ giỏ hàng)**  
Quyền: Đăng nhập  
*Lưu ý kỹ thuật*: Backend phải thực hiện xử lý qua **Database Transaction**: Kiểm tra tồn kho → Trừ tồn kho → Tạo đơn hàng → Làm trống giỏ hàng để tránh race-condition (bán âm tồn kho).

| Vị trí | Tên | Kiểu | Bắt buộc | Mô tả / Ràng buộc |
|---|---|---|---|---|
| body | fullName | string | Có | VD: `Nguyễn Văn A` |
| body | phone | string | Có | VD: `0901234567` |
| body | address | string | Có | VD: `12 Nguyễn Huệ` |
| body | paymentMethod | string | Có | Giá trị: `COD`, `BANK_TRANSFER` |
| body | note | string | Không |  |

Ví dụ request body:
```json
{
  "fullName": "Nguyễn Văn A",
  "phone": "0901234567",
  "address": "12 Nguyễn Huệ",
  "paymentMethod": "COD",
  "note": "Giao giờ hành chính"
}
```

| Mã | Ý nghĩa |
|---|---|
| 201 | Đặt hàng thành công → `Order` |
| 400 | Giỏ hàng trống hoặc dữ liệu không hợp lệ → `Error` |
| 401 | Chưa đăng nhập → `Error` |
| 409 | Một hoặc nhiều sách không đủ tồn kho → `Error` |

#### 19. `GET /api/v1/orders/{id}`
**Chi tiết đơn hàng của tôi**  
Quyền: Đăng nhập

| Vị trí | Tên | Kiểu | Bắt buộc | Mô tả / Ràng buộc |
|---|---|---|---|---|
| path | id | string | Có | VD: `1` |

| Mã | Ý nghĩa |
|---|---|
| 200 | OK → `Order` |
| 401 | Chưa đăng nhập → `Error` |
| 404 | Không tìm thấy đơn hàng → `Error` |

#### 20. `POST /api/v1/orders/{id}/cancel`
**Hủy đơn hàng**  
Quyền: Đăng nhập  
Chỉ hủy được khi đơn đang `PENDING`. Tồn kho được hoàn lại đầy đủ qua Transaction.

| Vị trí | Tên | Kiểu | Bắt buộc | Mô tả / Ràng buộc |
|---|---|---|---|---|
| path | id | string | Có | VD: `1` |

| Mã | Ý nghĩa |
|---|---|
| 200 | Đã hủy → `Order` |
| 401 | Chưa đăng nhập → `Error` |
| 404 | Không tìm thấy đơn hàng → `Error` |
| 409 | Đơn không thể hủy ở trạng thái hiện tại → `Error` |

---

### 3.5. Admin - Quản trị (MANAGER/ADMIN)

#### 21. `GET /api/v1/admin/users`
**Danh sách người dùng (ADMIN)**  
Quyền: Đăng nhập (ADMIN)

| Vị trí | Tên | Kiểu | Bắt buộc | Mô tả / Ràng buộc |
|---|---|---|---|---|
| query | role | Role | Không | Giá trị: `USER`, `MANAGER`, `ADMIN` |
| query | active | boolean | Không |  |
| query | page | integer | Không | min = 1; mặc định = 1 |
| query | limit | integer | Không | min = 1; max = 50; mặc định = 10 |

| Mã | Ý nghĩa |
|---|---|
| 200 | OK → `{ items: array<User>, totalItems: integer, totalPages: integer, page: integer, limit: integer }` |
| 401 | Chưa đăng nhập → `Error` |
| 403 | Không đủ quyền → `Error` |

#### 22. `PATCH /api/v1/admin/users/{id}`
**Gán vai trò, khóa hoặc mở tài khoản (ADMIN)**  
Quyền: Đăng nhập (ADMIN)

| Vị trí | Tên | Kiểu | Bắt buộc | Mô tả / Ràng buộc |
|---|---|---|---|---|
| path | id | string | Có | VD: `1` |
| body | role | Role | Không | Giá trị: `USER`, `MANAGER`, `ADMIN` |
| body | active | boolean | Không |  |

Ví dụ request body:
```json
{
  "role": "MANAGER",
  "active": true
}
```

| Mã | Ý nghĩa |
|---|---|
| 200 | Đã cập nhật → `User` |
| 401 | Chưa đăng nhập → `Error` |
| 403 | Không đủ quyền → `Error` |
| 404 | Không tìm thấy người dùng → `Error` |

#### 23. `GET /api/v1/admin/orders`
**Tất cả đơn hàng (MANAGER/ADMIN)**  
Quyền: Đăng nhập (MANAGER/ADMIN)

| Vị trí | Tên | Kiểu | Bắt buộc | Mô tả / Ràng buộc |
|---|---|---|---|---|
| query | status | OrderStatus | Không | Giá trị: `PENDING`, `CONFIRMED`, `SHIPPING`, `COMPLETED`, `CANCELLED` |
| query | page | integer | Không | min = 1; mặc định = 1 |
| query | limit | integer | Không | min = 1; max = 50; mặc định = 10 |

| Mã | Ý nghĩa |
|---|---|
| 200 | OK → `{ items: array<Order>, totalItems: integer, totalPages: integer, page: integer, limit: integer }` |
| 401 | Chưa đăng nhập → `Error` |
| 403 | Không đủ quyền → `Error` |

#### 24. `PATCH /api/v1/admin/orders/{id}/status`
**Cập nhật trạng thái đơn (MANAGER/ADMIN)**  
Quyền: Đăng nhập (MANAGER/ADMIN)

| Vị trí | Tên | Kiểu | Bắt buộc | Mô tả / Ràng buộc |
|---|---|---|---|---|
| path | id | string | Có | VD: `1` |
| body | status | OrderStatus | Có | Giá trị: `PENDING`, `CONFIRMED`, `SHIPPING`, `COMPLETED`, `CANCELLED` |

Ví dụ request body:
```json
{
  "status": "CONFIRMED"
}
```

| Mã | Ý nghĩa |
|---|---|
| 200 | Đã cập nhật → `Order` |
| 401 | Chưa đăng nhập → `Error` |
| 403 | Không đủ quyền → `Error` |
| 404 | Không tìm thấy đơn hàng → `Error` |

---

## 4. Cấu trúc dữ liệu (Schemas)

### Error
| Trường | Kiểu | Mô tả / Ràng buộc |
|---|---|---|
| `message` | string | VD: `Không tìm thấy dữ liệu.` |
| `errors` | array<object> | Trả về danh sách chi tiết các trường bị lỗi validator |

### Role
Giá trị: `USER`, `MANAGER`, `ADMIN`

### OrderStatus
Giá trị: `PENDING`, `CONFIRMED`, `SHIPPING`, `COMPLETED`, `CANCELLED`

### User
| Trường | Kiểu | Mô tả / Ràng buộc |
|---|---|---|
| `id` | string | VD: `1730000000000` |
| `name` | string |  |
| `email` | string (email) |  |
| `role` | Role | Giá trị: `USER`, `MANAGER`, `ADMIN` |
| `active` | boolean |  |
| `createdAt` | string (date-time) |  |

### Book
| Trường | Kiểu | Mô tả / Ràng buộc |
|---|---|---|
| `id` | string | VD: `1` |
| `title` | string | VD: `Lập trình Node.js căn bản` |
| `author` | string | VD: `Nguyễn Văn A` |
| `price` | integer | Đơn vị VND \| VD: `120000` |
| `category` | string | VD: `Lập trình` |
| `image` | string | URL ảnh bìa |
| `description` | string |  |
| `stock` | integer | VD: `10` |
| `active` | boolean |  |
| `createdAt` | string (date-time) |  |

### Review
| Trường | Kiểu | Mô tả / Ràng buộc |
|---|---|---|
| `id` | string |  |
| `bookId` | string |  |
| `userName` | string |  |
| `rating` | integer | min = 1; max = 5 |
| `comment` | string |  |
| `createdAt` | string (date-time) |  |

### CartItem
| Trường | Kiểu | Mô tả / Ràng buộc |
|---|---|---|
| `book` | Book | Object thông tin cuốn sách |
| `quantity` | integer | Số lượng trong giỏ |
| `subtotal` | integer | Thành tiền (`price * quantity`) |

### Cart
| Trường | Kiểu | Mô tả / Ràng buộc |
|---|---|---|
| `items` | array<CartItem> |  |
| `totalQuantity` | integer | Tổng số lượng sản phẩm |
| `total` | integer | Tổng tiền VND |

### OrderItem
| Trường | Kiểu | Mô tả / Ràng buộc |
|---|---|---|
| `book` | string | Id sách |
| `title` | string |  |
| `price` | integer | Giá tại thời điểm mua |
| `quantity` | integer |  |

### Order
| Trường | Kiểu | Mô tả / Ràng buộc |
|---|---|---|
| `id` | string |  |
| `user` | string | Id người đặt |
| `items` | array<OrderItem> |  |
| `total` | integer | Tổng tiền đơn hàng |
| `status` | OrderStatus | Giá trị: `PENDING`, `CONFIRMED`, `SHIPPING`, `COMPLETED`, `CANCELLED` |
| `paymentMethod` | string | Giá trị: `COD`, `BANK_TRANSFER` |
| `fullName` | string |  |
| `phone` | string |  |
| `address` | string |  |
| `note` | string |  |
| `createdAt` | string (date-time) |  |

---

## 5. Mã phản hồi chung

| Mã | Ý nghĩa |
|---|---|
| 200 | Thành công |
| 201 | Tạo mới thành công |
| 204 | Thành công, không có nội dung |
| 400 | Dữ liệu không hợp lệ |
| 401 | Chưa đăng nhập |
| 403 | Không đủ quyền |
| 404 | Không tìm thấy |
| 409 | Xung đột dữ liệu / Vi phạm ràng buộc kinh doanh |

---

## 6. Lưu ý triển khai backend

- Các route API này thiết kế cho SPA (React/Vite) hoặc mobile app giao tiếp với Express.js backend.
- Các thao tác cập nhật số lượng kho và trạng thái đơn hàng (`POST /orders`, `POST /orders/{id}/cancel`) bắt buộc phải bọc trong **Database Transaction** để chống race-condition.
- Khi người dùng đăng nhập (`POST /auth/login`), gộp toàn bộ sản phẩm ở `req.session.cart` (khách) vào giỏ hàng cá nhân trong session mới.