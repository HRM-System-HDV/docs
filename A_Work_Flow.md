# Work_flow_PTPMDV.md — Người A (nhóm trưởng) — Môn Phát triển phần mềm hướng dịch vụ

> File này dành cho AI agent. Đọc hết trước khi sửa code hoặc tạo file. Nguồn: `PTPMDV_PhanCong.docx`.
> Mục có nhãn **(suy ra)** hoặc **(chưa quy định)** không có trong tài liệu phân công gốc. Không coi đó là yêu cầu chắc chắn; hỏi lại người A khi cần.
> Khi làm việc, không đổi cổng, envelope, tên trường hay cấu trúc thư mục chung.

---

## 0. Tóm tắt nhanh

| Mục | Nội dung |
|---|---|
| Môn | Phát triển phần mềm theo hướng dịch vụ (PTPMDV) |
| Đề tài | Hệ thống HRM theo kiến trúc hướng dịch vụ (SOA): nhiều service độc lập giao tiếp qua REST và SOAP |
| Thành viên | A (nhóm trưởng), B, D, E |
| Phần A sở hữu | `backend/api-gateway` (4000), `backend/identity-service` (4001), `mobile/` (Flutter), `frontend/` (Web Portal) |
| Nhiệm vụ cuối | Kết nối, demo REST idempotent và SOAP end-to-end trên cả web và mobile, optimize |

---

## 1. Phạm vi: được sửa và không được sửa

### Được sửa (thuộc A)
- `backend/api-gateway/`
- `backend/identity-service/`
- `mobile/`
- `frontend/` (Web Portal — Task 5) **(suy ra, chưa có trong tài liệu gốc)**
- `docker-compose.yml` (ở gốc dự án)

### Không sửa (thuộc thành viên khác)
| Thư mục | Cổng | Chủ sở hữu | Chức năng chính |
|---|---|---|---|
| `backend/organization-service/` | 4002 | B | Danh mục tổ chức và nhân sự |
| `backend/work-service/` | 4003 | B | Ca, chấm công, tác vụ |
| `backend/payroll-service/` | 4004 | D | Lương, lệnh chi, phiếu lương |
| `backend/integration-service/` | 4005 | E | SOAP ngân hàng, yêu cầu, thông báo, bảng tin, nội quy, Wi-Fi |

Nếu cần thay đổi ở service của người khác (thiếu endpoint, sai schema, sai envelope), không tự sửa. Ghi lại vấn đề, nêu rõ service, endpoint, kỳ vọng và thực tế, rồi báo cho người A để trao đổi với chủ service.

---

## 2. Chuẩn kỹ thuật bắt buộc

### 2.1 Cấu trúc service
Mỗi service nằm trong `backend/<service-name>/` theo **Flat 3-Tier (Mẫu A)** — luồng chuẩn `Route -> Controller -> Service -> Prisma`:

```text
backend/<service-name>/
├── prisma/                # schema.prisma + migration + seed riêng
├── src/
│   ├── config/            # database.js + index.js (đọc env)
│   ├── clients/           # (nếu có) HTTP client gọi service khác, timeout 5000ms
│   ├── controllers/       # nhận req/res, gọi Service, trả JSON
│   ├── middlewares/       # xác thực, phân quyền, xử lý lỗi
│   ├── routes/            # khai báo route Express
│   ├── services/          # business logic + gọi trực tiếp Prisma (không qua repository)
│   ├── validators/        # (nếu có) kiểm tra đầu vào
│   ├── utils/             # errors.js, logger.js, ...
│   └── app.js             # khởi tạo Express, CORS, parser, gắn route, health
├── tests/
│   ├── unit/
│   └── integration/
├── .env.example
├── package.json
└── server.js
```

Áp dụng cho `api-gateway` và `identity-service`. Riêng `api-gateway` chỉ cần `src/config/` + `src/middlewares/`, stateless không DB, không có `prisma/`/`services/`.

Cấm `config/` ở root, `src/api/`, `src/domain/`, `src/infrastructure/`, tầng `repositories` riêng.

### 2.2 Định dạng response
- Thành công: `{ "data": ... }`
- Lỗi: `{ "error": { "code": "...", "message": "..." } }`

### 2.3 Health check
Mỗi service (kể cả gateway) có `GET /health`. Nghiệm thu yêu cầu chạy được ở đủ **6 điểm**: gateway + 5 service.

Tài liệu PTPMDV không nêu định dạng body của `/health`. Nên giữ thống nhất với chuẩn chung của dự án **(suy ra)**:

```json
{ "status": "ok", "service": "<service-name>", "time": "<thời gian hiện tại>" }
```

### 2.4 Gọi liên service (ràng buộc riêng của môn này)
- Gọi qua HTTP, timeout tối đa **5000ms**.
- **Không import chéo mã nguồn** giữa các service. Mỗi service độc lập, chỉ giao tiếp qua REST hoặc SOAP.
- Áp dụng cho các lệnh gọi từ gateway đến service phía sau **(suy ra)**.

### 2.5 Mobile (Flutter)
- Điều hướng: `go_router`.
- Màu: `AppColors`.
- Font: Public Sans.
- **Không dùng `withOpacity`.**

---

## 3. Git workflow

- Nhánh gốc làm việc: `dev`.
- Tạo nhánh theo mẫu `feat/<service>-<tên>` (ví dụ `feat/api-gateway-proxy`, `feat/identity-auth`, `feat/mobile-salary`).
- Xong thì tạo PR về `dev`.
- **Không commit thẳng vào `dev` hoặc `main`.**
- Mỗi PR nên gọn theo một service hoặc một chức năng.

---

## 4. Danh sách việc của A

### Task 1 — api-gateway (cổng 4000) — ✅ XONG (PR `feat/api-gateway-proxy`)
- [x] Dựng gateway proxy về các service 4001–4005.
- [x] Cấu hình CORS.
- [x] Xử lý lỗi tập trung, trả đúng envelope `{error: {code, message}}`.
- [x] `GET /health` cho gateway.
- [x] Viết `docker-compose` dựng hạ tầng DB local: **5 Postgres riêng** (mỗi service 1 container/volume/port `5433`–`5437`); app service chạy ngoài Docker bằng `npm run dev`.
- [x] Đặt timeout tối đa 5000ms khi proxy về service phía sau (suy ra từ mục 2.4).
- [x] Rate-limiting và xác thực JWT đầu vào tại gateway (TECHS.md §6.3) **(đã xong trên `feat/gateway-hardening`: `auth.js` verify JWT + cookie `hrm-session`, `rateLimit.js` 429, forward `x-user-*`, E2E login qua gateway pass)**.

Gợi ý kỹ thuật **(suy ra)**: phiên dùng cookie `hrm-session`, nên CORS cần cho phép credentials và chỉ định origin cụ thể, không dùng `*`.

### Task 2 — identity-service (cổng 4001) — ✅ XONG (PR `feat/identity-auth`)
- [x] `POST /api/auth/login`
- [x] `POST /api/auth/logout`
- [x] `GET /api/auth/me`
- [x] Middleware phiên `hrm-session`.
- [x] Phân quyền theo vai trò: `admin`, `manager`, `staff`.
- [x] Seed 3 tài khoản: `admin`, `manager`, `staff`.
- [x] `GET /health`.
- [x] `POST /api/auth/refresh` — cấp lại access token bằng refresh token (TECHS.md §6.1) **(đã xong trên feat/identity-auth)**.
- [x] `POST /api/auth/change-password` — đổi mật khẩu qua bcrypt + PostgreSQL/memory **(đã xong trên feat/identity-auth)**.
- [x] `POST /api/auth/forgot-password` — đặt lại mật khẩu về mặc định 123456 **(đã xong trên feat/identity-auth)**.
- [x] `GET /api/auth/devices` và `DELETE /api/auth/devices/:id` — danh sách và gỡ thiết bị/phiên đăng nhập **(đã xong trên feat/identity-auth)**.

> Quyết định đã chốt: DB dùng **5 Postgres riêng trên 5 link Supabase riêng (database-per-service)** — `hrm_identity` (A/4001), `hrm_organization` (B/4002), `hrm_work` (B/4003), `hrm_payroll` (D/4004), `hrm_integration` (E/4005), mỗi DB 1 Prisma schema + `DATABASE_URL` riêng trỏ link Supabase riêng, cấm join xuyên cơ sở dữ liệu; auth dùng **JWT + cookie `hrm-session` HttpOnly**; seed pass mặc định `123456`. Test unit + integration 10/10 pass.

### Task 3 — Mobile (Flutter, thư mục `mobile/`) — ✅ XONG TÍCH HỢP (PR `feat/mobile-integration`)
- [x] Đồng bộ models mobile với schema backend (mục 6) — đã thêm `payslip/payout/leave_request/app_notification/attendance/device_session` (PR `feat/mobile-api`).
- [x] Tích hợp màn hình `salary` về API thật qua gateway (kèm fallback mock khi payroll của D chưa có).
- [x] Tích hợp màn hình `notifications` về API thật (`/api/notifications`, mark read, delete; PR `feat/mobile-integration`).
- [x] Tích hợp màn hình `leave_request` về API thật (`GET /api/requests?type=leave`, `POST /api/requests`; PR `feat/mobile-integration`).
- [x] Tích hợp màn hình `approvals` về API thật (`GET /api/requests?status=pending`, duyệt/từ chối; PR `feat/mobile-integration`).
- [x] Tích hợp màn hình `salary_advance` về API thật (`POST /api/requests` với `type: advance`; PR `feat/mobile-integration`).
- [x] Tích hợp màn hình `tasks` về API thật (`GET/PUT /api/tasks`; PR `feat/mobile-integration`).
- [x] Tích hợp màn hình `attendance` (checkin/checkout) về API thật (`POST /api/attendance/checkin`, `checkout`; PR `feat/mobile-integration`).
- [x] Tích hợp màn hình `general_schedule` + `staff_monitor` về API thật (`GET /api/shifts`, `GET /api/attendance`; PR `feat/mobile-integration`).
- [x] Tích hợp màn hình `shift_assignment` (phân ca) + `schedule_registration` (đăng ký ca) về API thật (`/api/shifts/assign`, `/register`; PR `feat/mobile-integration`).
- [x] Tích hợp màn hình `news` về API thật (`GET /api/news`; PR `feat/mobile-integration`).
- [x] Tích hợp màn hình `regulations` về API thật (`GET /api/regulations`; PR `feat/mobile-integration`).
- [x] Tích hợp màn hình `wifi_config` về API thật (`GET/POST/PUT/DELETE /api/wifi-configs`; PR `feat/mobile-integration`).
- [x] Cấu hình base URL và company, trỏ về gateway; app chỉ chạy iOS nên dùng `localhost` cho iOS Simulator và IP LAN cho thiết bị thật.
- [x] Tích hợp màn hình login + splash về API thật (phiên `hrm-session`, PR `feat/mobile-auth`).
- [x] Tích hợp màn hình `profile/password.dart` (đổi mật khẩu) về API thật `POST /api/auth/change-password`, bỏ mock `Future.delayed` (nhánh `feat/mobile-auth`).
- [x] Tích hợp màn hình `profile/devices.dart` (quản lý thiết bị) về API thật `GET/DELETE /api/auth/devices`, bỏ mock (nhánh `feat/mobile-auth`).
- [x] Nối nút "Quên mật khẩu?" trên login mobile vào `POST /api/auth/forgot-password` thật, bỏ mock (nhánh `feat/mobile-auth`).
- [x] Tích hợp màn hình `schedule` (lịch cá nhân, `shift_detail`, `shift_form`) về API thật (`ShiftRepository().getShifts()`; PR `feat/mobile-integration`).
- [x] Tích hợp màn hình `home` (nút check-in/check-out) về API `attendance` thật (`AttendanceRepository`; PR `feat/mobile-integration`).
- [x] Tích hợp màn hình `attendance/adjustment` (điều chỉnh chấm công) về API thật (`type: work_supplement` qua `LeaveRepository`; PR `feat/mobile-integration`).
- [x] Tích hợp màn hình `notifications/request_detail` về API thật (duyệt/từ chối qua `ApprovalRepository`; PR `feat/mobile-integration`).
- [x] Tích hợp màn hình `help` — xác nhận nội dung FAQ tĩnh chuẩn HRM.
- [ ] Build và kiểm thử bản iOS.
- [x] `flutter analyze` pass (0 issues).
- [x] `flutter test` pass.

> Ngoài phạm vi gốc nhưng đã được A duyệt: tích hợp login web về API thật (PR `feat/frontend-auth`); sửa lỗi iOS AppIcon thiếu; sửa base URL tự nhận diện platform; bỏ mock đổi mật khẩu và thiết bị mobile.

### Task 4 — Kết nối, demo và optimize (làm cuối) — ⏳ CHƯA LÀM (chờ D xong payouts 4004, E xong SOAP/requests 4005)
- [ ] Kết nối toàn hệ thống khi các service khác đã xong.
- [ ] Demo **REST idempotent** end-to-end trên cả web và mobile: gửi lệnh chi hai lần với cùng `idempotencyKey`, lần hai trả bản ghi cũ kèm `deduped: true`.
- [ ] Demo **SOAP end-to-end** trên cả web và mobile: gọi `/soap/payroll`, tạo được lệnh chi, lỗi trả `soap:Fault`.
- [ ] Optimize sau khi luồng chạy đúng.
- [ ] Demo **SAGA compensating transaction** cho luồng chi lương khi thất bại giữa chừng (TECHS.md §6.3, phối hợp cùng D và E) **(suy ra)**.
- [ ] **API Composition** cho trang dashboard web (gộp organization + work + payroll) và **BFF** payload gọn cho mobile (TECHS.md §6.3) **(suy ra)**.

### Task 5 — Web Portal (`frontend/`) — ✅ XONG — Phụ trách: **A** (TECHS.md yêu cầu demo trên web)
> Tài liệu phân công gốc không giao web cho ai; A nhận phụ trách toàn diện.
- [x] Chủ sở hữu `frontend/`: **A**.
- [x] Tích hợp đăng nhập web qua gateway, phiên cookie `hrm-session` thật, bỏ mock (nhánh `feat/frontend-auth`).
- [x] Nút "Quên mật khẩu?" của web nối với `POST /api/auth/forgot-password` qua gateway, bỏ mock (nhánh `feat/frontend-auth`).
- [x] [Agent 1 – Task A1] Tích hợp trang employees về API thật `GET/POST/PUT/DELETE /api/employees` qua gateway (nhánh `feat/frontend-org-api`).
- [x] [Agent 1 – Task A2] Tích hợp trang branches về API thật `GET/POST/PUT/DELETE /api/branches` qua gateway (nhánh `feat/frontend-branches`).
- [x] [Agent 1 – Task A3] Tích hợp trang departments về API thật `GET/POST/PUT/DELETE /api/departments` qua gateway (nhánh `feat/frontend-departments`).
- [x] [Agent 1 – Task A4] Tích hợp trang shifts về API thật `GET/POST/PUT/DELETE /api/shifts`, `/assign`, `/registrations` qua gateway (nhánh `feat/frontend-shifts`).
- [x] [Agent 1 – Task A5] Tích hợp trang tasks về API thật `GET/POST/PUT/DELETE /api/tasks` qua gateway (nhánh `feat/frontend-tasks`).
- [x] [Agent 1 – Task A6] Tích hợp trang requests (phê duyệt đơn/từ chối/thông báo) về API thật qua gateway (nhánh `feat/frontend-requests`).
- [x] [Agent 2 – Task B1] Tích hợp trang bank (Ngân hàng & Chi lương: REST Idempotent + SOAP) về API thật qua gateway (nhánh `feat/frontend-payroll-content-api`).
- [x] [Agent 2 – Task B2] Tích hợp trang payslips (điều chỉnh thưởng/phạt, chốt đơn + hàng loạt, tạo phiếu tháng) về API thật qua gateway (nhánh `feat/frontend-payslips`).
- [x] [Agent 2 – Task B3] Tích hợp trang news (tạo/sửa/xóa bài viết) về API thật qua gateway (nhánh `feat/frontend-news`).
- [x] [Agent 2 – Task B4] Tích hợp trang regulations (tạo/sửa/xóa/ghim văn bản) về API thật qua gateway (nhánh `feat/frontend-regulations`).
- [x] [Agent 2 – Task B5] Tích hợp trang wifi (thêm/sửa/xóa cấu hình chấm công) về API thật qua gateway (nhánh `feat/frontend-wifi`).
- [x] Tích hợp toàn bộ 11/11 trang dashboard về API thật qua gateway (trang tổng `/dashboard` chuyển hướng về `/dashboard/employees`).
- [x] RBAC menu theo `buildMenuItems(role)` nối với role thật từ identity-service qua `AuthContext`.
- [x] Chốt ma trận phân quyền theo vai trò cho từng endpoint (Admin toàn quyền hệ thống; Manager phụ trách chi nhánh; Staff cá nhân).


---

## 5. Bảng route qua gateway

Gateway proxy các đường dẫn sau về service tương ứng. Đường dẫn ghi trong tài liệu gốc được đánh dấu ✔. Đường dẫn còn lại là **(suy ra)** từ tên chức năng, cần xác nhận với chủ service trước khi cấu hình.

| Đường dẫn | Service (cổng) | Nguồn |
|---|---|---|
| `/api/auth/*` | identity (4001) | ✔ |
| `/api/branches`, `/api/departments`, `/api/employees` | organization (4002) | ✔ |
| `/api/shifts`, `/api/attendance` | work (4003) | (suy ra: chỉ ghi "Shifts, attendance checkin/checkout, tasks CRUD") |
| `/api/tasks` | work (4003) | (suy ra) |
| `/api/payroll/*` (`bank-accounts`, `payouts`, `payslips`) | payroll (4004) | ✔ |
| `/soap/payroll` (`?wsdl` và POST) | integration (4005) | ✔ |
| `/api/news`, `/api/regulations`, `/api/wifi-configs` | integration (4005) | ✔ |
| `/api/requests` | integration (4005) | (suy ra) |
| `/api/notifications` | integration (4005) | (suy ra) |

Lưu ý: `/soap/payroll` nhận và trả XML, và trả `soap:Fault` khi lỗi. Gateway phải chuyển tiếp nguyên vẹn body, header và status của SOAP, không bọc lại thành `{data}` hay `{error}`.

---

## 6. Schema dữ liệu thống nhất (mobile và web đều phải khớp)

Đây là chuẩn chung SSOT. Không tự đổi tên trường. Mọi thay đổi schema phải cập nhật file này trước.

### Khuôn SOAP
- **Request:** `PayoutRequest` gồm `idempotencyKey, debitAccount, content, totalAmount, beneficiaryCount`.
- **Response:** `PayoutResponse` gồm `transactionId (TXN-xxxxxx), bankReference (BANK-xxxxxxxx), status`.
- **Lỗi:** trả `soap:Fault` với `faultcode = soap:Client` và `faultstring` mô tả lỗi (ví dụ: "Số dư không đủ").

### Các thực thể

- **Branch:** `id, name, slug (mã chi nhánh do hệ thống tự sinh CN-0001, client không tự nhập), address, manager (tên hiển thị), status (hoạt động/vô hiệu hóa), staff (số lượng nhân sự)`

- **Department:** `id, name, code (mã phòng ban do hệ thống tự sinh PB-0001, client không tự nhập), description, manager (tên hiển thị), status (hoạt động/tạm dừng), staff, createdAt`

- **Employee:** `id (mã nhân viên do hệ thống tự sinh NV-0001, client không tự nhập), name, email, phone, gender, birthDate, province, ward, street, cccd, issueDate, issuePlace, cccdFront, cccdBack, branch (slug chi nhánh, ví dụ "CN-0001"), department (tên phòng ban), role (chức danh hiển thị), systemRole (admin/manager/staff), status (đang làm/vô hiệu hóa), joinDate, baseSalary, salaryType (hourly/monthly), hourlySalary, bankName, bankAccountNumber, bankAccountName`

- **Shift (lịch sử ca / phân ca):** `id, employeeId, branch (slug), date (DD-MM-YYYY), template (tên ca), scheduledStart (HH:MM), scheduledEnd (HH:MM), checkIn (HH:MM), checkOut (HH:MM), status (hoàn thành/đang làm/vắng/trễ)`

- **Attendance (chấm công):** `id, employeeId, shiftId, date (DD-MM-YYYY), checkIn (HH:MM), checkOut (HH:MM), penaltyAmount, penaltyNote, status (present/absent/late/early_leave)`

- **AttendanceConfig:** `latePenaltyAmount (số tiền/lần, mặc định 20000), earlyLeavePenaltyAmount, gracePeriodMinutes (mặc định 5), autoCloseShift (bool), shiftSwapMode (string)`
  > Chủ sở hữu: **B** (work-service 4003). Endpoint gợi ý: `GET /api/attendance/config`, `PUT /api/attendance/config`.

- **Task:** `id, title, description, assignedTo (employeeId), branchSlug, dueDate, status (pending/in_progress/done), createdAt`

- **Payslip:** `id, employeeId, month (MM-YYYY), baseSalary, bonus, totalPenalty, netSalary (= baseSalary + bonus - totalPenalty), payoutId, status (chưa chốt/đã chốt), issuedAt`
  > Ghi chú: `bonus` được bổ sung so với tài liệu gốc để khớp với frontend và mobile.

- **Payout:** `id (TXN-xxxxxx), bankReference (BANK-xxxxxxxx), debitAccount, totalAmount, content, beneficiaryCount, idempotencyKey, status, createdAt`

- **Yêu cầu nội bộ (Request):** `id, type, employeeId, branchSlug, title, content, attachmentUrl, status (pending/approved/rejected), reviewedBy, reviewNote, createdAt, updatedAt`
  > Các giá trị `type` hợp lệ: `leave` (nghỉ phép), `overtime` (tăng ca), `advance` (tạm ứng lương), `shift_swap` (đổi ca), `work_supplement` (bổ sung công), `other`.
  > Chủ sở hữu: **E** (integration-service 4005). Lưu ý: `shift_swap` cần tham chiếu `shiftId` nguồn và đích — xác nhận với B trước.

- **Thông báo (Notification):** `id, targetEmployeeId (null = toàn hệ thống), branchSlug, title, body, isRead, createdAt`

- **Bảng tin (Announcement/News):** `id, title, summary, content, date, tag, tagTone (danger/warning/success/primary/gray), pinned (bool)`

- **Nội quy (Regulation):** `id, code, title, category, summary, content, status (hiệu lực/dự thảo/hết hiệu lực), scope (Toàn công ty hoặc slug chi nhánh), effectiveDate (DD-MM-YYYY), expiryDate, createdAt, updatedAt, version, pinned, attachments (số file đính kèm)`

- **Wi-Fi chấm công:** `id, ssid, bssid, branch (slug chi nhánh), status (hoạt động/vô hiệu hóa)`

Ghi chú định dạng ngày: `date` và `joinDate` dùng `DD-MM-YYYY`; `Payslip.month` dùng `MM-YYYY`; query attendance dùng `month=YYYY-MM`. Ba định dạng này khác nhau — chú ý khi parse.

Quy tắc cần biết khi tích hợp:
- Payout trùng `idempotencyKey` trả lại bản ghi cũ kèm `deduped: true`.
- Payout hết số dư trả HTTP `422`.
- Duyệt hoặc từ chối yêu cầu tự sinh thông báo. Màn hình `approvals` và `notifications` phản ánh đúng liên động này.
- Chỉ xóa yêu cầu khi đang `pending`.
- `netSalary = baseSalary + bonus - totalPenalty`.

---

## 7. Mốc kiểm tra

| Mốc | Việc của A | Việc của người khác (phụ thuộc) |
|---|---|---|
| 1 | Xong gateway + identity + tích hợp API mobile | B xong organization + work; D xong lệnh chi; E xong requests, notifications, SOAP |
| 2 | Xong build và kiểm thử iOS, kết nối, demo REST idempotent và SOAP end-to-end, rồi optimize | E xong bảng tin, nội quy, Wi-Fi |

Thứ tự ưu tiên: hoàn thành gateway và identity trước, vì các thành viên khác và mobile phụ thuộc vào chúng.

> **Tiến độ A (cập nhật 01-10-2026):** Task 1 + Task 2 xong và đã lên Supabase thật; Task 3 toàn bộ 15 màn hình mobile đã hoàn tất tích hợp API qua API Gateway (4000) kèm fallback mock mượt mà (PR `feat/mobile-integration`), `flutter analyze` 0 issues, `flutter test` pass; Task 5 toàn bộ 11 trang Web Portal Dashboard + Auth + RBAC đã hoàn tất kết nối API thật qua gateway và merge vào dev; Task 4 demo end-to-end chờ D và E. Chi tiết theo từng checkbox ở mục 4.

---

## 8. Nghiệm thu (Definition of Done)

- Gateway và xác thực chạy.
- `GET /health` đủ 6 điểm.
- Compose chạy full hệ thống.
- App mobile gọi API lương, phiếu lương, thông báo pass.
- `flutter analyze` và `flutter test` pass.
- Demo pass.

### Checklist trước khi mở PR
- [ ] Đúng cấu trúc thư mục service (mục 2.1).
- [ ] Response đúng envelope `{data}` hoặc `{error: {code, message}}`.
- [ ] Có `GET /health`.
- [ ] Có `.env.example` (mặc định trỏ Postgres local của chính service; Supabase để dạng comment, mỗi service 1 link riêng).
- [ ] Gọi liên service qua HTTP, timeout tối đa 5000ms, không import chéo mã nguồn.
- [ ] Nhánh đặt tên `feat/<service>-<tên>`, PR về `dev`.
- [ ] Mobile không dùng `withOpacity`, dùng `AppColors`, `go_router`, Public Sans.
- [ ] Không sửa file thuộc service của người khác.
- [ ] Cập nhật tiến độ vào file `Work_flow` của mình (đánh dấu `[x]`, ghi PR liên quan) trước khi nhờ review/merge PR.

---

## 9. Điểm chưa quy định

Các mục sau không có trong tài liệu phân công. Không tự quyết định rồi coi như đã chốt. Hỏi người A hoặc ghi rõ giả định trong PR.

- **Cơ sở dữ liệu và ORM (đã chốt 5 DB trên 5 link Supabase riêng):** mỗi service 1 Postgres + Prisma riêng — A giữ `hrm_identity`; B giữ `hrm_organization` + `hrm_work`; D giữ `hrm_payroll`; E giữ `hrm_integration`, mỗi DB 1 `DATABASE_URL` riêng trỏ link Supabase riêng. Lớp lưu trữ nằm trong `src/services/` gọi trực tiếp Prisma (Mẫu A — không có tầng `repositories` riêng hay `src/infrastructure`). Cấm join xuyên cơ sở dữ liệu, cần dữ liệu chéo thì gọi HTTP timeout 5000ms. `api-gateway` (4000) không cần DB — chỉ routing và proxy. B duy nhất sở hữu 2 DB (`hrm_organization` và `hrm_work`).
- **Ai soạn schema chung:** không có người được giao riêng việc thiết kế dữ liệu. Mỗi người tự lo model của service mình, còn mục 6 là chuẩn chung.
- **Body request và response của `/api/auth/login`, `/api/auth/me`:** không được quy định chi tiết, chỉ bắt buộc đúng envelope.
- **Mật khẩu và thông tin 3 tài khoản seed:** không được nêu.
- **Đường dẫn chính xác của shifts, attendance, tasks, requests, notifications:** xem mục 5, cần xác nhận.
- **Định dạng body của `/health`:** tài liệu PTPMDV không nêu, xem mục 2.3.
- **Ứng dụng web:** tài liệu nhắc demo trên web nhưng không giao ai làm web. Đã phân công cho **A** (Task 5) **(suy ra)**.
- **API versioning `/api/v1/`:** TECHS.md §6.3 yêu cầu versioning nhưng toàn bộ route ở mục 5 dùng `/api/...`. Cần chốt trước nghiệm thu **(suy ra)**. **Phụ trách: A** (cấu hình route ở gateway).
- **Công thức `netSalary` đang lệch 3 bản:** A §6 và B ghi `baseSalary + bonus - totalPenalty`; D ghi `baseSalary - totalPenalty`; `AGENTS.md` §2.6 ghi `baseSalary - totalPenalties + allowances`. Cần chốt một bản duy nhất **(đã chỉnh D theo A §6 ngày 24-09-2026)**. **Phụ trách chốt: A** cùng D.

---

## 10. Quy tắc làm việc cho agent

1. Đọc mục 1 trước. Chỉ sửa trong phạm vi của A.
2. Không tự bịa endpoint, tên trường hay hành vi. Nếu tài liệu không nêu, dùng nhãn **(suy ra)** và hỏi lại.
3. Bám đúng envelope, tên trường và định dạng ngày ở mục 6.
4. Không import chéo mã nguồn giữa các service. Mọi giao tiếp qua HTTP, timeout tối đa 5000ms.
5. Không commit thẳng vào `dev` hoặc `main`. Mỗi thay đổi đi qua nhánh `feat/...` và PR về `dev`.
6. Sau mỗi thay đổi ở mobile, chạy `flutter analyze` và `flutter test`. Sau mỗi thay đổi ở backend, kiểm tra `GET /health` và chạy test của service.
7. Khi báo kết quả, nói rõ đã chạy lệnh nào và kết quả ra sao. Không báo "pass" khi chưa chạy.
8. Ưu tiên theo thứ tự: gateway + identity, đến tích hợp API mobile, đến build iOS, cuối cùng là kết nối, demo REST idempotent và SOAP end-to-end, optimize.
9. **Luật cập nhật tiến độ (bắt buộc):** sau khi hoàn thành mỗi task và trước khi nhờ review/merge PR, phải cập nhật file `Work_flow` của mình — đánh dấu `[x]` các việc đã xong, ghi rõ nhánh/PR liên quan và việc nào đang chờ thành viên khác. File `Work_flow` là kênh thông báo tiến độ chính cho cả nhóm; không để người khác phải hỏi mới biết làm đến đâu.
