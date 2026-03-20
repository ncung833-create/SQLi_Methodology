

**Metadata:**
* **Target:** [PortSwigger Lab: Visible error-based SQL injection](https://portswigger.net/web-security/sql-injection/error-based/lab-visible-error-based)

---

## 1. Reconnaissance & Information Gathering (Thu thập thông tin)

* **Mục tiêu bài Lab:** Khai thác thông báo lỗi để trích xuất mật khẩu của người dùng `administrator` và đăng nhập.
* **Bề mặt tấn công (Attack Surface):** Tương tự các bài Lab trước, lỗ hổng nằm ở tham số `TrackingId` bên trong Cookie. 
* **Hành vi hệ thống:** Kết quả của câu truy vấn chứa Cookie không được hiển thị trên giao diện, nhưng mọi lỗi phát sinh trong quá trình truy vấn SQL đều được hệ thống "vô tình" in ra màn hình một cách chi tiết.

## 2. Vulnerability Analysis (Phân tích lỗ hổng)

Ta thực hiện kiểm chứng khả năng rò rỉ thông tin qua lỗi (Verbose Error Leakage):

* **Bước 1 (Fuzzing):** Chèn dấu nháy đơn `'` vào cuối `TrackingId`. 
  -> **Kết quả:** Server trả về lỗi SQL chi tiết. Điều này xác nhận hệ thống cho phép hiển thị lỗi nội bộ (Verbose Errors).
* **Bước 2 (Kiểm tra cơ chế ép kiểu):** Ta thử nghiệm kỹ thuật ép kiểu (**Type Casting**) trên PostgreSQL để ép hệ thống in ra dữ liệu. 
  * Payload thử nghiệm: `TrackingId=xyz' AND 1=CAST((SELECT 'test') AS int)--`
  * **Phân tích:** Ta ép chuỗi `'test'` thành kiểu số nguyên (`int`). Do `'test'` không phải là số, PostgreSQL sẽ ném ra lỗi: 
    `invalid input syntax for type integer: "test"`

=> **Kết luận:** Ta có thể gài bất kỳ truy vấn nào vào vị trí của chuỗi `'test'` để lấy dữ liệu qua thông báo lỗi.

## 3. Payload Crafting & Exploitation (Xây dựng Payload và Khai thác)

Chúng ta sẽ thực hiện trích xuất mật khẩu của `administrator` thông qua lỗi ép kiểu.

### Bước 1: Trích xuất mật khẩu (Exploit via CAST)
Ta xây dựng Payload để truy vấn mật khẩu từ bảng `users`:
* **Payload:** `' AND 1=CAST((SELECT password FROM users WHERE username='administrator') AS int)--`

* **Thực thi:** Gửi Request thông qua **Burp Suite Repeater**. 
* **Phân tích phản hồi (Response):** Máy chủ sẽ phản hồi lỗi có dạng:
  `invalid input syntax for type integer: "h4ck3r_p4ssw0rd_123..."`
  *(Mật khẩu của admin đã xuất hiện ngay trong nội dung lỗi).*

### Bước 2: Xử lý giới hạn ký tự (Nếu cần)
Trong một số trường hợp, thông báo lỗi có thể bị cắt ngắn (Truncated). Nếu mật khẩu quá dài, ta cần sử dụng thêm hàm `SUBSTRING` như đã học ở phần lý thuyết:
* **Payload cắt chuỗi:**
  `' AND 1=CAST((SELECT SUBSTRING(password,1,32) FROM users...) AS int)--`

### Bước 3: Đăng nhập (Final Step)
Sử dụng mật khẩu vừa trích xuất được từ thông báo lỗi để đăng nhập với quyền `administrator`, hoàn thành bài Lab.

## 4. Root Cause Analysis - RCA (Phân tích Nguyên nhân Cốt lõi)

Lỗ hổng này là sự kết hợp của hai sai phạm nghiêm trọng:
1. **SQL Injection:** Không sử dụng **Parameterized Queries**, cho phép đầu vào từ Cookie can thiệp vào logic truy vấn.
2. **Improper Error Handling:** Cấu hình hệ thống cho phép in các lỗi chi tiết (Stack traces/Database errors) ra môi trường Production. Đây là một lỗi cấu hình an ninh (**Security Misconfiguration**) phổ biến, cung cấp "lối đi" cho kẻ tấn công trích xuất dữ liệu khi các kênh khác bị chặn.

## 5. Mitigation & Remediation (Chiến lược Khắc phục)

* **Ưu tiên số 1:** Sử dụng **Prepared Statements** để triệt tiêu lỗ hổng SQLi gốc.
* **Ưu tiên số 2:** Cấu hình lại cơ chế xử lý lỗi (Generic Error Messages). Hệ thống chỉ nên trả về các thông báo lỗi chung chung (ví dụ: *"Something went wrong"*) và ghi lại lỗi chi tiết vào Server Logs nội bộ thay vì hiển thị cho người dùng cuối.