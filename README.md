# PROJECT I - CallCenter

## Mục lục
- [0. Tài liệu thêm](#0-tài-liệu-thêm)
- [1. Giới thiệu](#1-giới-thiệu)
- [2. Yêu cầu cấu hình và thư viện](#2-yêu-cầu-cấu-hình-và-thư-viện)
- [3. Hướng dẫn cài đặt và khởi chạy](#3-hướng-dẫn-cài-đặt-và-khởi-chạy)
  - [Bước 1: Clone project](#bước-1-clone-project)
  - [Bước 2: Tạo môi trường ảo và cài đặt thư viện](#bước-2-tạo-môi-trường-ảo-và-cài-đặt-thư-viện)
  - [Bước 3: Cấu hình môi trường](#bước-3-cấu-hình-môi-trường)
  - [Bước 4: Khởi tạo Database và chạy Migration](#bước-4-khởi-tạo-database-và-chạy-migration)
  - [Bước 5: Khởi chạy ứng dụng](#bước-5-khởi-chạy-ứng-dụng)
- [4. Cấu trúc thư mục](#4-cấu-trúc-thư-mục)

---

## 0. Tài liệu thêm

Để tìm hiểu chi tiết hơn về mô hình nghiệp vụ, luồng dữ liệu và thiết kế từng phân hệ trong hệ thống, vui lòng tham khảo các tài liệu chuyên sâu trong thư mục `docs/`:

* **[Tổng quan hệ thống (`docs/Overview.md`)](docs/Overview.md):** Trình bày bài toán điều phối hai hàng đợi FIFO ghép cặp 1–1, đặc tả dữ liệu đầu vào/đầu ra, bảng ánh xạ các trạng thái kết quả (`Success`, `Miss`, `Break`, `Rejected`, `Abandoned`) và sơ đồ luồng hoạt động tổng thể của `CallCenterEngine`.
* **[Kiến trúc & Thiết kế chi tiết (`docs/Document.md`)](docs/Document.md):** Phân tích kiến trúc phân tầng và đi sâu vào thiết kế kỹ thuật của từng phân hệ (`config/`, `core/`, `generator/`, `database/`, `services/`, `storage/`, `gui/`).

---

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
| **pyinstaller** | 6.0+ | Đóng gói ứng dụng |

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

Tạo file `.env`, sau đó sao chép từ file `.env.example` (thay mật khẩu PostgreSQL của bạn vào):
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
python main.py                                      # chạy ứng dụng
python main.py --selftest                           # tự kiểm tra môi trường/đóng gói (tài nguyên, múi giờ, kết nối CSDL) rồi thoát
python main.py export --day yesterday [--out DIR]   # xuất CSV một ngày, không mở giao diện (cho Task Scheduler)
```

---

## 4. Cấu trúc thư mục

```text
ProjectI---CallCenter/
├── config/
│   ├── settings.py
│   ├── runtime.py
│   └── timeutils.py
├── core/
│   ├── models.py
│   ├── queue_manager.py
│   ├── call_processor.py
│   ├── employee_manager.py
│   ├── event_dispatcher.py
│   └── engine.py
├── generator/
│   ├── test_case_generator.py
│   └── input_reader.py
├── database/
│   ├── migrations/
│   ├── connection.py
│   ├── migrate.py
│   ├── models.py
│   ├── repositories.py
│   └── setup.py
├── services/
│   ├── simulation.py
│   ├── persistence.py
│   └── automation.py
├── storage/
│   ├── csv_exporter.py
│   ├── scheduler.py
│   ├── archive_service.py
│   └── export_cli.py
├── gui/
│   ├── controllers/
│   ├── views/
│   ├── widgets/
│   ├── dialogs/
│   ├── main_window.py
│   ├── theme.py
│   └── gc_guard.py
├── docs/
│   ├── Overview.md
│   └── Document.md
├── tests/
├── main.py
├── requirements.txt
├── pytest.ini
└── .env.example
```
