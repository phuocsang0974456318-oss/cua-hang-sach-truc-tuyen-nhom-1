# BÁO CÁO THIẾT KẾ CHI TIẾT RESTFUL API VÀ MÃ NGUỒN FULL-STACK (BACKEND & FRONTEND)

## MÔN HỌC: PHÂN TÍCH VÀ THIẾT KẾ HỆ THỐNG THÔNG TIN
### Đề tài: Hệ thống Quản lý và Cửa hàng Sách Trực tuyến (Online Bookstore)
* **Phiên bản API**: `v1.0.0`
* **Base URL**: `http://localhost:3000/api/v1`
* **Kiến trúc hệ thống**: Client-Server 3-Tier Layered Architecture
  - **Frontend**: ReactJS (Vite, Axios Client, State Management)
  - **Backend**: Node.js, Express.js (Router, Controller, Middleware)
  - **Database**: MongoDB với Mongoose ODM (Transactions, ACID Locking, Indexes)

---

## 1. QUY ƯỚC KỸ THUẬT CHUNG VÀ CHUẨN HÓA HỆ THỐNG

### 1.1. Chuẩn hóa Phân trang (Pagination Standard)
Các API trả về danh sách có phân trang (`GET /books`, `GET /orders`, `GET /admin/users`, `GET /admin/orders`) bắt buộc trả về định dạng:

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

### 1.2. Chuẩn hóa Cấu trúc Phản hồi Lỗi (Error Handling Standard)
Tất cả ngoại lệ `4xx` và `5xx` đều tuân theo định dạng chuẩn:

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

## 2. MA TRẬN PHÂN QUYỀN VÀ DANH MỤC ENDPOINTS

| STT | Phân hệ | Method | Endpoint | Tên chức năng | Quyền | Success Status |
|---|---|---|---|---|---|---|
| **1** | Auth | `POST` | `/auth/register` | Đăng ký tài khoản | Công khai | `201 Created` |
| **2** | Auth | `POST` | `/auth/login` | Đăng nhập hệ thống | Công khai | `200 OK` |
| **3** | Auth | `POST` | `/auth/logout` | Đăng xuất phiên làm việc | Logged-in | `204 No Content` |
| **4** | Auth | `GET` | `/auth/me` | Lấy thông tin cá nhân | Logged-in | `200 OK` |
| **5** | Books | `GET` | `/books` | Danh sách sách (Tìm kiếm/Lọc/Phân trang) | Công khai | `200 OK` |
| **6** | Books | `POST` | `/books` | Thêm sách mới | `MANAGER`, `ADMIN` | `201 Created` |
| **7** | Books | `GET` | `/books/:id` | Chi tiết 1 cuốn sách | Công khai | `200 OK` |
| **8** | Books | `PUT` | `/books/:id` | Cập nhật thông tin sách | `MANAGER`, `ADMIN` | `200 OK` |
| **9** | Books | `DELETE` | `/books/:id` | Xóa sách (Soft Delete) | `ADMIN` | `204 No Content` |
| **10** | Cart | `GET` | `/cart` | Xem chi tiết giỏ hàng | Công khai | `200 OK` |
| **11** | Cart | `POST` | `/cart/items` | Thêm sách vào giỏ hàng | Công khai | `200 OK` |
| **12** | Cart | `PATCH` | `/cart/items/:bookId` | Cập nhật số lượng sách trong giỏ | Công khai | `200 OK` |
| **13** | Cart | `DELETE` | `/cart/items/:bookId` | Xóa 1 sách khỏi giỏ | Công khai | `200 OK` |
| **14** | Orders | `POST` | `/orders` | Tạo đơn hàng (Checkout Transaction) | Logged-in | `201 Created` |
| **15** | Orders | `GET` | `/orders` | Lịch sử đơn hàng cá nhân | Logged-in | `200 OK` |
| **16** | Orders | `POST` | `/orders/:id/cancel` | Hủy đơn hàng | Logged-in | `200 OK` |
| **17** | Admin | `PATCH` | `/admin/orders/:id/status` | Cập nhật trạng thái đơn (State Machine) | `MANAGER`, `ADMIN` | `200 OK` |

---

## 3. THIẾT KẾ CHI TIẾT CÁC API CỐT LÕI (FULL-STACK INTEGRATION)

### 3.1. API Đăng Nhập Hệ Thống (`POST /auth/login`)

#### A. Bảng Đặc Tả Đầu Vào (Input) & Đầu Ra (Output)

* **Tham số Đầu vào (Request Body)**:

| Tên trường | Kiểu dữ liệu | Bắt buộc | Ràng buộc / Miền giá trị | Mô tả |
|---|---|---|---|---|
| `email` | String | Có | Format email hợp lệ, max 100 ký tự | Email đăng ký của tài khoản |
| `password` | String | Có | Độ dài từ 6 - 50 ký tự | Mật khẩu tài khoản |

* **Cấu trúc Đầu ra khi THÀNH CÔNG (`200 OK`)**:

| Tên trường | Kiểu dữ liệu | Mô tả |
|---|---|---|
| `id` | String | Identifier duy nhất của User |
| `name` | String | Họ và tên người dùng |
| `email` | String | Địa chỉ email người dùng |
| `role` | String | Vai trò (`USER`, `MANAGER`, `ADMIN`) |
| `active` | Boolean | Trạng thái hoạt động tài khoản (`true` / `false`) |
| `createdAt` | String (ISO Date) | Thời gian tạo tài khoản |

* **Cấu trúc Đầu ra khi THẤT BẠI**:
  - `400 Bad Request`: Thiếu thông tin `email` hoặc `password`.
  - `401 Unauthorized`: Sai `email` hoặc `password`.
  - `403 Forbidden`: Tài khoản bị khóa (`active: false`).

#### B. Mã nguồn Backend (`src/controllers/auth.controller.js`)

##### 1. Khối code Xử lý THÀNH CÔNG (Success Path):
```javascript
// BACKEND: Khối logic Xử lý Đăng nhập THÀNH CÔNG
exports.loginSuccessHandler = async (req, res, user) => {
  // Regenerate Session để phòng chống tấn công Session Fixation Attack
  req.session.regenerate(async (err) => {
    if (err) {
      return res.status(500).json({
        statusCode: 500,
        message: "Không thể khởi tạo phiên làm việc",
        error: "Internal Server Error"
      });
    }

    // Lưu thông tin người dùng vào Session
    req.session.userId = user._id;
    req.session.role = user.role;

    // Logic gộp giỏ hàng vãng lai (guestCart) vào CSDL nếu có
    if (req.session.guestCart && req.session.guestCart.length > 0) {
      await mergeGuestCartToUser(user._id, req.session.guestCart);
      req.session.guestCart = null;
    }

    // TRẢ VỀ RESPONSE THÀNH CÔNG (200 OK)
    return res.status(200).json({
      id: user._id,
      name: user.name,
      email: user.email,
      role: user.role,
      active: user.active,
      createdAt: user.createdAt
    });
  });
};
```

##### 2. Khối code Xử lý THẤT BẠI (Failure & Validation Path):
```javascript
// BACKEND: Khối logic Kiểm tra Ràng buộc & Bắt lỗi THẤT BẠI
const User = require('../models/User.model');
const bcrypt = require('bcryptjs');

exports.login = async (req, res) => {
  try {
    const { email, password } = req.body;

    // THẤT BẠI 1: Thiếu thông tin đầu vào (400 Bad Request)
    if (!email || !password) {
      return res.status(400).json({
        statusCode: 400,
        message: "Email và mật khẩu không được để trống",
        error: "Bad Request",
        errors: [{ field: !email ? "email" : "password", message: "Trường này là bắt buộc" }]
      });
    }

    // Tìm kiếm người dùng theo Email
    const user = await User.findOne({ email: email.toLowerCase().trim() });

    // THẤT BẠI 2: Sai email (401 Unauthorized)
    if (!user) {
      return res.status(401).json({
        statusCode: 401,
        message: "Email hoặc mật khẩu không chính xác",
        error: "Unauthorized"
      });
    }

    // Kiểm tra mật khẩu mã hóa Bcrypt
    const isPasswordValid = await bcrypt.compare(password, user.password);

    // THẤT BẠI 3: Sai mật khẩu (401 Unauthorized)
    if (!isPasswordValid) {
      return res.status(401).json({
        statusCode: 401,
        message: "Email hoặc mật khẩu không chính xác",
        error: "Unauthorized"
      });
    }

    // THẤT BẠI 4: Tài khoản bị khóa (403 Forbidden)
    if (user.active === false) {
      return res.status(403).json({
        statusCode: 403,
        message: "Tài khoản của bạn đã bị khóa bởi quản trị viên",
        error: "Forbidden"
      });
    }

    // NẾU TẤT CẢ HỢP LỆ -> CHUYỂN SANG KHỐI XỬ LÝ THÀNH CÔNG
    return exports.loginSuccessHandler(req, res, user);

  } catch (error) {
    // THẤT BẠI 5: Lỗi máy chủ (500 Internal Server Error)
    return res.status(500).json({
      statusCode: 500,
      message: "Lỗi hệ thống máy chủ",
      error: error.message
    });
  }
};
```

#### C. Mã nguồn Frontend React Component (`src/pages/LoginPage.jsx`)

##### 1. Khối code Frontend Xử lý THÀNH CÔNG:
```jsx
// FRONTEND: Khối Xử lý UI/UX khi Đăng nhập THÀNH CÔNG
import React, { useState } from 'react';
import axiosClient from '../api/axiosClient';

export function handleLoginSuccess(userData, navigate, setAuthContext) {
  // 1. Cập nhật trạng thái người dùng toàn ứng dụng (Context API)
  setAuthContext({ user: userData, isAuthenticated: true });

  // 2. Thông báo thành công
  alert(`Đăng nhập thành công! Chào mừng ${userData.name}`);

  // 3. Điều hướng người dùng dựa vào vai trò (Role-based Navigation)
  if (userData.role === 'ADMIN' || userData.role === 'MANAGER') {
    navigate('/admin/dashboard');
  } else {
    navigate('/');
  }
}
```

##### 2. Khối code Frontend Xử lý THẤT BẠI & Render Form:
```jsx
// FRONTEND: Khối Render Form & Bắt lỗi THẤT BẠI từ API
export default function LoginPage({ navigate, setAuthContext }) {
  const [formData, setFormData] = useState({ email: '', password: '' });
  const [errorMessage, setErrorMessage] = useState('');
  const [isLoading, setIsLoading] = useState(false);

  const handleSubmit = async (e) => {
    e.preventDefault();
    setIsLoading(true);
    setErrorMessage('');

    try {
      // Gửi Yêu cầu Đăng nhập đến Backend
      const response = await axiosClient.post('/auth/login', formData);
      
      // GỌI KHỐI XỬ LÝ THÀNH CÔNG
      handleLoginSuccess(response, navigate, setAuthContext);

    } catch (error) {
      // GỌI KHỐI XỬ LÝ THẤT BẠI (Bắt mã lỗi HTTP Status)
      if (error.statusCode === 400) {
        setErrorMessage('Vui lòng điền đầy đủ email và mật khẩu.');
      } else if (error.statusCode === 401) {
        setErrorMessage('Email hoặc mật khẩu không đúng.');
      } else if (error.statusCode === 403) {
        setErrorMessage(error.message || 'Tài khoản của bạn hiện đã bị khóa.');
      } else {
        setErrorMessage('Không thể kết nối đến máy chủ. Vui lòng thử lại sau.');
      }
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <div className="login-container">
      <h2>Đăng Nhập Tài Khoản</h2>
      
      {/* THẤT BẠI: Hiển thị thông báo lỗi lên giao diện */}
      {errorMessage && <div className="alert alert-danger">{errorMessage}</div>}

      <form onSubmit={handleSubmit}>
        <div className="form-group">
          <label>Email Address</label>
          <input 
            type="email" 
            value={formData.email} 
            onChange={(e) => setFormData({ ...formData, email: e.target.value })}
            required 
          />
        </div>
        <div className="form-group">
          <label>Password</label>
          <input 
            type="password" 
            value={formData.password} 
            onChange={(e) => setFormData({ ...formData, password: e.target.value })}
            required 
          />
        </div>
        <button type="submit" disabled={isLoading} className="btn-submit">
          {isLoading ? 'Đang xác thực...' : 'Đăng Nhập'}
        </button>
      </form>
    </div>
  );
}
```

---

### 3.2. API Thêm Sản Phẩm Vào Giỏ Hàng (`POST /cart/items`)

#### A. Bảng Đặc Tả Đầu Vào (Input) & Đầu Ra (Output)

* **Tham số Đầu vào (Request Body)**:

| Tên trường | Kiểu dữ liệu | Bắt buộc | Ràng buộc / Miền giá trị | Mô tả |
|---|---|---|---|---|
| `bookId` | String | Có | Bắt buộc đúng format ObjectId | Mã ID cuốn sách cần thêm |
| `quantity` | Integer | Có | Min = 1, Max = 99 | Số lượng sách muốn thêm |

* **Cấu trúc Đầu ra khi THÀNH CÔNG (`200 OK`)**:

| Tên trường | Kiểu dữ liệu | Mô tả |
|---|---|---|
| `statusCode` | Integer | Giá trị `200` |
| `message` | String | Thông báo `"Thêm vào giỏ hàng thành công"` |
| `data.items` | Array | Danh sách các món hàng trong giỏ sau khi cập nhật |
| `data.totalQuantity` | Integer | Tổng số lượng tất cả sách trong giỏ |
| `data.totalAmount` | Number | Tổng thành tiền của giỏ hàng |

* **Cấu trúc Đầu ra khi THẤT BẠI**:
  - `400 Bad Request`: `bookId` không hợp lệ hoặc `quantity` $\le 0$.
  - `404 Not Found`: Không tìm thấy sách trong CSDL.
  - `409 Conflict`: Số lượng yêu cầu thêm vượt quá số lượng tồn kho (`stock`).

#### B. Mã nguồn Backend (`src/controllers/cart.controller.js`)

##### 1. Khối code Xử lý THÀNH CÔNG (Success Path):
```javascript
// BACKEND: Khối Xử lý Cập nhật Giỏ hàng THÀNH CÔNG
exports.addToCartSuccessHandler = (req, res, cart, book, quantity, existingIndex) => {
  // Nếu sách đã có trong giỏ -> Cộng dồn số lượng
  if (existingIndex > -1) {
    cart.items[existingIndex].quantity += quantity;
    cart.items[existingIndex].subtotal = cart.items[existingIndex].quantity * book.price;
  } else {
    // Nếu sách chưa có -> Thêm mới item vào giỏ
    cart.items.push({
      bookId: book._id.toString(),
      title: book.title,
      price: book.price,
      image: book.image,
      quantity: quantity,
      subtotal: book.price * quantity
    });
  }

  // Tính toán lại tổng tiền và tổng số lượng giỏ hàng
  cart.totalQuantity = cart.items.reduce((sum, item) => sum + item.quantity, 0);
  cart.totalAmount = cart.items.reduce((sum, item) => sum + item.subtotal, 0);

  // Lưu giỏ hàng vào Session
  req.session.cart = cart;

  // TRẢ VỀ RESPONSE THÀNH CÔNG (200 OK)
  return res.status(200).json({
    statusCode: 200,
    message: "Thêm vào giỏ hàng thành công",
    data: cart
  });
};
```

##### 2. Khối code Xử lý THẤT BẠI (Failure & Validation Path):
```javascript
// BACKEND: Khối Kiểm tra Tồn kho & Bắt lỗi THẤT BẠI
const Book = require('../models/Book.model');

exports.addToCart = async (req, res) => {
  try {
    const { bookId, quantity } = req.body;

    // THẤT BẠI 1: Dữ liệu đầu vào sai (400 Bad Request)
    if (!bookId || !Number.isInteger(quantity) || quantity <= 0) {
      return res.status(400).json({
        statusCode: 400,
        message: "ID sách không hợp lệ hoặc số lượng phải lớn hơn 0",
        error: "Bad Request"
      });
    }

    const book = await Book.findById(bookId);

    // THẤT BẠI 2: Sách không tồn tại hoặc đã bị ẩn (404 Not Found)
    if (!book || book.active === false) {
      return res.status(404).json({
        statusCode: 404,
        message: "Sách không tồn tại hoặc đã ngừng kinh doanh",
        error: "Not Found"
      });
    }

    // Khởi tạo giỏ hàng nếu chưa tồn tại
    if (!req.session.cart) {
      req.session.cart = { items: [], totalQuantity: 0, totalAmount: 0 };
    }

    const cart = req.session.cart;
    const existingIndex = cart.items.findIndex(item => item.bookId === bookId);
    const currentQtyInCart = existingIndex > -1 ? cart.items[existingIndex].quantity : 0;
    const requestedTotalQty = currentQtyInCart + quantity;

    // THẤT BẠI 3: Vượt quá tồn kho thực tế (409 Conflict)
    if (requestedTotalQty > book.stock) {
      return res.status(409).json({
        statusCode: 409,
        message: `Không thể thêm. Kho chỉ còn ${book.stock} cuốn (Bạn đã có ${currentQtyInCart} cuốn trong giỏ).`,
        error: "Conflict"
      });
    }

    // NẾU HỢP LỆ -> CHUYỂN SANG KHỐI XỬ LÝ THÀNH CÔNG
    return exports.addToCartSuccessHandler(req, res, cart, book, quantity, existingIndex);

  } catch (error) {
    // THẤT BẠI 4: Lỗi Server (500 Internal Server Error)
    return res.status(500).json({
      statusCode: 500,
      message: "Lỗi hệ thống khi thêm sản phẩm vào giỏ",
      error: error.message
    });
  }
};
```

#### C. Mã nguồn Frontend React Component (`src/components/AddToCartButton.jsx`)

##### 1. Khối code Frontend Xử lý THÀNH CÔNG:
```jsx
// FRONTEND: Khối Xử lý THÀNH CÔNG khi thêm sản phẩm vào giỏ
export function handleAddToCartSuccess(updatedCart, updateCartState) {
  // 1. Cập nhật State giỏ hàng trên Header/Navbar
  updateCartState(updatedCart);

  // 2. Hiển thị Toast thông báo thành công
  alert("Đã thêm sản phẩm vào giỏ hàng thành công!");
}
```

##### 2. Khối code Frontend Xử lý THẤT BẠI & Render Component:
```jsx
// FRONTEND: Khối Nút Thêm Vào Giỏ & Bắt lỗi THẤT BẠI
import React, { useState } from 'react';
import axiosClient from '../api/axiosClient';

export default function AddToCartButton({ bookId, currentStock, updateCartState }) {
  const [quantity, setQuantity] = useState(1);
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleAddToCart = async () => {
    setIsSubmitting(true);
    try {
      // Gửi Request thêm giỏ hàng
      const response = await axiosClient.post('/cart/items', {
        bookId: bookId,
        quantity: quantity
      });

      // KHỐI THÀNH CÔNG
      handleAddToCartSuccess(response.data, updateCartState);

    } catch (error) {
      // KHỐI THẤT BẠI: Bắt mã lỗi từ Backend
      if (error.statusCode === 409) {
        alert(`[CẢNH BÁO TỒN KHO]: ${error.message}`);
      } else if (error.statusCode === 404) {
        alert("Sản phẩm này không còn tồn tại.");
      } else {
        alert("Lỗi hệ thống! Vui lòng thử lại sau.");
      }
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <div className="add-to-cart-widget">
      <input 
        type="number" 
        min="1" 
        max={currentStock} 
        value={quantity} 
        onChange={(e) => setQuantity(parseInt(e.target.value) || 1)}
      />
      <button 
        onClick={handleAddToCart} 
        disabled={isSubmitting || currentStock <= 0}
        className="btn-primary"
      >
        {currentStock <= 0 ? 'Hết hàng' : isSubmitting ? 'Đang thêm...' : 'Thêm Vào Giỏ'}
      </button>
    </div>
  );
}
```

---

### 3.3. API Tạo Đơn Hàng / Checkout (`POST /orders`)

#### A. Bảng Đặc Tả Đầu Vào (Input) & Đầu Ra (Output)

* **Tham số Đầu vào (Request Body)**:

| Tên trường | Kiểu dữ liệu | Bắt buộc | Ràng buộc / Miền giá trị | Mô tả |
|---|---|---|---|---|
| `fullName` | String | Có | Độ dài từ 2 - 100 ký tự | Họ tên người nhận hàng |
| `phone` | String | Có | Chuỗi 10 chữ số hợp lệ | Số điện thoại liên hệ |
| `address` | String | Có | Độ dài từ 10 - 300 ký tự | Địa chỉ giao hàng chi tiết |
| `paymentMethod`| String | Có | `COD` hoặc `BANK` | Phương thức thanh toán |
| `note` | String | Không | Max 500 ký tự | Ghi chú đơn hàng |

* **Cấu trúc Đầu ra khi THÀNH CÔNG (`201 Created`)**:

| Tên trường | Kiểu dữ liệu | Mô tả |
|---|---|---|
| `statusCode` | Integer | Giá trị `201` |
| `message` | String | `"Tạo đơn hàng thành công"` |
| `data.id` | String | Mã đơn hàng vừa khởi tạo |
| `data.totalAmount` | Number | Tổng tiền thanh toán của đơn hàng |
| `data.status` | String | Trạng thái ban đầu: `"PENDING"` |

* **Cấu trúc Đầu ra khi THẤT BẠI**:
  - `400 Bad Request`: Giỏ hàng trống hoặc thiếu thông tin nhận hàng.
  - `401 Unauthorized`: Chưa đăng nhập.
  - `409 Conflict`: Tranh chấp tồn kho (Sách bị hết hàng trong lúc thao tác checkout).

#### B. Mã nguồn Backend (`src/controllers/order.controller.js`) - ACID Transaction

##### 1. Khối code Xử lý THÀNH CÔNG (Success Path):
```javascript
// BACKEND: Khối Xử lý Tạo Đơn hàng & Commit Transaction THÀNH CÔNG
exports.createOrderSuccessHandler = async (req, res, session, userId, orderItems, totalAmount, shippingInfo) => {
  const Order = require('../models/Order.model');

  // Khởi tạo đơn hàng mới trong CSDL
  const newOrder = new Order({
    user: userId,
    items: orderItems,
    totalAmount: totalAmount,
    status: "PENDING",
    paymentMethod: shippingInfo.paymentMethod || "COD",
    fullName: shippingInfo.fullName,
    phone: shippingInfo.phone,
    address: shippingInfo.address,
    note: shippingInfo.note || ""
  });

  await newOrder.save({ session });

  // Làm rỗng giỏ hàng Session sau khi mua hàng thành công
  req.session.cart = { items: [], totalQuantity: 0, totalAmount: 0 };

  // COMMIT TRANSACTION
  await session.commitTransaction();
  session.endSession();

  // TRẢ VỀ RESPONSE THÀNH CÔNG (201 Created)
  return res.status(201).json({
    statusCode: 201,
    message: "Tạo đơn hàng thành công",
    data: newOrder
  });
};
```

##### 2. Khối code Xử lý THẤT BẠI & Rollback (Failure & Validation Path):
```javascript
// BACKEND: Khối Kiểm tra Tranh chấp Kho & Rollback Transaction THẤT BẠI
const mongoose = require('mongoose');
const Book = require('../models/Book.model');

exports.createOrder = async (req, res) => {
  const session = await mongoose.startSession();
  session.startTransaction();

  try {
    const { fullName, phone, address, paymentMethod, note } = req.body;
    const userId = req.session.userId;

    // THẤT BẠI 1: Thiếu thông tin người nhận (400 Bad Request)
    if (!fullName || !phone || !address) {
      await session.endSession();
      return res.status(400).json({
        statusCode: 400,
        message: "Vui lòng cung cấp đầy đủ tên, số điện thoại và địa chỉ giao hàng",
        error: "Bad Request"
      });
    }

    const cart = req.session.cart;

    // THẤT BẠI 2: Giỏ hàng rỗng (400 Bad Request)
    if (!cart || !cart.items || cart.items.length === 0) {
      await session.endSession();
      return res.status(400).json({
        statusCode: 400,
        message: "Giỏ hàng rỗng, không thể tiến hành đặt hàng",
        error: "Bad Request"
      });
    }

    let totalAmount = 0;
    const orderItems = [];

    // DUYỆT QUA TỪNG ITEM VÀ KHÓA DÒNG TỒN KHO (FOR UPDATE LOCKING)
    for (const item of cart.items) {
      const book = await Book.findById(item.bookId).session(session);

      if (!book || !book.active) {
        await session.abortTransaction();
        session.endSession();
        return res.status(404).json({
          statusCode: 404,
          message: `Sách '${item.title}' không tồn tại hoặc đã ngưng bán`
        });
      }

      // THẤT BẠI 3: Tranh chấp kho hàng - Bị hết hàng (409 Conflict)
      if (item.quantity > book.stock) {
        await session.abortTransaction();
        session.endSession();
        return res.status(409).json({
          statusCode: 409,
          message: `Thanh toán thất bại! Cuốn sách '${book.title}' chỉ còn ${book.stock} cuốn trong kho.`,
          error: "Conflict"
        });
      }

      // Trừ số lượng tồn kho
      book.stock -= item.quantity;
      await book.save({ session });

      totalAmount += book.price * item.quantity;
      orderItems.push({
        bookId: book._id,
        title: book.title,
        price: book.price,
        quantity: item.quantity,
        subtotal: book.price * item.quantity
      });
    }

    // NẾU MỌI THỨ HỢP LỆ -> CHUYỂN SANG KHỐI THÀNH CÔNG VÀ COMMIT
    return await exports.createOrderSuccessHandler(
      req, res, session, userId, orderItems, totalAmount, 
      { fullName, phone, address, paymentMethod, note }
    );

  } catch (error) {
    // THẤT BẠI 4: Lỗi bất ngờ -> ROLLBACK TOÀN BỘ TRANSACTIONS
    await session.abortTransaction();
    session.endSession();
    return res.status(500).json({
      statusCode: 500,
      message: "Lỗi hệ thống khi xử lý thanh toán",
      error: error.message
    });
  }
};
```

#### C. Mã nguồn Frontend React Component (`src/pages/CheckoutPage.jsx`)

##### 1. Khối code Frontend Xử lý THÀNH CÔNG:
```jsx
// FRONTEND: Khối Xử lý THÀNH CÔNG sau khi Đặt Hàng
export function handleCheckoutSuccess(orderData, navigate) {
  alert(`Đặt hàng thành công! Mã đơn hàng của bạn là: ${orderData.id}`);
  // Điều hướng về trang chi tiết đơn hàng
  navigate(`/orders/${orderData.id}`);
}
```

##### 2. Khối code Frontend Xử lý THẤT BẠI & Render Form:
```jsx
// FRONTEND: Màn hình Checkout & Bắt lỗi THẤT BẠI
import React, { useState } from 'react';
import axiosClient from '../api/axiosClient';

export default function CheckoutPage({ navigate }) {
  const [formData, setFormData] = useState({
    fullName: '',
    phone: '',
    address: '',
    paymentMethod: 'COD',
    note: ''
  });
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [errorMsg, setErrorMsg] = useState('');

  const handleCheckout = async (e) => {
    e.preventDefault();
    setIsSubmitting(true);
    setErrorMsg('');

    try {
      // Gửi Yêu cầu Checkout
      const response = await axiosClient.post('/orders', formData);
      
      // KHỐI THÀNH CÔNG
      handleCheckoutSuccess(response.data, navigate);

    } catch (error) {
      // KHỐI THẤT BẠI: Hiển thị lỗi từ Backend
      if (error.statusCode === 409) {
        setErrorMsg(`[XUNG ĐỘT KHO HÀNG]: ${error.message}`);
      } else if (error.statusCode === 400) {
        setErrorMsg(error.message || 'Vui lòng kiểm tra lại thông tin nhận hàng.');
      } else {
        setErrorMsg('Không thể xử lý đơn hàng lúc này. Vui lòng thử lại sau.');
      }
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <div className="checkout-container">
      <h2>Thanh Toán Đơn Hàng</h2>
      
      {/* KHỐI THẤT BẠI: Render Alert lỗi */}
      {errorMsg && <div className="alert alert-danger">{errorMsg}</div>}

      <form onSubmit={handleCheckout}>
        <div className="form-group">
          <label>Họ và Tên Người Nhận (*)</label>
          <input 
            type="text" 
            value={formData.fullName} 
            onChange={(e) => setFormData({ ...formData, fullName: e.target.value })}
            required 
          />
        </div>
        <div className="form-group">
          <label>Số Điện Thoại (*)</label>
          <input 
            type="text" 
            value={formData.phone} 
            onChange={(e) => setFormData({ ...formData, phone: e.target.value })}
            required 
          />
        </div>
        <div className="form-group">
          <label>Địa Chỉ Giao Hàng (*)</label>
          <textarea 
            value={formData.address} 
            onChange={(e) => setFormData({ ...formData, address: e.target.value })}
            required 
          />
        </div>
        <button type="submit" disabled={isSubmitting} className="btn-checkout">
          {isSubmitting ? 'Đang giao dịch...' : 'Xác Nhận Đặt Hàng'}
        </button>
      </form>
    </div>
  );
}
```

---

### 3.4. API Chuyển Trạng Thái Đơn Hàng Admin (`PATCH /admin/orders/:id/status`)

#### A. Bảng Đặc Tả Đầu Vào (Input) & Đầu Ra (Output)

* **Tham số Đầu vào (Request Body & Path)**:

| Tên trường | Vị trí | Kiểu dữ liệu | Bắt buộc | Ràng buộc / Miền giá trị | Mô tả |
|---|---|---|---|---|---|
| `id` | Path | String | Có | ID đơn hàng hợp lệ | Mã đơn hàng cần chuyển |
| `status` | Body | String | Có | `CONFIRMED`, `SHIPPING`, `COMPLETED`, `CANCELLED` | Trạng thái mới của đơn hàng |

* **Cấu trúc Đầu ra khi THÀNH CÔNG (`200 OK`)**:

| Tên trường | Kiểu dữ liệu | Mô tả |
|---|---|---|
| `statusCode` | Integer | Giá trị `200` |
| `message` | String | `"Cập nhật trạng thái đơn hàng thành công"` |
| `data.id` | String | Mã đơn hàng |
| `data.status` | String | Trạng thái mới vừa cập nhật thành công |

* **Cấu trúc Đầu ra khi THẤT BẠI**:
  - `400 Bad Request`: Vi phạm quy trình State Machine (VD: Chuyển trực tiếp từ `PENDING` sang `COMPLETED`).
  - `403 Forbidden`: Người dùng không có quyền Admin/Manager.
  - `404 Not Found`: Không tìm thấy mã đơn hàng.

#### B. Mã nguồn Backend (`src/controllers/adminOrder.controller.js`)

##### 1. Khối code Xử lý THÀNH CÔNG (Success Path):
```javascript
// BACKEND: Khối Xử lý Cập nhật Trạng thái THÀNH CÔNG
exports.updateStatusSuccessHandler = async (req, res, session, order, newStatus) => {
  const Book = require('../models/Book.model');

  // NẾU HỦY ĐƠN HÀNG (CANCELLED) -> CỘNG HOÀN TRẢ SỐ LƯỢNG VÀO KHO SÁCH
  if (newStatus === 'CANCELLED') {
    for (const item of order.items) {
      await Book.findByIdAndUpdate(
        item.bookId,
        { $inc: { stock: item.quantity } },
        { session }
      );
    }
  }

  // Cập nhật trạng thái mới
  order.status = newStatus;
  await order.save({ session });

  await session.commitTransaction();
  session.endSession();

  // TRẢ VỀ RESPONSE THÀNH CÔNG (200 OK)
  return res.status(200).json({
    statusCode: 200,
    message: `Cập nhật trạng thái đơn hàng sang ${newStatus} thành công`,
    data: order
  });
};
```

##### 2. Khối code Xử lý THẤT BẠI (Failure & Validation Path):
```javascript
// BACKEND: Khối Kiểm tra State Machine & Bắt lỗi THẤT BẠI
const Order = require('../models/Order.model');
const mongoose = require('mongoose');

exports.updateOrderStatus = async (req, res) => {
  const session = await mongoose.startSession();
  session.startTransaction();

  try {
    const { id } = req.params;
    const { status: newStatus } = req.body;

    // MA TRẬN CHUYỂN TRẠNG THÁI HỢP LỆ (STATE MACHINE)
    const STATE_TRANSITIONS = {
      PENDING: ['CONFIRMED', 'CANCELLED'],
      CONFIRMED: ['SHIPPING', 'CANCELLED'],
      SHIPPING: ['COMPLETED'],
      COMPLETED: [],
      CANCELLED: []
    };

    const order = await Order.findById(id).session(session);

    // THẤT BẠI 1: Không tìm thấy đơn hàng (404 Not Found)
    if (!order) {
      await session.endSession();
      return res.status(404).json({
        statusCode: 404,
        message: "Không tìm thấy đơn hàng với ID này",
        error: "Not Found"
      });
    }

    const currentStatus = order.status;
    const allowedStatuses = STATE_TRANSITIONS[currentStatus] || [];

    // THẤT BẠI 2: Vi phạm luồng chuyển trạng thái (400 Bad Request)
    if (!allowedStatuses.includes(newStatus)) {
      await session.endSession();
      return res.status(400).json({
        statusCode: 400,
        message: `Chuyển trạng thái sai quy trình! Đơn hàng đang ở '${currentStatus}' không thể chuyển sang '${newStatus}'.`,
        error: "Bad Request"
      });
    }

    // NẾU HỢP LỆ -> CHUYỂN SANG KHỐI THÀNH CÔNG
    return await exports.updateStatusSuccessHandler(req, res, session, order, newStatus);

  } catch (error) {
    await session.abortTransaction();
    session.endSession();
    return res.status(500).json({
      statusCode: 500,
      message: "Lỗi máy chủ khi cập nhật đơn hàng",
      error: error.message
    });
  }
};
```

#### C. Mã nguồn Frontend React Component (`src/pages/AdminOrderDetail.jsx`)

##### 1. Khối code Frontend Xử lý THÀNH CÔNG:
```jsx
// FRONTEND: Khối Xử lý THÀNH CÔNG phía Admin UI
export function handleStatusUpdateSuccess(updatedOrder, setOrderState) {
  setOrderState(updatedOrder);
  alert(`Đã chuyển trạng thái đơn hàng sang: ${updatedOrder.status}`);
}
```

##### 2. Khối code Frontend Xử lý THẤT BẠI & Render Component:
```jsx
// FRONTEND: Giao diện Admin Đổi Trạng Thái & Bắt Lỗi THẤT BẠI
import React, { useState } from 'react';
import axiosClient from '../api/axiosClient';

export default function AdminOrderStatusControl({ order, setOrderState }) {
  const [selectedStatus, setSelectedStatus] = useState('');
  const [isUpdating, setIsUpdating] = useState(false);

  const handleUpdateStatus = async () => {
    if (!selectedStatus) return;
    setIsUpdating(true);

    try {
      // Gửi Yêu cầu Cập nhật Trạng thái
      const response = await axiosClient.patch(`/admin/orders/${order.id}/status`, {
        status: selectedStatus
      });

      // KHỐI THÀNH CÔNG
      handleStatusUpdateSuccess(response.data, setOrderState);

    } catch (error) {
      // KHỐI THẤT BẠI: Bắt mã lỗi từ Server
      if (error.statusCode === 400) {
        alert(`[VI PHẠM QUY TRÌNH]: ${error.message}`);
      } else if (error.statusCode === 403) {
        alert("Bạn không có quyền Admin để thực hiện thao tác này.");
      } else {
        alert("Cập nhật thất bại. Vui lòng thử lại sau.");
      }
    } finally {
      setIsUpdating(false);
    }
  };

  return (
    <div className="admin-status-control">
      <p>Trạng thái hiện tại: <strong>{order.status}</strong></p>
      
      <select value={selectedStatus} onChange={(e) => setSelectedStatus(e.target.value)}>
        <option value="">-- Chọn trạng thái mới --</option>
        <option value="CONFIRMED">CONFIRMED (Xác nhận)</option>
        <option value="SHIPPING">SHIPPING (Đang giao)</option>
        <option value="COMPLETED">COMPLETED (Hoàn thành)</option>
        <option value="CANCELLED">CANCELLED (Hủy đơn)</option>
      </select>

      <button onClick={handleUpdateStatus} disabled={isUpdating || !selectedStatus}>
        {isUpdating ? 'Đang cập nhật...' : 'Cập Nhật Trạng Thái'}
      </button>
    </div>
  );
}
```

---

## 4. BẢNG MÁP MÃ HÀNH ĐỘNG GIỮA BACKEND VÀ FRONTEND (HTTP STATUS UI MAPPING)

| HTTP Code | Ngữ cảnh Backend | Logic Xử lý Backend | Hành động UI Phía Frontend |
|---|---|---|---|
| **`200 OK`** | Truy vấn/Cập nhật thành công | Trả về dữ liệu JSON dạng object/array | Render dữ liệu mới, tắt icon Loading |
| **`201 Created`** | Tạo mới bản ghi thành công | Tạo bản ghi mới trong CSDL (`POST`) | Thông báo thành công, reset Form, điều hướng trang |
| **`400 Bad Request`** | Dữ liệu sai định dạng | Vi phạm Validation, sai State Machine | Hiển thị thông báo lỗi màu đỏ tại Form |
| **`401 Unauthorized`** | Chưa đăng nhập / Sai mật khẩu | Cookie hết hạn hoặc Credential sai | Xóa state user, chuyển hướng về `/login` |
| **`403 Forbidden`** | Không đủ thẩm quyền | Role không đạt hoặc tài khoản bị khóa | Hiển thị Toast Alert: *"Bạn không có quyền truy cập"* |
| **`404 Not Found`** | Tài nguyên không tồn tại | ID truyền trên URL không có trong DB | Chuyển sang màn hình giao diện Lỗi 404 |
| **`409 Conflict`** | Tranh chấp dữ liệu / Hết hàng | Hết tồn kho, trùng Email đăng ký | Hiển thị Alert cảnh báo tồn kho, vô hiệu hóa nút bấm |
| **`500 Internal Error`** | Lỗi hệ thống server | Lỗi đứt kết nối CSDL hoặc crash code | Hiển thị Banner: *"Hệ thống gặp sự cố, thử lại sau"* |
