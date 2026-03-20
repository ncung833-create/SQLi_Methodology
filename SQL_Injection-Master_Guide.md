
## I. Overview of Web Architecture and Database Systems

Trong kiến trúc của **Modern Web Applications**, hệ thống lưu trữ dữ liệu đóng vai trò cốt lõi. Dữ liệu người dùng và các tài nguyên hệ thống được quản lý tập trung tại **Database Management Systems - DBMS**. Để bảo vệ và tối ưu hóa Network Traffic truy cập vào các hệ thống này, các kiến trúc hiện đại thường triển khai các giải pháp trung gian như **Content Delivery Network (CDN)** và **Web Application Firewall (WAF)** (điển hình như Cloudflare).

Về mặt cấu trúc lưu trữ, **Database** được phân chia thành hai hệ sinh thái chính:

* **Relational Database (Cơ sở dữ liệu quan hệ - RDBMS):** Quản lý dữ liệu dưới dạng các bảng (Tables) có tính liên kết chặt chẽ và tuân thủ các nguyên tắc toàn vẹn dữ liệu. Các nền tảng phổ biến bao gồm **MySQL**, **Microsoft SQL Server (MSSQL)**, **PostgreSQL**, và **Oracle Database**.
* **Non-Relational Database (Cơ sở dữ liệu phi quan hệ - NoSQL):** Quản lý dữ liệu dưới dạng tài liệu (Document-oriented) hoặc Key-Value, mang tính linh hoạt cao, điển hình như **MongoDB**.

Đối với mô hình **RDBMS**, ngôn ngữ giao tiếp tiêu chuẩn là **Structured Query Language (SQL)**. Tuy nhiên, trong thực tiễn kiểm thử xâm nhập (**Penetration Testing**), cần lưu ý rằng mỗi nền tảng DBMS sẽ triển khai một hệ phương ngữ (**SQL Dialect**) riêng biệt, dẫn đến sự khác biệt về cú pháp và Built-in Functions.

---

## II. SQL Injection (SQLi) Vulnerability & Root Cause Analysis

Bản chất của các câu lệnh **SQL** là phương thức để **Web Application** tương tác hợp lệ với **DBMS** (thực hiện các luồng truy xuất, thêm, sửa, xóa dữ liệu). Tuy nhiên, lỗ hổng **SQL Injection** sẽ phá vỡ hoàn toàn nguyên tắc giao tiếp này.

**Root Cause :**
Lỗ hổng phát sinh chủ yếu từ các sai lầm trong quy trình phát triển phần mềm an toàn (Secure Software Development Life Cycle - SSDLC). Cụ thể:

* **Thiếu cơ chế kiểm soát đầu vào hoặc kiểm soát đầu vào yếu(Lack of Input Validation/Sanitization):** Ứng dụng tiếp nhận **User Input** một cách tuyệt đối mà không thực hiện các bước làm sạch dữ liệu.
* **Ghép chuỗi động (Dynamic String Concatenation):** Thay vì tách biệt ranh giới logic giữa "lệnh điều khiển" (Code) và "dữ liệu" (Data), các lập trình viên thường nối trực tiếp **User Input** vào thẳng câu truy vấn gốc.

Hệ quả : Kẻ tấn công có thể thao túng cấu trúc logic bằng cách chèn các đoạn mã độc hại (**Malicious SQL Payloads**). Hệ thống **DBMS** không thể phân biệt được đâu là lệnh hợp lệ từ ứng dụng và đâu là mã khai thác, dẫn đến việc thực thi toàn bộ luồng truy vấn ngoài ý muốn.

---


## III. SQL Injection Taxonomy (Phân loại Kỹ thuật Khai thác)

Dựa trên phương thức trích xuất dữ liệu (Data Extraction Method) và luồng phản hồi từ máy chủ, các kỹ thuật khai thác **SQLi** được phân rã thành 3 nhánh kiến trúc chính:

### 1. In-band SQLi (Classic SQLi)
Đây là phương thức tấn công trực diện nhất. Kẻ tấn công sử dụng cùng một kênh giao tiếp (Communication Channel) để vừa gửi **Payload** vừa nhận kết quả trả về. Kỹ thuật này thường được chia thành 2 biến thể:
* **Union-based SQLi:** Tận dụng toán tử `UNION` để gộp kết quả của câu truy vấn độc hại vào tập kết quả (Result Set) của câu truy vấn gốc, sau đó hiển thị trực tiếp trên giao diện **Web Application**.
* **Error-based SQLi:** Chủ động chèn các **Payload** gây lỗi cú pháp hoặc lỗi logic, ép **DBMS** ném ra các thông báo lỗi (Error Messages) chứa đựng thông tin nhạy cảm về cấu trúc cơ sở dữ liệu.

### 2. Inferential SQLi (Blind SQLi)
Khi ứng dụng được cấu hình an toàn để không trả về bất kỳ kết quả truy vấn hay thông báo lỗi nào lên giao diện, kỹ thuật **In-band** sẽ vô hiệu. Lúc này, quá trình khai thác chuyển sang dạng "mù" (Blind), dựa trên việc quan sát hành vi của hệ thống để suy luận dữ liệu:
* **Boolean-based (Logic-based):** Kẻ tấn công gửi các câu truy vấn chứa mệnh đề điều kiện (`TRUE` hoặc `FALSE`). Dữ liệu được trích xuất từng byte một thông qua việc phân tích sự khác biệt trong nội dung **HTTP Response** giữa hai trạng thái logic này.
* **Time-based:** Kẻ tấn công chèn các hàm gây trễ hệ thống (như `SLEEP()` hoặc `pg_sleep()`). Dữ liệu được suy luận dựa trên thời gian máy chủ hoàn thành và trả về **HTTP Response**.

### 3. Out-of-Band SQLi (OOB SQLi)
Kỹ thuật này được áp dụng khi cả **In-band** và **Inferential** đều không khả thi (thường do các truy vấn được thực thi bất đồng bộ - Asynchronously). Kẻ tấn công buộc **DBMS** của hệ thống mục tiêu tạo ra một luồng kết nối mạng ngoại vi (ví dụ: truy vấn **DNS** hoặc **HTTP**) gửi trực tiếp dữ liệu đánh cắp được đến một máy chủ do kẻ tấn công kiểm soát (như **Burp Collaborator**).

---

## IV. Practical Exploitation & Lab Walkthroughs (Thực hành Khai thác)

Phần này tài liệu hóa chi tiết quá trình kiểm thử xâm nhập đối với các mô hình **SQLi** đã phân loại ở trên, áp dụng trực tiếp trên hệ thống phòng thí nghiệm của **PortSwigger Web Security Academy**. Mỗi bài thực hành sẽ tuân thủ nghiêm ngặt quy trình 5 bước: Thu thập, Phân tích, Khai thác, Truy vết nguyên nhân và Đề xuất khắc phục.
### A. Nhóm kỹ thuật: In-band SQLi
[[In-Band_Content]]

### B. Nhóm kỹ thuật: Blind SQLi
[[Blind_Content]]

### C. Nhóm kĩ thuật Out-Band SQLi
