# USER STORY: US-01 - Đăng ký tài khoản người dùng

**Mã US:** US-01  
**Epic:** E01. Quản lý Tài khoản & Phân quyền  
**Độ ưu tiên:** High  
**Thành viên đảm nhận:** Thành viên 1 (Admin)  

---

## 1. Mô tả (Description)
* **As a** Khách hàng mới,
* **I want to** đăng ký tài khoản hệ thống bằng email và mật khẩu,
* **So that** tôi có thể thực hiện mua sách, lưu giỏ hàng và theo dõi tiến độ đơn hàng.

---

## 2. Tiêu chí chấp nhận (Acceptance Criteria)
- [ ] **AC01:** Hiển thị form đăng ký gồm các trường: `Họ tên`, `Email`, `Mật khẩu`, `Nhập lại mật khẩu`, `Số điện thoại`.
- [ ] **AC02:** Kiểm tra định dạng Email hợp lệ và Mật khẩu có độ dài tối thiểu 8 ký tự.
- [ ] **AC03:** Kiểm tra trùng lặp Email; nếu Email đã tồn tại trong cơ sở dữ liệu, hiển thị thông báo lỗi phù hợp.
- [ ] **AC04:** Mã hóa mật khẩu bằng **Bcrypt** trước khi lưu vào bảng `Users`.
- [ ] **AC05:** Sau khi đăng ký thành công, tự động chuyển hướng người dùng tới trang Đăng nhập và hiển thị thông báo thành công.

---

## 3. Thông tin Kỹ thuật (Technical Notes)
* **Endpoint API:** `POST /api/auth/register`
* **Model liên quan:** `Users`
* **Tham số Request:**
  ```json
  {
    "fullName": "Nguyen Van A",
    "email": "nguyenvana@example.com",
    "password": "Password123",
    "phone": "0901234567"
  }
  ```