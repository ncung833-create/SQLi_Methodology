# PortSwigger Web Security Academy: SQL Injection 

## Project Overview
Dự án này là hệ thống tài liệu hóa (Technical Documentation) quá trình nghiên cứu và thực thi các bài kiểm thử xâm nhập (**Penetration Testing**) chuyên sâu về lỗ hổng **SQL Injection (SQLi)** trên nền tảng PortSwigger. 

Mục tiêu cốt lõi của Repository này không chỉ là đưa ra các lời giải (**Walkthroughs**), mà là xây dựng một hệ thống phương pháp luận giúp phân tích và khai thác lỗ hổng một cách có hệ thống trên nhiều hệ quản trị cơ sở dữ liệu khác nhau như **MySQL, PostgreSQL, MSSQL, và Oracle**.

##  Repository Structure

Hệ thống tài liệu được phân cấp theo mô hình **PKM (Personal Knowledge Management)** để tối ưu hóa việc tra cứu:

### 1. 🧠 Core Knowledge
* **[SQL Injection - Master Guide](./SQL_Injection-Master_Guide.md)**: Bản đồ tri thức tổng quan, phân loại các biến thể SQLi và nguyên lý khai thác cốt lõi.

### 2. 🛠️ Methodology (The "How-To" Engine)
Đây là phần quan trọng nhất, chứa các kỹ thuật nền tảng để đối đầu với các hệ thống thực tế:
* **[Database Identification](./Methodology/Methodology_Database_Identification.md)**: Kỹ thuật nhận diện DBMS qua phương ngữ SQL (SQL Dialect) và dấu hiệu đặc trưng.
* **[Column & Data Type Enumeration](./Methodology/Methodology_Column&Data_Type_Enumeration.md)**: Quy trình xác định cấu trúc bảng và kiểu dữ liệu bằng `ORDER BY` và `UNION`.
* **[Error-Based Vector Attack](./Methodology/Error-Based_Vector_Attack.md)**: Tổng hợp các Payload khai thác qua cơ chế XML Parsing và ép kiểu dữ liệu.
* **[System Catalog (Data Directory)](./Methodology/SYSTEM_CATALOG(DATA_DIRECTIONARY).md)**: "Bản đồ" các bảng hệ thống của Oracle, MySQL, PostgreSQL.

### 3. 🧪 Hands-on Writeups (The Labs)
Danh sách các bài Lab đã được giải quyết, chia theo nhóm kỹ thuật:

#### 🔹 In-Band SQLi (Union & Error-based)
* `WarmUp Labs`: Khởi động với các lỗ hổng cơ bản trong mệnh đề WHERE.
* `Union-Based Series`: Khai thác trích xuất dữ liệu qua tập kết quả gộp.
* `Visible Error-Based`: Kỹ thuật leak mật khẩu trực tiếp qua thông báo lỗi của DBMS.

#### 🔹 Blind SQLi (Inference-based)
* `Conditional Responses`: Khai thác dựa trên thay đổi nội dung trang web.
* `Time-Based Delays`: Kỹ thuật khai thác "mù" tuyệt đối thông qua độ trễ thời gian phản hồi.


##  Key Technical Skills Demonstrated
* **Vulnerability Research**: Phân tích luồng dữ liệu không tin cậy (**Untrusted Data**) từ HTTP Request đến DBMS.
* **Advanced Exploitation**: Thành thạo kỹ thuật khai thác Blind SQLi qua độ trễ thời gian (**Time-delays**) và phản hồi logic (**Conditional responses**).
* **Database Forensics**: Liệt kê cấu trúc hệ thống qua `information_schema` và các bảng đặc thù của Oracle.
* **Remediation Advice**: Đề xuất giải pháp lập trình an toàn dựa trên **Parameterized Queries** và **Input Validation**.

---
*Maintained by **CuongNguyen** 