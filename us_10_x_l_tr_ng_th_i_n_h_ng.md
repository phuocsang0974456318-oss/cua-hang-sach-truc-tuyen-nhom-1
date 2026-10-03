# USER STORY: US-10 - Xử lý trạng thái đơn hàng

**Mã US:** US-10  
**Epic:** E06. Quản lý Đơn hàng  
**Độ ưu tiên:** High  
**Thành viên đảm nhận:** Thành viên 2 (Quản lý)  

---

## 1. Mô tả (Description)

* **As a** Quản lý (Manager),
* **I want to** xem danh sách và cập nhật trạng thái đơn hàng,
* **So that** tôi có thể theo dõi và đảm bảo tiến độ giao hàng cho khách.

---

## 2. Tiêu chí chấp nhận (Acceptance Criteria)

- Hiển thị danh sách đơn hàng có thể lọc theo trạng thái: Chờ xử lý -> Đang xử lý -> Đang giao -> Hoàn tất / Đã hủy.
- Quản lý có thể cập nhật chuyển trạng thái đơn hàng theo đúng luồng logic (không được nhảy bước sai quy trình).
- Nếu đơn hàng bị Hủy, tự động cộng lại số lượng sách vào tồn kho.