
**Metadata:**
* **Target:** [PortSwigger Lab: Querying the database type and version on MySQL and Microsoft](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-mysql-microsoft)
***Prerequisite Knowledge:** Cần nắm vững kỹ thuật tìm số lượng cột và cột chứa kiểu String ở bài viết [Methodology_Column&Data_Type_Enumeration](./Methodology_Column&Data_Type_Enumeration.md) và Xác định hệ quản trị ở bài viết [Methodology_Column&Data_Type_Enumeration](./Methodology_Column&Data_Type_Enumeration.md)

---

## 1. Objective & Attack Surface 

* **Mục tiêu:** Sử dụng kỹ thuật **UNION attack** để trích xuất chuỗi thông tin phiên bản của hệ quản trị cơ sở dữ liệu **MySQL** hoặc **Microsoft SQL Server (MSSQL)**.
* **Tham số Mục tiêu:** Lỗ hổng nằm ở tính năng lọc sản phẩm qua tham số `category` trên thanh URL:
  `GET /filter?category=Gifts HTTP/1.1`

## 2. MySQL & MSSQL Specifics 

Sự khác biệt cốt lõi (The Delta) so với Oracle SQL nằm ở 2 điểm:
* **Không bắt buộc mệnh đề `FROM`:** Trên MySQL và MSSQL, bạn hoàn toàn có thể thực thi lệnh `SELECT` độc lập để in ra một giá trị mà không cần truy vấn từ bảng nào (Không cần dùng `FROM DUAL` như Oracle).
* **Biến hệ thống lưu phiên bản:** Cả MySQL và MSSQL đều sử dụng chung một biến hệ thống (System Variable) là `@@version` để lưu trữ thông tin phiên bản.
* **Ký tự Comment:** MySQL thường dùng dấu `#` (URL Encode là `%23`) hoặc `-- ` (phải có khoảng trắng, URL Encode là `--+`). MSSQL dùng `--`.

## 3. Payload Crafting & Exploitation (Xây dựng Payload)

Dựa số cột đã biết ( tham khảo link ở trên) , ta tiến hành tiêm mã để trích xuất biến `@@version`:

* **Payload:** `'+UNION+SELECT+@@version,+NULL#` 
  *(Hoặc dùng `'+UNION+SELECT+@@version,+NULL--+`)*

* **Thực thi vào URL:**
  `https://YOUR-LAB-ID.web-security-academy.net/filter?category=Gifts'+UNION+SELECT+@@version,+NULL%23`

* **Kết quả:**
  Lệnh `SELECT @@version` được thực thi trực tiếp mà không cần bảng phụ. **DBMS** sẽ gộp chuỗi phiên bản vào cột đầu tiên của kết quả trả về, hiển thị trực tiếp lên giao diện Web.