# USER STORY: US-08 - Thêm, Sửa, Xóa đầu sách (CRUD)

**Mã US:** US-08  
**Epic:** E05. Quản lý Kho & Sản phẩm  
**Độ ưu tiên:** Medium  
**Thành viên đảm nhận:** Thành viên 2 (Quản lý)  

---

## 1. Mô tả (Description)

* **As a** Quản lý Kho (Manager),
* **I want to** thêm mới, cập nhật thông tin và xóa các đầu sách,
* **So that** tôi có thể duy trì danh mục sản phẩm chính xác và phong phú trên website.

---

## 2. Tiêu chí chấp nhận (Acceptance Criteria)

- Form thêm/sửa sách gồm: Tên sách, Tác giả, Nhà xuất bản, Danh mục, Giá bán, Mô tả, Hình ảnh bìa sách.
- Cho phép upload file ảnh bìa sách lên hệ thống/cloud storage (Cloudinary/S3).
- Chỉ thực hiện Xóa mềm (Soft Delete) đối với sách đã từng có trong đơn hàng để giữ tính toàn vẹn dữ liệu.