# USER STORY: US-05 - Quản lý Giỏ hàng

**Mã US:** US-05  
**Epic:** E03. Giỏ hàng & Đặt hàng  
**Độ ưu tiên:** High  
**Thành viên đảm nhận:** Thành viên 3 (Kỹ thuật)  

---

## 1. Mô tả (Description)
* **As a** Khách hàng,
* **I want to** thêm sách vào giỏ, thay đổi số lượng hoặc xóa sản phẩm,
* **So that** tôi có thể điều chỉnh danh sách sách muốn mua trước khi thanh toán.

---

## 2. Tiêu chí chấp nhận (Acceptance Criteria)
- [ ] **AC01:** Nút "Thêm vào giỏ" xuất hiện ở danh sách sản phẩm và trang chi tiết sách.
- [ ] **AC02:** Không cho phép thêm số lượng vượt quá số lượng tồn kho hiện tại (`stockQuantity`).
- [ ] **AC03:** Người dùng có thể cập nhật tăng/giảm số lượng từng món trong trang Giỏ hàng.
- [ ] **AC04:** Người dùng có thể xóa một hoặc tất cả sản phẩm khỏi Giỏ hàng.
- [ ] **AC05:** Tự động cập nhật Tổng tiền giỏ hàng sau mỗi thao tác.

---

## 3. Thông tin Kỹ thuật (Technical Notes)
* **Endpoints API:** 
  * `GET /api/cart`
  * `POST /api/cart/add`
  * `PUT /api/cart/update`
  * `DELETE /api/cart/item/:id`
* **Models liên quan:** `Cart`, `Books`