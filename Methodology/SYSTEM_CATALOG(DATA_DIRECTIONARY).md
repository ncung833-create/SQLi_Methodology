## 1. Tổng quan về System Catalog

**System Catalog** là một tập hợp các bảng (hoặc **Views**) lưu trữ **Metadata** (dữ liệu về dữ liệu) của toàn bộ **DBMS**. Nó chứa thông tin về người dùng, bảng, cột, chỉ mục (Indexes), quyền hạn (Permissions) và các tham số cấu hình hệ thống.

### Bảng đối chiếu nhanh (Quick Reference)

|**Đối tượng**|**MySQL / PostgreSQL / MSSQL**|**Oracle**|
|---|---|---|
|**Tiêu chuẩn**|`INFORMATION_SCHEMA`|**Data Dictionary Views**|
|**Danh sách Bảng**|`tables`|`all_tables`|
|**Danh sách Cột**|`columns`|`all_tab_columns`|
|**Lọc theo DB**|`table_schema`|`owner`|

---

## 2. Chi tiết cấu trúc từng DBMS

### 2.1. MySQL & PostgreSQL (Chuẩn INFORMATION_SCHEMA)

Cả hai đều tuân thủ tiêu chuẩn ANSI SQL bằng cách cung cấp cơ sở dữ liệu ảo `information_schema`.

- **Bảng `information_schema.tables`**: Lưu trữ danh sách toàn bộ các bảng.
    
    - `table_schema`: Tên của cơ sở dữ liệu (Database name).
        
    - `table_name`: Tên bảng.
        
    - `table_type`: Loại bảng (BASE TABLE hoặc VIEW).
        
- **Bảng `information_schema.columns`**: Lưu trữ thông tin chi tiết về các cột.
    
    - `table_name`: Bảng chứa cột đó.
        
    - `column_name`: Tên cột.
        
    - `data_type`: Kiểu dữ liệu (int, varchar, etc.).
        

> **Lưu ý:** Trong PostgreSQL, bạn cũng có thể sử dụng `pg_catalog` (ví dụ: `pg_tables`) để truy vấn sâu hơn về cấu trúc nội bộ của hệ thống.

---

### 2.2. Microsoft SQL Server (MSSQL)

Ngoài `information_schema`, MSSQL sử dụng các **System Views** bắt đầu bằng tiền tố `sys.`.

- **`sys.databases`**: Danh sách các cơ sở dữ liệu trên Server.
    
- **`sys.tables`**: Danh sách các bảng trong database hiện tại.
    
- **`sys.columns`**: Danh sách các cột.
    
    - Để lấy tên bảng cho từng cột, cần `JOIN` với `sys.tables` qua trường `object_id`.
        

---

### 2.3. Oracle Database (Data Dictionary)

Oracle không hỗ trợ `information_schema` mặc định. Thay vào đó, nó chia quyền truy cập Metadata theo 3 cấp độ:

1. **`USER_`**: Chỉ những đối tượng thuộc sở hữu của người dùng hiện tại (ví dụ: `user_tables`).
    
2. **`ALL_`**: Tất cả các đối tượng mà người dùng hiện tại có quyền truy cập (ví dụ: `all_tables`).
    
3. **`DBA_`**: Toàn bộ đối tượng trong hệ thống (Yêu cầu quyền Administrator).
    

**Các bảng quan trọng:**

- **`all_tables`**: Chứa cột `table_name` và `owner`.
    
- **`all_tab_columns`**: Chứa `table_name`, `column_name`, và `data_type`.
    
- **`v$version`**: Thông tin phiên bản (thường dùng trong **Error-based SQLi**).