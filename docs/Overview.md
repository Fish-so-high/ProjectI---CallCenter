# Tổng quan về CallCenter

## 1. Giới thiệu

**CallCenter** là hệ thống mô phỏng hoạt động điều phối cuộc gọi thời gian thực tại trung tâm chăm sóc khách hàng theo mô hình **hai hàng đợi FIFO ghép cặp 1–1**:
* Cuộc gọi đến được đưa vào hàng đợi chờ phục vụ (`CallingQueue`) với sức chứa giới hạn.
* Nhân viên trực tổng đài ở trạng thái rảnh được xếp vào hàng đợi phục vụ (`ServiceQueue`) theo thứ tự FIFO.
* Ngay khi cả hai hàng đợi đều có phần tử, hệ thống ghép cặp cuộc gọi đầu hàng với nhân viên rảnh lâu nhất để xử lý song song và cập nhật trạng thái trực quan lên giao diện Desktop (PySide6) cũng như lưu trữ bền vững vào PostgreSQL.

## 2. Dữ liệu đầu vào (Input)
* **Dữ liệu cuộc gọi (`IncomingCall` / file `input_testcase.txt` hoặc bộ sinh ngẫu nhiên):**
  Mỗi cuộc gọi gồm 4 thuộc tính chính:
  1. `phone_number` (`str`): Số điện thoại khách hàng gồm **10 chữ số**, bắt đầu bằng đầu số `09`, `08`, `02`, hoặc `03`[cite: 1].
  2. `event` (`CallEvent`): Kịch bản của cuộc gọi, nhận 1 trong 3 giá trị:
     * `Success`: Khách hàng và nhân viên trao đổi thành công[cite: 1].
     * `Miss`: Cuộc gọi nhỡ (thời lượng xử lý thực tế bị ép về `0.0s`)[cite: 1].
     * `Break`: Khách hàng hoặc cuộc gọi bị ngắt giữa chừng sau một khoảng thời gian ngắn[cite: 1].
  3. `time_call` (`float`): Thời lượng cuộc gọi dự kiến (tính bằng giây, $\ge 0$). Trong chế độ sinh ngẫu nhiên: `Success` $\in [0, 3]\text{s}$, `Break` $\in [0, 1]\text{s}$, `Miss` $= 0\text{s}$[cite: 1].
  4. `received_at` (`datetime`) & `call_id` (`str` UUID): Thời điểm cuộc gọi chạm tới tổng đài và mã định danh duy nhất[cite: 1].
  *(Định dạng dòng trong file text: `phone_number|event|time_call`, ví dụ: `0912345678|Success|2.45`)*[cite: 1].

* **Dữ liệu nhân viên (`Employee` / file `data/employees.txt`):**
  * Định dạng khai báo: `employee_id|employee_name[|status]` (ví dụ: `NV001|Nguyen Van A|AVAILABLE`)[cite: 1].
  * `employee_id`: Mã duy nhất, không chứa khoảng trắng hoặc ký tự `|`[cite: 1].
  * `status` (`EmployeeStatus`): `AVAILABLE` (Rảnh - mặc định), `BUSY` (Đang nghe máy - chỉ phát sinh khi chạy), `BREAK` (Tạm nghỉ), `OFFLINE` (Ngừng hoạt động)[cite: 1].
  * Có cơ chế `pending_status`: Nếu nhân viên đang `BUSY` mà bấm Tạm nghỉ/Offline, hệ thống ghi nhận trạng thái chờ và tự động chuyển trạng thái ngay khi nhân viên kết thúc cuộc gọi hiện tại[cite: 1].

## 3. Dữ liệu đầu ra (Output)
Sau khi `CallCenterEngine` xử lý xong, mỗi cuộc gọi tạo ra một bản ghi `CallResult` (lưu vào bảng `call_records` trong PostgreSQL và xuất ra file CSV `calls-DD-MM-YYYY.csv`):
* **Các trường dữ liệu:** `call_id`, `phone_number`, `employee_id` (`None` nếu bị từ chối/cúp máy khi chờ), `received_at`, `started_at`, `ended_at`, `duration_seconds`, `status` (`CallStatus`), và `tag` (`CallTag`)[cite: 1].
* **Bảng ánh xạ kết quả xử lý:**

| Tình huống xảy ra | `CallStatus` (Trạng thái) | `CallTag` (Nhãn Output) | `employee_id` | Thời lượng (`duration_seconds`) |
| :--- | :--- | :--- | :--- | :--- |
| Phục vụ thành công (`event=Success`) | `COMPLETED` | `Success` | Có mã NV | `time_call` |
| Cuộc gọi nhỡ (`event=Miss`) | `MISSED` | `Miss` | Có mã NV | `0.0` |
| Ngắt giữa chừng khi đang phục vụ (`event=Break`) | `ABANDONED` | `Break` | Có mã NV | `time_call` |
| Hàng đợi `CallingQueue` đầy (`>= max_queue_size`) | `REJECTED` | `Rejected` | `None` | `0.0` |
| Khách tự cúp máy khi đang xếp hàng chờ | `ABANDONED` | `Abandoned` | `None` | `0.0` |

Ngoài ra, các cuộc gọi có `tag = Success` sẽ tự động cộng dồn số cuộc gọi và tổng thời gian đàm thoại theo từng nhân viên và từng tháng (`YYYY-MM`) vào bảng `employee_monthly_stats`[cite: 1].

## 4. Sơ đồ hoạt động của toàn bộ CallCenter Engine

```text
  [Nguồn cuộc gọi: RandomSource / ReplaySource]
                       │
                       ▼ (GeneratorWorker sinh các cuộc gọi ngẫu nhiên)
         ┌───────────────────────────┐
         │     CallCenterEngine      │
         │  (Kiểm tra trùng call_id) │
         └─────────────┬─────────────┘
                       │
         ┌─────────────▼─────────────┐      Đầy hàng đợi
         │       CallingQueue        ├──────────────────────► Tag: Rejected
         │    (Hàng đợi cuộc gọi)    │      Khách cúp máy
         └─────────────┬─────────────┘──────────────────────► Tag: Abandoned
                       │
                       ▼ (Ghép cặp 1-1: đầu CallingQueue + đầu ServiceQueue)
         ┌───────────────────────────┐      Nhân viên rảnh
         │       ServiceQueue        │◄─────────────────────┐
         │    (Hàng đợi NV rảnh)     │                      │
         └─────────────┬─────────────┘                      │
                       │ (NV chuyển sang BUSY)              │
                       ▼                                    │
         ┌───────────────────────────┐                      │
         │       CallProcessor       │                      │
         │  (Min-Heap theo end_at)   │                      │
         └─────────────┬─────────────┘                      │
                       │ (tick() khi now >= end_at)         │
                       ▼                                    │
         ┌───────────────────────────┐      Chính sách      │
         │   Hoàn tất & Giải phóng   ├──────────────────────┘
         │      Tạo CallResult       │  (hoặc chuyển BREAK / OFFLINE)
         │  (Success / Miss / Break) │
         └─────────────┬─────────────┘
                       │ (EventDispatcher phát CALL_FINISHED)
          ┌────────────┴────────────┐
          ▼                         ▼
 [AppController -> GUI]    [PersistenceService]
 (Cập nhật bảng & biểu đồ)  (Ghi batch vào PostgreSQL)
                                    │
                                    ▼ ( Tự động:19:00 & 00:00 hoặc Export thủ công)
                           [ArchiveService / CSV]
                           (Xuất calls-DD-MM-YYYY.csv)
```

