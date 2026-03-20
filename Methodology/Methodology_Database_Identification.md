
**Metadata:**
* **Parent Link:** [[SQL_Injection-Master_Guide]]

## 1. Objective (Mục tiêu Kỹ thuật)
Mỗi hệ quản trị cơ sở dữ liệu (**DBMS**) triển khai một hệ phương ngữ (**SQL Dialect**) với các hàm nội trú (built-in functions) và cú pháp riêng biệt. Việc xác định chính xác nền tảng **DBMS** ở Backend là bước tiền quyết để định hình các **Payload** khai thác trong các giai đoạn tiếp theo.

## 2. Technique 1: String Concatenation Fuzzing (Kiểm tra qua Ghép chuỗi)
Cách an toàn và tĩnh lặng nhất để nhận diện **DBMS** là gửi các biểu thức ghép chuỗi (String Concatenation) và quan sát xem máy chủ có xử lý thành công hay không.

| Tên DBMS                 | Cú pháp Ghép chuỗi (Payload) | Phân tích Logic                               |
| :----------------------- | :--------------------------- | :-------------------------------------------- |
| **Oracle**               | `'foo'`\|\| `'bar'`          | Chỉ hỗ trợ toán tử \|\|                       |
| **PostgreSQL**           | `'foo'`\|\| `'bar'`\|\|      | Chỉ hỗ trợ toán tử \|\|                       |
| **Microsoft SQL Server** | `'foo'+'bar'`                | Sử dụng toán tử `+`.                          |
| **MySQL**                | `'foo' 'bar'` (Khoảng trắng) | Chỉ cần một khoảng trắng giữa hai chuỗi tĩnh. |

*Cách áp dụng:* Nếu bạn tiêm `?category=Gifts'||'Gifts` và trang web vẫn hiển thị danh mục Gifts bình thường, Backend chắc chắn là **Oracle** hoặc **PostgreSQL**.

## 3. Technique 2: Version Extraction (Kiểm tra qua Hàm Phiên bản)
Nếu lỗ hổng cho phép hiển thị kết quả truy vấn (như **UNION-based SQLi**), ta có thể trực tiếp gọi các biến hoặc hàm hệ thống lưu trữ thông tin phiên bản.

| Tên DBMS | Payload Trích xuất Phiên bản | Ghi chú (Đặc thù) |
| :--- | :--- | :--- |
| **Microsoft SQL Server** | `SELECT @@version` | Dùng biến hệ thống. |
| **MySQL** | `SELECT @@version` | Dùng biến hệ thống. |
| **PostgreSQL** | `SELECT version()` | Dùng hàm nội trú. |
| **Oracle** | `SELECT * FROM v$version` | Bắt buộc truy vấn từ bảng hệ thống `v$version`. |

## 4. Technique 3: Comment Syntax (Nhận diện qua cú pháp chú thích)
Một phương pháp rà quét (Fuzzing) thụ động nhưng rất hiệu quả là quan sát cách máy chủ xử lý các ký tự chú thích (Comment indicators) ở cuối chuỗi Payload. 

*(Lưu ý: Các ký tự này bắt buộc phải được **URL Encode** trước khi gửi qua HTTP Request để tránh bị trình duyệt hiểu nhầm).*

| Tên DBMS                 | Cú pháp Line Comment             | Ký tự URL Encode thực tế | Ghi chú (Đặc thù)                                                                                              |
| :----------------------- | :------------------------------- | :----------------------- | :------------------------------------------------------------------------------------------------------------- |
| **MySQL**                | `#` hoặc `-- ` (có khoảng trắng) | `%23` hoặc `--+`         | Dấu `#` là đặc trưng nhận diện độc quyền của MySQL. Nếu dùng `--`, bắt buộc phải có một khoảng trắng phía sau. |
| **PostgreSQL**           | `--`                             | `--`                     | Chấp nhận `--` liền kề không cần khoảng trắng.                                                                 |
| **Microsoft SQL Server** | `--`                             | `--`                     | Chấp nhận `--` liền kề.                                                                                        |
| **Oracle**               | `--`                             | `--`                     | Chấp nhận `--` liền kề.                                                                                        |

**Luồng Suy luận Loại trừ (Elimination Logic):**
1. Chèn Payload kết thúc bằng `%23` (Ví dụ: `' OR 1=1%23`). Nếu truy vấn chạy thành công mà không báo lỗi cú pháp $\rightarrow$ **100% là MySQL**.
2. Nếu `%23` báo lỗi (HTTP 500), tiếp tục thử `--`. Nếu `--` chạy thành công $\rightarrow$ Backend có thể là **Oracle, PostgreSQL, hoặc MSSQL**. (Lúc này quay lại áp dụng Technique 1 hoặc Technique 2 để thu hẹp phạm vi).

## 5. The Oracle "DUAL" Table Rule (Luật Bảng DUAL)
* **Quy tắc cốt lõi:** Mọi câu lệnh `SELECT` trong **Oracle SQL** bắt buộc phải có mệnh đề `FROM`. 
* **Hệ quả:** Khi kiểm tra cấu trúc bằng `UNION SELECT NULL`, nếu hệ thống ném ra lỗi cú pháp, bạn phải thử nối thêm `FROM DUAL` (một bảng ảo mặc định của Oracle). 
* **Ví dụ:** `' UNION SELECT NULL FROM DUAL--`