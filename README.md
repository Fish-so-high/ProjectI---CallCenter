# PROJECT I - CallCenter 

## 1. Giới thiệu

**CallCenter** là hệ thống mô phỏng hoạt động điều phối cuộc gọi thời gian thực tại trung tâm chăm sóc khách hàng theo mô hình **hai hàng đợi FIFO ghép cặp 1–1**:
* Cuộc gọi đến được đưa vào hàng đợi chờ phục vụ (`CallingQueue`) với sức chứa giới hạn.
* Nhân viên trực tổng đài ở trạng thái rảnh được xếp vào hàng đợi phục vụ (`ServiceQueue`) theo thứ tự FIFO.
* Ngay khi cả hai hàng đợi đều có phần tử, hệ thống ghép cặp cuộc gọi đầu hàng với nhân viên rảnh lâu nhất để xử lý song song và cập nhật trạng thái trực quan lên giao diện Desktop (PySide6) cũng như lưu trữ bền vững vào PostgreSQL.

## 2. Yêu cầu cấu hình và thư viện
**Hệ điều hành**: Windows 10/11, Linux hoặc macOS.

**Python**: Phiên bản 3.10 trở lên (khuyến nghị Python 3.11+).

**Cơ sở dữ liệu**: PostgreSQL 14 trở lên.

**Các thư viện Python (`requirements.txt`):**

| Gói | Phiên bản | Mô tả |
| :--- | :--- | :--- |
| **PySide6** | 6.6+ | Giao diện người dùng đồ họa (Qt6) |
| **SQLAlchemy** | 2.0+ | ORM và quản lý kết nối CSDL |
| **psycopg[binary]** | 3.1+ | Driver kết nối PostgreSQL hiệu năng cao |
| **python-dotenv** | 1.0+ | Đọc cấu hình từ file `.env` |
| **tzdata** | 2024.1+ | Dữ liệu múi giờ chuẩn (`Asia/Ho_Chi_Minh`) |
| **pytest** | 8.0+ | Kiểm thử tự động |
| **pyinstaller** | 8.0+ | Đóng gói ứng dụng |

## 3. Hướng dẫn cài đặt và khởi chạy

### Bước 1: Clone project
```bash
git clone <repository-url>
cd ProjectI-CallCenter
```

Hoặc tải file ZIP và giải nén.

---

### Bước 2: Tạo môi trường ảo và cài đặt thư viện

Tạo môi trường ảo
```bash
python -m venv .venv
# Kích hoạt môi trường ảo:
# Windows: .venv\Scripts\activate
# Linux/macOS: source .venv/bin/activate
```

Cài đặt thư viện
```bash
pip install -r requirements.txt
```

---

### Bước 3: Cấu hình môi trường

Tạo file `.env`, sau đó sao chép file `.env.example` ( thay mật khẩu POstgreSQL của bạn vào )
```bash
DB_HOST=localhost
DB_PORT=5432
DB_NAME=callcenter_db
DB_USER=callcenter
DB_PASSWORD=mat_khau_cua_ban
```
---

### Bước 4: Khởi tạo Database và chạy Migration
```bash
python -m database.setup
```
---

### Bước 5: Khởi chạy ứng dụng

```bash
python main.py                                      #chạy ứng dụng
python main.py --selftest                           #tự kiểm tra môi trường/đóng gói (tài nguyên, múi giờ, kết nối CSDL) rồi thoát
python main.py export --day yesterday [--out DIR]   #xuất CSV một ngày, không mở giao diện (cho Task Scheduler)
```
