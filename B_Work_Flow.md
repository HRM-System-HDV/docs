# Work_flow_PTPMDV_B.md — Thành viên B — Môn Phát triển phần mềm hướng dịch vụ

> File này dành cho AI agent. Đọc hết trước khi sửa code hoặc tạo file. Nguồn: `PTPMDV_PhanCong.docx`.
> Mục có nhãn **(suy ra)** hoặc **(chưa quy định)** không có trong tài liệu phân công gốc. Không coi đó là yêu cầu chắc chắn; hỏi lại khi cần.
> Không đổi cổng, envelope, tên trường hay cấu trúc thư mục chung.

---

## 0. Tóm tắt nhanh

| Mục | Nội dung |
|---|---|
| Môn | Phát triển phần mềm hướng dịch vụ |
| Thành viên nhóm | A (nhóm trưởng), B, D, E |
| Phần B sở hữu | `backend/organization-service` (4002) + `backend/work-service` (4003) |
| Chức năng chính | Danh mục tổ chức và nhân sự; ca, chấm công, tác vụ |

---

## 1. Phạm vi: được sửa và không được sửa

### Được sửa (thuộc B)
- `backend/organization-service/` (cổng 4002): Danh mục tổ chức và nhân sự
- `backend/work-service/` (cổng 4003): Ca, chấm công, tác vụ

### Không sửa (thuộc thành viên khác)
| Thư mục | Cổng | Chủ sở hữu | Chức năng chính |
|---|---|---|---|
| `backend/api-gateway/`, `backend/identity-service/`, `mobile/`, `docker-compose.yml` | 4000, 4001 | A | Gateway, xác thực, mobile, compose |
| `backend/payroll-service/` | 4004 | D | Lương, lệnh chi, phiếu lương |
| `backend/integration-service/` | 4005 | E | SOAP ngân hàng, yêu cầu, thông báo, bảng tin, nội quy, Wi-Fi |

Nếu cần thay đổi ở phần của người khác (thiếu endpoint, sai schema, sai envelope), không tự sửa. Ghi lại vấn đề, nêu rõ service, endpoint, kỳ vọng và thực tế, rồi báo cho B để trao đổi với chủ phần đó.

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

Cấm `config/` ở root, `src/api/`, `src/domain/`, `src/infrastructure/`, tầng `repositories` riêng.

### 2.2 Định dạng response
- Thành công: `{ "data": ... }`
- Lỗi: `{ "error": { "code": "...", "message": "..." } }`

### 2.3 Health check
Mỗi service có `GET /health`. Tài liệu PTPMDV không nêu định dạng body. Nên giữ thống nhất với chuẩn chung của dự án **(suy ra)**:

```json
{ "status": "ok", "service": "<service-name>", "time": "<thời gian hiện tại>" }
```

### 2.4 Gọi liên service (ràng buộc riêng của môn này)
- Gọi qua HTTP, timeout tối đa **5000ms**.
- **Không import chéo mã nguồn** giữa các service. Mỗi service độc lập, chỉ giao tiếp qua REST hoặc SOAP.

---

## 3. Git workflow

- Nhánh gốc làm việc: `dev`.
- Tạo nhánh theo mẫu `feat/<service>-<tên>` (ví dụ `feat/organization-employees`, `feat/work-attendance`).
- Xong thì tạo PR về `dev`.
- **Không commit thẳng vào `dev` hoặc `main`.**
- Mỗi PR nên gọn theo một service hoặc một chức năng.

---

## 4. Danh sách việc của B

Tài liệu PTPMDV chỉ nêu phần việc của B ở mức chức năng. Chi tiết endpoint dưới đây **(suy ra)** từ danh sách chức năng ở mục 0 và schema mục 5.

### Task 1 — organization-service (cổng 4002) — ✅ XONG (đã merge PR #13)
- [x] CRUD **branches**: `GET /api/branches`, `GET /api/branches/:slug`, `POST /api/branches`, `PUT /api/branches/:slug`, `DELETE /api/branches/:slug`.
- [x] CRUD **departments**: `GET /api/departments`, `GET /api/departments/:id`, `POST`, `PUT`, `DELETE`.
- [x] CRUD **employees** đúng schema, validate email và systemRole, phân quyền.
  - `GET /api/employees?branchSlug=&departmentId=&systemRole=`
  - `GET /api/employees/:id`
  - `POST /api/employees`, `PUT /api/employees/:id`, `DELETE /api/employees/:id`
  - Email không trùng; `systemRole` chỉ nhận `admin`/`manager`/`staff`.
- [x] `GET /health`.

### Task 2 — work-service (cổng 4003) — ✅ XONG (đã merge PR #13)
- [x] **Shifts (lịch ca / phân ca):**
  - `GET /api/shifts?branchSlug=&date=YYYY-MM-DD&employeeId=` — danh sách ca.
  - `POST /api/shifts` — tạo ca mới (admin/manager).
  - `PUT /api/shifts/:id` — cập nhật ca (admin/manager).
  - `DELETE /api/shifts/:id` — xóa ca (admin/manager).
  - `POST /api/shifts/:id/assign` — phân công ca cho nhân viên (chức năng `shift_assignment`).
  - `POST /api/shifts/register` — nhân viên tự đăng ký ca (chức năng `schedule_registration`).
  - `GET /api/shifts/registrations?branchSlug=&week=` — danh sách nguyện vọng đăng ký ca của nhân viên, phục vụ bảng "Quản lý đăng ký ca" trên web (`RegTimetable`, `WeekWishes`) **(suy ra)**.
- [x] **Attendance (chấm công):**
  - `POST /api/attendance/checkin` — check-in.
  - `POST /api/attendance/checkout` — check-out, tính phạt tự động.
  - `GET /api/attendance?employeeId=&month=YYYY-MM` — lịch sử chấm công.
  - `GET /api/attendance?branchSlug=&date=YYYY-MM-DD` — giám sát toàn bộ nhân sự trong ngày (chức năng `staff_monitor`).
  - `GET /api/attendance/config` — lấy cấu hình phạt chấm công.
  - `PUT /api/attendance/config` — cập nhật cấu hình phạt (admin).
- [x] **Tasks (tác vụ):** CRUD toàn bộ.
  - `GET /api/tasks?branchSlug=&assignedTo=&status=`
  - `POST /api/tasks`, `PUT /api/tasks/:id`, `DELETE /api/tasks/:id`.
- [x] `GET /health`.

---

## 5. Schema dữ liệu liên quan (phải khớp — SSOT tại `A_Work_Flow.md §6`)

Không tự đổi tên trường. Mọi thay đổi phải cập nhật `A_Work_Flow.md §6` trước.

- **Branch:** `id, name, slug (mã chi nhánh do hệ thống tự sinh CN-0001, client không tự nhập), address, manager (tên hiển thị), status (hoạt động/vô hiệu hóa), staff (số lượng nhân sự)`

- **Department:** `id, name, code (mã phòng ban do hệ thống tự sinh PB-0001, client không tự nhập), description, manager (tên hiển thị), status (hoạt động/tạm dừng), staff, createdAt`

- **Employee:** `id (mã nhân viên do hệ thống tự sinh NV-0001, client không tự nhập), name, email, phone, gender, birthDate, province, ward, street, cccd, issueDate, issuePlace, cccdFront, cccdBack, branch (slug chi nhánh, ví dụ "CN-0001"), department (tên phòng ban), role (chức danh hiển thị), systemRole (admin/manager/staff), status (đang làm/vô hiệu hóa), joinDate, baseSalary, salaryType (hourly/monthly), hourlySalary, bankName, bankAccountNumber, bankAccountName`

- **Shift:** `id, employeeId, branch (slug), date (DD-MM-YYYY), template (tên ca), scheduledStart (HH:MM), scheduledEnd (HH:MM), checkIn (HH:MM), checkOut (HH:MM), status (hoàn thành/đang làm/vắng/trễ)`

- **Attendance:** `id, employeeId, shiftId, date (DD-MM-YYYY), checkIn (HH:MM), checkOut (HH:MM), penaltyAmount, penaltyNote, status (present/absent/late/early_leave)`

- **AttendanceConfig:** `latePenaltyAmount (mặc định 20000), earlyLeavePenaltyAmount, gracePeriodMinutes (mặc định 5), autoCloseShift (bool), shiftSwapMode (string)`

- **Task:** `id, title, description, assignedTo (employeeId), branchSlug, dueDate, status (pending/in_progress/done), createdAt`

Ghi chú định dạng ngày: `date` và `joinDate` dùng `DD-MM-YYYY`; query attendance dùng `month=YYYY-MM`.

Quy tắc tính phạt chấm công (theo `AttendanceConfig`):
- Đi trễ > `gracePeriodMinutes` phút: phạt `latePenaltyAmount`/lần, ghi `status = late`.
- Về sớm: phạt `earlyLeavePenaltyAmount`/lần, ghi `status = early_leave`.
- Vắng mặt: ghi `status = absent`, phạt theo quy định nhóm.
- `netSalary = baseSalary + bonus - totalPenalty`.

---

## 6. Phụ thuộc và phối hợp

- Mọi request từ web và mobile đi qua `api-gateway` (4000) của A. Client không gọi thẳng vào service của bạn.
- Đăng nhập và phiên `hrm-session` do `identity-service` (4001) của A xử lý.
- **D (payroll-service, 4004):** D sinh phiếu lương bằng cách "tổng hợp phạt theo tháng". Dữ liệu phạt nằm ở work-service của bạn. Cách D lấy dữ liệu là gọi HTTP đến attendance của bạn **(suy ra)**. Giữ `GET /api/attendance?employeeId=&month=YYYY-MM` ổn định và trả `penaltyAmount` đúng.
- **E (integration-service, 4005):** Yêu cầu `shift_swap` do E tiếp nhận và xử lý approval. Khi approval, E có thể gọi về work-service của bạn để cập nhật ca **(suy ra)**. Xác nhận luồng này với E trước khi làm.
- **A:** cần route `/api/branches`, `/api/departments`, `/api/employees`, `/api/shifts`, `/api/attendance`, `/api/attendance/config`, `/api/tasks` cấu hình trên gateway. Mobile của A gọi các màn `general_schedule`, `staff_monitor`, `shift_assignment`, `schedule_registration`, `attendance`, `tasks` từ service của bạn.

---

## 7. Mốc kiểm tra

| Mốc | Việc của B |
|---|---|
| 1 | Xong organization-service + work-service |

Mốc đầu tiên của A là gateway + identity. Cần bám sát để chạy được qua gateway khi tích hợp.

---

## 8. Nghiệm thu (Definition of Done)

- Danh mục, nhân sự, chấm công, tác vụ đúng chuẩn.

### Checklist trước khi mở PR
- [ ] Đúng cấu trúc thư mục service (mục 2.1).
- [ ] Response đúng envelope `{data}` hoặc `{error: {code, message}}`.
- [ ] Có `GET /health`.
- [ ] Có `.env.example` (mặc định trỏ Postgres local của chính service; Supabase để dạng comment, mỗi service 1 link riêng).
- [ ] Email trùng bị từ chối, `role` chỉ nhận `admin`/`manager`/`staff`.
- [ ] Chấm công tính phạt đúng; `Attendance.date` dùng `DD-MM-YYYY`.
- [ ] Gọi liên service (nếu có) qua HTTP, timeout tối đa 5000ms, không import chéo mã nguồn.
- [ ] Nhánh đặt tên `feat/<service>-<tên>`, PR về `dev`.
- [ ] Không sửa file thuộc phần của người khác.

---

## 9. Điểm chưa quy định

Các mục sau không có trong tài liệu phân công. Không tự quyết định rồi coi như đã chốt. Hỏi A (nhóm trưởng) hoặc ghi rõ giả định trong PR.

- **Cơ sở dữ liệu và ORM (đã chốt 5 DB trên 5 link Supabase riêng):** B sở hữu 2 DB riêng `hrm_organization` (4002) và `hrm_work` (4003), mỗi DB 1 Prisma schema + migration + `DATABASE_URL` riêng trỏ link Supabase riêng. Lớp lưu trữ nằm trong `src/services/` gọi trực tiếp Prisma (Mẫu A — không có tầng `repositories` riêng hay `src/infrastructure`). Cấm join xuyên cơ sở dữ liệu, cần dữ liệu chéo thì gọi HTTP timeout 5000ms.
- **Cách service nhận danh tính người dùng (vai trò `admin`/`manager`/`staff`):** tài liệu chỉ nêu middleware `hrm-session` gắn `req.user` ở identity-service. Cách các service khác lấy được danh tính và vai trò chưa được quy định. Hỏi A trước khi làm phần phân quyền.
- **Luật tính phạt chấm công:** mức phạt cụ thể lưu trong `AttendanceConfig`. Xem mục 5 để biết schema. Giá trị mặc định: `latePenaltyAmount=20000`, `gracePeriodMinutes=5`.
- **Luồng `shift_swap`:** khi E tiếp nhận yêu cầu đổi ca và duyệt, có thể cần gọi vào `PUT /api/shifts/:id` của bạn để cập nhật phân công. Xác nhận luồng này với E trước khi làm.
- **Ma trận phân quyền chi tiết theo vai trò cho từng endpoint:** không được nêu. **Phụ trách: A** (xem `A_Work_Flow.md` Task 5) **(suy ra)**.
- **Định dạng body của `/health`:** tài liệu PTPMDV không nêu, xem mục 2.3.

---

## 10. Quy tắc làm việc cho agent

1. Đọc mục 1 trước. Chỉ sửa trong phạm vi của B.
2. Không tự bịa endpoint, tên trường hay hành vi. Nếu tài liệu không nêu, dùng nhãn **(suy ra)** và hỏi lại.
3. Bám đúng envelope, tên trường và định dạng ngày ở mục 5.
4. Không commit thẳng vào `dev` hoặc `main`. Mỗi thay đổi đi qua nhánh `feat/...` và PR về `dev`.
5. Không import chéo mã nguồn giữa các service. Muốn dùng dữ liệu của service khác thì gọi qua HTTP.
6. Sau mỗi thay đổi, kiểm tra `GET /health` và chạy test của service (`tests/`).
7. Khi báo kết quả, nói rõ đã chạy lệnh nào và kết quả ra sao. Không báo "pass" khi chưa chạy.
8. Ưu tiên: hoàn thành cả organization-service và work-service trước mốc 1, vì D và mobile phụ thuộc vào dữ liệu này.

---

## 11. Tiến độ đợt "Lưu trữ" (hoàn thành 10-11-2026)

Workspace hiện không phải git repo nên không tạo nhánh/PR; thay đổi nằm thẳng trong cây mã nguồn.

- [x] work-service: `ArchivedTask` + migration `archived_tasks`; `ArchiveService/Controller/routes` cho `/api/archive/work/*` (preview/run batch ≤2000/list/detail/restore, restore trùng id trả 409). Test `tests/integration/archive.test.js` pass; toàn bộ suite work 7/7.
- [x] work-service: `src/clients/IntegrationClient.js` (timeout 5000ms); `deleteTask` gọi xoá `proofPhotoUrl` best-effort sau khi xoá row; env `INTEGRATION_SERVICE_URL`.
- [x] organization-service: `src/clients/IntegrationClient.js`; `removeEmployee` dọn avatar/cccdFront/cccdBack bằng `Promise.allSettled` best-effort; env `INTEGRATION_SERVICE_URL`. Suite org 3/3 pass.
- [x] E2e: xoá task qua gateway kéo theo xoá file trên đĩa + soft-delete `FileAsset` (đối soát ở integration).
