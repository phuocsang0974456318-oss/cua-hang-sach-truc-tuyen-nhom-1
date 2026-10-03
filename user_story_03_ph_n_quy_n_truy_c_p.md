# USER STORY: US-03 - Phân quyền truy cập Actor

**Mã US:** US-03  
**Epic:** E01. Quản lý Tài khoản & Phân quyền  
**Độ ưu tiên:** High  
**Thành viên đảm nhận:** Thành viên 1 (Admin)  

---

## 1. Mô tả (Description)
* **As an** Admin hệ thống,
* **I want to** hệ thống tự động kiểm tra và phân quyền truy cập theo vai trò (Actor),
* **So that** người dùng chỉ có thể thực hiện đúng các thao tác được phép.

---

## 2. Tiêu chí chấp nhận (Acceptance Criteria)
- [ ] **AC01:** **Customer:** Được quyền Xem/Tìm kiếm sách, Đặt hàng, Xem lịch sử đơn hàng, Viết đánh giá.
- [ ] **AC02:** **Manager:** Được quyền Quản lý Kho sách (CRUD), Cập nhật trạng thái đơn hàng, Duyệt đánh giá.
- [ ] **AC03:** **Admin:** Có đầy đủ mọi quyền + Quản lý người dùng + Xem báo cáo doanh thu.
- [ ] **AC04:** Xây dựng Middleware xác thực token và kiểm tra role tại các Endpoint API Backend.
- [ ] **AC05:** Trả về mã lỗi `403 Forbidden` khi người dùng cố tình truy cập tính năng không đúng thẩm quyền.

---

## 3. Thông tin Kỹ thuật (Technical Notes)
* **Middlewares:** `authMiddleware.js`, `roleMiddleware.js`