# Kiến trúc Mô hình CallCenter

## Mục lục
1. [Tổng quan Kiến trúc Mô hình CallCenter](#1-tổng-quan-kiến-trúc-mô-hình-callcenter)
2. [Phân hệ Cấu hình (`config/`)](#2-phân-hệ-cấu-hình-config)
3. [Phân hệ Lõi (`core/`)](#3-phân-hệ-lõi-core)
4. [Phân hệ Sinh cuộc gọi (`generator/`)](#4-phân-hệ-sinh-cuộc-gọi-generator)
5. [Phân hệ Cơ sở dữ liệu (`database/`)](#5-phân-hệ-cơ-sở-dữ-liệu-database)
6. [Phân hệ Dịch vụ nền (`services/`)](#6-phân-hệ-dịch-vụ-nền-services)
7. [Phân hệ Lưu trữ & Lập lịch (`storage/`)](#7-phân-hệ-lưu-trữ--lập-lịch-storage)
8. [Phân hệ Giao diện Người dùng (`gui/`)](#8-phân-hệ-giao-diện-người-dùng-gui)

---

## 1. Tổng quan Kiến trúc Mô hình CallCenter

### 1.1. Kiến trúc phân tầng (Layered Architecture)
Hệ thống **CallCenter** được thiết kế theo kiến trúc tách biệt hoàn toàn giữa nghiệp vụ lõi, tầng dịch vụ nền, tầng lưu trữ và giao diện người dùng:

1. **Tầng Lõi (`core/`):** Thuần Python, không phụ thuộc UI hay CSDL. Chứa máy trạng thái (State Machine) của cuộc gọi và nhân viên, quản lý hai hàng đợi FIFO (`CallingQueue`, `ServiceQueue`) và bộ lập lịch xử lý cuộc gọi đồng thời bằng Min-Heap (`CallProcessor`).
2. **Tầng Sinh Dữ Liệu (`generator/`):** Chạy trên luồng nền riêng (`GeneratorWorker`), đẩy các lô cuộc gọi (`IncomingCall`) vào hàng đợi trung gian `_inbox` với cơ chế tự động giảm tải một nửa lưu lượng khi tổng đài quá tải (`waiting > heavy_threshold`).
3. **Tầng Dịch Vụ (`services/`):**
   * `SimulationService`: Điều phối vòng lặp thời gian thực của Engine (`tick` mỗi 20ms).
   * `PersistenceService`: Hàng đợi đệm bất đồng bộ (tối đa 100.000 bản ghi) giúp hệ thống tiếp tục vận hành trơn tru ngay cả khi PostgreSQL mất kết nối tạm thời.
   * `AutomationService`: Quản lý các tác vụ định kỳ theo ngày (xuất CSV lúc 19:00 và chốt sổ phân kỳ lúc 00:00).
4. **Tầng Dữ Liệu & Lưu Trữ (`database/`, `storage/`):** Quản lý kết nối PostgreSQL qua SQLAlchemy, tự động chạy migration khi khởi động và xuất file CSV chuẩn UTF-8 BOM.
5. **Tầng Giao Diện (`gui/`):** Xây dựng bằng PySide6 theo mô hình MVC/Signal-Slot. Mọi sự kiện từ luồng nền đều đi qua `AppController` bằng `Qt.QueuedConnection` để đảm bảo an toàn luồng (Thread-Safe).

### 1.2. Vòng đời của một cuộc gọi (Call Lifecycle)
1. **Tiếp nhận (`CALL_RECEIVED`):** `CallCenterEngine.submit()` kiểm tra trùng lặp `call_id` thông qua bộ nhớ đệm `_RecentIds`.
2. **Xếp hàng (`CALL_QUEUED`) hoặc Từ chối (`REJECTED`):** Nếu `CallingQueue` đã đạt `max_queue_size`, cuộc gọi bị từ chối ngay lập tức với nhãn `Rejected`.
3. **Phân phối (`CALL_STARTED`):** Khi có đồng thời cuộc gọi chờ và nhân viên rảnh trong `ServiceQueue`, Engine lấy cặp đầu hàng, chuyển nhân viên sang `BUSY` và đưa vào `CallProcessor`.
4. **Xử lý & Hoàn tất (`CALL_FINISHED`):** Đến thời điểm `ended_at`, `CallProcessor` rút cuộc gọi ra khỏi Min-Heap, tạo `CallResult` (`Success`, `Miss` hoặc `Break`) và giải phóng nhân viên theo chính sách cấu hình (`requeue` hoặc `to_break`).

---

## 2. Phân hệ Cấu hình (`config/`)

Phân hệ `config/` quản lý toàn bộ tham số môi trường, tham số vận hành động và chuẩn hóa thời gian cho toàn hệ thống.

### 2.1. `config/settings.py` — Cấu hình tĩnh từ môi trường
* Xác định thư mục gốc `BASE_DIR` (tự động nhận diện khi chạy mã nguồn hoặc đóng gói `.exe` bằng PyInstaller).
* `DatabaseSettings`: Đọc và kiểm tra tính hợp lệ của các biến trong `.env` (`DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`, `DB_CONNECT_TIMEOUT`). Mật khẩu được ẩn khỏi `repr()` để tránh lộ thông tin trong log.
* `AppSettings` & `load_settings()`: Nạp cấu hình CSDL, thư mục xuất báo cáo (`CALLCENTER_EXPORT_DIR`) và múi giờ (`CALLCENTER_TIMEZONE`, mặc định `Asia/Ho_Chi_Minh`).

### 2.2. `config/runtime.py` — Cấu hình vận hành động (`settings.json`)
* Lớp dataclass `RuntimeConfig` lưu trữ các thông số người dùng có thể thay đổi trực tiếp trên giao diện:
  * **Engine:** `max_queue_size`, `miss_release_policy`, `break_release_policy`.
  * **Generator:** `max_call`, `min_interval`, `max_interval`, `heavy_threshold`, `event_weights`, `seed`, `source_mode` (`random` / `replay`), `replay_file`, `record_generated`.
  * **Export & Schedule:** `export_dir`, `auto_schedule_enabled`, `auto_export_time`.
* Hỗ trợ kiểm tra ràng buộc (`validate()`) và ghi file an toàn thông qua file tạm + `os.replace()` (`save()`) để tránh hỏng file cấu hình khi mất điện đột ngột.

### 2.3. `config/timeutils.py` — Tiện ích thời gian & Múi giờ
* `get_timezone()`: Nạp đối tượng `ZoneInfo` chuẩn IANA.
* `to_local()`: Chuẩn hóa `datetime` (naive hoặc aware) về múi giờ địa phương.
* `day_bounds()`: Tính mốc bắt đầu (`00:00:00`) và kết thúc (`00:00:00` ngày hôm sau) của một ngày lịch để truy vấn dữ liệu chính xác theo ngày.
* `year_month()`: Trích xuất chuỗi `YYYY-MM` phục vụ thống kê tháng.

---

## 3. Phân hệ Lõi (`core/`)

Phân hệ `core/` chứa toàn bộ logic mô phỏng trung tâm của tổng đài **CallCenter**, được bảo vệ đồng bộ đa luồng bằng `threading.RLock`.

### 3.1. `core/models.py` — Mô hình dữ liệu & Trạng thái
* **Enum trạng thái:** `CallEvent` (`Success`, `Miss`, `Break`), `CallStatus` (`WAITING`, `IN_PROGRESS`, `COMPLETED`, `MISSED`, `ABANDONED`, `REJECTED`), `CallTag` (`Success`, `Miss`, `Break`, `Rejected`, `Abandoned`), và `EmployeeStatus` (`AVAILABLE`, `BUSY`, `BREAK`, `OFFLINE`).
* `IncomingCall`: Dữ liệu đầu vào bất biến (frozen), tự động kiểm tra định dạng số điện thoại 10 số (`09`, `08`, `02`, `03`) và thời lượng gọi không âm.
* `CallResult`: Bản ghi kết quả cuối cùng của một cuộc gọi sau khi xử lý xong.
* `EngineSnapshot`: Ảnh chụp trạng thái tức thời (hàng đợi chờ, hàng đợi nhân viên, thẻ nhân viên, thống kê) dùng để đẩy sang GUI.

### 3.2. `core/queue_manager.py` — Quản lý hàng đợi FIFO
* `CallingQueue`: Hàng đợi cuộc gọi chờ xử lý (`deque` + `set` tra cứu nhanh $O(1)$), giới hạn bởi `capacity`. Hỗ trợ xóa cuộc gọi ở giữa hàng khi khách tự cúp máy (`remove()`).
* `ServiceQueue`: Hàng đợi các mã nhân viên đang `AVAILABLE`, đảm bảo không trùng lặp và hỗ trợ truy vấn vị trí xếp hàng (`position()`).
* `QueueManager`: Điều phối ghép cặp đầu hàng (`pop_pair()`) và kiểm tra bất biến `check_invariant()` (không bao giờ tồn tại đồng thời cuộc gọi đang chờ và nhân viên đang rảnh).

### 3.3. `core/call_processor.py` — Bộ xử lý cuộc gọi đồng thời
* Quản lý các cuộc gọi đang diễn ra (`ActiveCall`) bằng cấu trúc **Min-Heap** (`heapq`) sắp xếp theo thời điểm kết thúc `end_at`.
* `effective_duration()`: Ép thời lượng cuộc gọi `Miss` về `0.0s`.
* `pop_due(now)`: Rút toàn bộ các cuộc gọi đã đến hạn kết thúc tại thời điểm `now` với độ phức tạp $O(\log N)$.
* `shift(delta)`: Dịch chuyển mốc thời gian khi hệ thống tạm dừng (`pause`) và tiếp tục (`resume`).

### 3.4. `core/employee_manager.py` — Quản lý nhân viên & File TXT
* Quản lý vòng đời trạng thái nhân viên (`add`, `update`, `mark_busy`, `release`).
* Hỗ trợ cơ chế **trạng thái chờ (`pending_status`)**: Khi nhân viên đang `BUSY`, yêu cầu chuyển sang `BREAK` hoặc `OFFLINE` sẽ được hoãn lại cho đến khi cuộc gọi hiện tại kết thúc.
* Cung cấp các hàm đọc/ghi file `data/employees.txt` (`load_employees`, `parse_employees`, `save_employees`).

### 3.5. `core/engine.py` & `core/event_dispatcher.py` — Bộ điều phối trung tâm
* `CallCenterEngine`: Kết nối các thành phần trên qua các phương thức chính: `submit()` (nhận cuộc gọi), `tick()` (cập nhật tiến trình theo thời gian), `abandon_waiting()`, `pause()`, `resume()`, và các thao tác thay đổi trạng thái nhân viên.
* `EventDispatcher`: Cơ chế Publish/Subscribe nội bộ phát đi các sự kiện (`CALL_RECEIVED`, `CALL_QUEUED`, `CALL_STARTED`, `CALL_FINISHED`, `EMPLOYEE_STATUS_CHANGED`, `STATE_CHANGED`).

---

## 4. Phân hệ Sinh cuộc gọi (`generator/`)

Phân hệ `generator/` chịu trách nhiệm tạo luồng cuộc gọi đầu vào cho hệ thống mô phỏng theo thời gian thực.

### 4.1. `generator/test_case_generator.py` — Bộ sinh ngẫu nhiên & Luồng nền
* `GeneratorConfig`: Cấu hình số cuộc gọi tối đa mỗi đợt (`max_call`), khoảng nghỉ giữa các đợt (`min_interval` – `max_interval`), trọng số phân phối sự kiện (`Success`/`Miss`/`Break`), hạt giống ngẫu nhiên (`seed`) và ngưỡng quá tải (`heavy_threshold`).
  * **Cơ chế tự điều tiết tải:** Khi số cuộc gọi đang chờ vượt `heavy_threshold`, hàm `effective_max_call()` tự động giảm một nửa `max_call` để tránh làm tràn hàng đợi.
* `RandomSource`: Sinh ngẫu nhiên số điện thoại hợp lệ (`09`, `08`, `02`, `03` + 8 chữ số), chọn `CallEvent` theo trọng số và sinh `time_call` tương ứng (`Success`: $0\text{–}3\text{s}$, `Break`: $0\text{–}1\text{s}$, `Miss`: $0\text{s}$).
* `GeneratorWorker`: Luồng nền (`threading.Thread`) định kỳ gọi `make_batch()`, đẩy cuộc gọi vào hàng đợi `_inbox` và ghi lại kịch bản ra file text nếu bật `record_generated`.

### 4.2. `generator/input_reader.py` — Đọc & Phát lại kịch bản
* `parse_line()` & `load_rows()`: Phân tích cú pháp từng dòng trong file kịch bản (`phone_number|event|time_call`), bỏ qua dòng trống và dòng chú thích `#`.
* `ReplaySource`: Phát lại tuần tự các cuộc gọi từ file kịch bản có sẵn (như `data/input_testcase.txt`) theo nhịp thời gian cấu hình, tự động báo hiệu `exhausted = True` khi phát hết file.

---

## 5. Phân hệ Cơ sở dữ liệu (`database/`)

Phân hệ `database/` quản lý kết nối PostgreSQL, lược đồ bảng (Schema), migration và các lớp truy xuất dữ liệu (Repositories).

### 5.1. `database/connection.py` — Quản lý kết nối & Tự động thử lại
* `create_db_engine()`: Khởi tạo SQLAlchemy Engine với connection pool (`pool_size=5`, `pool_pre_ping=True`).
* `Database.run()`: Thực thi giao dịch (Transaction) với cơ chế tự động thử lại (Retry with backoff) đối với các lỗi mạng tạm thời, đồng thời ngắt ngay đối với lỗi xác thực hoặc sai cú pháp SQL (`SQLSTATE` nhóm `28`, `3D`, `42`).

### 5.2. `database/migrate.py` & `migrations/0001_initial.sql` — Quản lý Schema
* `apply_migrations()`: Sử dụng khóa tư vấn PostgreSQL (`pg_advisory_xact_lock`) để chống xung đột khi nhiều tiến trình cùng khởi động, kiểm tra mã băm SHA-256 của từng file `.sql` và lưu lịch sử vào bảng `schema_migrations`.
* **Cấu trúc 3 bảng chính (`0001_initial.sql`):**
  1. `employees`: Lưu thông tin nhân viên (`employee_id`, `employee_name`, `status`, `is_active`).
  2. `call_records`: Lưu lịch sử cuộc gọi kèm các `CHECK CONSTRAINT` kiểm tra định dạng SĐT, nhãn `Tag` và ràng buộc logic thời gian (`ended_at >= started_at`). Đánh chỉ mục tối ưu theo `(COALESCE(ended_at, received_at), call_id)`.
  3. `employee_monthly_stats`: Bảng tổng hợp hiệu suất nhân viên theo tháng (`successful_calls`, `total_talk_seconds`).

### 5.3. `database/repositories.py` — Tầng truy xuất dữ liệu
* `EmployeeRepository`: Đồng bộ danh sách nhân viên bằng câu lệnh `INSERT ... ON CONFLICT DO UPDATE` (`upsert_many`).
* `CallRecordRepository`: Lưu kết quả cuộc gọi (`save`, `save_many`) chống trùng lặp theo `call_id`; nếu cuộc gọi có `tag == 'Success'`, hệ thống tự động cộng dồn chỉ số vào `employee_monthly_stats` trong cùng một transaction. Hỗ trợ phân trang theo keyset (`iter_by_day`) để xuất hàng chục nghìn bản ghi mà không tốn RAM.
* `MonthlyStatsRepository`: Truy vấn và tính toán lại (`rebuild_month`) thống kê tháng từ bảng gốc.

### 5.4. `database/setup.py` — Công cụ khởi tạo CSDL
* Cho phép chạy độc lập (`python -m database.setup`) để tự động tạo database `callcenter_db` nếu chưa tồn tại và áp dụng toàn bộ migration.

---

## 6. Phân hệ Dịch vụ nền (`services/`)

Phân hệ `services/` đóng vai trò cầu nối điều phối đa luồng giữa lõi mô phỏng (`core`), bộ sinh (`generator`), cơ sở dữ liệu (`database`) và giao diện (`gui`).

### 6.1. `services/simulation.py` — Dịch vụ Mô phỏng (`SimulationService`)
* Quản lý vòng lặp chính của tổng đài (`_loop`) chạy ở tần suất cao (`loop_interval = 0.02s`):
  1. Rút tối đa 500 cuộc gọi từ `_inbox` đưa vào `CallCenterEngine.submit()`.
  2. Gọi `CallCenterEngine.tick(now)` để hoàn tất các cuộc gọi đến hạn.
  3. Phát `EngineSnapshot` cho giao diện mỗi `0.1s` khi có thay đổi trạng thái (`_dirty = True`).
* Cung cấp các API điều khiển an toàn luồng: `start()`, `pause()`, `toggle()`, `set_generator_enabled()`, `apply_config()`, các thao tác quản lý nhân viên và xuất CSV bất đồng bộ qua `ThreadPoolExecutor`.

### 6.2. `services/persistence.py` — Dịch vụ Lưu trữ Bền vững (`PersistenceService`)
* Chạy một luồng nền riêng (`callcenter-persistence`) để ghi dữ liệu vào PostgreSQL mà không làm chậm vòng lặp mô phỏng thời gian thực.
* **Bộ đệm chịu lỗi (Resilient Buffer):** Sử dụng hàng đợi `deque` tối đa 100.000 bản ghi (`buffer_limit`). Nếu PostgreSQL bị ngắt kết nối, dữ liệu được giữ trong RAM và tự động xả theo lô (`batch_size = 200`) ngay khi kết nối khôi phục.
* **Cô lập bản ghi lỗi (Poison Record Isolation):** Nếu một lô gặp lỗi ràng buộc dữ liệu, hệ thống tự động tách từng bản ghi để lưu các bản ghi hợp lệ và chỉ loại bỏ bản ghi lỗi.

### 6.3. `services/automation.py` — Dịch vụ Tự động hóa (`AutomationService`)
* Khởi tạo và quản lý `DailyScheduler` với 2 tác vụ mặc định:
  1. `daily-export`: Xuất báo cáo CSV trong ngày vào giờ cấu hình (mặc định `19:00`).
  2. `day-rollover`: Chốt dữ liệu ngày cũ lúc `00:00` và gọi `SimulationService.start_new_day()` để làm mới bảng thống kê trên giao diện cho ngày mới.

---

## 7. Phân hệ Lưu trữ & Lập lịch (`storage/`)

Phân hệ `storage/` xử lý việc kết xuất báo cáo CSV chuẩn hóa và lập lịch thực thi tác vụ tự động theo ngày.

### 7.1. `storage/csv_exporter.py` — Kết xuất CSV an toàn
* Định dạng tên file chuẩn: `calls-DD-MM-YYYY.csv` (ví dụ: `calls-18-11-2026.csv`).
* Xuất đầy đủ 6 cột tiếng Việt với mã hóa **UTF-8 with BOM (`utf-8-sig`)** để hiển thị chuẩn tiếng Việt khi mở trực tiếp bằng Microsoft Excel.
* Tích hợp chống CSV Formula Injection (`_safe()`) và ghi qua file tạm + `os.fsync()` + `os.replace()` để đảm bảo tính nguyên tử (Atomic Write).

### 7.2. `storage/scheduler.py` & `storage/archive_service.py` — Bộ lập lịch & Lưu trữ
* `SchedulerState`: Lưu vết ngày hoàn thành gần nhất của từng tác vụ vào `data/scheduler_state.json`.
* `DailyScheduler`: Kiểm tra các tác vụ đến hạn mỗi giây. Nếu ứng dụng bị tắt vào giờ chạy lịch, khi mở lại hệ thống sẽ tự động **chạy bù (Catch-up)** tối đa 7 ngày gần nhất và tự động thử lại nếu PostgreSQL chưa sẵn sàng.
* `ArchiveService`: Đợi `PersistenceService` ghi hết dữ liệu tồn đọng (`wait_idle`) trước khi gọi `export_day()` để đảm bảo file CSV không bị thiếu các cuộc gọi vừa kết thúc.

### 7.3. `storage/export_cli.py` — Công cụ xuất báo cáo dòng lệnh (CLI)
* Cho phép xuất dữ liệu độc lập thông qua lệnh `python main.py export --day <today|yesterday|DD-MM-YYYY>` để tích hợp với Windows Task Scheduler hoặc Cronjob ngay cả khi ứng dụng desktop đang đóng.

---

## 8. Phân hệ Giao diện Người dùng (`gui/`)

Phân hệ `gui/` được phát triển trên nền tảng **PySide6 (Qt6)** với giao diện sáng (Light Theme) hiện đại và kiến trúc tách biệt hoàn toàn khỏi luồng tính toán nền.

### 8.1. `gui/controllers/app_controller.py` & `gui/gc_guard.py` — An toàn luồng
* `AppController`: Tiếp nhận callback từ các luồng nền (`SimulationService`, `PersistenceService`, `AutomationService`), chuyển đổi qua các Signal nội bộ sử dụng `Qt.QueuedConnection` để mọi cập nhật giao diện đều thực thi trên GUI Thread. Tích hợp bộ đệm gom kết quả (`FLUSH_MS = 100ms`) giúp giao diện mượt mà ngay cả khi có hàng trăm cuộc gọi kết thúc mỗi giây.
* `GuiGarbageCollector`: Tắt bộ thu gom rác tự động của Python và ép chạy định kỳ trên GUI Thread bằng `QTimer` nhằm ngăn chặn lỗi xung đột hủy widget Qt đa luồng.

### 8.2. `gui/main_window.py` & `gui/theme.py` — Cửa sổ chính & Giao diện
* `MainWindow`: Bố cục chia 3 vùng chính bằng `QSplitter`: thanh điều hướng bên trái (`Sidebar`), khu vực giám sát hàng đợi phía trên (`ServiceQueueView` + `CallingQueueView`) và bảng lịch sử phía dưới (`HistoryView`).
* `theme.py`: Định nghĩa bảng màu chuẩn và QSS Stylesheet đồng nhất cho toàn bộ ứng dụng.

### 8.3. Các thành phần hiển thị (`gui/views/`, `gui/widgets/`, `gui/dialogs/`)
* `ServiceQueueView` & `AgentCard`: Hiển thị thẻ trực quan cho từng nhân viên theo màu trạng thái (Xanh lá: Rảnh, Đỏ: Đang xử lý, Vàng: Tạm nghỉ, Xám: Offline) kèm menu chuột phải đổi trạng thái nhanh.
* `CallingQueueView`: Hiển thị danh sách cuộc gọi đang chờ theo thứ tự FIFO và cho phép chuột phải mô phỏng khách hàng cúp máy.
* `HistoryView`: Bảng lịch sử cuộc gọi sử dụng `QAbstractTableModel` giới hạn hiển thị 5.000 dòng gần nhất để tối ưu hiệu năng cuộn.
* `Sidebar` & `ToastManager`: Thanh điều khiển chạy/dừng mô phỏng, bật/tắt bộ sinh, thống kê trực tiếp theo từng `Tag`, chip trạng thái kết nối PostgreSQL và hệ thống thông báo nổi góc dưới màn hình.
* **Hộp thoại chức năng (`gui/dialogs/`):** `EmployeeDialog` (thêm/sửa nhân viên), `ExportDialog` (chọn ngày trích xuất CSV) và `SettingsDialog` (cấu hình 4 tab cho toàn bộ hệ thống).
