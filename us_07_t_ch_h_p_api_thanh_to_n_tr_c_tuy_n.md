# USER STORY: US-07 - Tích hợp API Thanh toán trực tuyến

**Mã US:** US-07  
**Epic:** E04. Thanh toán & Tích hợp API Payment  
**Độ ưu tiên:** High  
**Thành viên đảm nhận:** Thành viên 3 (Kỹ thuật)  

---

## 1. Mô tả (Description)

* **As a** Khách hàng,
* **I want to** thanh toán đơn hàng qua Cổng thanh toán trực tuyến (Momo/VNPay/ZaloPay),
* **So that** tôi có thể hoàn tất giao dịch nhanh chóng mà không cần dùng tiền mặt.

---

## 2. Tiêu chí chấp nhận (Acceptance Criteria)

- Tích hợp API Payment (Momo/VNPay Sandbox/Live).
- Khi chọn thanh toán trực tuyến, hệ thống sinh URL thanh toán và chuyển hướng người dùng sang Cổng thanh toán.
- Xử lý webhook/callback từ Cổng thanh toán:
  - **Nối thành công:** Cập nhật trạng thái thanh toán đơn hàng thành Đã thanh toán (Paid).
  - **Thất bại/Hủy:** Cập nhật trạng thái thành Thanh toán thất bại và hoàn trả lại số lượng tồn kho.