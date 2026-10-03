# USER STORY: US-06 - Đặt hàng & Xác nhận đơn hàng

**Mã US:** US-06  
**Epic:** E03. Giỏ hàng & Đặt hàng  
**Độ ưu tiên:** High  
**Thành viên đảm nhận:** Thành viên 2 (Quản lý)  

---

## 1. Mô tả (Description)

* **As a** Khách hàng,
* **I want to** nhập thông tin giao hàng và xác nhận đặt hàng,
* **So that** tôi có thể hoàn tất giao dịch mua sách.

---

## 2. Tiêu chí chấp nhận (Acceptance Criteria)

- Form đặt hàng yêu cầu nhập đầy đủ: Tên người nhận, Số điện thoại người nhận, Địa chỉ giao hàng chi tiết, Ghi chú đơn hàng (nếu có).
- Cho phép người dùng lựa chọn phương thức thanh toán:
  - Thanh toán khi nhận hàng (COD).
  - Thanh toán trực tuyến (Payment API).
- Hiển thị bản tóm tắt đơn hàng bao gồm: Danh sách sản phẩm, Số lượng, Đơn giá, Phí vận chuyển, Tổng tiền thanh toán trước khi người dùng bấm xác nhận.
- Khi người dùng bấm "Xác nhận đặt hàng":
  - Hệ thống tự động tạo bản ghi Đơn hàng mới với trạng thái mặc định: **Chờ xử lý (Pending)**.
  - Tự động trừ số lượng tồn kho tương ứng của từng sản phẩm trong đơn hàng.
  - Xóa toàn bộ các mục đã đặt khỏi Giỏ hàng của người dùng.
  - Hiển thị màn hình thông báo Đặt hàng thành công kèm mã đơn hàng.