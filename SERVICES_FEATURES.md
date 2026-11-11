# TỔNG HỢP TÍNH NĂNG CÁC MICROSERVICES TRONG HỆ THỐNG HRM ENTERPRISE

> **Kiến trúc:** Monorepo 3 tầng, On-Premises SOA, Flat 3-Tier Architecture (Mẫu A), Database-per-service (5 PostgreSQL độc lập).  
> **Cổng vào tập trung:** API Gateway (`:4000`) -> Các Core Services (`:4001` - `:4005`).

---

## 1. BẢNG TỔNG HỢP CÁC SERVICE

| STT | Tên Service | Port | Database riêng | Công nghệ / Giao thức | Trách nhiệm chính |
|:---:|---|:---:|---|---|---|
| **0** | **API Gateway** | `4000` | *Không (Stateless)* | Express, HTTP Proxy, JWT Verify, Rate Limit | Cửa ngõ duy nhất, xác thực tập trung, định tuyến, bảo mật |
| **1** | **Identity Service** | `4001` | `hrm_identity` (5433) | Express, Prisma, JWT, bcryptjs, Cookie | Quản lý danh tính, tài khoản, phiên làm việc, thiết bị |
| **2** | **Organization Service** | `4002` | `hrm_organization` (5434) | Express, Prisma | Cơ cấu tổ chức, chi nhánh, phòng ban, hồ sơ nhân sự |
| **3** | **Work Service** | `4003` | `hrm_work` (5435) | Express, Prisma | Ca kíp mẫu, đăng ký/xếp ca, chấm công, giao việc |
| **4** | **Payroll Service** | `4004` | `hrm_payroll` (5436) | Express, Prisma, SAGA | Tài khoản DN, tính lương, phiếu lương, lệnh chi Idempotent |
| **5** | **Integration Service** | `4005` | `hrm_integration` (5437) | Express, Prisma, SOAP XML, lưu trữ file nội bộ | Cổng ngân hàng SOAP, đơn từ, Wi-Fi, bảng tin, quy định, upload, kho lưu trữ dữ liệu cũ |

---

## 2. CHI TIẾT TÍNH NĂNG TỪNG SERVICE

### 🚪 2.0. API Gateway (Port: 4000)
- **Điểm vào duy nhất (Single Point of Entry):** Đón nhận toàn bộ request từ Web Portal (Next.js) và Mobile App (Flutter).
- **Định tuyến thông minh (Reverse Proxy):** Chuyển tiếp request đến đúng 5 microservices phía sau dựa theo prefix đường dẫn (`/api/auth`, `/api/branches`, `/api/shifts`, `/api/payroll`, `/soap/payroll`...).
- **Xác thực JWT tập trung (Centralized Authentication):**
  - Kiểm tra tính hợp lệ và chữ ký của Access Token hoặc cookie `hrm-session`.
  - Trích xuất thông tin người dùng (`userId`, `role`, `branchSlug`) và tự động gắn vào các header nội bộ: `x-user-id`, `x-user-role`, `x-branch-slug` để các service con không cần query DB lại.
- **Chống lạm dụng (Rate Limiting):** Giới hạn tần suất request (mặc định 300 req / phút) để bảo vệ hệ thống trước tấn công spam API/DDoS.
- **Quản lý CORS & Cookie:** Hỗ trợ CORS có credentials cho trình duyệt web.
- **Xử lý sự cố & Timeout:** Cấu hình timeout mặc định 5000ms; tự động trả về `502 Bad Gateway` chuẩn hóa khi một service bị ngắt kết nối.

---

### 🔐 2.1. Identity Service (Port: 4001)
- **Database:** `hrm_identity` (Port container: 5433)
- **Xác thực người dùng (Authentication):**
  - `POST /api/auth/login`: Đăng nhập bằng Email & Mật khẩu (kiểm tra hash bcrypt).
  - `POST /api/auth/logout`: Đăng xuất, xóa phiên và làm sạch cookie phiên.
  - `POST /api/auth/refresh`: Cấp mới Access Token (15 phút) bằng Refresh Token (7 ngày).
  - `POST /api/auth/forgot-password`: Quy trình hỗ trợ cấp lại mật khẩu.
- **Phân quyền vai trò (RBAC):**
  - Quản trị viên (`admin`): Toàn quyền trên toàn hệ sinh thái doanh nghiệp.
  - Quản lý chi nhánh (`manager`): Xem và vận hành phạm vi chi nhánh của mình.
  - Nhân viên (`staff`): Thao tác ca kíp, chấm công, nộp đơn từ, xem lương.
- **Quản lý phiên & Thiết bị (Sessions & Devices):**
  - `GET /api/auth/me`: Trả về thông tin danh tính của người dùng đang đăng nhập.
  - `POST /api/auth/change-password`: Đổi mật khẩu cá nhân.
  - `GET /api/auth/devices`: Liệt kê các thiết bị đang đăng nhập tài khoản.
  - `DELETE /api/auth/devices/:id`: Đăng xuất và thu hồi phiên từ xa theo từng thiết bị.

---

### 🏢 2.2. Organization Service (Port: 4002)
- **Database:** `hrm_organization` (Port container: 5434)
- **Quản lý Chi nhánh (Branches):**
  - `GET /api/branches`: Lấy danh sách toàn bộ chi nhánh.
  - `GET /api/branches/:slug`: Xem thông tin chi tiết một chi nhánh theo mã slug (`HN-1`, `HN-2`, `DN-1`...).
  - `POST /api/branches`, `PUT /api/branches/:slug`, `DELETE /api/branches/:slug`: Thêm, sửa, xóa/khóa chi nhánh.
- **Quản lý Phòng ban (Departments):**
  - `GET /api/departments`: Danh sách phòng ban công ty.
  - `POST /api/departments`, `PUT /api/departments/:id`, `DELETE /api/departments/:id`: Thêm mới, cập nhật tên/mô tả, bổ nhiệm trưởng phòng ban.
- **Hồ sơ Nhân sự (Employees):**
  - `GET /api/employees`: Danh sách nhân viên (hỗ trợ lọc theo phòng ban, chi nhánh, trạng thái).
  - `GET /api/employees/:id`: Chi tiết hồ sơ nhân sự (CCCD, hợp đồng, chi nhánh, ngân hàng).
  - `POST /api/employees`: Thêm nhân viên mới, thiết lập hình thức tính lương (`hourly` / `monthly`).
  - `PUT /api/employees/:id`: Cập nhật thông tin nhân viên, tài khoản chi trả lương.
  - `POST /api/employees/:id/leave`: Xử lý thủ tục thôi việc / đình chỉ công tác nhân sự.

---

### ⏱️ 2.3. Work Service (Port: 4003)
- **Database:** `hrm_work` (Port container: 5435)
- **Khung ca mẫu (Shift Templates):**
  - `GET /api/shift-templates`: Danh sách các khung ca mẫu (Ca sáng, Ca chiều, Ca tối, Ca hành chính...).
  - `POST`, `PUT`, `DELETE /api/shift-templates`: Thiết lập giờ bắt đầu, kết thúc, thời gian nghỉ, hệ số lương.
- **Đăng ký & Phân bổ Ca kíp (Work Shifts & Registration):**
  - `POST /api/shifts/register`: Nhân viên đăng ký nguyện vọng làm việc theo tuần.
  - `GET /api/shifts/registrations`: Quản lý duyệt nguyện vọng đăng ký ca.
  - `GET /api/shifts`: Lấy danh sách lịch ca làm việc theo ngày/tuần/chi nhánh.
  - `POST /api/shifts`: Tạo ca làm việc theo ngày.
  - `POST /api/shifts/:id/assign`: Điều phối/xếp nhân sự vào ca cụ thể.
- **Chấm công thông minh (Attendance Tracking):**
  - `GET /api/attendance/config`: Xem cấu hình chấm công (dung sai đi muộn, về sớm, bắt buộc Wi-Fi).
  - `PUT /api/attendance/config`: Quản trị viên cập nhật quy chuẩn chấm công.
  - `POST /api/attendance/checkin`: Nhân viên điểm danh vào ca (xác thực thời gian, Wi-Fi).
  - `POST /api/attendance/checkout`: Nhân viên điểm danh tan ca, hệ thống tính số giờ làm thực tế.
  - `GET /api/attendance`: Thống kê nhật ký chấm công phục vụ tính lương.
- **Quản lý Giao việc & Nhiệm vụ (Tasks):**
  - `GET /api/tasks`: Danh sách nhiệm vụ theo ca kíp/chi nhánh.
  - `POST /api/tasks`: Quản lý giao việc mới cho nhân viên kèm hạn hoàn thành (deadline).
  - `PUT /api/tasks/:id`: Cập nhật tiến độ, trạng thái (Chờ làm, Đang làm, Hoàn thành, Quá hạn), đính kèm bằng chứng ảnh chụp camera.

---

### 💰 2.4. Payroll Service (Port: 4004)
- **Database:** `hrm_payroll` (Port container: 5436)
- **Tài khoản Nguồn Doanh nghiệp (Bank Accounts):**
  - `GET /api/payroll/bank-accounts`: Quản lý tài khoản ngân hàng của công ty dùng để giải ngân.
  - `POST /api/payroll/bank-accounts`: Thêm tài khoản ngân hàng liên kết.
  - `PUT /api/payroll/bank-accounts/:id`: Cập nhật số dư, chỉ định Tài khoản chi lương chính (`isPrimary`).
- **Bảng lương & Phiếu lương (Payslips):**
  - `POST /api/payroll/payslips/generate`: Tự động tổng hợp dữ liệu giờ công từ `work-service` và mức lương từ `organization-service` để sinh bảng lương tháng.
  - `GET /api/payroll/payslips`: Danh sách phiếu lương theo tháng/kỳ.
  - `PUT /api/payroll/payslips/:id`: Điều chỉnh các khoản phụ cấp, thưởng, phạt đi muộn, trừ bảo hiểm.
  - `PUT /api/payroll/payslips/:id/status`: Chốt phiếu lương (chuyển sang trạng thái khóa/đã duyệt).
- **Lệnh chi lương Idempotent & SAGA (Payouts):**
  - `POST /api/payroll/payouts`: Tạo lệnh chi trả lương tự động.
  - **Idempotency Key:** Sử dụng khoá `idempotencyKey` đảm bảo mỗi lệnh chi chỉ được thực thi duy nhất 1 lần, chống chi trùng ngân sách.
  - **Quy trình SAGA chi lương:** Trích nợ tài khoản công ty -> Gửi lệnh sang `integration-service` (cổng SOAP) -> Nhận mã đối soát `BANK-*` -> Cập nhật trạng thái thành công hoặc phát sinh giao dịch bù trừ nếu thất bại.

---

### 🌐 2.5. Integration Service (Port: 4005)
- **Database:** `hrm_integration` (Port container: 5437)
- **Cổng thanh toán liên ngân hàng SOAP (Banking SOAP Gateway):**
  - `GET /soap/payroll?wsdl`: Cung cấp file đặc tả WSDL chuẩn XML cho đối tác ngân hàng.
  - `POST /soap/payroll`: Tiếp nhận và xử lý điện chuyển tiền XML (`PayoutRequest` / `PayoutResponse`).
  - `GET`, `PUT /api/soap-config`: Cấu hình kết nối ngân hàng (endpoint, WSDL, chứng chỉ mTLS, IP cho phép); `POST /api/soap-config/ping` đo thật độ trễ và trạng thái endpoint.
- **Quản lý Đơn từ & Phê duyệt (Requests):**
  - `GET /api/requests`: Danh sách đơn từ (nghỉ phép, đổi ca, giải trình công).
  - `POST /api/requests`: Nhân viên tạo đơn mới.
  - `PUT /api/requests/:id/approve`: Quản lý phê duyệt đơn.
  - `PUT /api/requests/:id/reject`: Quản lý từ chối đơn kèm lý do.
- **Cấu hình Wi-Fi Chấm công (Wi-Fi Configs):**
  - `GET /api/wifi-configs`: Danh sách điểm truy cập Wi-Fi hợp lệ theo từng chi nhánh.
  - `POST`, `PUT`, `DELETE /api/wifi-configs`: Thêm/sửa SSID, BSSID (địa chỉ MAC router) để phục vụ định danh vị trí chấm công trên Mobile App.
- **Bảng tin & Thông báo nội bộ (News & Notifications):**
  - `GET /api/news`: Đọc tin tức truyền thông nội bộ, chính sách mới.
  - `POST /api/news`: Đăng bài thông báo công ty, gắn cờ bài viết quan trọng.
  - `GET /api/notifications`: Danh sách thông báo cá nhân.
  - `PUT /api/notifications/:id/read`: Đánh dấu đã đọc thông báo.
- **Tải ảnh vào lưu trữ nội bộ (Internal File Storage):**
  - `POST /api/upload`: Nhận ảnh Base64/Data URL từ client, kiểm tra MIME + magic bytes, lưu vào ổ đĩa máy chủ (`STORAGE_DIR`) và trả về link tuyệt đối qua Gateway (dùng cho ảnh đại diện, ảnh 2 mặt CCCD, ảnh bằng chứng hoàn thành nhiệm vụ).
  - `GET /api/files/:folder/:name`: Phục vụ file sau xác thực; `DELETE` để xóa file; `POST /api/files/scan-orphans` + `purge-orphans` đối soát ảnh mồ côi (admin).
- **Kho lưu trữ dữ liệu cũ (Archive — chỉ admin):**
  - `POST /api/archive/integration/preview|run`: Chuyển đơn từ đã duyệt/từ chối và thông báo đã đọc quá hạn vào bảng archive theo batch, vẫn xem lại và khôi phục được (`GET .../requests|notifications`, `POST .../:id/restore`). Work-service có tuyến tương tự `/api/archive/work/*` cho nhiệm vụ hoàn thành.
- **Văn bản Nội quy & Quy định (Regulations):**
  - `GET /api/regulations`: Tra cứu tài liệu nội quy công ty, phân loại theo danh mục.
  - `POST`, `PUT`, `DELETE /api/regulations`: Ban hành, cập nhật tài liệu quy chế doanh nghiệp.

---

## 3. CƠ CHẾ GIAO TIẾP LIÊN DỊCH VỤ (IPC)

1. **Giao tiếp REST nội bộ qua HTTP Client:**
   - Các service gọi nhau qua `src/clients/` (ví dụ `PayrollService` gọi `OrgClient` và `WorkClient`).
   - Timeout quy định nghiêm ngặt: tối đa **5000ms**, có cơ chế bắt lỗi và fallback.
2. **Nguyên tắc Database-Per-Service:**
   - Không JOIN bảng chéo giữa các database.
   - Các bảng chỉ lưu tham chiếu chéo dưới dạng chuỗi (`employeeId`, `branchSlug`, `shiftId`).
3. **Mô hình Giao dịch Phân tán (Distributed Transactions / SAGA):**
   - Áp dụng trong quy trình chi lương: `payroll-service` -> `integration-service` (cổng SOAP) -> ghi có tài khoản nhân viên.
   - Lưu trữ nhật ký giao dịch từng bước kèm Idempotency Key để đảm bảo tính toàn vẹn tài chính.
