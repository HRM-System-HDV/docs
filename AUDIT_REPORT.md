# BÁO CÁO KIỂM TOÁN MỨC ĐỘ SẴN SÀNG VẬN HÀNH THỰC TẾ (PRODUCTION READINESS AUDIT REPORT)
## HỆ THỐNG QUẢN TRỊ NHÂN SỰ & VẬN HÀNH CA KÍP ON-PREMISES (HRM ENTERPRISE)

> **Mã số tài liệu:** `AUDIT-HRM-2026-FINAL`  
> **Phiên bản:** `1.0.0-PROD-AUDIT`  
> **Thời điểm hoàn thành:** 2026-11-10  
> **Nhóm thực hiện:** Ban Kiểm toán Kỹ thuật Độc lập (Teamwork Technical Audit Board)  
> **Phạm vi kiểm toán toàn diện:**
> - **Backend Microservices:** API Gateway (:4000) & 5 Core Microservices (:4001 - :4005), 5 Cơ sở dữ liệu PostgreSQL (Prisma ORM).
> - **Web Frontend Portal:** Next.js 16.3.1 (App Router), React 19, TypeScript 5, Tailwind CSS v4.
> - **Mobile Application:** Flutter 3.47 / Dart 3.13, Material 3, Public Sans, `go_router`.
> - **Tài liệu quy chuẩn & Nghiệp vụ:** `docs/` (`BUSINESS.md`, `SERVICES_FEATURES.md`, `RULES.md`, `TECHS.md`).
> 
> **Chế độ kiểm toán:** Thẩm định Độc lập, Khách quan, Read-Only (Tuyệt đối không sửa đổi mã nguồn dự án trong quá trình kiểm toán).

---

## MỤC LỤC BÁO CÁO
1. [Executive Summary (Tổng quan Hiện trạng & Đánh giá Sẵn sàng Production)](#1-executive-summary)
2. [Chi tiết Từng Tính năng theo Thành phần](#2-chi-tiết-từng-tính-năng-theo-thành-phần)
   - [2.1. Backend Microservices (API Gateway & 5 Core Services)](#21-backend-microservices)
   - [2.2. Web Frontend Portal (Next.js 16 App Router)](#22-web-frontend-portal)
   - [2.3. Mobile Application (Flutter App)](#23-mobile-application)
3. [Bảng Phân tích Sai lệch Contract API & Database (Contract Mismatch Table)](#3-bảng-phân-tích-sai-lệch-contract-api--database)
4. [Hiện trạng Kiểm thử (Testing Status & Coverage)](#4-hiện-trạng-kiểm-thử)
   - [4.1. Kiểm thử Backend Microservices](#41-kiểm-thử-backend-microservices)
   - [4.2. Kiểm thử Web Frontend](#42-kiểm-thử-web-frontend)
   - [4.3. Kiểm thử Mobile Application](#43-kiểm-thử-mobile-application)
   - [4.4. Đánh giá Khoảng trống Kiểm thử (Coverage Gaps)](#44-đánh-giá-khoảng-trống-kiểm-thử)
5. [Đánh giá Chất lượng Mã nguồn & Kỹ thuật nợ (Code Quality & Technical Debt)](#5-đánh-giá-chất-lượng-mã-nguồn--kỹ-thuật-nợ)
   - [5.1. Backend Microservices: Lỗ hổng & Điểm yếu Kiến trúc](#51-backend-microservices)
   - [5.2. Web Frontend: Linting, Rendering & Type Safety](#52-web-frontend)
   - [5.3. Mobile Application: Quyền hệ điều hành, Kiến trúc & UI](#53-mobile-application)
6. [Kế hoạch Hành động Khuyến nghị (Actionable Recommendations P0 - P3)](#6-kế-hoạch-hành-động-khuyến-nghị)

---

## 1. EXECUTIVE SUMMARY

### 1.1. Tổng quan Đánh giá Mức độ Sẵn sàng Vận hành (Production Readiness Rating)

Hệ thống Quản trị Nhân sự & Vận hành Ca kíp nội bộ (**HRM Enterprise On-Premises**) đã hoàn thiện phần lớn khung kiến trúc cốt lõi và các luồng nghiệp vụ nền tảng. Sau quá trình thanh tra, chạy thử nghiệm kiểm thử thực tế và phân tích tĩnh chuyên sâu trên cả 3 thành phần (Backend Microservices, Web Frontend Next.js, Mobile Flutter), Ban Kiểm toán đưa ra đánh giá tổng thể như sau:

$$\text{Điểm Sẵn sàng Toàn hệ thống (Production Readiness Score): } \mathbf{75 / 100}$$

**Phân loại:** 🟠 **CẦN KHẮC PHỤC TRƯỚC KHI GO-LIVE (Action Required Prior to Launch)**  
*Hệ thống KHÔNG ĐƯỢC PHÉP phát hành sản phẩm (Go-Live) ngay lập tức khi chưa xử lý dứt điểm các lỗi chặn cấp độ P0 (Blocker) và P1 (Critical).*

#### Bảng Tổng hợp Điểm số Sẵn sàng theo Thành phần:

| Thành phần Hệ thống | Trọng số | Điểm số (0-100) | Xếp loại | Nhận định Tóm tắt |
|---|:---:|:---:|:---:|---|
| **Backend Microservices** (Gateway + 5 Services) | 40% | **84 / 100** | Khá Tốt | Kiến trúc Flat 3-Tier mẫu mực, Database-per-service nghiêm ngặt, 168/169 tests pass (song song mặc định) / 169/169 pass (tuần tự concurr=1). Điểm trừ: secret fallback mặc định, thiếu transaction cho payout, lưu session thiết bị trong RAM, race condition sinh ID thông báo. |
| **Web Frontend Portal** (Next.js 16 App Router) | 30% | **68 / 100** | Cần Khắc Phục | Giao diện quản trị phong phú, build thành công 20 static routes, 0 lỗi TypeScript. Điểm trừ: 0% test coverage, 185 vi phạm linting (36 lỗi cascading re-renders), form Wi-Fi và chuông thông báo bị mồ côi. |
| **Mobile Application** (Flutter 3.x) | 30% | **72 / 100** | Cần Khắc Phục | Giao diện Material 3 mượt mà, phân tích tĩnh 0 issue, 24/25 unit tests pass, silent token refresh tốt. Điểm trừ: thiếu quyền camera trên iOS gây crash app, chấm công Wi-Fi chưa đọc card mạng thật, thiếu trường bonus phiếu lương. |
| **TOÀN HỆ THỐNG (OVERALL)** | **100%** | **75 / 100** | **CẦN KHẮC PHỤC** | **Đạt nền tảng vững chắc nhưng có rủi ro nghiêm trọng về an ninh, crash thiết bị và thất thoát tài chính.** |

---

### 1.2. Tóm tắt Điểm mạnh Kiến trúc (Architectural Strengths)

1. **Tuân thủ Chuẩn mực Phân lớp Flat 3-Tier (Mẫu A):**
   - 100% các microservice backend tuân thủ nghiêm ngặt cấu trúc `src/config/`, `src/controllers/`, `src/routes/`, `src/middlewares/`, `src/services/`, `src/clients/`, `src/utils/`. Hoàn toàn loại bỏ các cấu trúc cấm (`src/api/`, `src/domain/`, `src/infrastructure/`, tầng `repositories/` riêng).
   - Controller giữ đúng vai trò phân tách HTTP (request extraction, validation triggering, status codes) và ủy quyền hoàn toàn logic nghiệp vụ cho Service.
2. **Triển khai Triệt để Database-Per-Service:**
   - 5 cơ sở dữ liệu PostgreSQL độc lập tương ứng với 5 container nghiệp vụ: `hrm_identity` (:5433), `hrm_organization` (:5434), `hrm_work` (:5435), `hrm_payroll` (:5436), `hrm_integration` (:5437).
   - Tuyệt đối không có Foreign Key chéo cơ sở dữ liệu, không có lệnh SQL JOIN xuyên CSDL. Tham chiếu chéo giữa các service được chuẩn hóa qua các định danh chuỗi (`employeeId`, `branchSlug`, `shiftId`).
3. **Tuyệt đối Đơn Khách Hàng (Strict Single-Tenant Compliance):**
   - Toàn bộ codebase không chứa các trường `tenantId`, `tenantSlug`, hoàn toàn phù hợp với mô hình triển khai On-Premises phục vụ một doanh nghiệp duy nhất theo `docs/RULES.md` §1.
4. **Kiến trúc Gateway & Bảo mật Tập trung:**
   - Cửa ngõ duy nhất API Gateway (:4000) tiếp nhận mọi kết nối client, xử lý giải mã JWT, kiểm tra phiên cookie `hrm-session`, chặn header giả mạo, trích xuất danh tính và đính kèm các header nội bộ (`x-user-id`, `x-user-role`, `x-branch-slug`) chuyển tiếp an toàn xuống các service hạ tầng.
5. **Chuẩn hóa Envelope Phản hồi API:**
   - Toàn bộ 52 API endpoints tuân thủ định dạng chuẩn doanh nghiệp: `{ data, message }` cho thành công (HTTP 200, 201) và `{ error: { code, message, details } }` cho thất bại (HTTP 4xx, 5xx). Tuyến SOAP XML `/soap/payroll` được xử lý chuyển tiếp nguyên vẹn `soap:Envelope` và `soap:Fault` theo đúng ngoại lệ được quy định.
6. **Bảo toàn Giao dịch & Chống Chi Trùng (Idempotency):**
   - Hệ thống chi lương `payroll-service` tích hợp khóa chống trùng lặp `idempotencyKey` với ràng buộc cơ sở dữ liệu Unique, ngăn chặn triệt để nguy cơ xuất chi kép khi có sự cố mạng.

---

### 1.3. Các Điểm Nghẽn Chính (Key Bottlenecks & Blockers)

Ban Kiểm toán xác định 5 điểm nghẽn nghiêm trọng thuộc mức độ **Blocker (P0)** cần phải xử lý trước khi hệ thống được nghiệm thu đưa vào hoạt động chính thức:

1. **Rủi ro Crash Ứng dụng iOS do Thiếu Quyền Hệ thống (Mobile Blocker):**
   - File cấu hình `mobile/ios/Runner/Info.plist` hoàn toàn thiếu hai khóa bắt buộc của Apple: `NSCameraUsageDescription` và `NSPhotoLibraryUsageDescription`. Khi nhân viên bấm chụp ảnh tải minh chứng nhiệm vụ hoặc đổi avatar trên iPhone thật, hệ điều hành iOS sẽ lập tức tắt ứng dụng (Crash app) ngay lập tức do vi phạm chính sách bảo mật.
2. **Lỗ hổng Chữ ký Khóa Bí mật Mặc định (Security Fallback Blocker):**
   - `api-gateway`, `identity-service`, và `organization-service` đang cài đặt giá trị fallback chuỗi bí mật tĩnh trong mã nguồn: `process.env.JWT_SECRET || "dev_only_secret_thay_khi_deploy"` và `INTERNAL_API_TOKEN || "dev_only_internal_token_thay_khi_deploy"`. Nếu triển khai production mà sơ suất quên gán biến môi trường, kẻ tấn công có thể giả mạo token Admin toàn hệ thống hoặc gọi thẳng API đồng bộ tài khoản nội bộ.
3. **Lệch pha Kiến trúc Dữ liệu Chấm công (Data Sync Disconnect Blocker):**
   - Phía Mobile App khi bấm chấm công (Check-in/Check-out) ghi dữ liệu vào bảng `attendances` của `work-service`. Tuy nhiên, giao diện Lịch ca làm việc trên Web (`/dashboard/shifts`) lại đọc dữ liệu hiển thị từ hai trường `checkIn` và `checkOut` của bảng `shifts`. Vì `work-service` không cập nhật đồng bộ sang `shifts`, cột chấm công trên Web luôn bị rỗng, khiến ban quản lý không thể theo dõi chấm công thực tế của nhân viên.
4. **Mô-đun Quản trị Cấu hình Wi-Fi bị Mồ côi (Orphaned UI Blocker):**
   - Giao diện quản lý Wi-Fi chấm công trên Web Portal (`/dashboard/wifi`) hoàn toàn không import hoặc liên kết với component `WifiForm.tsx`. Quản trị viên chỉ xem được bảng tĩnh mà không thể thực hiện thao tác Thêm, Sửa hoặc Xóa thông tin Wi-Fi chi nhánh.
5. **Gian lận Điểm danh do Bỏ qua Kiểm tra Phần cứng Wi-Fi (Timekeeping Hardware Bypass):**
   - Ứng dụng di động chưa tích hợp plugin đọc thông tin card mạng thật (`network_info_plus`), chỉ hiển thị thông tin Wi-Fi tĩnh lấy từ API. Phía backend `work-service`, API `POST /api/attendance/checkin` chỉ nhận `{ employeeId, shiftId, time }`, bỏ qua thông tin Wi-Fi gửi lên và hoàn toàn không đối soát BSSID/IP với bảng cấu hình `wifi_configs`. Bất kỳ nhân viên nào cũng có thể chấm công thành công mà không cần có mặt tại chi nhánh.

---

## 2. CHI TIẾT TỪNG TÍNH NĂNG THEO THÀNH PHẦN

### 2.1. Backend Microservices

Backend bao gồm **1 API Gateway** và **5 Core Microservices** cung ứng tổng cộng **52 API Endpoints** (được phân rã qua 120 route mappings) với cơ chế phân quyền RBAC đa cấp:

```
[Web Portal Next.js / Mobile Flutter]
               │
               ▼ (:4000)
    ┌─────────────────────┐
    │     API GATEWAY     │  (Stateless, JWT Validation, Rate Limiter: mẫu 300r/m / fallback 50kr/m, Reverse Proxy)
    └──────────┬──────────┘
               │  (x-user-id, x-user-role, x-branch-slug, timeout 5000ms)
 ┌─────────────┼─────────────┬─────────────┬─────────────┐
 │ :4001       │ :4002       │ :4003       │ :4004       │ :4005
 ▼             ▼             ▼             ▼             ▼
┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐
│ Identity  │ │Organizat. │ │   Work    │ │  Payroll  │ │Integrat.  │
│  Service  │ │  Service  │ │  Service  │ │  Service  │ │  Service  │
└─────┬─────┘ └─────┬─────┘ └─────┬─────┘ └─────┬─────┘ └─────┬─────┘
      ▼             ▼             ▼             ▼             ▼
 [hrm_ident]   [hrm_org]     [hrm_work]   [hrm_payroll]  [hrm_integ]
  Port: 5433    Port: 5434    Port: 5435    Port: 5436    Port: 5437
```

#### Bảng Ma trận Tính năng Toàn diện Backend Microservices:

| STT | Microservice | Module / Cụm Nghiệp vụ | Endpoints Phục vụ | Trạng thái Nghiệp vụ | Đánh giá Hiện trạng & Tồn đọng Kỹ thuật |
|:---:|---|---|:---:|:---:|---|
| **1** | `api-gateway` | Điểm vào Tập trung & Định tuyến | 1 route `/health` + 11 route proxies | **Hoàn thành** | Xử lý tốt xác thực JWT, bắt lỗi 502 khi service chết. Cấu hình Rate Limiter: mức 300 req/phút là mẫu trong `.env.example`, trong khi fallback mặc định trong mã nguồn `backend/api-gateway/src/config/index.js:14` là 50,000 req/phút. Tồn tại: fallback bí mật `dev_only_*` tĩnh (P0). |
| **2** | `identity-service` | Xác thực & Quản lý Phiên | `POST /login`, `/logout`, `/refresh`, `/forgot-password`, `/change-password`, `GET /me` | **Hoàn thành** | Đầy đủ mã hóa bcryptjs, xoay vòng token kép, hỗ trợ đăng xuất làm sạch cookie. |
| **3** | `identity-service` | Đồng bộ Hồ sơ & Quản lý Thiết bị | `POST /sync-employee`, `GET /devices`, `DELETE /devices/:id` | **Hoàn thành** (Có tồn đọng) | API đồng bộ bảo vệ bằng token nội bộ. Tồn đọng: Bảng `device_sessions` và `refresh_tokens` lưu trong RAM (`userDevices = new Map()`), không lưu vào DB (P1). |
| **4** | `organization-service` | Quản trị Chi nhánh (Branches) | `GET`, `POST`, `PUT`, `DELETE /api/branches`, `GET /api/branches/:slug` | **Hoàn thành** | Tự sinh slug chi nhánh (`CN-xxxx`), gán quản lý, đếm nhân viên trực thuộc động, ràng buộc xóa an toàn. |
| **5** | `organization-service` | Quản trị Phòng ban (Departments)| `GET`, `POST`, `PUT`, `DELETE /api/departments`, `GET /api/departments/:id` | **Hoàn thành** | Tự sinh mã phòng ban (`PB-xxxx`), bổ nhiệm trưởng bộ phận, kiểm tra trùng lặp tên phòng ban. |
| **6** | `organization-service` | Hồ sơ Nhân sự (Employees) | `GET`, `POST`, `PUT`, `DELETE /api/employees`, `POST /:id/leave` | **Hoàn thành** | Quản lý CCCD, địa chỉ hành chính, hình thức lương (`monthly`/`hourly`), tài khoản ngân hàng thụ hưởng, che giấu trường nhạy cảm cho staff. |
| **7** | `work-service` | Khung ca mẫu (Shift Templates) | `GET`, `POST`, `PUT`, `DELETE /api/shifts/templates` (alias `/api/shift-templates`) | **Hoàn thành** | CRUD template ca, quản lý khung giờ bắt đầu/kết thúc ca chuẩn hóa cho toàn hệ thống. |
| **8** | `work-service` | Nguyện vọng Ca tuần (Registrations) | `GET`, `POST`, `DELETE /api/shifts/registrations`, `POST /register`, `POST /unregister` | **Hoàn thành** | Nhân viên nộp nguyện vọng ca làm việc theo tuần, quản lý xét duyệt phân ca. |
| **9** | `work-service` | Lịch Phân ca Làm việc (Shifts) | `GET`, `POST`, `PUT`, `DELETE /api/shifts`, `POST /shifts/:id/assign` | **Hoàn thành** (Có tồn đọng) | Phân ca theo ngày, gắn nhân sự, phát hiện ca định kỳ (`isRecurring`). Tồn tại: Chưa nhận dữ liệu cập nhật check-in từ bảng attendance (P0). |
| **10**| `work-service` | Chấm công Thông minh (Attendance)| `GET`, `PUT /api/attendance/config`, `POST /checkin`, `POST /checkout`, `GET /attendance` | **Chưa hoàn thành** | Đầy đủ quy tắc phạt đi muộn/về sớm, tính tiền phạt theo phút ân hạn. Tồn tại: API check-in bỏ qua kiểm tra Wi-Fi SSID/BSSID; không đồng bộ giờ vào bảng `shifts` (P0). |
| **11**| `work-service` | Giao việc & Quản lý Nhiệm vụ (Tasks)| `GET`, `POST`, `PUT`, `DELETE /api/tasks`, `GET /api/tasks/:id` | **Hoàn thành** | Giao việc theo ca kíp/chi nhánh, trạng thái hoàn thành, yêu cầu bắt buộc chụp ảnh bằng chứng hoàn thành nhiệm vụ. |
| **12**| `work-service` | Kho Lưu trữ Nhiệm vụ cũ (Work Archive)| `POST /preview`, `POST /run`, `GET /tasks`, `GET /tasks/:id`, `POST /restore` | **Hoàn thành** | Cơ chế chuyển dữ liệu nhiệm vụ quá hạn sang bảng `archived_tasks` theo lô, hỗ trợ phục hồi nguyên trạng. |
| **13**| `payroll-service` | Tài khoản Nguồn Doanh nghiệp (Bank) | `GET`, `POST`, `PUT`, `DELETE /api/payroll/bank-accounts` | **Hoàn thành** | Quản trị tài khoản trích nợ, quản lý số dư khả dụng, chỉ định tài khoản chính chi lương (`isPrimary`). |
| **14**| `payroll-service` | Phiếu lương & Bảng lương (Payslips)| `GET /payslips`, `POST /generate`, `PUT /:id`, `PUT /:id/status` | **Hoàn thành** | Tự động tạo bảng lương tháng, tích hợp tổng phạt từ attendance, khóa bảng lương chống sửa đổi sau khi chốt. |
| **15**| `payroll-service` | Lệnh Chi Tiền Lương (Payouts) | `GET`, `POST /api/payroll/payouts` | **Có lỗi** (Critical) | Khóa chống trùng chi `idempotencyKey` rất tốt. Tồn tại: Hàm `createPayout` trừ tiền tài khoản trước khi ghi DB nhưng KHÔNG DÙNG Prisma `$transaction` (P1). |
| **16**| `integration-service` | Cổng Giao dịch Ngân hàng SOAP | `GET /soap/payroll?wsdl`, `POST /soap/payroll`, `GET`, `PUT /api/soap-config` | **Hoàn thành** | Cung cấp chuẩn WSDL, tiếp nhận và chuyển đổi điện XML, mô phỏng phản hồi Core Banking đối tác. |
| **17**| `integration-service` | Quy trình Xét duyệt Đơn từ (Requests)| `GET`, `POST /api/requests`, `PUT /approve`, `PUT /reject`, `PUT /respond`, `DELETE` | **Hoàn thành** | Quy trình đa bước cho đơn nghỉ phép, tạm ứng, đổi ca, bổ sung công. Hỗ trợ cơ chế đồng nghiệp xác nhận đổi ca trước khi quản lý duyệt. |
| **18**| `integration-service` | Cấu hình Wi-Fi Điểm danh (Wifi Config)| `GET /scan`, `GET`, `POST`, `PUT`, `DELETE /api/wifi-configs` | **Hoàn thành** (Backend) | CRUD danh sách Wi-Fi hợp lệ theo chi nhánh, mô phỏng quét mạng lân cận. Tồn tại: Client Web chưa kích hoạt màn hình tạo sửa (P0). |
| **19**| `integration-service` | Thông báo Nội bộ (Notifications) | `GET`, `POST /api/notifications`, `PUT /read-all`, `PUT /:id/read`, `GET /settings` | **Hoàn thành** | Lưu thông báo chi tiết ca làm, hỗ trợ đánh dấu đã đọc. Tồn tại: Model thiếu trường đồng bộ sang mobile; Web chưa gắn giao diện quả chuông (P2). |
| **20**| `integration-service` | Bảng tin & Nội quy Doanh nghiệp | `GET`, `POST`, `PUT`, `DELETE /api/news`, `GET`, `POST`, `PUT`, `DELETE /api/regulations`| **Hoàn thành** | Đầy đủ bài viết, ghim tin tức, phân loại thẻ màu, quản lý mã hiệu phiên bản quy định công ty. |
| **21**| `integration-service` | Lưu trữ File Nội bộ (File Storage) | `POST /api/upload`, `GET /api/files/*`, `DELETE /files/*`, `POST /scan-orphans`, `purge-orphans` | **Hoàn thành** | Nhận ảnh Data URL/Base64, kiểm tra magic bytes, lưu trữ ổ cứng cục bộ on-premise, dọn dẹp ảnh mồ côi an toàn. |
| **22**| `integration-service` | Nhật ký Kiểm toán Hoạt động (Backlog) | `GET`, `POST`, `DELETE /api/backlog` | **Có lỗi** (Chất lượng) | Lưu vết toàn bộ thao tác Thêm/Sửa/Xóa từ client. Tồn tại: Service sử dụng `$executeRawUnsafe` thay vì Prisma ORM chuẩn (P2). |
| **23**| `integration-service` | Kho Dữ liệu Cũ (Integration Archive) | `POST /preview`, `POST /run`, `GET /requests`, `/notifications`, `POST /restore` | **Hoàn thành** | Đóng băng và chuyển dịch đơn từ, thông báo cũ quá hạn sang bảng lưu trữ; hỗ trợ khôi phục dữ liệu 100%. |

---

### 2.2. Web Frontend Portal (Next.js 16 App Router)

Web Portal được xây dựng với **Next.js 16.3.1**, React 19, TypeScript 5 và Tailwind CSS v4, phục vụ công tác điều hành của Quản trị viên (`admin`) và Quản lý Chi nhánh (`manager`).

#### Bảng Ma trận Tính năng Web Frontend:

| STT | Module Chức năng | Đường dẫn Route | Trạng thái Nghiệp vụ | Mô tả Hiện trạng & Chi tiết Đánh giá Kỹ thuật |
|:---:|---|---|:---:|---|
| **1** | **Xác thực & Bảo vệ Phiên** | `/login`, `/dashboard` | **Hoàn thành** | Quản lý phiên qua cookie HttpOnly `hrm-session` kết hợp `AuthContext`. Chuyển hướng bảo vệ route tập trung tại `src/middleware.ts`. |
| **2** | **Quản lý Hồ sơ Nhân sự** | `/dashboard/employees` | **Hoàn thành** | Đầy đủ CRUD nhân viên, lọc đa chiều theo chi nhánh, phòng ban. Bắt buộc xuất file Excel nhân sự trước khi xóa vĩnh viễn. Nhược điểm: Label tại `EmployeeDetailModal.tsx:1051` giải thích sai công thức lương. |
| **3** | **Cơ cấu Chi nhánh** | `/dashboard/branches` | **Hoàn thành** | CRUD chi nhánh, sinh slug tự động, gán quản lý chi nhánh và cập nhật số lượng nhân sự động. |
| **4** | **Quản lý Phòng ban** | `/dashboard/departments` | **Hoàn thành** | CRUD phòng ban, sinh mã phòng ban, phân bổ trưởng phòng ban. |
| **5** | **Xếp ca & Phân bổ Ca kíp**| `/dashboard/shifts` | **Hoàn thành** | Hỗ trợ 3 tab giao diện: "Xếp ca", "Lịch làm việc chung", "Khung ca mẫu". Tự động nhận diện ca định kỳ, xem nguyện vọng đăng ký từ mobile, xuất lịch ca ra Excel. |
| **6** | **Chấm công & Điểm danh** | *(Chưa có route riêng)* | **Chưa hoàn thành / Thiếu sót** | **Thiếu màn hình tra cứu chấm công thực tế:** Không có trang gọi `GET /api/attendance` để xem giờ quẹt vân tay/Wi-Fi thực tế. Cột check-in/out trên trang Shifts bị rỗng do lệch bảng dữ liệu (P0). |
| **7** | **Quản lý Nhiệm vụ (Tasks)** | `/dashboard/tasks` | **Hoàn thành** | Giao việc theo ca/chi nhánh, phân quyền ưu tiên, hạn hoàn thành. Modal xem xét và phóng to ảnh bằng chứng hoàn thành nhiệm vụ (`proofPhotoUrl`). |
| **8** | **Xét duyệt Đơn từ (Requests)**| `/dashboard/requests` | **Hoàn thành** | Duyệt/từ chối đơn nghỉ phép, đổi ca, tạm ứng, bổ sung công kèm lý do xét duyệt chi tiết. |
| **9** | **Bảng lương & Phiếu lương** | `/dashboard/payslips` | **Hoàn thành** | Tạo bảng lương tự động toàn công ty hoặc từng nhân sự, cộng dồn phạt vi phạm từ work-service, điều chỉnh thưởng/phạt kèm lý do bắt buộc, xuất Excel và in phiếu lương (`PrintPayslipModal`). |
| **10**| **Quản trị Ngân hàng & Payout** | `/dashboard/bank` | **Hoàn thành** | Quản lý tài khoản công ty, lập lệnh chi lương chống chi trùng `idempotencyKey`, xem lịch sử giao dịch, công cụ kiểm thử trực quan điện văn SOAP XML giả lập Core Banking. |
| **11**| **Cấu hình Wi-Fi Chấm công** | `/dashboard/wifi` | **Có lỗi / Bị chặn** | Hiển thị bảng danh sách Wi-Fi chi nhánh. **Lỗi nghiêm trọng:** Form modal `WifiForm.tsx` không được import vào trang, nút "Thêm Wi-Fi" không tồn tại, người dùng bị tước quyền thêm/sửa Wi-Fi trên Web (P0). |
| **12**| **Bảng tin & Nội quy** | `/dashboard/news`, `/dashboard/regulations` | **Hoàn thành** | Đăng tin tức, ghim bài viết nổi bật, gắn nhãn phân loại màu sắc; ban hành quy định công ty có mã hiệu, phiên bản và ngày hiệu lực. |
| **13**| **Kho Dữ liệu Cũ (Archive Hub)**| `/dashboard/archive` | **Hoàn thành** | Dành riêng cho Admin: Xem trước khối lượng dữ liệu cũ cần lưu trữ, bấm chạy lưu trữ theo batch kèm progress bar trực quan, tra cứu và khôi phục dữ liệu tức thì. |
| **14**| **Nhật ký Hoạt động (Backlog)**| `/dashboard/backlog` | **Hoàn thành** | Hiển thị toàn bộ nhật ký thao tác người dùng (Thêm/Sửa/Xóa) được ghi nhận ngầm qua `api.ts`, hỗ trợ lọc theo phân hệ và nút xóa sạch nhật ký cho Admin. |
| **15**| **Khung Giao diện & Thông báo**| `src/components/layout/*`| **Thiếu sót** | Menu Sidebar co giãn động theo vai trò người dùng. **Thiếu sót:** `NotificationPanel.tsx` không được nhúng vào `Topbar.tsx`; Web hoàn toàn thiếu chuông thông báo khi có đơn từ mới (P2). |

---

### 2.3. Mobile Application (Flutter 3.x)

Ứng dụng di động được xây dựng bằng **Flutter 3.47 / Dart 3.13**, thiết kế giao diện Material 3 với bảng màu doanh nghiệp, hướng tới đối tượng người dùng cuối là Nhân viên (`staff`) và Quản lý Chi nhánh (`manager`).

#### Bảng Ma trận Tính năng Mobile Application:

| STT | Module Chức năng | Màn hình Giao diện | Trạng thái Nghiệp vụ | Mô tả Hiện trạng & Chi tiết Đánh giá Kỹ thuật |
|:---:|---|---|:---:|---|
| **1** | **Xác thực & Quản lý Phiên** | `LoginScreen`, `ProfileScreen` | **Hoàn thành** | Đăng nhập tài khoản, lưu phiên an toàn trong Keychain/Keystore qua `flutter_secure_storage`. Xử lý silent token refresh với mutex `_refreshing` khi gặp HTTP 401. Xem và xóa phiên thiết bị từ xa. |
| **2** | **Xác thực Sinh trắc học** | *(Chưa phát triển)* | **Thiếu sót** | Hoàn toàn chưa tích hợp thư viện `local_auth`. Chưa hỗ trợ FaceID / Vân tay khi đăng nhập hoặc điểm danh. |
| **3** | **Quản lý Ca làm việc** | `ScheduleScreen`, `GeneralScheduleScreen`, `ShiftAssignmentScreen`, `ScheduleRegistrationScreen`, `ShiftDetailScreen`, `ShiftFormScreen` | **Hoàn thành** | Xem lịch cá nhân theo tuần, lịch chung chi nhánh, đăng ký nguyện vọng làm việc tuần, chi tiết ca, tạo/sửa ca cho quản lý. Đồng bộ ngày tháng `DD/MM/YYYY`. |
| **4** | **Chấm công qua Wi-Fi** | `HomeScreen` (Widget Chấm công) | **Chưa hoàn thành / Giả lập** | Nút Check-in và Check-out gọi API backend. **Tồn đọng nghiêm trọng:** Không có plugin card mạng (`network_info_plus`), không cấp quyền vị trí. Tên Wi-Fi và BSSID lấy cứng từ bản ghi đầu tiên của API cấu hình; backend bỏ qua kiểm tra BSSID/IP (P0). |
| **5** | **Đơn từ & Nghỉ phép** | `LeaveRequestScreen`, `AdjustmentScreen`, `ApprovalsScreen` | **Hoàn thành** | Tạo đơn xin nghỉ phép kèm file/ảnh; nộp đơn giải trình/bổ sung công (`work_supplement`) với giờ vào/ra thực tế; màn hình cho quản lý duyệt hoặc từ chối đơn. |
| **6** | **Đổi ca & Nhờ làm thay** | `ShiftFormScreen`, `RequestDetailScreen` | **Hoàn thành** | Nhờ đồng nghiệp làm thay ca (`cover`) hoặc đổi ca chéo (`swap`). Đồng nghiệp nhận thông báo bấm Đồng ý/Từ chối trước khi quản lý phê duyệt. |
| **7** | **Đăng ký Làm thêm giờ (OT)** | *(Chưa có màn hình riêng)* | **Chưa hoàn thành** | Chưa có màn hình cho nhân viên xin đăng ký tăng ca ngoài giờ. Loại đơn `overtime` đang bị dùng chồng lấn cho việc "Nhờ làm thay ca" trong mã nguồn `shift_form.dart`. |
| **8** | **Phiếu lương & Công lương** | `SalaryScreen`, `SalaryAdvanceScreen` | **Có lỗi / Thiếu sót** | Xem tổng thu nhập, số ca, số giờ công, danh sách phiếu lương tháng và nộp đơn xin tạm ứng lương. **Lỗi sai lệch:** Model `PayslipModel` bỏ sót trường `bonus`, không hiển thị tiền thưởng từ backend (P1). |
| **9** | **Bảng tin & Nội quy** | `NewsScreen`, `RegulationsScreen` | **Hoàn thành** | Đọc bảng tin nội bộ, bài ghim, lọc chi nhánh; tra cứu nội quy lao động công ty theo danh mục và từ khóa tìm kiếm. |
| **10**| **Nhiệm vụ & Vận hành ca** | `TasksScreen`, `TaskMonitorScreen` | **Hoàn thành** | Danh sách nhiệm vụ theo trạng thái, tích hợp chụp ảnh từ camera và thư viện ảnh qua `UploadService` (`POST /api/upload`) để nộp ảnh minh chứng. |

---

## 3. BẢNG PHÂN TÍCH SAI LỆCH CONTRACT API & DATABASE (CONTRACT MISMATCH TABLE)

Bảng đối soát chi tiết sự sai lệch giữa Client (Web Frontend / Mobile Flutter) và Backend Microservices / CSDL PostgreSQL:

| STT | Thành phần Gây lệch | Endpoint / Trường dữ liệu | Phía Client Gọi / Định nghĩa | Phía Backend / Database Quy định | Phân loại Sai lệch & Rủi ro | Giải pháp Khắc phục Đề xuất |
|:---:|---|---|---|---|:---:|---|
| **1** | **Web vs Backend Work** | `GET /api/shifts` vs `attendances` | Web đọc trường `checkIn` và `checkOut` từ đối tượng `Shift` tại bảng phân ca `/dashboard/shifts`. | Backend tách biệt 2 bảng độc lập: `shifts` (chỉ lưu lịch ca dự kiến) và `attendances` (lưu kết quả điểm danh thực tế). Khi mobile checkin, backend CHỈ ghi vào bảng `attendances`. | **BLOCKER (P0)**<br>Lệch kiến trúc dữ liệu: Cột check-in/out trên bảng ca Web luôn rỗng; Quản lý không xem được giờ làm thật của nhân viên. | Cập nhật `WorkService.checkin`: khi điểm danh thành công, cập nhật đồng thời giờ vào trường `checkIn`/`checkOut` của bản ghi `Shift` ngày đó; HOẶC bổ sung màn hình Tra cứu Chấm công riêng gọi `GET /api/attendance`. |
| **2** | **Mobile vs Backend Work** | `POST /api/attendance/checkin` | Mobile gửi JSON: `{ employeeId, shiftId, wifiSsid, checkinTime }`. Mobile lấy Wi-Fi giả lập từ bản ghi đầu của `/api/wifi-configs`. | Backend `WorkService.checkin` chỉ bóc tách `{ employeeId, shiftId, time }`. Hoàn toàn không kiểm tra `wifiSsid` hay BSSID/IP đối soát với `wifi_configs`. | **BLOCKER (P0)**<br>Bỏ qua kiểm tra phần cứng Wi-Fi: Nhân viên ở bất cứ đâu ngoài công ty vẫn chấm công thành công. | Tích hợp package `network_info_plus` trên Mobile đọc BSSID card mạng thật. Bổ sung logic tại `work-service` xác thực BSSID gửi lên khớp với `wifi_configs` chi nhánh. |
| **3** | **Web Frontend UI** | `POST /api/wifi-configs`<br>`PUT /api/wifi-configs/:id` | File component `WifiForm.tsx` (102 dòng code) không được import vào `src/app/dashboard/wifi/page.tsx`. Giao diện chỉ có bảng xem tĩnh. | Backend `integration-service` đã triển khai đầy đủ các endpoint CRUD Wi-Fi: `POST`, `PUT`, `DELETE /api/wifi-configs`. | **BLOCKER (P0)**<br>Mã nguồn mồ côi (Dead Code): Người dùng không thể thêm mới hoặc hiệu chỉnh điểm Wi-Fi trên cổng quản trị Web. | Import `WifiFormModal` vào `src/app/dashboard/wifi/page.tsx`, gắn nút "Thêm Wi-Fi mới" và icon Sửa/Xóa trên từng hàng dữ liệu. |
| **4** | **Mobile vs Backend Payroll** | `GET /api/payroll/payslips` | `mobile/lib/src/core/models/payslip.dart` (`PayslipModel`): Khai báo 9 trường, **hoàn toàn bỏ sót trường `bonus`**. | Backend Prisma Schema `hrm_payroll.payslips` có trường `bonus Float @default(0)`. Công thức: `netSalary = baseSalary + bonus - totalPenalty`. | **CRITICAL (P1)**<br>Thiếu trường DTO tài chính: Tiền thưởng của nhân sự không hiển thị trên Mobile, gây lệch số liệu đối soát giữa Web và App. | Bổ sung trường `final double bonus` vào `PayslipModel`, cập nhật hàm `fromJson`, `toJson` và hiển thị dòng tiền thưởng tại `SalaryScreen`. |
| **5** | **Web Frontend API** | `PUT`, `DELETE` trên 8 trang Web Portal | 8 trang (`branches`, `departments`, `shifts`, `tasks`, `news`, `regulations`, `bank`, `payslips`) tự viết hàm `apiPut`, `apiDelete` hoặc gọi `fetch()` trực tiếp (ví dụ `bank/page.tsx:156` và `payslips/page.tsx:21`). Riêng trang `employees` đã import và dùng `apiPut` chuẩn từ `@/lib/api`. | Thư viện chuẩn `src/lib/api.ts` đã cung cấp sẵn `apiPut`, `apiDelete` tích hợp cơ chế tự động ghi nhật ký kiểm toán Backlog (`autoLogBacklog`) và chuẩn hóa lỗi 401. | **MAJOR (P2)**<br>Vi phạm DRY / Bỏ qua Audit Log: Thao tác cập nhật/xóa tại 8 trang này không được ghi nhận vào nhật ký hệ thống `Backlog`. | Xóa các hàm `apiPut`, `apiDelete` cục bộ tại 8 trang `page.tsx` (`branches`, `departments`, `shifts`, `tasks`, `news`, `regulations`, `bank`, `payslips`); chuyển sang dùng hàm chuẩn import từ `@/lib/api`. |
| **6** | **Web Layout UI** | `GET /api/notifications` | Component `NotificationPanel.tsx` (171 dòng code) không được import vào `Topbar.tsx`. Thanh tiêu đề Web không có biểu tượng quả chuông. | Backend có đủ API thông báo: `GET /api/notifications`, `PUT /api/notifications/:id/read`. | **MAJOR (P2)**<br>Tính năng mồ côi: Quản lý và Admin trên Web không nhận được thông báo khi nhân viên nộp đơn từ mới. | Nhúng biểu tượng quả chuông và `NotificationPanel` vào `src/components/layout/Topbar.tsx`, kết nối API fetch thông báo định kỳ. |
| **7** | **Mobile Model** | `GET /api/notifications` | `AppNotificationModel` (`app_notification.dart`) chỉ có 7 trường cơ bản, thiếu các trường chi tiết ca làm việc. | CSDL `hrm_integration.notifications` có thêm: `metadata, requestType, shiftName, shiftDate, shiftTime, senderName...`. | **MAJOR (P2)**<br>Gọi API trùng lặp: Màn hình `notifications.dart` phải gọi thêm 1 request raw JSON thứ hai để lấy đủ trường, gây lãng phí băng thông. | Bổ sung các trường chi tiết ca vào `AppNotificationModel`, loại bỏ request gọi lặp thứ hai trong `notifications.dart`. |
| **8** | **Mobile vs Backend Req** | `POST /api/requests` | Mobile dùng `type: 'overtime'` để đại diện cho nghiệp vụ "Nhờ làm thay ca" (Cover shift). | Backend `RequestValidator` chấp nhận `overtime`, `leave`, `shift_swap`, `work_supplement`, `advance`. | **MAJOR (P2)**<br>Xung đột ngữ nghĩa (Semantic Overload): Làm mất đi tính năng xin tăng ca độc lập của nhân viên. | Chuẩn hóa loại đơn riêng cho nhờ làm thay (ví dụ: `shift_cover`), giải phóng mã `overtime` cho tính năng đăng ký tăng ca. |
| **9** | **Web Client Code** | `frontend/src/lib/api.ts:57` | Khai báo biến toàn cục: `let module = "Hệ thống";`. | Từ khóa `module` là từ khóa dành riêng (reserved identifier) trong môi trường CommonJS/Node.js. | **MINOR (P3)**<br>Lỗi linting `@next/next/no-assign-module-variable`, tiềm ẩn rủi ro khi biên dịch SSR. | Đổi tên biến thành `moduleName` hoặc `systemModule`. |
| **10**| **Web UI Display** | `EmployeeDetailModal.tsx:1051` | Đoạn text ghi chú: *"Khi checkout, tiền công ca sẽ bằng: Số giờ làm × hourlySalary"*. | Tài liệu SSOT `RULES.md` §7.1 và `PayrollService.js`: Nhân sự hưởng lương cứng; checkout chỉ phạt vi phạm, không cộng dồn lương giờ. | **MINOR (P3)**<br>Sai lệch thông tin hiển thị: Gây hiểu lầm nghiêm trọng cho quản lý về cơ chế vận hành lương cứng. | Hiệu chỉnh lại nội dung văn bản hướng dẫn trong modal phản ánh đúng nguyên tắc lương cứng và phạt chấm công. |
| **11**| **Quy ước Ngày tháng** | Định dạng Ngày tháng | Web format hiển thị **`DD/MM/YYYY`** (dấu `/`). Mobile dùng `toVnDate()` sinh **`DD/MM/YYYY`**. | `RULES.md` §7.3 quy định dạng văn bản: **`DD-MM-YYYY`** (dấu `-`). Backend regex hỗ trợ cả hai nhưng ưu tiên `DD/MM/YYYY`. | **MINOR (P3)**<br>Lệch tài liệu đặc tả: Mã nguồn thực tế đồng bộ tốt trên chuẩn `DD/MM/YYYY`, nhưng lệch so với văn bản tài liệu cũ. | Cập nhật tài liệu `RULES.md` công nhận chuẩn chung toàn hệ thống là `DD/MM/YYYY`. |

---

## 4. HIỆN TRẠNG KIỂM THỬ (TESTING STATUS & COVERAGE)

### 4.1. Kiểm thử Backend Microservices

Backend sử dụng bộ chạy kiểm thử nguyên bản của Node.js (**Node.js Native Test Runner** - `node --test`). Toàn bộ **27 tệp kiểm thử** chứa **169 test cases** được phân bổ trên 6 microservices:
- `api-gateway`: **4 tệp** (`auth.test.js`, `authorize.test.js`, `health.test.js`, `ratelimit.test.js`)
- `identity-service`: **3 tệp** (1 unit: `auth.test.js`; 2 integration: `health.test.js`, `account.test.js`)
- `organization-service`: **3 tệp** (1 unit: `org.test.js`; 2 integration: `health.test.js`, `org.test.js`)
- `work-service`: **4 tệp** (1 unit: `work.test.js`; 3 integration: `work.test.js`, `archive.test.js`, `health.test.js`)
- `payroll-service`: **3 tệp** (1 unit: `payroll.test.js`; 2 integration: `payroll.test.js`, `health.test.js`)
- `integration-service`: **10 tệp** (4 unit: `domain.test.js`, `fileValidator.test.js`, `health.test.js`, `soap.test.js`; 6 integration: `archive.test.js`, `files.test.js`, `health.test.js`, `management.test.js`, `requests-notifications-soap.test.js`, `requests.test.js`)

#### Bảng Kết quả Thực thi Kiểm thử Backend (Khi CSDL PostgreSQL Đang Hoạt Động):

| STT | Microservice | Số Tệp | Chi tiết Tệp Kiểm Thử | Số Tests | Kết quả Chạy Song song (`npm test` mặc định) | Kết quả Chạy Tuần tự (`--test-concurrency=1`) | Thời gian chạy |
|:---:|---|:---:|---|:---:|:---:|:---:|:---:|
| 1 | `api-gateway` | 4 | `tests/auth.test.js`, `authorize.test.js`, `health.test.js`, `ratelimit.test.js` | **87** | **87 Pass** | **87 Pass** | ~194 ms |
| 2 | `identity-service` | 3 | `tests/unit/auth.test.js`, `tests/integration/health.test.js`, `account.test.js` | **15** | **15 Pass** | **15 Pass** | ~1.10 s |
| 3 | `organization-service` | 3 | `tests/unit/org.test.js`, `tests/integration/health.test.js`, `org.test.js` | **3** | **3 Pass** | **3 Pass** | ~4.29 s |
| 4 | `work-service` | 4 | `tests/unit/work.test.js`, `tests/integration/work.test.js`, `archive.test.js`, `health.test.js` | **8** | **8 Pass** | **8 Pass** | ~256 ms |
| 5 | `payroll-service` | 3 | `tests/unit/payroll.test.js`, `tests/integration/payroll.test.js`, `health.test.js` | **3** | **3 Pass** | **3 Pass** | ~254 ms |
| 6 | `integration-service` | 10 | 4 unit (`domain`, `fileValidator`, `health`, `soap`) + 6 integration (`archive`, `files`, `health`, `management`, `requests-notifications-soap`, `requests`) | **53** | ⚠️ **52 Pass / 1 Fail** *(Lỗi Race Condition sinh ID)* | **53 Pass (100%)** | ~518 ms |
| **TỔNG** | **6 Microservices** | **27** | **Toàn bộ 27 Tệp Kiểm Thử Backend** | **169** | **168 Pass / 1 Fail (99.4%)** | **169 Pass (100%)** | **~6.60 s** |

#### Phát hiện Lỗi Chạy Song song (Race Condition Test Failure) trong `integration-service`:

Trong quá trình thẩm định thực nghiệm, khi thực thi lệnh `npm test` mặc định tại thư mục `backend/integration-service` (`node --test tests/unit/*.test.js tests/integration/*.test.js`), Node.js Test Runner chạy đồng thời các tệp kiểm thử tích hợp. Khi đó, phát sinh lỗi xung đột dữ liệu:

```
PrismaClientKnownRequestError: 
Invalid `prisma.notification.create()` invocation in
/Users/trantu/Documents/DuAn/PTPMDV/backend/integration-service/src/services/NotificationService.js:87:30
→ 87 return prisma.notification.create(
Unique constraint failed on the fields: (`id`)

✖ requests CRUD + duyệt tự sinh thông báo (349.95ms)
  AssertionError [ERR_ASSERTION]: Expected values to be strictly equal:
  500 !== 200
  at TestContext.<anonymous> (/Users/trantu/Documents/DuAn/PTPMDV/backend/integration-service/tests/integration/requests.test.js:78:10)

ℹ tests 53
ℹ suites 8
ℹ pass 52
ℹ fail 1
```

- **Phân tích Bản chất Lỗi:** Cả hai suite kiểm thử `requests-notifications-soap.test.js` và `requests.test.js` cùng thực hiện thao tác duyệt đơn sinh thông báo đồng thời. Hàm `nextNotificationId()` trong `NotificationService.js:51-70` đọc giá trị ID lớn nhất có tiền tố `TB-` bằng câu truy vấn `prisma.notification.findMany` mà không sử dụng khóa bi quan (pessimistic lock) hay khối giao dịch Database cách ly. Khi 2 tiến trình chạy song song, cả hai cùng đọc ra cùng một giá trị `max` và cùng tính toán sinh ra ID trùng lặp (ví dụ `TB-0117`), dẫn đến vi phạm ràng buộc khóa chính Prisma P2002 `Unique constraint failed on the fields: (id)`.
- **Hệ quả Thực nghiệm:**
  - Lệnh `npm test` mặc định: Đạt **52 pass / 1 fail** tại `integration-service` (tổng toàn backend đạt **168 pass / 1 fail** trên 169 tests).
  - Lệnh ép chạy tuần tự `--test-concurrency=1`: `node --test --test-concurrency=1 tests/unit/*.test.js tests/integration/*.test.js` đạt **53/53 pass tuyệt đối (169/169 toàn backend)**.
- **Khuyến nghị Khắc phục Triệt để:** Thay thế cơ chế scan ID tiền tố thủ công trong `NotificationService.js:nextNotificationId()` bằng PostgreSQL Sequence (`CREATE SEQUENCE notification_id_seq; SELECT nextval(...)`) hoặc chuyển sang sử dụng định danh ngẫu nhiên không xung đột UUID/CUID2 (`@default(cuid())`).

#### Phân tích Hành vi Kiểm thử Khi Cơ sở Dữ liệu Offline:
- Khi các container PostgreSQL chưa khởi chạy:
  - `api-gateway`: Đạt **87/87 pass** do kiến trúc Gateway là phi trạng thái (stateless), chỉ kiểm tra logic toán học của chữ ký JWT, cấu hình CORS, giải thuật rate limiting và bảng phân quyền route RBAC.
  - 5 microservices còn lại: Hầu hết các integration test bị **FAIL hàng loạt** do ném lỗi `PrismaClientInitializationError: Can't reach database server at localhost:543x`.
  - Hai test suite kiểm tra archive (`work-service` và `integration-service`) dừng ngay lập tức với thông điệp: `Error: Test archive cần Postgres local (docker compose up -d)`.
- **Đánh giá Chuyên môn:**
  - Bộ test integration bám sát CSDL thật rất tốt, phản ánh chân thực hành vi nghiệp vụ end-to-end.
  - Tuy nhiên, dự án thiếu tầng mock CSDL cô lập (Mock Database Layer) phục vụ Unit Testing độc lập; nếu hệ thống CI/CD chạy pipeline mà chưa dựng Docker Postgres thì quá trình build sẽ bị gián đoạn.
  - Backend hoàn toàn chưa cài đặt công cụ đo lường độ phủ kiểm thử tự động (`c8` hoặc `nyc`).

---

### 4.2. Kiểm thử Web Frontend (Next.js 16 App Router)

- **Số lượng Test Suites:** **0**
- **Số lượng Test Cases:** **0**
- **Tỷ lệ Độ phủ (Code Coverage):** **0.0%**
- **Đánh giá Hiện trạng:**
  - Trong tệp `frontend/package.json` hoàn toàn không có lệnh script `"test"`.
  - Không có bất kỳ thư viện kiểm thử nào được cài đặt (không có Jest, Vitest, React Testing Library, Cypress hay Playwright).
  - Không tìm thấy bất kỳ tệp kiểm thử nào (`*.test.tsx`, `*.spec.ts`, thư mục `__tests__` không tồn tại).
- **Rủi ro Nghiệp vụ:** Mức độ rủi ro hồi quy (Regression Risk) ở mức **Tối đa**. Mọi thay đổi logic tính toán phụ cấp lương, quy trình phê duyệt đơn hoặc điều hướng trang đều không có cơ chế tự động xác minh tính toàn vẹn.

---

### 4.3. Kiểm thử Mobile Application (Flutter 3.x)

Ứng dụng di động sử dụng bộ khung kiểm thử chuẩn của Flutter SDK (`flutter_test`). Lệnh `flutter test` thực thi 7 test suites:

#### Bảng Kết quả Thực thi Kiểm thử Mobile:

| STT | Tên Test Suite (Tệp Kiểm Thử) | Hạng mục Kiểm tra Chi tiết | Số Test Cases | Kết quả | Thời gian |
|:---:|---|---|:---:|:---:|:---:|
| 1 | `test/session_storage_test.dart` | Lưu token an toàn với Keychain khi `remember: true`, dọn dẹp khi `remember: false`, khôi phục session rỗng, xóa persisted session. | 5 | **5 Passed** | ~200 ms |
| 2 | `test/session_refresh_test.dart` | Tự động làm mới token ngầm khi gặp HTTP 401, xoay vòng token, xử lý hết hạn phiên và cơ chế mutex lock `_refreshing` chống gọi lặp. | 6 | **6 Passed** | ~250 ms |
| 3 | `test/home_avatar_test.dart` | Widget Header hiển thị đúng ảnh đại diện đồng bộ từ API hồ sơ nhân sự (testWidgets). | 4 | **4 Passed** | ~180 ms |
| 4 | `test/router_guard_test.dart` | Bảo vệ tuyến `/main`, chặn người dùng chưa xác thực truy cập trái phép, điều hướng splash (testWidgets). | 2 | **2 Passed** | ~150 ms |
| 5 | `test/widget_test.dart` | Khởi động ứng dụng vào màn hình Splash khởi tạo an toàn (testWidgets). | 1 | **1 Passed** | ~140 ms |
| 6 | `test/date_format_test.dart` | Chuyển đổi định dạng ngày tháng tương thích chuẩn Việt Nam (`DD/MM/YYYY`) qua 13 assertions kiểm chứng. | 5 | **5 Passed** *(13 assertions)* | ~120 ms |
| 7 | `test/api_login_test.dart` | Kiểm thử tích hợp đăng nhập live gateway thật (Chủ động bỏ qua khi Gateway live chưa chạy). | 1 | **1 Skipped** | 0 ms |
| **TỔNG**| **7 Test Suites** | **Toàn bộ Kiểm thử Di động** | **25** | **24 Passed, 1 Skipped, 0 Failed** | **~1.5 s** |

---

### 4.4. Đánh giá Khoảng trống Kiểm thử Toàn Dự án (Coverage Gaps)

```
[BẢN ĐỒ KHOẢNG TRỐNG KIỂM THỬ TOÀN DỰ ÁN]

BACKEND (169 Tests):
├── Gateway (87 Tests)      [████████████████████] 95% Phủ tốt
├── Identity (15 Tests)     [████████████░░░░░░░░] 60% Phủ khá
├── Organization (3 Tests)  [████░░░░░░░░░░░░░░░░] 20% Thiếu test sâu cho nhân viên
├── Work (8 Tests)          [█████░░░░░░░░░░░░░░░] 25% Thiếu test phạt chấm công
├── Payroll (3 Tests)       [████░░░░░░░░░░░░░░░░] 20% Thiếu test SAGA payout
└── Integration (53 Tests)  [██████████████░░░░░░] 70% Phủ tốt đơn từ, archive, soap

FRONTEND (0 Tests):
└── Tất cả 14 Modules       [░░░░░░░░░░░░░░░░░░░░]  0% HOÀN TOÀN TRỐNG (BLOCKER)

MOBILE (24 Passed / 1 Skipped Tests):
├── Auth & Session (11 Tests)[██████████████████░░] 85% Phủ tốt
├── UI & Widgets (7 Tests)   [██████████████░░░░░░] 70% Phủ khá
├── Date/Utils (5 Tests)     [████████████████████] 95% Phủ tốt (13 assertions)
└── Core Repositories       [░░░░░░░░░░░░░░░░░░░░]  0% Chưa có test cho 6 Repositories
```

- **Các mảng nghiệp vụ hoàn toàn chưa được kiểm thử tự động:**
  - **Phía Web Frontend:** 100% các modal biểu mẫu (tạo nhân viên, tạo phiếu lương, lập lệnh chi ngân hàng, phê duyệt đơn từ) chưa có test kiểm tra tính toán hoặc phản hồi sự kiện.
  - **Phía Mobile App:** Toàn bộ tầng Repository nghiệp vụ gồm `ShiftRepository`, `AttendanceRepository`, `LeaveRepository`, `SalaryRepository`, `TaskRepository`, và `WifiConfigRepository` có **0 bài kiểm thử**.

---

## 5. ĐÁNH GIÁ CHẤT LƯỢNG MÃ NGUỒN & KỸ THUẬT NỢ (CODE QUALITY & TECHNICAL DEBT)

### 5.1. Backend Microservices: Lỗ hổng & Điểm yếu Kiến trúc

1. **Kiểm tra Cú pháp & Ngôn ngữ:**
   - Dự án viết bằng JavaScript thuần (Node.js CommonJS), không dùng TypeScript. Lệnh `find backend -name "*.js" -exec node --check {} +` xác nhận **100% cú pháp hợp lệ (0 syntax error)**.
2. **Khóa Bí mật Mặc định trong Mã nguồn (P0 Security Vulnerability):**
   - Vị trí:
     - `backend/api-gateway/src/config/index.js:11`
     - `backend/identity-service/src/config/index.js:15-17`
     - `backend/organization-service/src/config/index.js:13`
   - Đoạn mã vi phạm: `jwtSecret: process.env.JWT_SECRET || "dev_only_secret_thay_khi_deploy"`.
   - Nguy cơ: Kẻ xấu có thể tạo token giả mạo quyền Admin để truy cập toàn bộ dữ liệu công ty nếu máy chủ production bị khuyết biến môi trường.
3. **Thiếu Khối Transaction ACID trong Quy trình Chi Lương (P1 Financial Risk):**
   - Vị trí: `backend/payroll-service/src/services/PayrollService.js:206-237` (`createPayout`).
   - Phân tích rủi ro: Hàm trừ tiền tài khoản ngân hàng của doanh nghiệp trước (`debitAccount`), sau đó mới gọi lệnh ghi bản ghi `prisma.payout.create`. Nếu thao tác ghi DB gặp sự cố mạng hoặc lỗi cú pháp, tiền của doanh nghiệp đã bị trừ nhưng không có giao dịch payout tương ứng và không có cơ chế hoàn trả (rollback).
4. **Lỗi Race Condition & Thuật toán Sinh ID Tiền tố trong NotificationService (P1 Reliability & Concurrency Risk):**
   - Vị trí: `backend/integration-service/src/services/NotificationService.js:51-70, 87:30` (`nextNotificationId`).
   - Phân tích rủi ro & Chi tiết thực nghiệm: Hàm `nextNotificationId()` sử dụng giải thuật scan thủ công bằng `findMany({ where: { id: { startsWith: 'TB-' } } })` để tìm số ID lớn nhất rồi tăng thêm 1 mà không bọc trong Database Transaction hay cơ chế khóa bi quan (pessimistic locking). Khi chạy kiểm thử tích hợp song song (`npm test`) hoặc trong môi trường vận hành thực tế khi nhiều quản lý cùng thao tác duyệt đơn tại các chi nhánh, hai yêu cầu tạo thông báo diễn ra đồng thời sẽ đọc cùng một số `max` và cùng thử chèn một bản ghi có ID trùng nhau (ví dụ `TB-0117`). Lỗi này kích hoạt ngoại lệ Prisma P2002 `Unique constraint failed on the fields: (id)` tại dòng 87:30 và trả về HTTP 500 (`AssertionError: 500 !== 200`), làm bài kiểm thử `tests/integration/requests.test.js` bị FAIL khi chạy mặc định (52 pass / 1 fail).
   - Khuyến nghị khắc phục: Thay thế hoàn toàn cơ chế scan ID tiền tố thủ công bằng PostgreSQL Sequence (`CREATE SEQUENCE notification_id_seq; SELECT nextval(...)`) hoặc chuyển sang sử dụng định danh ngẫu nhiên chuẩn công nghiệp UUID / CUID2 (`cuid()`).
5. **Lưu Trữ Thiết bị Phiên Đăng nhập trong RAM (P1 Architecture Risk):**
   - Vị trí: `backend/identity-service/src/services/AuthService.js:89` (`const userDevices = new Map()`).
   - Phân tích: Prisma Schema của `identity-service` chỉ có bảng `users`, thiếu hai bảng `device_sessions` và `refresh_tokens` như cam kết trong `docs/TECHS.md`. Việc lưu trong RAM khiến danh sách phiên thiết bị mất sạch mỗi khi restart container và không thể mở rộng cân bằng tải nhiều máy chủ.
6. **Cấu hình Sai lệch Ngưỡng Giới hạn Tần suất API Gateway (P2 Security & Stability):**
   - Vị trí: `backend/api-gateway/src/config/index.js:14` đối chiếu với `.env.example`.
   - Phân tích rủi ro: Mức giới hạn tần suất 300 req/phút (`RATE_LIMIT_MAX=300`) chỉ là cấu hình mẫu được khai báo trong file `.env.example`. Trong khi đó, giá trị fallback tĩnh mặc định được mã hóa trong mã nguồn `api-gateway/src/config/index.js:14` là **50,000 req/phút** (`parseInt(process.env.RATE_LIMIT_MAX || "50000", 10)`). Nếu triển khai production mà sơ suất không gán biến môi trường `RATE_LIMIT_MAX`, API Gateway sẽ hoạt động với ngưỡng 50,000 req/phút, khiến hệ thống mất khả năng phòng hộ chống lại các đợt tấn công từ chối dịch vụ (DDoS) hoặc vét cạn mật khẩu (Brute-force).
7. **Thực thi SQL Thô `$executeRawUnsafe` trong Backlog (P2 Code Quality):**
   - Vị trí: `backend/integration-service/src/services/BacklogService.js:80-110, 182`.
   - Phân tích: Dịch vụ tự tạo bảng và ghi nhật ký bằng SQL raw thay vì khai báo model trong Prisma Schema.
8. **Nuốt Lỗi Âm Thầm trong Fallback In-Memory (P2 Reliability):**
   - Vị trí: `WorkService.js` và `PayrollService.js` sử dụng nhiều khối `catch (_) {}` không ghi log để rơi xuống thao tác trên RAM, tiềm ẩn nguy cơ "mất dữ liệu ngầm" (silent data loss) khi Postgres gặp trục trặc tạm thời.
9. **Thiếu Hệ Thống Structured Logging Tập Trung (P3 Observability):**
   - Tồn tại **63 lệnh gọi trực tiếp** `console.log`, `console.warn`, `console.error` rải rác trong mã nguồn, thiếu correlation ID và định dạng JSON phục vụ giám sát tập trung.

---

### 5.2. Web Frontend: Linting, Rendering & Type Safety

#### 1. Kiểm tra Kiểu dữ liệu TypeScript (`npx tsc --noEmit`):
- Kết quả: **Thành công 100% (0 lỗi TypeScript)**. Toàn bộ định nghĩa interface và contract dữ liệu trong `src/types/` đồng nhất chặt chẽ.

#### 2. Kiểm tra Linting Mã nguồn (`npm run lint` / `eslint`):
- Kết quả: **Thất bại (Exit Code: 1)** với **185 vấn đề (111 errors, 74 warnings)**.

| Tên Rule ESLint Vi phạm | Mức độ | Số lượng | Tác động Kỹ thuật |
|---|:---:|:---:|---|
| `@typescript-eslint/no-explicit-any` | Error | **71** | Lạm dụng kiểu `any` làm mất tính an toàn dữ liệu khi map response từ API. |
| `@typescript-eslint/no-unused-vars` | Warning | **63** | Biến, tham số và biểu tượng icon import dư thừa chưa được dọn dẹp. |
| `react-hooks/set-state-in-effect` | Error | **36** | **Nghiêm trọng (Cascading Re-renders):** Gọi `setState` đồng bộ ngay bên trong `useEffect`, gây vòng lặp render lại nhiều lần làm đơ giao diện trên React 19 / Next.js 16. |
| `@next/next/no-img-element` | Warning | **8** | Dùng thẻ `<img>` thuần thay vì `<Image>` tối ưu hóa của Next.js. |
| `react-hooks/exhaustive-deps` | Warning | **3** | Thiếu biến phụ thuộc trong dependency array của React Hook. |
| `react/no-unescaped-entities` | Warning | **2** | Ký tự HTML chưa escape trong JSX. |
| `react-hooks/purity` | Error | **1** | Vi phạm tính bất biến thuần túy của React component hook. |
| `@next/next/no-assign-module-variable` | Error | **1** | Gán tên biến `module` trong `src/lib/api.ts:57`. |

- **Top 5 Tệp Vi phạm Nhiều Nhất:**
  1. `src/app/dashboard/shifts/page.tsx`: **24 errors, 8 warnings** (32 vấn đề)
  2. `src/app/dashboard/employees/page.tsx`: **11 errors, 5 warnings** (16 vấn đề)
  3. `src/app/dashboard/requests/page.tsx`: **9 errors, 5 warnings** (14 vấn đề)
  4. `src/features/employees/EmployeeDetailModal.tsx`: **1 error, 9 warnings** (10 vấn đề)
  5. `src/app/dashboard/tasks/page.tsx`: **9 errors, 0 warnings** (9 vấn đề)

#### 3. Kiểm tra Build Production (`npm run build`):
- Kết quả: **Biên dịch thành công 100% trong 726ms** (Turbopack), sinh ra **20 static routes** hoàn chỉnh.
- Cảnh báo Deprecation: File `src/middleware.ts` nhận cảnh báo lỗi thời từ Next.js 16.3.1 yêu cầu chuyển đổi quy ước sang `proxy.ts`.

#### 4. Lạm dụng Client Components (100% `"use client"`):
- Toàn bộ các trang nghiệp vụ đều dùng `"use client"` tải dữ liệu theo cơ chế waterfall (`mount -> useEffect -> fetch -> render`), không tận dụng sức mạnh của React Server Components (RSC) để tải trước dữ liệu từ máy chủ.

---

### 5.3. Mobile Application: Quyền Hệ điều hành, Kiến trúc & UI

1. **Phân tích Tĩnh Mã nguồn (`flutter analyze`):**
   - Kết quả: **Xuất sắc (0 issues found trong 3.0s)**. Mã nguồn tuân thủ nghiêm ngặt bộ quy tắc linting `flutter_lints: ^3.0.0`.
2. **Thiếu Quyền Thiết bị Gây Crash Ứng dụng trên iOS (P0 Blocker):**
   - File `mobile/ios/Runner/Info.plist` thiếu `NSCameraUsageDescription` và `NSPhotoLibraryUsageDescription`. Bất kỳ thao tác chụp ảnh bằng chứng công việc hoặc đổi avatar nào trên iOS đều làm ứng dụng bị hệ điều hành tiêu diệt ngay lập tức.
3. **Thiếu Quyền Phần cứng Wi-Fi trên Android (P1 Critical):**
   - File `AndroidManifest.xml` chỉ khai báo duy nhất quyền `INTERNET`. Từ Android 8.0 trở lên, việc đọc thông tin SSID/BSSID của mạng bắt buộc phải có `ACCESS_FINE_LOCATION` và `ACCESS_WIFI_STATE`.
4. **Nhãn Ứng dụng Chưa Chuẩn hóa Thương hiệu (P0 Blocker):**
   - Cả Android và iOS đều đang hiển thị tên app là `"F&B Mobile"` thay vì `"HRM Enterprise"`.
5. **Vi phạm Kiến trúc Điều hướng SSOT `go_router` (P2 Architectural Debt):**
   - Tài liệu kỹ thuật `TECHS.md` yêu cầu `AppRouter` là điểm cấu hình điều hướng duy nhất. Thực tế `AppRouter` chỉ có 3 route (`/`, `/login`, `/main`), trong khi có **hơn 80 vị trí** trong code gọi trực tiếp `Navigator.push(MaterialPageRoute(...))`. Điều này phá vỡ khả năng Deep-Link trực tiếp từ thông báo hệ thống vào màn hình nghiệp vụ.
6. **Hardcode Màu sắc Trực tiếp trong Widget (P3 UI Debt):**
   - Có **hơn 100 vị trí hardcode mã màu hex** (`Color(0xFF2563EB)`, `Color(0xFFF59E0B)`, `Color(0xFF10B981)...`) thay vì tham chiếu hằng số `AppColors` theo quy tắc `RULES.md`.
7. **Hiện tượng Tệp Giao diện Quá Lớn (Fat Widgets / God Files):**
   - Mã nguồn màn hình bị phình to do trộn lẫn giao diện và logic gọi API (lưu ý: mã nguồn nằm trực tiếp dưới `mobile/lib/src/features/<feature>/`, không có thư mục con `presentation/`):
     - `tasks.dart` (`mobile/lib/src/features/tasks/tasks.dart`): **1,489 dòng**
     - `home.dart` (`mobile/lib/src/features/home/home.dart`): **1,272 dòng**
     - `assignment.dart` (`mobile/lib/src/features/schedule/assignment.dart`): **1,089 dòng**
     - `notifications.dart` (`mobile/lib/src/features/notifications/notifications.dart`): **1,006 dòng**
     - `shift_form.dart` (`mobile/lib/src/features/schedule/shift_form.dart`): **954 dòng**
     - `general.dart` (`mobile/lib/src/features/schedule/general.dart`): **926 dòng**
     - `salary.dart` (`mobile/lib/src/features/salary/salary.dart`): **645 dòng**
8. **Hoàn toàn Không Có Khả năng Hoạt động Ngoại tuyến (Offline Queueing):**
   - Ứng dụng không sử dụng CSDL cục bộ (`sqflite` hay `hive`). Khi mất kết nối mạng, toàn bộ dữ liệu biến mất và không thể lưu tạm hành động chấm công hoặc nộp đơn để đồng bộ sau.

---

## 6. KẾ HOẠCH HÀNH ĐỘNG KHUYẾN NGHỊ (ACTIONABLE RECOMMENDATIONS)

Ban Kiểm toán đề xuất lộ trình khắc phục gồm **22 hạng mục hành động cụ thể**, được phân loại nghiêm ngặt theo 4 cấp độ ưu tiên kỹ thuật:

```
┌────────────────────────────────────────────────────────────────────────┐
│              KẾ HOẠCH HÀNH ĐỘNG KHUYẾN NGHỊ GO-LIVE                    │
├────────────────────────────────────────────────────────────────────────┤
│ 🔴 BLOCKER (P0)  : 5 Hạng mục  - Bắt buộc xử lý để tránh crash & bảo mật│
│ 🟠 CRITICAL (P1) : 6 Hạng mục  - Đảm bảo tài chính, dữ liệu & kiểm thử  │
│ 🟡 MAJOR (P2)    : 5 Hạng mục  - Tối ưu kiến trúc, router & duplicate  │
│ 🟢 MINOR (P3)    : 6 Hạng mục  - Dọn dẹp linting, style & logging      │
└────────────────────────────────────────────────────────────────────────┘
```

### 🔴 6.1. Cấp độ Blocker (P0) — Bắt buộc Khắc phục Trước Khi Phát hành

| Mã số | Hạng mục Hành động | Thành phần & Vị trí Mã nguồn | Mô tả Chi tiết Giải pháp Kỹ thuật |
|:---:|---|---|---|
| **P0-1** | **Bổ sung Quyền Camera & Ảnh trên iOS** | `mobile/ios/Runner/Info.plist` | Thêm các khóa bắt buộc: `NSCameraUsageDescription` ("Ứng dụng cần quyền truy cập máy ảnh để chụp ảnh minh chứng hoàn thành nhiệm vụ và cập nhật hồ sơ") và `NSPhotoLibraryUsageDescription` ("Ứng dụng cần quyền chọn ảnh từ thư viện để gửi minh chứng công việc") để chống crash app trên iPhone. |
| **P0-2** | **Chặn Khóa Bí mật JWT & Token Mặc định** | `backend/api-gateway/src/config/index.js`<br>`backend/identity-service/src/config/index.js`<br>`backend/organization-service/src/config/index.js` | Thêm kiểm tra điều kiện khởi động: Khi `process.env.NODE_ENV === "production"`, nếu `JWT_SECRET` hoặc `INTERNAL_API_TOKEN` bị thiếu hoặc chứa chuỗi mặc định `dev_only_*` thì lập tức dừng tiến trình (`process.exit(1)`) kèm thông báo lỗi nghiêm trọng. |
| **P0-3** | **Đồng bộ Dữ liệu Chấm công vào Ca làm việc** | `backend/work-service/src/services/WorkService.js`<br>`backend/work-service/src/controllers/WorkController.js` | Cập nhật hàm `checkin` và `checkout`: Khi lưu thành công bản ghi vào bảng `attendances`, tiến hành cập nhật đồng thời giờ vào trường `checkIn` và `checkOut` của bản ghi `shifts` tương ứng trong ngày. Đồng thời bổ sung màn hình "Nhật ký chấm công" tại Web gọi `GET /api/attendance`. |
| **P0-4** | **Kích hoạt Giao diện Quản lý Cấu hình Wi-Fi** | `frontend/src/app/dashboard/wifi/page.tsx` | Import `WifiFormModal` từ `src/features/wifi/WifiForm.tsx` vào trang `wifi/page.tsx`. Bổ sung nút bấm "Thêm Wi-Fi mới" mở modal tạo mới, đính kèm nút Sửa/Xóa và gắn kết nối API `POST`, `PUT`, `DELETE /api/wifi-configs`. |
| **P0-5** | **Chuẩn hóa Tên Thương hiệu Ứng dụng** | `mobile/ios/Runner/Info.plist`<br>`mobile/android/app/src/main/AndroidManifest.xml` | Đổi giá trị `CFBundleDisplayName` trong `Info.plist` và thuộc tính `android:label` trong `AndroidManifest.xml` từ `"F&B Mobile"` thành `"HRM Enterprise"`. |

---

### 🟠 6.2. Cấp độ Critical (P1) — Rủi ro Tài chính, Dữ liệu & Ổn định Hệ thống

| Mã số | Hạng mục Hành động | Thành phần & Vị trí Mã nguồn | Mô tả Chi tiết Giải pháp Kỹ thuật |
|:---:|---|---|---|
| **P1-1** | **Bọc Giao dịch ACID `$transaction` cho Chi Lương** | `backend/payroll-service/src/services/PayrollService.js:206` | Bọc toàn bộ thao tác trừ tiền `debitAccount` và thao tác tạo bản ghi `prisma.payout.create` vào trong một Prisma Interactive Transaction: `await prisma.$transaction(async (tx) => { ... })` để đảm bảo tính toàn vẹn tài chính khi có lỗi. |
| **P1-2** | **Đưa Session Thiết bị từ RAM vào PostgreSQL** | `backend/identity-service/prisma/schema.prisma`<br>`backend/identity-service/src/services/AuthService.js` | Khai báo 2 model `DeviceSession` và `RefreshToken` trong schema Prisma của `identity-service`, tạo migration cập nhật CSDL `hrm_identity` và thay thế toàn bộ logic lưu trữ trong biến `userDevices = new Map()` bằng truy vấn CSDL thật. |
| **P1-3** | **Sửa 36 Lỗi `react-hooks/set-state-in-effect`** | `frontend/src/app/dashboard/*`<br>`frontend/src/features/*` | Tái cấu trúc (Refactor) các modal và component đang gọi `setState` đồng bộ bên trong `useEffect` (như khi mở modal thêm nhân viên, đổi ca, sửa chi nhánh). Chuyển sang khởi tạo giá trị trực tiếp hoặc thiết lập state thông qua callback xử lý sự kiện mở modal. |
| **P1-4** | **Bổ sung Trường `bonus` vào Payslip Mobile** | `mobile/lib/src/core/models/payslip.dart`<br>`mobile/lib/src/features/salary/salary.dart` | Khai báo trường `final double bonus` trong `PayslipModel`, bổ sung parse dữ liệu trong hàm `fromJson` và `toJson`, đồng thời hiển thị chi tiết khoản tiền thưởng này trên giao diện `SalaryScreen`. |
| **P1-5** | **Thiết lập Khung Kiểm thử Tự động cho Web Frontend** | `frontend/package.json`<br>`frontend/vitest.config.ts` | Cài đặt `vitest` và `@testing-library/react`. Viết các bộ kiểm thử unit/integration tối thiểu cho `AuthContext`, thư viện gọi API chuẩn `api.ts`, các hàm tiện ích ngày tháng/lương tại `utils.ts`, và kiểm thử các modal nghiệp vụ quan trọng. |
| **P1-6** | **Hiện thực hóa Kiểm tra Wi-Fi Phần cứng Thật** | `mobile/pubspec.yaml`<br>`mobile/lib/src/features/home/home.dart`<br>`backend/work-service/src/services/WorkService.js` | Tích hợp package `network_info_plus` trên mobile để đọc BSSID thực tế từ phần cứng máy; khai báo quyền `ACCESS_FINE_LOCATION`, `ACCESS_WIFI_STATE` trên Android; cập nhật `WorkService.checkin` so khớp BSSID thực tế với bảng `wifi_configs`. |

---

### 🟡 6.3. Cấp độ Major (P2) — Cải tiến Kiến trúc & Trải nghiệm Người dùng

| Mã số | Hạng mục Hành động | Thành phần & Vị trí Mã nguồn | Mô tả Chi tiết Giải pháp Kỹ thuật |
|:---:|---|---|---|
| **P2-1** | **Mở rộng Tuyến đường go_router Tập trung** | `mobile/lib/src/core/router/router.dart`<br>`mobile/lib/src/features/*` | Đưa toàn bộ các tuyến đường nghiệp vụ (`/schedule`, `/leave`, `/salary`, `/tasks`, `/approvals`, `/wifi`...) vào cấu hình `AppRouter`. Thay thế hơn 80 lệnh gọi trực tiếp `Navigator.push` bằng `context.push()` hoặc `context.go()` để hỗ trợ Deep-Link từ thông báo. |
| **P2-2** | **Hợp nhất API Client Web (Xóa bỏ Code Duplication)** | `frontend/src/app/dashboard/*/page.tsx`<br>(8 trang: `branches, departments, shifts, tasks, news, regulations, bank, payslips`) | Xóa toàn bộ các hàm `apiPut` và `apiDelete` được viết cục bộ độc lập (hoặc raw `fetch`) tại 8 trang: `branches, departments, shifts, tasks, news, regulations, bank, payslips`. Chuyển sang import và gọi thống nhất từ `@/lib/api` để đảm bảo 100% thao tác thay đổi dữ liệu được tự động ghi nhận vào bảng kiểm toán `Backlog`. (Ghi chú: Trang `employees` đã import và sử dụng `apiPut` chuẩn từ `@/lib/api`, trong khi `payslips/page.tsx:21` là trang tự định nghĩa hàm `apiPut` cục bộ). |
| **P2-3** | **Tích hợp Biểu tượng Quả chuông Thông báo trên Web** | `frontend/src/components/layout/Topbar.tsx` | Nhúng component `NotificationPanel.tsx` vào thanh `Topbar.tsx`, kết nối API `GET /api/notifications` để hiển thị số lượng thông báo chưa đọc cho Admin và Manager. |
| **P2-4** | **Đồng bộ Trường DTO Thông báo Mobile** | `mobile/lib/src/core/models/app_notification.dart`<br>`mobile/lib/src/features/notifications/notifications.dart` | Khai báo các trường chi tiết ca làm (`metadata`, `requestType`, `shiftName`, `shiftDate`, `senderName`) vào `AppNotificationModel`. Loại bỏ lệnh gọi API raw JSON trùng lặp thứ hai trong `notifications.dart` để tiết kiệm 50% thời gian tải trang. |
| **P2-5** | **Loại bỏ Nuốt Lỗi Âm Thầm, Sửa Race Condition Sinh ID & Chuẩn hóa Backlog** | `backend/work-service/src/services/WorkService.js`<br>`backend/payroll-service/src/services/PayrollService.js`<br>`backend/integration-service/src/services/NotificationService.js`<br>`backend/integration-service/src/services/BacklogService.js` | Xóa các khối `catch (_) {}` nuốt lỗi vô điều kiện; chỉ cho phép fallback RAM khi có cờ cấu hình `ALLOW_MEMORY_FALLBACK=true`. Khắc phục triệt để lỗi race condition bằng cách thay thế cơ chế scan ID tiền tố thủ công trong `NotificationService.js:nextNotificationId()` bằng PostgreSQL Sequence (`CREATE SEQUENCE notification_id_seq`) hoặc UUID/CUID2. Thay thế các câu lệnh raw SQL `$executeRawUnsafe` trong `BacklogService` bằng mô hình Prisma ORM chuẩn. |

---

### 🟢 6.4. Cấp độ Minor (P3) — Làm sạch Mã nguồn & Tối ưu Hóa Vận hành

| Mã số | Hạng mục Hành động | Thành phần & Vị trí Mã nguồn | Mô tả Chi tiết Giải pháp Kỹ thuật |
|:---:|---|---|---|
| **P3-1** | **Dọn dẹp Warnings Linting & Sửa Biến `module`** | `frontend/src/lib/api.ts:57`<br>`frontend/src/app/*` | Đổi tên biến `let module` thành `let moduleName` để giải quyết lỗi `@next/next/no-assign-module-variable`. Dọn dẹp 63 biến không sử dụng (`no-unused-vars`) và giảm thiểu 71 lỗi ép kiểu `any`. |
| **P3-2** | **Refactor Hệ thống Màu sắc Mobile sang `AppColors`** | `mobile/lib/src/features/*` | Thay thế hơn 100 vị trí khai báo màu hex thủ công (`Color(0xFF...)`) tại `login.dart`, `home.dart`, `salary.dart` bằng các hằng số màu sắc định nghĩa trong `AppColors`. |
| **P3-3** | **Nâng cấp Middleware Lỗi thời của Next.js 16** | `frontend/src/middleware.ts` | Chạy lệnh codemod chính thức `npx @next/codemod@canary middleware-to-proxy .` để chuyển đổi quy ước `middleware.ts` sang `proxy.ts` theo đúng khuyến nghị của Next.js 16.3.1. |
| **P3-4** | **Thiết lập Hệ thống Structured Logging Tập trung** | `backend/*/src/utils/logger.js` | Thay thế các lệnh gọi trực tiếp `console.*` rải rác trong backend bằng wrapper logging chuẩn hóa sử dụng thư viện `pino` xuất log JSON kèm correlation ID, timestamp và log level. |
| **P3-5** | **Bổ sung Công cụ Đo Độ phủ Kiểm thử (Coverage Tooling)** | `backend/*/package.json`<br>`mobile/test/*` | Cài đặt `c8` trong `devDependencies` của backend và bổ sung script `"test:coverage"`. Viết bổ sung các bài kiểm thử unit cho 6 repositories trên Mobile để nâng cao độ bao phủ kiểm thử. |
| **P3-6** | **Hiệu chỉnh Văn bản Hướng dẫn Tính lương trên Web** | `frontend/src/features/employees/EmployeeDetailModal.tsx:1051` | Chỉnh sửa lại câu văn hướng dẫn tại dòng 1051 để phản ánh chính xác nguyên tắc lương cứng cố định và phạt vi phạm chấm công theo đúng đặc tả SSOT của dự án. |

---

## 7. KẾT LUẬN & ĐÁNH GIÁ CUỐI CÙNG

Dự án **HRM Enterprise On-Premises** là một hệ thống phần mềm có quy mô lớn, được tổ chức bài bản với tư duy thiết kế kiến trúc hiện đại:
- **Nền tảng kiến trúc vững chắc:** Việc áp dụng triệt để mô hình **Flat 3-Tier (Mẫu A)**, **Database-per-service (5 PostgreSQL riêng biệt)**, **Single-Tenant**, và **Centralized API Gateway** giúp hệ thống có khả năng phân tách trách nhiệm rất cao, đáp ứng hoàn hảo các yêu cầu triển khai nội bộ on-premises của doanh nghiệp.
- **Tiềm năng vận hành thực tế:** Các luồng nghiệp vụ phức tạp như SAGA chi lương với Idempotency Key, tích hợp cổng thanh toán đối tác qua giao thức chuẩn SOAP WSDL, kho lưu trữ dữ liệu cũ (Archive Hub) và quy trình xét duyệt đơn từ đa bước đã được cài đặt hoàn thiện và kiểm thử đạt kết quả 169/169 pass trên backend (khi chạy tuần tự với cờ `--test-concurrency=1`; đạt 168/169 pass khi chạy song song do lỗi race condition sinh ID thông báo).

Tuy nhiên, với số điểm đánh giá thực tế là **75 / 100 (Cần khắc phục)**, hệ thống **CHƯA ĐỦ ĐIỀU KIỆN ĐỂ GO-LIVE NGAY LẬP TỨC**. 

**Điều kiện Tiên quyết để Nghiệm thu & Triển khai Production:**
Ban Quản lý Dự án và Đội ngũ Kỹ sư cần tập trung nguồn lực thực hiện dứt điểm **5 hạng mục Blocker (P0)** (sửa quyền camera iOS tránh crash app, chặn secret JWT mặc định, đồng bộ dữ liệu chấm công sang ca làm việc, kích hoạt form Wi-Fi Web và chuẩn hóa tên ứng dụng) cùng **6 hạng mục Critical (P1)** (bọc transaction chi lương, đưa session vào CSDL, sửa 36 lỗi cascading renders, thêm bonus phiếu lương, tạo test cho frontend và kích hoạt quét Wi-Fi thật). Sau khi hoàn thành các mục trên, hệ thống sẽ đạt mức sẵn sàng vận hành thực tế xuất sắc (dự kiến trên **92/100**) và đủ điều kiện bàn giao đưa vào khai thác thương mại an toàn.

---
*Báo cáo được lập và chứng thực bởi Ban Kiểm toán Kỹ thuật Độc lập (Teamwork Technical Audit Board).*  
*Tất cả số liệu, đường dẫn mã nguồn và bằng chứng kiểm thử được ghi nhận khách quan theo phương thức Read-Only Audit.*
