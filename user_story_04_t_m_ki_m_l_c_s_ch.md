# USER STORY: US-04 - Tìm kiếm & Lọc sách

**Mã US:** US-04  
**Epic:** E02. Duyệt & Tìm kiếm Sách  
**Độ ưu tiên:** High  
**Thành viên đảm nhận:** Thành viên 2 (Quản lý)  

---

## 1. Mô tả (Description)
* **As a** Khách hàng,
* **I want to** tìm kiếm sách theo từ khóa và lọc theo danh mục hoặc giá cả,
* **So that** tôi có thể dễ dàng tìm ra cuốn sách phù hợp với nhu cầu.

---

## 2. Tiêu chí chấp nhận (Acceptance Criteria)
- [ ] **AC01:** Thanh tìm kiếm cho phép nhập từ khóa (Tên sách hoặc Tên tác giả).
- [ ] **AC02:** Bộ lọc cho phép chọn theo Danh mục sách (Category) và Khoảng giá (Min Price - Max Price).
- [ ] **AC03:** Kết quả trả về được phân trang (Pagination: 10/20 sản phẩm trên một trang).
- [ ] **AC04:** Ẩn các sản phẩm có thuộc tính `isDeleted = true`.
- [ ] **AC05:** Thời gian phản hồi của API tìm kiếm < 1 giây.

---

## 3. Thông tin Kỹ thuật (Technical Notes)
* **Endpoint API:** `GET /api/books?search=...&categoryId=...&minPrice=...&maxPrice=...&page=1`
* **Models liên quan:** `Books`, `Categories`