# USER STORY: US-02 - Đăng nhập hệ thống

**Mã US:** US-02  
**Epic:** E01. Quản lý Tài khoản & Phân quyền  
**Độ ưu tiên:** High  
**Thành viên đảm nhận:** Thành viên 1 (Admin)  

---

## 1. Mô tả (Description)
* **As a** Người dùng đã có tài khoản,
* **I want to** đăng nhập bằng Email và Mật khẩu,
* **So that** tôi có thể truy cập vào các chức năng dành riêng cho tài khoản của mình.

---

## 2. Tiêu chí chấp nhận (Acceptance Criteria)
- [ ] **AC01:** Form đăng nhập gồm các trường `Email` và `Mật khẩu`.
- [ ] **AC02:** Kiểm tra xác thực thông tin đăng nhập đối chiếu với bảng `Users`.
- [ ] **AC03:** Trả về thông báo lỗi "Email hoặc mật khẩu không chính xác" nếu thông tin nhập sai.
- [ ] **AC04:** Khi đăng nhập thành công, hệ thống sinh ra chuỗi **JWT Token** lưu thông tin phiên làm việc.
- [ ] **AC05:** Tự động điều hướng theo vai trò (Role): `Customer` sang Trang chủ, `Manager`/`Admin` sang Trang quản trị.

---

## 3. Thông tin Kỹ thuật (Technical Notes)
* **Endpoint API:** `POST /api/auth/login`
* **Model liên quan:** `Users`
* **Response:** JWT Token + Thông tin User cơ bản (id, fullName, role).