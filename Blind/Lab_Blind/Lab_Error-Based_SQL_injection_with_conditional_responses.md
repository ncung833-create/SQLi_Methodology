

**Link lab:** [Blind SQL injection with conditional responses](https://portswigger.net/web-security/sql-injection/blind/lab-conditional-responses)

---
## 1. Reconnaissance & Information Gathering (Thu thập thông tin)

* **Mục tiêu bài Lab:** Tìm mật khẩu của tài khoản `administrator` và đăng nhập thành công.
* **Phân tích Bề mặt Tấn công (Attack Surface Analysis):** Ứng dụng sử dụng một cookie có tên là `TrackingId` để theo dõi phân tích. Khi gửi Request, hệ thống sẽ thực thi một câu lệnh SQL để kiểm tra xem `TrackingId` này có tồn tại trong cơ sở dữ liệu hay không.
* **Cơ chế phản hồi (Conditional Behavior):** * Nếu truy vấn trả về kết quả (ID tồn tại): Trang web hiển thị dòng chữ **"Welcome back"**.
    * Nếu truy vấn không trả về kết quả: Dòng chữ **"Welcome back"** sẽ biến mất.
    * Đây chính là "tín hiệu" (Signal) để ta thực hiện khai thác **Blind SQLi**.

## 2. Vulnerability Analysis (Phân tích lỗ hổng)

Ta cần xác nhận xem tham số `TrackingId` có thực sự bị tiêm nhiễm SQL hay không bằng cách gửi các điều kiện logic:

* **Test TRUE:** `TrackingId=XYZ' AND 1=1--` 
  -> Kết quả: Xuất hiện **"Welcome back"**.
* **Test FALSE:** `TrackingId=XYZ' AND 1=2--` 
  -> Kết quả: **Mất** dòng chữ "Welcome back".

=> Hệ thống bị lỗ hổng **Blind SQLi**. Ta không thể "đọc" dữ liệu, nhưng có thể "hỏi" dữ liệu thông qua các mệnh đề logic.

## 3. Payload Crafting & Exploitation (Xây dựng Payload và Khai thác)

Quy trình khai thác Blind SQLi diễn ra theo từng bước nhỏ (Enumeration):
### Bước 1: Kiểm tra loại cơ sở dữ liệu
	*Dựa vào tài liệu Xác định hệ quản trị trước đó cùng với kết quả trả về của trang web để kết luận*

### Bước 2: Kiểm tra sự tồn tại của bảng users
* **Payload:** `TrackingId=XYZ' AND (SELECT 'a' FROM users LIMIT 1)='a'--`
* **Kết quả:** Thấy "Welcome back" -> Bảng `users` tồn tại.

### Bước 3: Kiểm tra username 'administrator'
* **Payload:** `TrackingId=XYZ' AND (SELECT 'a' FROM users WHERE username='administrator')='a'--`
* **Kết quả:** Thấy "Welcome back" -> User `administrator` tồn tại.

### Bước 4: Xác định độ dài mật khẩu (Password Length)
Ta sử dụng hàm `LENGTH()` để hỏi độ dài của mật khẩu:
* **Payload:** `TrackingId=XYZ' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>19)='a'--`
* **Quá trình:** Tăng dần con số cho đến khi mất dòng "Welcome back". Giả sử mật khẩu dài 20 ký tự.

### Bước 4: Trích xuất mật khẩu (Brute-forcing characters)
Sử dụng hàm `SUBSTR()` để cắt từng ký tự của mật khẩu và so sánh với bảng chữ cái.
* **Payload mẫu (Kiểm tra ký tự thứ 1):**
  `TrackingId=XYZ' AND (SELECT SUBSTR(password,1,1) FROM users WHERE username='administrator')='a'--`
* **Logic:** * Nếu trang web hiện "Welcome back" -> Ký tự thứ 1 là 'a'.
    * Nếu không hiện -> Thử tiếp 'b', 'c', '1', '2'...

*(Mẹo: Trong thực tế, bạn nên sử dụng tính năng **Automate** trong Caido hoặc viết một script **Python** để tự động hóa việc thử hàng trăm trường hợp này).*

## 4. Root Cause Analysis - RCA (Phân tích Nguyên nhân Cốt lõi)

Lỗ hổng phát sinh do ứng dụng tin tưởng hoàn toàn vào giá trị của Cookie và đưa trực tiếp vào mệnh đề `WHERE` của câu truy vấn SQL mà không qua xác thực. 
```sql
-- Câu lệnh giả định ở Backend
SELECT tracking_id FROM tracking_table WHERE tracking_id = '[User_Cookie]'