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

## 3. Mã nguồn Chi tiết các API Trọng tâm (Input, Output, Thành công & Thất bại)

### 3.1. API Đăng nhập (`POST /auth/login`)

#### Đầu vào (Input - Request Body)
```json
{
  "email": "nguyenvana@example.com",
  "password": "Password123!"
}
```

#### Đầu ra THÀNH CÔNG (`200 OK`)
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

#### Đầu ra THẤT BẠI (`401 Unauthorized` / `403 Forbidden`)
```json
{
  "statusCode": 403,
  "message": "Tài khoản của bạn đã bị khóa bởi quản trị viên",
  "error": "Forbidden",
  "timestamp": "2026-11-07T14:30:00.000Z"
}
```

#### Node.js / Express Controller Implementation
```javascript
// POST /api/v1/auth/login
async function login(req, res) {
  try {
    // 1. INPUT
    const { email, password } = req.body;

    if (!email || !password) {
      return res.status(400).json({
        statusCode: 400,
        message: "Email và mật khẩu không được để trống",
        error: "Bad Request"
      });
    }

    // 2. CHECK USER EXISTENCE
    const user = await User.findOne({ email });
    if (!user) {
      // THẤT BẠI: Sai email
      return res.status(401).json({
        statusCode: 401,
        message: "Email hoặc mật khẩu không chính xác",
        error: "Unauthorized"
      });
    }

    // 3. CHECK PASSWORD
    const isPasswordValid = await bcrypt.compare(password, user.password);
    if (!isPasswordValid) {
      // THẤT BẠI: Sai mật khẩu
      return res.status(401).json({
        statusCode: 401,
        message: "Email hoặc mật khẩu không chính xác",
        error: "Unauthorized"
      });
    }

    // 4. CHECK USER ACTIVE STATUS
    if (user.active === false) {
      // THẤT BẠI: Tài khoản bị khóa
      return res.status(403).json({
        statusCode: 403,
        message: "Tài khoản của bạn đã bị khóa bởi quản trị viên",
        error: "Forbidden"
      });
    }

    // 5. SESSION REGENERATION & CART MERGE (SUCCESS)
    req.session.regenerate(async (err) => {
      if (err) {
        return res.status(500).json({
          statusCode: 500,
          message: "Lỗi tạo phiên làm việc",
          error: "Internal Server Error"
        });
      }

      req.session.userId = user._id;
      req.session.role = user.role;

      // Merge guest cart from session into DB cart
      if (req.session.guestCart && req.session.guestCart.length > 0) {
        await mergeCartToUserDatabase(user._id, req.session.guestCart);
        req.session.guestCart = null;
      }

      // OUTPUT SUCCESS
      return res.status(200).json({
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role,
        active: user.active,
        createdAt: user.createdAt
      });
    });

  } catch (error) {
    return res.status(500).json({
      statusCode: 500,
      message: "Lỗi server nội bộ",
      error: error.message
    });
  }
}
```

---

### 3.2. API Thêm sản phẩm vào giỏ (`POST /cart/items`)

#### Đầu vào (Input - Request Body)
```json
{
  "bookId": "bk_101",
  "quantity": 2
}
```

#### Đầu ra THÀNH CÔNG (`200 OK`)
```json
{
  "statusCode": 200,
  "message": "Thêm vào giỏ hàng thành công",
  "data": {
    "items": [
      {
        "bookId": "bk_101",
        "title": "Lập trình Node.js Chuyên nghiệp",
        "price": 180000,
        "quantity": 2,
        "subtotal": 360000
      }
    ],
    "totalQuantity": 2,
    "totalAmount": 360000
  }
}
```

#### Đầu ra THẤT BẠI (`409 Conflict`)
```json
{
  "statusCode": 409,
  "message": "Số lượng yêu cầu vượt quá tồn kho. Kho chỉ còn 1 cuốn.",
  "error": "Conflict",
  "timestamp": "2026-11-07T14:30:00.000Z"
}
```

#### Node.js / Express Controller Implementation
```javascript
// POST /api/v1/cart/items
async function addToCart(req, res) {
  try {
    // 1. INPUT
    const { bookId, quantity } = req.body;
    if (!bookId || !quantity || quantity < 1) {
      return res.status(400).json({
        statusCode: 400,
        message: "Mã sách và số lượng mua không hợp lệ",
        error: "Bad Request"
      });
    }

    const cart = req.session.cart || { items: [], totalAmount: 0, totalQuantity: 0 };
    const book = await Book.findById(bookId);
    
    // THẤT BẠI 1: Không tìm thấy sách
    if (!book || !book.active) {
      return res.status(404).json({
        statusCode: 404,
        message: "Sách không tồn tại hoặc đã ngừng kinh doanh",
        error: "Not Found"
      });
    }

    const existingIndex = cart.items.findIndex(item => item.bookId === bookId);
    const currentQty = existingIndex > -1 ? cart.items[existingIndex].quantity : 0;
    const requestedQty = currentQty + quantity;

    // THẤT BẠI 2: Vượt tồn kho
    if (requestedQty > book.stock) {
      return res.status(409).json({
        statusCode: 409,
        message: `Số lượng yêu cầu vượt quá tồn kho. Kho chỉ còn ${book.stock} cuốn.`,
        error: "Conflict"
      });
    }

    // 2. SUCCESS PROCESSING
    if (existingIndex > -1) {
      cart.items[existingIndex].quantity = requestedQty;
      cart.items[existingIndex].subtotal = requestedQty * book.price;
    } else {
      cart.items.push({
        bookId: book._id,
        title: book.title,
        price: book.price,
        quantity,
        subtotal: book.price * quantity
      });
    }

    cart.totalAmount = cart.items.reduce((sum, item) => sum + item.subtotal, 0);
    cart.totalQuantity = cart.items.reduce((sum, item) => sum + item.quantity, 0);
    req.session.cart = cart;

    // OUTPUT SUCCESS
    return res.status(200).json({
      statusCode: 200,
      message: "Thêm vào giỏ hàng thành công",
      data: cart
    });

  } catch (error) {
    return res.status(500).json({
      statusCode: 500,
      message: "Lỗi hệ thống khi cập nhật giỏ hàng",
      error: error.message
    });
  }
}
```

---

### 3.3. API Tạo đơn hàng / Checkout (`POST /orders`)

#### Đầu vào (Input - Request Body)
```json
{
  "fullName": "Nguyễn Văn A",
  "phone": "0901234567",
  "address": "Số 12 Nguyễn Huệ, Phường Bến Nghé, Quận 1, TP. Hồ Chí Minh",
  "paymentMethod": "COD",
  "note": "Giao vào giờ hành chính"
}
```

#### Đầu ra THÀNH CÔNG (`201 Created`)
```json
{
  "statusCode": 201,
  "message": "Đặt hàng thành công",
  "data": {
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
}
```

#### Đầu ra THẤT BẠI (`409 Conflict`)
```json
{
  "statusCode": 409,
  "message": "Sách 'Lập trình Node.js Chuyên nghiệp' chỉ còn 1 cuốn trong kho.",
  "error": "Conflict",
  "timestamp": "2026-11-07T14:35:00.000Z"
}
```

#### Node.js / Express Controller Implementation (Dùng DB Transaction)
```javascript
// POST /api/v1/orders
async function createOrder(req, res) {
  const sessionDb = await mongoose.startSession();
  sessionDb.startTransaction();

  try {
    // 1. INPUT
    const { fullName, phone, address, paymentMethod, note } = req.body;
    const userId = req.session.userId;

    if (!fullName || !phone || !address) {
      await sessionDb.endSession();
      return res.status(400).json({
        statusCode: 400,
        message: "Vui lòng nhập đầy đủ thông tin giao hàng",
        error: "Bad Request"
      });
    }

    const cart = req.session.cart;
    // THẤT BẠI 1: Giỏ hàng trống
    if (!cart || !cart.items || cart.items.length === 0) {
      await sessionDb.endSession();
      return res.status(400).json({
        statusCode: 400,
        message: "Giỏ hàng của bạn đang trống",
        error: "Bad Request"
      });
    }

    let totalAmount = 0;
    const orderItems = [];

    // 2. STOCK CHECK & ROW LOCKING
    for (const item of cart.items) {
      const book = await Book.findById(item.bookId).session(sessionDb);

      // THẤT BẠI 2: Sách không tồn tại
      if (!book) {
        await sessionDb.abortTransaction();
        sessionDb.endSession();
        return res.status(404).json({
          statusCode: 404,
          message: `Sách mã ${item.bookId} không tìm thấy`,
          error: "Not Found"
        });
      }

      // THẤT BẠI 3: Khống chế tồn kho
      if (item.quantity > book.stock) {
        await sessionDb.abortTransaction();
        sessionDb.endSession();
        return res.status(409).json({
          statusCode: 409,
          message: `Sách '${book.title}' chỉ còn ${book.stock} cuốn trong kho.`,
          error: "Conflict"
        });
      }

      // Trừ kho
      book.stock -= item.quantity;
      await book.save({ session: sessionDb });

      totalAmount += book.price * item.quantity;
      orderItems.push({
        bookId: book._id,
        title: book.title,
        price: book.price,
        quantity: item.quantity,
        subtotal: book.price * item.quantity
      });
    }

    // 3. CREATE ORDER RECORD & COMMIT
    const newOrder = new Order({
      user: userId,
      items: orderItems,
      totalAmount,
      status: "PENDING",
      paymentMethod: paymentMethod || "COD",
      fullName,
      phone,
      address,
      note
    });

    await newOrder.save({ session: sessionDb });
    req.session.cart = { items: [], totalAmount: 0, totalQuantity: 0 };

    await sessionDb.commitTransaction();
    sessionDb.endSession();

    // OUTPUT SUCCESS
    return res.status(201).json({
      statusCode: 201,
      message: "Đặt hàng thành công",
      data: newOrder
    });

  } catch (error) {
    await sessionDb.abortTransaction();
    sessionDb.endSession();

    return res.status(500).json({
      statusCode: 500,
      message: "Lỗi hệ thống khi thanh toán",
      error: error.message
    });
  }
}
```

---

## 4. Bảng Mã Lỗi và Hướng dẫn Xử lý

| HTTP Code | Ý nghĩa | Ngữ cảnh sử dụng |
|---|---|---|
| **200 OK** | Thành công | Trả dữ liệu cho `GET`, `PUT`, `PATCH`, `DELETE`. |
| **201 Created** | Tạo thành công | Tạo mới tài nguyên (`POST /register`, `POST /orders`, `POST /books`). |
| **204 No Content** | Thành công rỗng | Xóa thành công hoặc đăng xuất thành công. |
| **400 Bad Request** | Lỗi dữ liệu | Thiếu trường dữ liệu, sai định dạng JSON/Query. |
| **401 Unauthorized** | Chưa xác thực | Chưa gửi cookie session hoặc phiên hết hạn. |
| **403 Forbidden** | Không đủ quyền | User thông thường truy cập API của Admin/Manager. |
| **404 Not Found** | Không tìm thấy | ID sản phẩm, đơn hàng không tồn tại trong DB. |
| **409 Conflict** | Xung đột dữ liệu | Trùng email, vượt quá tồn kho, hủy đơn sai trạng thái. |
| **500 Internal Error** | Lỗi Server | Đứt kết nối database, lỗi hệ thống server backend. |