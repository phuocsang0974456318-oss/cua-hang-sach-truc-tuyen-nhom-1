# Thiết kế Chi tiết REST API & Phân tích Đánh giá Code (Nhóm 1)

> Tài liệu kỹ thuật toàn diện dành cho Backend và Frontend Developer.  
> Phiên bản API: `v1.0.0` | Base URL: `http://localhost:3000/api/v1`

---

## 1. Quy ước Kỹ thuật & Chuẩn hóa Hệ thống

### 1.1. Transport & Headers

* **Content-Type**: `application/json; charset=utf-8` (Cho tất cả các request có Body và Response).
* **Authentication**: Session-cookie (`connect.sid`) được cấp tự động từ server qua header `Set-Cookie` khi đăng nhập.
* **CORS & Credentials**: Phía Client (React/Axios/Fetch) bắt buộc thiết lập `withCredentials: true` để trình duyệt tự động đính kèm cookie session trong các request cross-origin.

### 1.2. Bảng Ma trận Phân quyền (RBAC Authorization Matrix)

| Nhóm chức năng | Endpoint | Guest | USER | MANAGER | ADMIN |
|---|---|:---:|:---:|:---:|:---:|
| Auth | `POST /auth/register`, `POST /auth/login` | ✅ | ❌ | ❌ | ❌ |
| Auth | `POST /auth/logout`, `GET /auth/me` | ❌ | ✅ | ✅ | ✅ |
| Sách | `GET /books`, `GET /books/{id}`, `GET /books/{id}/reviews` | ✅ | ✅ | ✅ | ✅ |
| Quản lý sách | `POST /books`, `PUT /books/{id}` | ❌ | ❌ | ✅ | ✅ |
| Xóa sách | `DELETE /books/{id}` | ❌ | ❌ | ❌ | ✅ |
| Đánh giá sách | `POST /books/{id}/reviews` | ❌ | ✅ (Đã mua) | ✅ | ✅ |
| Giỏ hàng | `GET /cart`, `POST /cart/items`, `PATCH /cart/items/{id}`, `DELETE /cart/items/{id}`, `DELETE /cart` | ✅ | ✅ | ✅ | ✅ |
| Đơn hàng cá nhân | `GET /orders`, `POST /orders`, `GET /orders/{id}`, `POST /orders/{id}/cancel` | ❌ | ✅ | ✅ | ✅ |
| Admin - Quản lý User | `GET /admin/users`, `PATCH /admin/users/{id}` | ❌ | ❌ | ❌ | ✅ |
| Admin - Quản lý Đơn hàng | `GET /admin/orders`, `PATCH /admin/orders/{id}/status` | ❌ | ❌ | ✅ | ✅ |

### 1.3. Chuẩn hóa Cấu trúc Response Phân trang (Pagination Standard)

Mọi API truy vấn danh sách bắt buộc trả về theo cấu trúc chuẩn:

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

### 1.4. Chuẩn hóa Cấu trúc Xử lý Lỗi (Error Response Standard)

Tất cả response lỗi HTTP status `4xx` và `5xx` phải tuân theo cấu trúc JSON:

```json
{
  "statusCode": 400,
  "message": "Dữ liệu gửi lên không hợp lệ",
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

## 2. Bảng Tổng hợp & Phân tích Code: Code Hiện tại vs Code REST API Chuẩn

### 2.1. Bảng So sánh Tổng quan Lỗi Kiến trúc trong Code Hiện tại

| Vấn đề / Tính năng | Code hiện tại (EJS/SSR) | Code REST API Chuẩn | Hậu quả nếu không sửa |
|---|---|---|---|
| **Response Format** | Trả về EJS Render HTML (`res.render`) | Trả về JSON Data (`res.json`) | Frontend React / Mobile App không parse được dữ liệu. |
| **Session Security** | Gán `req.session.userId` trực tiếp | Sử dụng `req.session.regenerate()` | Dễ bị tấn công **Session Fixation**. |
| **Giỏ hàng khi Đăng nhập** | Ghi đè session làm mất giỏ Guest | Merge items từ Session Guest vào User Cart | Người dùng bị mất sạch sản phẩm đã chọn trước khi đăng nhập. |
| **Kiểm tra Review** | Cho phép mọi user đã đăng nhập viết review | Kiểm tra order có `status == COMPLETED` | Bị spam đánh giá ảo, review giả mạo từ đối thủ. |
| **Xử lý Đặt hàng (Checkout)** | Đọc & trừ kho tuần tự không qua Transaction | Sử dụng **Database Transaction & Atomic Update** | Bán âm tồn kho khi nhiều người mua cùng lúc (Race Condition). |
| **Admin Pagination** | Trả mảng `[]` không có metadata, thiếu `limit` | Trả object `{ items, pagination: { totalPages, ... } }` | Frontend không làm được UI phân trang (Pagination Bar). |

---

### 2.2. Phân tích So sánh Mã nguồn Chi tiết

#### Lỗi 1: Authentication & Mất giỏ hàng Guest khi Login

##### Bảng so sánh phương thức Login
| Tiêu chí | Code hiện tại (SAI) | Code REST API Chuẩn (ĐÚNG) |
|---|---|---|
| Output | `res.render('login', ...)` | `res.status(200).json(user)` |
| Bảo mật Session | Giữ nguyên Session ID cũ | `req.session.regenerate()` |
| Xử lý Giỏ hàng | Bị xóa mất giỏ hàng tạm | Tự động gộp giỏ hàng Guest vào tài khoản |

* **Code hiện tại (SAI)**:
```javascript
// routes/auth.js (Code hiện tại)
router.post('/login', async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ email });
  if (!user || !user.comparePassword(password)) {
    // ❌ SAI: Render HTML EJS
    return res.render('login', { error: 'Email hoặc mật khẩu không đúng' });
  }
  // ❌ SAI: Không regenerate session & Làm mất giỏ hàng Guest
  req.session.userId = user._id;
  res.redirect('/');
});
```

* **Code REST API Chuẩn (ĐÚNG)**:
```javascript
// controllers/authController.js (Code ĐÚNG)
exports.login = async (req, res, next) => {
  try {
    const { email, password } = req.body;
    const user = await User.findOne({ email }).select('+password');

    if (!user || !(await user.comparePassword(password))) {
      return res.status(401).json({
        statusCode: 401,
        message: "Email hoặc mật khẩu không chính xác",
        error: "Unauthorized"
      });
    }

    if (!user.active) {
      return res.status(403).json({
        statusCode: 403,
        message: "Tài khoản của bạn đã bị khóa",
        error: "Forbidden"
      });
    }

    // 1. Lưu lại giỏ hàng guest
    const guestCart = req.session.cart || { items: [] };

    // 2. Regenerate Session ID chống Session Fixation
    req.session.regenerate(async (err) => {
      if (err) return next(err);

      // 3. Merge giỏ hàng Guest vào User Cart trong DB
      const userCart = await Cart.findOne({ user: user._id }) || new Cart({ user: user._id, items: [] });
      guestCart.items.forEach(guestItem => {
        const existingIndex = userCart.items.findIndex(i => i.book.toString() === guestItem.book.toString());
        if (existingIndex > -1) {
          userCart.items[existingIndex].quantity += guestItem.quantity;
        } else {
          userCart.items.push(guestItem);
        }
      });
      await userCart.save();

      // 4. Lưu thông tin xác thực
      req.session.user = { id: user._id, role: user.role, name: user.name, email: user.email };

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
    next(error);
  }
};
```

---

#### Lỗi 2: Không kiểm tra điều kiện mua hàng khi Viết Đánh giá (`POST /books/{id}/reviews`)

* **Code hiện tại (SAI)**:
```javascript
// routes/books.js (Code hiện tại)
router.post('/:id/reviews', isAuthenticated, async (req, res) => {
  const { rating, comment } = req.body;
  // ❌ SAI: Chỉ cần đăng nhập là được review -> Spam ảo
  const review = await Review.create({
    book: req.params.id,
    user: req.session.userId,
    rating, comment
  });
  res.redirect(`/books/${req.params.id}`);
});
```

* **Code REST API Chuẩn (ĐÚNG)**:
```javascript
// controllers/reviewController.js (Code ĐÚNG)
exports.createReview = async (req, res, next) => {
  try {
    const bookId = req.params.id;
    const userId = req.session.user.id;
    const { rating, comment } = req.body;

    // 1. Kiểm tra đơn hàng COMPLETED có chứa sách này không
    const hasPurchased = await Order.findOne({
      user: userId,
      status: 'COMPLETED',
      'items.book': bookId
    });

    if (!hasPurchased) {
      return res.status(403).json({
        statusCode: 403,
        message: "Bạn chỉ có thể đánh giá sản phẩm sau khi đã mua và hoàn thành đơn hàng.",
        error: "Forbidden"
      });
    }

    // 2. Kiểm tra trùng lặp review
    const existingReview = await Review.findOne({ book: bookId, user: userId });
    if (existingReview) {
      return res.status(409).json({
        statusCode: 409,
        message: "Bạn đã đánh giá sản phẩm này rồi.",
        error: "Conflict"
      });
    }

    const review = await Review.create({
      book: bookId,
      user: userId,
      userName: req.session.user.name,
      rating,
      comment
    });

    return res.status(201).json(review);
  } catch (error) {
    next(error);
  }
};
```

---

#### Lỗi 3: Đặt hàng thiếu Database Transaction (`POST /orders`)

* **Code hiện tại (SAI)**:
```javascript
// routes/orders.js (Code hiện tại)
router.post('/', isAuthenticated, async (req, res) => {
  const cart = req.session.cart;
  // ❌ SAI: Kiểm tra & trừ kho không có Transaction / Atomic Locking
  for (let item of cart.items) {
    const book = await Book.findById(item.bookId);
    if (book.stock < item.quantity) return res.status(400).send('Hết hàng');
    book.stock -= item.quantity; // ❌ Bị Race Condition nếu 2 người bấm cùng lúc
    await book.save();
  }
  const order = await Order.create({ ...req.body, items: cart.items });
  req.session.cart = null;
  res.json(order);
});
```

* **Code REST API Chuẩn (ĐÚNG)**:
```javascript
// controllers/orderController.js (Code ĐÚNG - Mongoose Transaction)
const mongoose = require('mongoose');

exports.createOrder = async (req, res, next) => {
  const session = await mongoose.startSession();
  session.startTransaction();

  try {
    const userId = req.session.user.id;
    const cart = await Cart.findOne({ user: userId }).session(session);

    if (!cart || cart.items.length === 0) {
      await session.abortTransaction();
      return res.status(400).json({ statusCode: 400, message: "Giỏ hàng trống.", error: "Bad Request" });
    }

    const orderItems = [];
    let totalAmount = 0;

    // 1. Trừ kho Atomic với điều kiện stock >= quantity
    for (const item of cart.items) {
      const updatedBook = await Book.findOneAndUpdate(
        { _id: item.book, stock: { $gte: item.quantity }, active: true },
        { $inc: { stock: -item.quantity } },
        { new: true, session }
      );

      if (!updatedBook) {
        await session.abortTransaction();
        return res.status(409).json({
          statusCode: 409,
          message: `Sản phẩm '${item.title}' không đủ tồn kho.`,
          error: "Conflict"
        });
      }

      const subtotal = updatedBook.price * item.quantity;
      totalAmount += subtotal;
      orderItems.push({
        book: updatedBook._id,
        title: updatedBook.title,
        price: updatedBook.price,
        quantity: item.quantity,
        subtotal
      });
    }

    // 2. Tạo đơn hàng
    const order = await Order.create([{
      user: userId,
      items: orderItems,
      totalAmount,
      fullName: req.body.fullName,
      phone: req.body.phone,
      address: req.body.address,
      paymentMethod: req.body.paymentMethod,
      note: req.body.note,
      status: 'PENDING'
    }], { session });

    // 3. Làm trống giỏ hàng
    cart.items = [];
    await cart.save({ session });

    await session.commitTransaction();
    session.endSession();

    return res.status(201).json(order[0]);
  } catch (error) {
    await session.abortTransaction();
    session.endSession();
    next(error);
  }
};
```

---

#### Lỗi 4: Danh sách Admin thiếu Phân trang & Metadata (`GET /admin/users`)

* **Code hiện tại (SAI)**:
```javascript
// routes/admin.js (Code hiện tại)
router.get('/users', isAdmin, async (req, res) => {
  // ❌ SAI: Trả về toàn bộ danh sách User gây chậm server
  const users = await User.find();
  res.json(users); // Trả mảng trực tiếp, thiếu pagination metadata
});
```

* **Code REST API Chuẩn (ĐÚNG)**:
```javascript
// controllers/adminController.js (Code ĐÚNG)
exports.getAdminUsers = async (req, res, next) => {
  try {
    const page = parseInt(req.query.page, 10) || 1;
    const limit = parseInt(req.query.limit, 10) || 10;
    const skip = (page - 1) * limit;

    const filter = {};
    if (req.query.role) filter.role = req.query.role;
    if (req.query.active !== undefined) filter.active = req.query.active === 'true';

    const [users, totalItems] = await Promise.all([
      User.find(filter).select('-password').skip(skip).limit(limit).sort({ createdAt: -1 }),
      User.countDocuments(filter)
    ]);

    const totalPages = Math.ceil(totalItems / limit);

    return res.status(200).json({
      items: users,
      pagination: {
        totalItems,
        totalPages,
        currentPage: page,
        limit,
        hasNextPage: page < totalPages,
        hasPrevPage: page > 1
      }
    });
  } catch (error) {
    next(error);
  }
};
```

---

## 3. Bảng Tổng quan 24 API Endpoints

Below is the master table summarizing all endpoints in the system:

| # | HTTP Method | Đường dẫn Endpoint | Tóm tắt Chức năng | Body Yêu cầu | Query Params Chi tiết | Mã Response Trả về |
|---|---|---|---|---|---|---|
| **1** | `POST` | `/api/v1/auth/register` | Đăng ký tài khoản mới | `name`, `email`, `password` | Không | `201`, `400`, `409` |
| **2** | `POST` | `/api/v1/auth/login` | Đăng nhập & Merge Cart | `email`, `password` | Không | `200`, `401`, `403` |
| **3** | `POST` | `/api/v1/auth/logout` | Đăng xuất hệ thống | Không | Không | `204`, `401` |
| **4** | `GET` | `/api/v1/auth/me` | Thông tin đang đăng nhập | Không | Không | `200`, `401` |
| **5** | `GET` | `/api/v1/books` | Danh sách sách & Lọc | Không | `q`, `category`, `minPrice`, `maxPrice`, `sort`, `page`, `limit` | `200` |
| **6** | `POST` | `/api/v1/books` | Thêm sách mới | `title`, `author`, `price`, `stock`, `...` | Không | `201`, `400`, `401`, `403` |
| **7** | `GET` | `/api/v1/books/{id}` | Chi tiết 1 cuốn sách | Không | Không | `200`, `404` |
| **8** | `PUT` | `/api/v1/books/{id}` | Cập nhật thông tin sách | `title`, `author`, `price`, `stock`, `...` | Không | `200`, `400`, `401`, `403`, `404` |
| **9** | `DELETE` | `/api/v1/books/{id}` | Xóa sách khỏi hệ thống | Không | Không | `204`, `401`, `403`, `404` |
| **10** | `GET` | `/api/v1/books/{id}/reviews` | Danh sách đánh giá sách | Không | Không | `200` |
| **11** | `POST` | `/api/v1/books/{id}/reviews` | Viết đánh giá (Đã mua) | `rating`, `comment` | Không | `201`, `400`, `401`, `403`, `409` |
| **12** | `GET` | `/api/v1/cart` | Xem nội dung giỏ hàng | Không | Không | `200` |
| **13** | `DELETE` | `/api/v1/cart` | Xóa toàn bộ giỏ hàng | Không | Không | `200` |
| **14** | `POST` | `/api/v1/cart/items` | Thêm sản phẩm vào giỏ | `bookId`, `quantity` | Không | `200`, `404`, `409` |
| **15** | `PATCH` | `/api/v1/cart/items/{bookId}` | Sửa số lượng sản phẩm | `quantity` | Không | `200`, `404`, `409` |
| **16** | `DELETE` | `/api/v1/cart/items/{bookId}` | Xóa 1 món khỏi giỏ hàng | Không | Không | `200` |
| **17** | `GET` | `/api/v1/orders` | Lịch sử đơn hàng cá nhân | Không | `status`, `page`, `limit` | `200`, `401` |
| **18** | `POST` | `/api/v1/orders` | Đặt hàng (Checkout DB Transaction) | `fullName`, `phone`, `address`, `paymentMethod`, `note` | Không | `201`, `400`, `401`, `409` |
| **19** | `GET` | `/api/v1/orders/{id}` | Xem chi tiết 1 đơn hàng | Không | Không | `200`, `401`, `403`, `404` |
| **20** | `POST` | `/api/v1/orders/{id}/cancel` | Hủy đơn hàng (Khi PENDING) | Không | Không | `200`, `401`, `404`, `409` |
| **21** | `GET` | `/api/v1/admin/users` | Danh sách tất cả người dùng | Không | `role`, `active`, `page`, `limit` | `200`, `401`, `403` |
| **22** | `PATCH` | `/api/v1/admin/users/{id}` | Gán vai trò / Khóa User | `role`, `active` | Không | `200`, `400`, `401`, `403`, `404` |
| **23** | `GET` | `/api/v1/admin/orders` | Tất cả đơn hàng toàn hệ thống | Không | `status`, `page`, `limit` | `200`, `401`, `403` |
| **24** | `PATCH` | `/api/v1/admin/orders/{id}/status` | Cập nhật trạng thái đơn | `status` | Không | `200`, `400`, `401`, `403`, `404` |

---

## 4. Chi tiết Mô tả Chi tiết Payload cho từng API

*(Phần này chi tiết Request / Response Payload JSON như đã thiết kế ở Mục 3)*

### Bảng Sơ đồ Máy trạng thái Đơn hàng (Order State Machine Table)

Khi gọi API `PATCH /api/v1/admin/orders/{id}/status`, trạng thái chỉ được phép chuyển đổi theo bảng quy tắc sau:

| Trạng thái hiện tại | Trạng thái mới HỢP LỆ | Trạng thái KHÔNG HỢP LỆ (Trả 400 Bad Request) |
|---|---|---|
| `PENDING` | `CONFIRMED`, `CANCELLED` | `SHIPPING`, `COMPLETED` |
| `CONFIRMED` | `SHIPPING`, `CANCELLED` | `PENDING`, `COMPLETED` |
| `SHIPPING` | `COMPLETED` | `PENDING`, `CONFIRMED`, `CANCELLED` |
| `COMPLETED` | *(Không thể thay đổi)* | Tất cả trạng thái khác |
| `CANCELLED` | *(Không thể thay đổi)* | Tất cả trạng thái khác |

---

## 5. Bảng Tra cứu Mã Lỗi HTTP Status Code

| Status Code | Tên chuẩn | Nguyên nhân trả về |
| ----- | ----- | ----- |
| **200 OK** | Success | Request thành công cho các phương thức `GET`, `PUT`, `PATCH`, `DELETE`. |
| **201 Created** | Created | Tạo mới tài nguyên thành công (`POST /register`, `POST /orders`, `POST /books`). |
| **204 No Content** | No Content | Đã xử lý thành công và không cần trả về body (`POST /auth/logout`, `DELETE /books/{id}`). |
| **400 Bad Request** | Bad Request | Dữ liệu gửi lên vi phạm Validation hoặc vi phạm State Machine đơn hàng. |
| **401 Unauthorized** | Unauthorized | Chưa đăng nhập hoặc Session ID không hợp lệ / hết hạn. |
| **403 Forbidden** | Forbidden | Không đủ quyền truy cập (User thường gọi API Admin) hoặc vi phạm nghiệp vụ (chưa mua sách đòi review). |
| **404 Not Found** | Not Found | ID tài nguyên không tồn tại trong DB. |
| **409 Conflict** | Conflict | Xung đột dữ liệu: Trùng Email đăng ký, hết hàng trong kho khi checkout, gửi trùng review. |
| **500 Internal Error** | Server Error | Lỗi hệ thống Backend chưa được catch (Database disconnection, Runtime crash). |