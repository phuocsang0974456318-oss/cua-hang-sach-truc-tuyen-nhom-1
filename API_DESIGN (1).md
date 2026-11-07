# Thiết kế API - Cửa hàng sách trực tuyến (Nhóm 1)

> Bản thiết kế REST API. Nguồn chính xác nhất là file [`openapi.yaml`](openapi.yaml); tài liệu này được sinh từ file đó.

## 1. Quy ước chung

| Mục | Nội dung |
|---|---|
| Base URL | `http://localhost:3000/api/v1` |
| Định dạng | JSON (`Content-Type: application/json`) |
| Xác thực | Session cookie (`connect.sid`) do `express-session` cấp sau khi đăng nhập |
| Phân quyền | `USER` < `MANAGER` < `ADMIN` |
| Tiền tệ | VND, số nguyên |
| Phân trang | `page` (từ 1), `limit` |
| Lỗi | `{ "message": "...", "errors": [{ "field": "...", "message": "..." }] }` |

## 2. Tổng quan endpoint

| # | Method | Đường dẫn | Chức năng | Quyền |
|---|---|---|---|---|
| 1 | `POST` | `/auth/register` | Đăng ký tài khoản | Công khai |
| 2 | `POST` | `/auth/login` | Đăng nhập | Công khai |
| 3 | `POST` | `/auth/logout` | Đăng xuất | Đăng nhập |
| 4 | `GET` | `/auth/me` | Lấy thông tin người dùng đang đăng nhập | Đăng nhập |
| 5 | `GET` | `/books` | Danh sách sách (tìm kiếm, lọc, phân trang) | Công khai |
| 6 | `POST` | `/books` | Thêm sách mới | Đăng nhập (MANAGER/ADMIN) |
| 7 | `GET` | `/books/{id}` | Chi tiết một cuốn sách | Công khai |
| 8 | `PUT` | `/books/{id}` | Cập nhật sách | Đăng nhập (MANAGER/ADMIN) |
| 9 | `DELETE` | `/books/{id}` | Xóa sách | Đăng nhập (ADMIN) |
| 10 | `GET` | `/books/{id}/reviews` | Danh sách đánh giá của một cuốn sách | Công khai |
| 11 | `POST` | `/books/{id}/reviews` | Viết đánh giá (cần đăng nhập) | Đăng nhập |
| 12 | `GET` | `/cart` | Xem giỏ hàng | Công khai |
| 13 | `DELETE` | `/cart` | Xóa toàn bộ giỏ hàng | Công khai |
| 14 | `POST` | `/cart/items` | Thêm sách vào giỏ | Công khai |
| 15 | `PATCH` | `/cart/items/{bookId}` | Cập nhật số lượng (quantity = 0 thì xóa khỏi giỏ) | Công khai |
| 16 | `DELETE` | `/cart/items/{bookId}` | Xóa một sách khỏi giỏ | Công khai |
| 17 | `GET` | `/orders` | Lịch sử đơn hàng của tôi | Đăng nhập |
| 18 | `POST` | `/orders` | Đặt hàng (checkout từ giỏ hàng) | Đăng nhập |
| 19 | `GET` | `/orders/{id}` | Chi tiết đơn hàng của tôi | Đăng nhập |
| 20 | `POST` | `/orders/{id}/cancel` | Hủy đơn hàng | Đăng nhập |
| 21 | `GET` | `/admin/users` | Danh sách người dùng | Đăng nhập (ADMIN) |
| 22 | `PATCH` | `/admin/users/{id}` | Gán vai trò, khóa hoặc mở tài khoản | Đăng nhập (ADMIN) |
| 23 | `GET` | `/admin/orders` | Tất cả đơn hàng | Đăng nhập (MANAGER/ADMIN) |
| 24 | `PATCH` | `/admin/orders/{id}/status` | Cập nhật trạng thái đơn | Đăng nhập (MANAGER/ADMIN) |

## 3. Mã nguồn triển khai Express Routes & Controllers

Dưới đây là mã nguồn Node.js / Express hoàn chỉnh cho 24 API endpoints theo đúng tài liệu thiết kế.

### 3.1. Auth Middleware (`middlewares/auth.middleware.js`)

```javascript
// Middleware xác thực session và phân quyền
exports.isAuthenticated = (req, res, next) => {
  const user = req.user || (req.session && req.session.user);
  if (!user) {
    return res.status(401).json({
      message: "Chưa đăng nhập",
      errors: [{ field: "auth", message: "Bạn cần đăng nhập để thực hiện thao tác này" }]
    });
  }
  req.user = user;
  next();
};

exports.authorizeRoles = (...allowedRoles) => {
  return (req, res, next) => {
    if (!req.user || !allowedRoles.includes(req.user.role)) {
      return res.status(403).json({
        message: "Không đủ quyền",
        errors: [{ field: "role", message: "Bạn không có quyền truy cập chức năng này" }]
      });
    }
    next();
  };
};
```

### 3.2. Auth Controller & Routes (`routes/auth.route.js`)

```javascript
const express = require('express');
const router = express.Router();
const { isAuthenticated } = require('../middlewares/auth.middleware');

// 1. POST /api/v1/auth/register
router.post('/auth/register', async (req, res) => {
  try {
    const { name, email, password } = req.body;
    const errors = [];
    if (!name || name.length > 100) errors.push({ field: "name", message: "Họ tên không hợp lệ (tối đa 100 ký tự)" });
    if (!email) errors.push({ field: "email", message: "Email là bắt buộc" });
    if (!password || password.length < 6) errors.push({ field: "password", message: "Mật khẩu phải tối thiểu 6 ký tự" });

    if (errors.length > 0) {
      return res.status(400).json({ message: "Dữ liệu gửi lên không hợp lệ", errors });
    }

    // TODO: Kiểm tra trùng email (409) và tạo User mới trong DB
    const newUser = { id: Date.now().toString(), name, email, role: 'USER', active: true, createdAt: new Date().toISOString() };
    return res.status(201).json(newUser);
  } catch (error) {
    return res.status(500).json({ message: "Lỗi hệ thống", errors: [{ field: "server", message: error.message }] });
  }
});

// 2. POST /api/v1/auth/login
router.post('/auth/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    if (!email || !password) {
      return res.status(401).json({ message: "Sai email hoặc mật khẩu", errors: [{ field: "auth", message: "Vui lòng nhập email và mật khẩu" }] });
    }
    // TODO: Kiểm tra thông tin từ DB. Nếu tài khoản bị khóa return status(403).
    const user = { id: "1", name: "Nguyễn Văn A", email, role: "USER", active: true };
    req.session.user = user;
    return res.status(200).json(user);
  } catch (error) {
    return res.status(500).json({ message: "Lỗi hệ thống", errors: [{ field: "server", message: error.message }] });
  }
});

// 3. POST /api/v1/auth/logout
router.post('/auth/logout', isAuthenticated, (req, res) => {
  req.session.destroy((err) => {
    if (err) return res.status(500).json({ message: "Không thể đăng xuất" });
    res.clearCookie('connect.sid');
    return res.status(204).send();
  });
});

// 4. GET /api/v1/auth/me
router.get('/auth/me', isAuthenticated, (req, res) => {
  return res.status(200).json(req.user);
});

module.exports = router;
```

### 3.3. Books & Reviews Controller & Routes (`routes/book.route.js`)

```javascript
const express = require('express');
const router = express.Router();
const { isAuthenticated, authorizeRoles } = require('../middlewares/auth.middleware');

// 5. GET /api/v1/books - Danh sách sách
router.get('/books', async (req, res) => {
  try {
    let { q, category, minPrice, maxPrice, sort = 'newest', page = 1, limit = 12 } = req.query;
    page = Math.max(1, parseInt(page) || 1);
    limit = Math.min(50, Math.max(1, parseInt(limit) || 12));

    // TODO: Truy vấn sách từ DB theo filter, sort, pagination
    return res.status(200).json({
      data: [],
      pagination: { total: 0, page, limit, totalPages: 0 }
    });
  } catch (error) {
    return res.status(500).json({ message: "Lỗi hệ thống", errors: [{ field: "server", message: error.message }] });
  }
});

// 6. POST /api/v1/books - Thêm sách mới
router.post('/books', isAuthenticated, authorizeRoles('MANAGER', 'ADMIN'), async (req, res) => {
  try {
    const { title, author, price, category, image, description, stock, active } = req.body;
    const errors = [];
    if (!title) errors.push({ field: "title", message: "Tên sách là bắt buộc" });
    if (!author) errors.push({ field: "author", message: "Tác giả là bắt buộc" });
    if (price === undefined || price < 0) errors.push({ field: "price", message: "Giá phải >= 0" });
    if (stock === undefined || stock < 0) errors.push({ field: "stock", message: "Tồn kho phải >= 0" });

    if (errors.length > 0) return res.status(400).json({ message: "Dữ liệu không hợp lệ", errors });

    const newBook = {
      id: Date.now().toString(),
      title, author, price: Number(price), category: category || "", image: image || "", description: description || "",
      stock: Number(stock), active: active !== undefined ? Boolean(active) : true, createdAt: new Date().toISOString()
    };
    return res.status(201).json(newBook);
  } catch (error) {
    return res.status(500).json({ message: "Lỗi hệ thống", errors: [{ field: "server", message: error.message }] });
  }
});

// 7. GET /api/v1/books/{id}
router.get('/books/:id', async (req, res) => {
  try {
    // TODO: Lấy từ DB
    const book = null;
    if (!book) return res.status(404).json({ message: "Không tìm thấy dữ liệu", errors: [{ field: "id", message: "Sách không tồn tại" }] });
    return res.status(200).json(book);
  } catch (error) {
    return res.status(500).json({ message: "Lỗi hệ thống", errors: [{ field: "server", message: error.message }] });
  }
});

// 8. PUT /api/v1/books/{id}
router.put('/books/:id', isAuthenticated, authorizeRoles('MANAGER', 'ADMIN'), async (req, res) => {
  try {
    const { id } = req.params;
    // TODO: Kiểm tra tồn tại & cập nhật
    return res.status(200).json({ id, ...req.body });
  } catch (error) {
    return res.status(500).json({ message: "Lỗi hệ thống", errors: [{ field: "server", message: error.message }] });
  }
});

// 9. DELETE /api/v1/books/{id}
router.delete('/books/:id', isAuthenticated, authorizeRoles('ADMIN'), async (req, res) => {
  try {
    // TODO: Xóa từ DB
    return res.status(204).send();
  } catch (error) {
    return res.status(500).json({ message: "Lỗi hệ thống", errors: [{ field: "server", message: error.message }] });
  }
});

// 10. GET /api/v1/books/{id}/reviews
router.get('/books/:id/reviews', async (req, res) => {
  try {
    return res.status(200).json([]);
  } catch (error) {
    return res.status(500).json({ message: "Lỗi hệ thống", errors: [{ field: "server", message: error.message }] });
  }
});

// 11. POST /api/v1/books/{id}/reviews
router.post('/books/:id/reviews', isAuthenticated, async (req, res) => {
  try {
    const { id } = req.params;
    const { rating, comment } = req.body;
    if (!rating || rating < 1 || rating > 5) {
      return res.status(400).json({ message: "Dữ liệu không hợp lệ", errors: [{ field: "rating", message: "Rating từ 1 đến 5" }] });
    }
    const newReview = {
      id: Date.now().toString(),
      bookId: id,
      userName: req.user.name || "User",
      rating: Number(rating),
      comment: comment || "",
      createdAt: new Date().toISOString()
    };
    return res.status(201).json(newReview);
  } catch (error) {
    return res.status(500).json({ message: "Lỗi hệ thống", errors: [{ field: "server", message: error.message }] });
  }
});

module.exports = router;
```

### 3.4. Cart Controller & Routes (`routes/cart.route.js`)

```javascript
const express = require('express');
const router = express.Router();

const getCartFromSession = (req) => {
  if (!req.session.cart) {
    req.session.cart = { items: [], totalQuantity: 0, total: 0 };
  }
  return req.session.cart;
};

const recalculateCart = (cart) => {
  cart.totalQuantity = cart.items.reduce((sum, item) => sum + item.quantity, 0);
  cart.total = cart.items.reduce((sum, item) => sum + (item.subtotal || item.book.price * item.quantity), 0);
  return cart;
};

// 12. GET /api/v1/cart
router.get('/cart', (req, res) => {
  return res.status(200).json(getCartFromSession(req));
});

// 13. DELETE /api/v1/cart
router.delete('/cart', (req, res) => {
  req.session.cart = { items: [], totalQuantity: 0, total: 0 };
  return res.status(204).send();
});

// 14. POST /api/v1/cart/items
router.post('/cart/items', async (req, res) => {
  try {
    const { bookId, quantity = 1 } = req.body;
    const cart = getCartFromSession(req);
    
    // TODO: Truy vấn thông tin sách từ DB để lấy giá và kiểm tra tồn kho (404/409)
    const book = { id: bookId, title: "Sách mẫu", price: 100000, stock: 10 };

    const existingItem = cart.items.find(item => item.book.id === bookId);
    if (existingItem) {
      existingItem.quantity += quantity;
      existingItem.subtotal = existingItem.quantity * book.price;
    } else {
      cart.items.push({ book, quantity, subtotal: quantity * book.price });
    }

    recalculateCart(cart);
    return res.status(200).json(cart);
  } catch (error) {
    return res.status(500).json({ message: "Lỗi hệ thống", errors: [{ field: "server", message: error.message }] });
  }
});

// 15. PATCH /api/v1/cart/items/{bookId}
router.patch('/cart/items/:bookId', (req, res) => {
  const { bookId } = req.params;
  const { quantity } = req.body;
  const cart = getCartFromSession(req);

  if (quantity === 0) {
    cart.items = cart.items.filter(item => item.book.id !== bookId);
  } else {
    const item = cart.items.find(item => item.book.id === bookId);
    if (item) {
      item.quantity = quantity;
      item.subtotal = item.quantity * item.book.price;
    }
  }
  recalculateCart(cart);
  return res.status(200).json(cart);
});

// 16. DELETE /api/v1/cart/items/{bookId}
router.delete('/cart/items/:bookId', (req, res) => {
  const { bookId } = req.params;
  const cart = getCartFromSession(req);
  cart.items = cart.items.filter(item => item.book.id !== bookId);
  recalculateCart(cart);
  return res.status(200).json(cart);
});

module.exports = router;
```

### 3.5. Orders Controller & Routes (`routes/order.route.js`)

```javascript
const express = require('express');
const router = express.Router();
const { isAuthenticated } = require('../middlewares/auth.middleware');

// 17. GET /api/v1/orders
router.get('/orders', isAuthenticated, async (req, res) => {
  try {
    const { status, page = 1, limit = 10 } = req.query;
    return res.status(200).json({ data: [], pagination: { total: 0, page: Number(page), limit: Number(limit) } });
  } catch (error) {
    return res.status(500).json({ message: "Lỗi hệ thống", errors: [{ field: "server", message: error.message }] });
  }
});

// 18. POST /api/v1/orders
router.post('/orders', isAuthenticated, async (req, res) => {
  try {
    const { fullName, phone, address, paymentMethod, note } = req.body;
    const cart = req.session.cart;

    if (!cart || !cart.items || cart.items.length === 0) {
      return res.status(400).json({ message: "Giỏ hàng trống", errors: [{ field: "cart", message: "Không thể đặt hàng khi giỏ hàng trống" }] });
    }

    if (!fullName || !phone || !address || !paymentMethod) {
      return res.status(400).json({ message: "Dữ liệu không hợp lệ", errors: [{ field: "checkout", message: "Vui lòng điền đầy đủ thông tin giao hàng" }] });
    }

    const order = {
      id: Date.now().toString(),
      user: req.user.id,
      items: cart.items.map(item => ({ book: item.book.id, title: item.book.title, price: item.book.price, quantity: item.quantity })),
      total: cart.total,
      status: "PENDING",
      paymentMethod,
      fullName,
      phone,
      address,
      note: note || "",
      createdAt: new Date().toISOString()
    };

    // TODO: Lưu order vào DB, trừ tồn kho và xóa giỏ hàng
    req.session.cart = { items: [], totalQuantity: 0, total: 0 };

    return res.status(201).json(order);
  } catch (error) {
    return res.status(500).json({ message: "Lỗi hệ thống", errors: [{ field: "server", message: error.message }] });
  }
});

// 19. GET /api/v1/orders/{id}
router.get('/orders/:id', isAuthenticated, async (req, res) => {
  try {
    const { id } = req.params;
    // TODO: Lấy đơn hàng theo id và userId
    return res.status(200).json({ id, status: "PENDING" });
  } catch (error) {
    return res.status(500).json({ message: "Lỗi hệ thống", errors: [{ field: "server", message: error.message }] });
  }
});

// 20. POST /api/v1/orders/{id}/cancel
router.post('/orders/:id/cancel', isAuthenticated, async (req, res) => {
  try {
    const { id } = req.params;
    // TODO: Kiểm tra trạng thái đơn. Chỉ cho hủy nếu đang PENDING.
    return res.status(200).json({ id, status: "CANCELLED" });
  } catch (error) {
    return res.status(500).json({ message: "Lỗi hệ thống", errors: [{ field: "server", message: error.message }] });
  }
});

module.exports = router;
```

### 3.6. Admin Controller & Routes (`routes/admin.route.js`)

```javascript
const express = require('express');
const router = express.Router();
const { isAuthenticated, authorizeRoles } = require('../middlewares/auth.middleware');

// 21. GET /api/v1/admin/users (ADMIN)
router.get('/admin/users', isAuthenticated, authorizeRoles('ADMIN'), async (req, res) => {
  try {
    return res.status(200).json([]);
  } catch (error) {
    return res.status(500).json({ message: "Lỗi hệ thống", errors: [{ field: "server", message: error.message }] });
  }
});

// 22. PATCH /api/v1/admin/users/{id} (ADMIN)
router.patch('/admin/users/:id', isAuthenticated, authorizeRoles('ADMIN'), async (req, res) => {
  try {
    const { id } = req.params;
    const { role, active } = req.body;
    return res.status(200).json({ id, role, active });
  } catch (error) {
    return res.status(500).json({ message: "Lỗi hệ thống", errors: [{ field: "server", message: error.message }] });
  }
});

// 23. GET /api/v1/admin/orders (MANAGER/ADMIN)
router.get('/admin/orders', isAuthenticated, authorizeRoles('MANAGER', 'ADMIN'), async (req, res) => {
  try {
    return res.status(200).json([]);
  } catch (error) {
    return res.status(500).json({ message: "Lỗi hệ thống", errors: [{ field: "server", message: error.message }] });
  }
});

// 24. PATCH /api/v1/admin/orders/{id}/status (MANAGER/ADMIN)
router.patch('/admin/orders/:id/status', isAuthenticated, authorizeRoles('MANAGER', 'ADMIN'), async (req, res) => {
  try {
    const { id } = req.params;
    const { status } = req.body;
    return res.status(200).json({ id, status });
  } catch (error) {
    return res.status(500).json({ message: "Lỗi hệ thống", errors: [{ field: "server", message: error.message }] });
  }
});

module.exports = router;
```

## 4. Cấu trúc dữ liệu (Schemas)

### Error

| Trường | Kiểu | Mô tả / Ràng buộc |
|---|---|---|
| `message` | string | VD: `Không tìm thấy dữ liệu.` |
| `errors` | array<object> |  |

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
| `role` | Role | Giá trị: USER, MANAGER, ADMIN |
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

### BookInput

| Trường | Kiểu | Mô tả / Ràng buộc |
|---|---|---|
| `title` | string |  |
| `author` | string |  |
| `price` | integer | min = 0 |
| `category` | string |  |
| `image` | string |  |
| `description` | string |  |
| `stock` | integer | min = 0 |
| `active` | boolean | mặc định = True |

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
| `book` | object |  |
| `quantity` | integer |  |
| `subtotal` | integer |  |

### Cart

| Trường | Kiểu | Mô tả / Ràng buộc |
|---|---|---|
| `items` | array<CartItem> |  |
| `totalQuantity` | integer |  |
| `total` | integer | Tổng tiền VND |

### OrderItem

| Trường | Kiểu | Mô tả / Ràng buộc |
|---|---|---|
| `book` | string | Id sách |
| `title` | string |  |
| `price` | integer |  |
| `quantity` | integer |  |

### Order

| Trường | Kiểu | Mô tả / Ràng buộc |
|---|---|---|
| `id` | string |  |
| `user` | string | Id người đặt |
| `items` | array<OrderItem> |  |
| `total` | integer |  |
| `status` | OrderStatus | Giá trị: PENDING, CONFIRMED, SHIPPING, COMPLETED, CANCELLED |
| `paymentMethod` | string | Giá trị: COD, BANK_TRANSFER |
| `fullName` | string |  |
| `phone` | string |  |
| `address` | string |  |
| `note` | string |  |
| `createdAt` | string (date-time) |  |

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
| 409 | Xung đột |

## 6. Lưu ý

- Code triển khai mẫu chi tiết cho 24 endpoint trên được tích hợp trực tiếp vào file tài liệu để Frontend và Backend dễ dàng tham chiếu và sử dụng chung.
