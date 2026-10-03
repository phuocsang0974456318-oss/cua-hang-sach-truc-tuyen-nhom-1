# TÀI LIỆU ĐẶC TẢ YÊU CẦU PHẦN MỀM (SRS)
**Hệ thống:** Cửa Hàng Sách Trực Tuyến  
**Tài liệu tham chiếu:** Sơ đồ tư duy phân tích hệ thống  

---

## 1. PHÂN TÍCH YÊU CẦU (REQUIREMENTS ANALYSIS)

### 1.1. Yêu cầu Chức năng (Functional Requirements)
* **1.1.1. Đăng ký & Đăng nhập**
  * Hệ thống cho phép người dùng đăng ký tài khoản mới với các thông tin cá nhân cơ bản.
  * Hỗ trợ đăng nhập, đăng xuất và khôi phục mật khẩu.
  * Phân quyền truy cập theo vai trò (Khách hàng, Quản lý, Admin).
* **1.1.2. Tìm kiếm & Lọc sách**
  * Tìm kiếm đầu sách theo từ khóa (tên sách, tác giả, nhà xuất bản).
  * Lọc sách theo nhiều tiêu chí: danh mục, khoảng giá, đánh giá sao, mức độ phổ biến.
* **1.1.3. Đặt hàng & Thanh toán**
  * Thêm/sửa/xóa sản phẩm trong giỏ hàng.
  * Đặt hàng và xác nhận địa chỉ giao hàng.
  * Tích hợp thanh toán trực tuyến đa hình thức.

### 1.2. Yêu cầu Phi chức năng (Non-Functional Requirements)
* **1.2.1. Bảo mật dữ liệu & Mã hóa**
  * Mã hóa mật khẩu người dùng (ví dụ: bcrypt/SHA-256) trước khi lưu vào cơ sở dữ liệu.
  * Đảm bảo an toàn thông tin giao dịch thanh toán trực tuyến (tuân thủ các chuẩn mã hóa HTTPS/SSL).
* **1.2.2. Tốc độ xử lý & UI/UX**
  * Tốc độ phản hồi hệ thống tối ưu (thời gian tải trang < 2 giây đối với các tác vụ thông thường).
  * Giao diện người dùng (UI) trực quan, thân thiện, tương thích responsive trên cả máy tính và thiết bị di động (UX).

---

## 2. BÀI TOÁN NGHIỆP VỤ (BUSINESS LOGIC)

### 2.1. Quản lý Kho & Danh mục
* **Thêm / Sửa / Xóa đầu sách:**
  * Cho phép thêm mới sách vào hệ thống với đầy đủ thông tin: Tiêu đề, Tác giả, Giá, Danh mục, Hình ảnh, Mô tả.
  * Cập nhật thông tin chi tiết hoặc ẩn/xóa đầu sách không còn kinh doanh.
* **Cập nhật số lượng tồn kho:**
  * Tự động trừ số lượng tồn kho khi đơn hàng được đặt thành công.
  * Cho phép người quản lý cập nhật thủ công số lượng nhập kho mới.

### 2.2. Quản lý Đơn hàng
* **Xử lý trạng thái đơn hàng:**
  * Theo dõi và cập nhật tiến trình đơn hàng theo vòng đời: `Chờ xác nhận` -> `Đang xử lý` -> `Đang giao hàng` -> `Hoàn tất` (hoặc `Đã hủy`).
* **Xác nhận thanh toán:**
  * Đối soát và xác nhận trạng thái thanh toán của đơn hàng (Đã thanh toán / Chưa thanh toán / Thất bại).

### 2.3. Quản lý Khách hàng
* **Lịch sử mua & Đánh giá:**
  * Cho phép khách hàng xem lại danh sách đơn hàng đã mua và trạng thái chi tiết.
  * Cho phép khách hàng gửi đánh giá (rating) và nhận xét (review) cho các đầu sách họ đã hoàn thành đơn mua.

---

## 3. THIẾT KẾ HỆ THỐNG (SYSTEM DESIGN)

### 3.1. Sơ đồ Use Case
* **Phân quyền Actor:**
  * **Khách hàng (Customer):** Tìm kiếm sách, đặt hàng, thanh toán, xem lịch sử mua hàng, viết đánh giá.
  * **Quản lý (Manager):** Quản lý kho, xử lý đơn hàng, kiểm duyệt đánh giá khách hàng.
  * **Admin:** Quản trị người dùng, phân quyền actor, xem báo cáo thống kê toàn hệ thống.

### 3.2. Sơ đồ Dữ liệu ERD
* **Cấu trúc Cơ sở Dữ liệu (7 Bảng & Khóa PK/FK):**
  1. `Users` (Mã người dùng, Họ tên, Email, Mật khẩu, Vai trò, ...)
  2. `Books` (Mã sách, Tên sách, Giá, Số lượng tồn, Mã danh mục, ...)
  3. `Categories` (Mã danh mục, Tên danh mục, ...)
  4. `Orders` (Mã đơn hàng, Mã khách hàng, Ngày đặt, Tổng tiền, Trạng thái, ...)
  5. `OrderDetails` (Mã đơn hàng, Mã sách, Số lượng, Đơn giá)
  6. `Payments` (Mã thanh toán, Mã đơn hàng, Phương thức, Trạng thái, Ngày thanh toán)
  7. `Reviews` (Mã đánh giá, Mã khách hàng, Mã sách, Số sao, Bình luận, Ngày tạo)

### 3.3. Sơ đồ Tuần tự (Sequence Diagram)
* **Luồng Đặt hàng:**
  1. Khách hàng chọn sản phẩm -> Thêm vào giỏ hàng.
  2. Khách hàng nhấn "Thanh toán" -> Hệ thống hiển thị giao diện nhập thông tin & chọn phương thức thanh toán.
  3. Khách hàng xác nhận đặt hàng -> Hệ thống gửi yêu cầu kiểm tra số lượng tồn kho.
  4. Nếu hợp lệ: Tạo đơn hàng -> Gọi API Thanh toán -> Cập nhật trạng thái đơn & Trừ số lượng kho -> Trả về thông báo thành công cho Khách hàng.

---

## 4. PHÂN CÔNG NHÓM (TEAM ASSIGNMENT)

| Thành viên | Vai trò | Nhiệm vụ chính |
| :--- | :--- | :--- |
| **Thành viên 1** | Admin | - Quản lý Người dùng (CRUD, Phân quyền)<br>- Quản lý & Xuất Báo cáo hệ thống |
| **Thành viên 2** | Quản lý | - Quản lý Kho & Danh mục sách<br>- Duyệt và quản lý Đánh giá của khách hàng |
| **Thành viên 3** | Kỹ thuật | - Thiết kế Database (ERD, Schema)<br>- Tích hợp API Payment & Luồng xử lý thanh toán |