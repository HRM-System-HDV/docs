# AUDIT BE, CSDL VÀ WEB — HRM Enterprise

> Cập nhật: 13/11/2026. Phạm vi: Gateway + 5 service BE, 5 DB PostgreSQL và đợt tái cấu trúc Web của đồ án SOA.
> Căn cứ nghiệp vụ: [BUSINESS.md](BUSINESS.md). Căn cứ cấu trúc dữ liệu: năm file Prisma trong backend.
> Báo cáo phân biệt code đã sửa, test đã chạy và kiểm chứng DB còn thiếu; không đồng nhất test memory/Prisma mock với test trên PostgreSQL thật.

## 1. Kết luận hiện tại

| Hạng mục | Trạng thái và bằng chứng |
|---|---|
| Tái cấu trúc 5 service và Gateway | Đã sửa code theo nghiệp vụ mới; xem mục 3. |
| Thiết kế CSDL | 25 bảng nghiệp vụ, UUID, Decimal, ngày có kiểu; cả 5 schema qua Prisma validate. |
| Migration | Có migration `20261112230000_hrm_uuid_redesign` cho mỗi service; đã deploy thêm `work-service/20261113120000_shift_partial_unique_cancelled` và `payroll-service/20261113100000_payout_allow_zero_amount` (DB trống/local). `prisma migrate status` 5 service up-to-date; history mồ côi `_prisma_migrations` đã dọn. |
| Kiểm thử BE | Chạy lại 13/11/2026 ~05:5x với `DATABASE_URL=""`: `npm.cmd test` trong backend → **236 test, 230 pass, 6 skip, 0 fail** (gateway 134, identity 21, org 15, work 19, payroll 12+6 skip, integration 28, flow 1). 6 skip payroll là các test cần DB, chạy riêng `test:db` → 6/6 pass ⇒ effectively **236/236, 0 fail**. |
| Kiểm thử HTTP xuyên service | Đã qua, chạy sáu tiến trình HTTP độc lập với dữ liệu memory cô lập, có luồng REST và SOAP thật giữa các service (`tests/service-flow.test.cjs` 1/1). |
| Kiểm chứng ghi/rollback trên DB thật | **Đã có bằng chứng bổ sung (13/11/2026):** `npm.cmd run test:database` → 5/5 service OK (rollback sạch, script đã điều chỉnh tương thích singleton/số dư sẵn có); `node scripts/verify-schema-drift.cjs` → 17 CHECK + 1 partial unique còn nguyên; `cd backend/payroll-service && npm.cmd run test:db` → 6/6 pass trên PostgreSQL thật (idempotency, concurrent, 422, `bankReference` sequence, GET chỉ đọc, payout 0đ). |
| Web | Đã đồng bộ contract BE mới. typecheck 0 lỗi, eslint 0 lỗi, Vitest **17 file / 62 test pass**, `next build` thành công; **E2E trình duyệt thật qua Gateway + DB thật**: đăng nhập 200, 15/15 trang dashboard, một lượt ghi thật `POST /api/tasks → 201` (xem §6.4). |
| Mobile | Đã đồng bộ theo contract UUID/envelope mới và **có verify thật** bằng Flutter SDK tại `D:\flutter`: `flutter analyze` **0 issue**, `flutter test` **125 pass / 1 skip / 0 fail** (xem §6.5). Chưa chạy trên thiết bị vật lý. |
| Bàn giao | BE + Web + Mobile đều xanh ở mức test tự động và E2E web; code review vẫn cần người xác nhận checklist §7. Phần chưa nghiệm thu còn lại giới hạn ở thiết bị vật lý và các quyết định sản phẩm ở §9.1. |

Báo cáo này thay thế các kết luận cũ như “27 bảng”, “giữ archive”, “BE chưa sửa”, “chưa chạy migration” và hướng dẫn bắt buộc reset. Không dùng các kết luận đợt cũ làm trạng thái hiện tại.

## 2. Phạm vi dữ liệu cuối

| DB/service | Số bảng | Model |
|---|---:|---|
| Identity | 3 | User, DeviceSession, RefreshToken |
| Organization | 4 | Branch, Department, Position, Employee |
| Work | 6 | ShiftTemplate, Shift, Attendance, ShiftRegistration, Task, AttendanceConfig |
| Payroll | 3 | BankAccount, Payout, Payslip |
| Integration | 9 | Request, Notification, NotificationSetting, SoapGatewayConfig, News, Regulation, WifiConfig, Backlog, FileAsset |
| Tổng | **25** | Không tính bảng quản lý migration của Prisma. |

Nguồn định nghĩa dữ liệu:

- [Identity Prisma](../backend/identity-service/prisma/schema.prisma)
- [Organization Prisma](../backend/organization-service/prisma/schema.prisma)
- [Work Prisma](../backend/work-service/prisma/schema.prisma)
- [Payroll Prisma](../backend/payroll-service/prisma/schema.prisma)
- [Integration Prisma](../backend/integration-service/prisma/schema.prisma)

Các điểm cần giữ khi review:

- User.id khác Employee.id; liên kết bằng User.employeeId, không suy luận hai ID giống nhau.
- Vai trò hệ thống thuộc Identity. Employee chỉ có vị trí công việc và thông tin nhân sự/lương.
- Employee chỉ có một mức tiền salary; monthly có standardWorkDays dương, hourly không có số công thỏa thuận.
- Shift không có branchId hoặc thuộc tính Wi-Fi; phạm vi nhân sự được lấy từ Organization.
- Unique nhân viên/ngày/ca mẫu **(ca hoạt động, không tính ca `đã hủy`)** chống phân trùng; Attendance.shiftId unique bảo đảm một dòng chấm công mỗi ca. (partial unique `shifts_employeeId_workDate_templateId_active_key`, migration work-service 20261113120000.)
- Task riêng gán trực tiếp employee; task chung theo branchId/workDate/templateId, dùng chung một kết quả hoàn thành.
- Payslip có người/thời điểm xác nhận, enum pending_confirmation/confirmed/paid.
- Nội dung có phạm vi chi nhánh hoặc toàn thể. Mỗi dòng Notification vẫn chỉ thuộc một người nhận.
- Không có slug, updatedAt hoặc bảng archive trong thiết kế hiện tại. UUID là khóa kỹ thuật; mã nghiệp vụ dùng để hiển thị/tra cứu.
- Không có FK, JOIN hoặc truy cập DB xuyên service.

## 3. Những phần code đã thay đổi

| Thành phần | Đã sửa | Nơi kiểm tra chính |
|---|---|---|
| Gateway | Truyền UUID user/employee/branch riêng; quyền hệ thống và xác nhận phiếu; staff token Mobile; loại header mạo danh/internal token; thêm positions, bỏ archive; SOAP passthrough. | [app](../backend/api-gateway/src/app.js), [auth](../backend/api-gateway/src/middlewares/auth.js), [authorize](../backend/api-gateway/src/middlewares/authorize.js) |
| Identity | systemRole; liên kết employee; thiết bị và lastSeenAt; staff Mobile/Web khác nhau; token có hạn, xoay vòng và thu hồi; quản trị tài khoản/reset mật khẩu chỉ admin; không tự tạo tài khoản lúc boot. | [AuthService](../backend/identity-service/src/services/AuthService.js), [routes](../backend/identity-service/src/routes/authRoutes.js) |
| Organization | CRUD UUID/mã nghiệp vụ, positions và FK; CCCD đầy đủ; kiểm tra salary/số công/ngày; staff tự xem/sửa trường được phép, manager theo chi nhánh; nghỉ việc giữ hồ sơ và đồng bộ Identity. | [OrgService](../backend/organization-service/src/services/OrgService.js), [validator](../backend/organization-service/src/validators/orgValidator.js), [access](../backend/organization-service/src/utils/employeeAccess.js) |
| Work | Phân ca chống trùng/chồng giờ, cho phép ca liền nhau; Wi-Fi preflight; check-in/out, kép, ca qua đêm, hạn checkout; admin/manager nhập giờ; phạt phần trăm; task chung/riêng. | [WorkService](../backend/work-service/src/services/WorkService.js), [routes](../backend/work-service/src/routes/workRoutes.js), [HTTP contract](../backend/work-service/README.md) |
| Payroll | Lương từ công hợp lệ, dùng Decimal/thời lượng gốc; cập nhật phiếu chưa xác nhận theo lương hiện tại; xác nhận đúng chủ thể; đóng băng kết quả đã xác nhận; chỉ chi phiếu hợp lệ; chống chi trùng và transaction số dư. | [PayrollService](../backend/payroll-service/src/services/PayrollService.js), [routes](../backend/payroll-service/src/routes/payrollRoutes.js) |
| Integration | Mẫu đơn không tự sửa chức năng khác; **kiểm tra đổi ca (phối hợp Q-B-01): lấy ca qua Work, xác minh chưa chấm công/chưa tới giờ/chưa hủy; thiếu attendance thì fallback `listShifts`, không xác định được trạng thái → 409 fail-closed**; thông báo fanout nguyên tử + phạm vi WHERE; nội dung chi nhánh/toàn thể; Wi-Fi cho Work; file/CCCD/PDF và nhật ký; SOAP cần payslipIds. | [IntegrationService](../backend/integration-service/src/services/IntegrationService.js), [SoapService](../backend/integration-service/src/services/SoapService.js), [FileService](../backend/integration-service/src/services/FileService.js), [app](../backend/integration-service/src/app.js) |

Các phần đã loại hoặc thay thế: archive source/routes/tests; thao tác bằng slug/tên thay ID; trường lương cũ baseSalary/hourlySalary trên hồ sơ; phạt tiền cố định; tự seed tài khoản khi khởi động; script backfill cũ và wrapper/formatter không còn dùng. Không xóa bảng theo dữ liệu người dùng để làm test qua.

Service có cấu hình DB nhưng kết nối/truy vấn lỗi không được âm thầm chuyển sang memory. Memory phục vụ chế độ dev/test không có DATABASE_URL; test phải chủ động cô lập cấu hình.

## 4. Hợp đồng API cần chú ý khi kiểm tra

| Luồng | API/dữ liệu đáng chú ý |
|---|---|
| Đăng nhập/làm mới token | POST `/api/auth/login`, `/api/auth/refresh`; clientPlatform web hoặc mobile. Staff dùng mobile và Bearer, không có cookie phiên Web. |
| Hồ sơ | UUID trong URL; salary, salaryType, standardWorkDays; không gửi Employee.systemRole, branchSlug hoặc hourlySalary. |
| Kiểm tra trước màn ca | POST `/api/attendance/preflight` với wifi.bssid và tùy chọn wifi.ssid; sai Wi-Fi bị từ chối trước khi trả danh sách ca. |
| Check-in/checkout | POST `/api/attendance/checkin` hoặc `/checkout`, có shiftId và wifi. Giờ nhân viên do server ghi nhận. |
| Chấm công kép | POST `/api/attendance/double` với shiftId ca trước, nextShiftId ca sau và wifi. |
| Admin/manager nhập giờ | PUT `/api/attendance/:shiftId`, checkIn/checkOut ISO có múi giờ; thao tác trên ca hiện có. |
| Cấu hình chấm công | GET/PUT `/api/attendance/config`, admin; peer service có credential nội bộ được đọc. |
| Xác nhận phiếu | POST/PUT `/api/payroll/payslips/:id/confirm`; owner, manager đúng phạm vi hoặc admin. Không trừ tiền tại bước này. |
| Chi lương REST | POST `/api/payroll/payouts`, payslipIds, idempotencyKey, debitAccountId hoặc debitAccount; tổng tiền/số người nếu gửi phải khớp dữ liệu suy ra. |
| Chi lương SOAP | POST `/soap/payroll` XML, các thẻ payslipId UUID; GET `?wsdl`. SOAP không được bọc envelope JSON. |
| Mẫu đơn | approve/reject hoặc accept/decline chỉ xử lý mẫu đơn, không tự sửa lịch, công hay lương. |
| Task riêng | assignedTo là Employee.id; không có API/bước nhận việc. |
| Task chung | branchId + workDate + templateId; một người hoàn thành thì cả nhóm ca thấy xong. |

Quyền Web/Mobile là quy ước ứng dụng qua clientPlatform đã ký trong token; test hiện xác nhận policy API/cookie, không phải nhận diện thiết bị vật lý. Phạm vi dữ liệu phải được service kiểm tra dù route đã qua Gateway.

## 5. Migration đã thực hiện

Cả năm service có thư mục `prisma/migrations/20261112230000_hrm_uuid_redesign/`. Đã bổ sung 2 migration: `work-service/prisma/migrations/20261113120000_shift_partial_unique_cancelled/` (partial unique ca hoạt động) và `payroll-service/prisma/migrations/20261113100000_payout_allow_zero_amount/` (nới `payout_amounts_valid` cho phép `totalAmount >= 0`).

Trước khi áp dụng redesign, đã kiểm tra DB local đều trống (cấu trúc cũ ID số nguyên). Sau đó `migrate deploy` thành công; lượt 13/11/2026 đã deploy thêm 2 migration bổ sung, dọn dòng `_prisma_migrations` mồ côi và xác nhận `migrate status` 5 service up-to-date.

Migration thực hiện trong transaction từng DB:

1. Khóa/kiểm tra các bảng nghiệp vụ cũ. Nếu xuất hiện dữ liệu thì báo lỗi và dừng.
2. **Thay cấu trúc bảng trống bằng `DROP TABLE IF EXISTS` + `CREATE TABLE` mới** (không dùng `ALTER` chuyển ID cũ→UUID); loại bảng archive cũ. DB có dữ liệu sẽ bị guard chặn.
3. Tạo hàm pad_code/sequence ở service **cần sinh mã** (thực tế: Organization `branch_code_seq`,`employee_code_seq`; Payroll `payout_ref_seq`→`GD-`). Work/Integration/Identity không có sequence — mã do caller gửi, service kiểm trùng app + DB unique.
4. Tạo unique, FK, CHECK và partial unique index theo thiết kế. **CHECK/partial unique là nợ drift** (Prisma không biểu diễn trong `schema.prisma`), chỉ tồn tại trong `migration.sql` — cấm `migrate dev/diff`, dùng `migrate deploy` và `node scripts/verify-schema-drift.cjs`.
5. Giữ bảng lịch sử `_prisma_migrations`; không chạy migrate reset, không tự seed.

Các ràng buộc SQL quan trọng đã có trong migration:

- Salary không âm; monthly có số công dương, hourly không có số công thỏa thuận.
- Giờ ra chỉ có khi có giờ vào và không trước giờ vào; tiền phạt không âm.
- Task có người được giao hoặc đủ phạm vi ca chung.
- Hai bảng cấu hình: singleton true + unique; số phút không âm, phần trăm trong 0–100.
- Phiếu đã xác nhận có người và thời điểm xác nhận; phiếu paid có payoutId.
- Kỳ lương đúng YYYY-MM; tiền trên phiếu khớp công thức; kiểu lương/giờ làm phù hợp.
- Số dư không âm; tối đa một tài khoản nguồn chính; lệnh chi có tiền và số người hưởng dương.
- Mẫu đơn có dữ liệu đúng loại và thông tin người xử lý phù hợp trạng thái.

**Cập nhật 13/11/2026:** đã dọn dữ liệu probe `hrm_organization`, dọn lịch sử migration mồ côi, deploy 2 migration bổ sung (work partial unique ca hoạt động, payroll `payout_amounts_valid >= 0`), `migrate status` 5/5 up-to-date. `node backend/scripts/verify-schema-drift.cjs` xác nhận 17 CHECK + 1 partial unique còn nguyên; `npm.cmd run test:database` 5/5 OK (rollback sạch); `payroll-service npm run test:db` 6/6 trên PostgreSQL thật. Migration redesign chỉ dùng cho **DB trống** (DROP+CREATE, guard), không phải công cụ chuyển dữ liệu cũ sang UUID. `docs/AUDIT.md` §6.1 vẫn ghi số liệu test của lượt trước (187/187) — lượt sửa lỗi gần nhất: `npm.cmd test` 231 pass, 0 fail, 1 skip; `test:flow` 1/1. Bằng chứng chi tiết và review 4 tầng: `GiaoTiep.md`.

## 6. Bằng chứng kiểm thử đã chạy

### 6.1. Bộ test BE gần nhất

Lệnh: `DATABASE_URL="" npm.cmd test` tại `backend/` (chạy 13/11/2026 05:45). Runner chạy
từng service rồi test HTTP xuyên service; `DATABASE_URL=""` để suite không ghi vào
PostgreSQL dùng chung.

| Bộ test | Qua | Fail | Phạm vi bằng chứng |
|---|---:|---:|---|
| Gateway | 134 | 0 | JWT/platform, quyền route, chống header mạo danh, proxy và SOAP XML. |
| Identity | 21 | 0 | Phiên/refresh, staff Mobile, admin tài khoản, sync và Prisma mock. |
| Organization | 15 | 0 | CRUD, salary/ngày/FK, phạm vi, nghỉ việc và lỗi Prisma. |
| Work | 19 | 0 | Wi-Fi, trùng/chồng giờ, kép/rollback mock, qua đêm, hạn checkout, manual và task. |
| Payroll | 12 (+6 skip) | 0 | Tính lương, xác nhận/freeze, số dư, idempotency, transaction mock. |
| Integration | 28 | 0 | Form-only, quyền/phạm vi, fanout, file/PDF, SOAP và Prisma mock. |
| HTTP xuyên service | 1 | 0 | Một kịch bản gồm nhiều bước thực hiện qua sáu tiến trình độc lập. |
| Tổng | **236** | **0** | 6 test payroll tự skip khi không có `DATABASE_URL`; xem §6.1.1. |

Bỏ số skip: `npm.cmd test` báo `fail 0`, `cancelled 0`, `skipped 6` trên cả bảy suite.

#### 6.1.1. Sáu test payroll cần PostgreSQL thật

`payroll-service/tests/integration/payout-idempotency.db.test.js` skip khi thiếu
`DATABASE_URL`. Chạy riêng với DB thật của payroll (13/11/2026 05:52):

```
cd backend/payroll-service && DATABASE_URL=<hrm_payroll> npm run test:db
→ tests 6  pass 6  fail 0  skipped 0
```

Sáu test gồm: đọc phiếu lương không ghi DB, gửi lại cùng `idempotencyKey` không trừ tiền
lần hai, hai lệnh chi song song chỉ trừ một lần, số dư không đủ → 422
`INSUFFICIENT_FUNDS`, `bankReference` lấy từ sequence Postgres, và lệnh chi 0đ đóng phiếu
thực lĩnh 0đ (Q-D-02). Test tự tạo bản ghi UUID ngẫu nhiên và xóa trong `after()`, không
chạy migration. **Tổng thực tế khi có DB: 236/236.**

Nguồn test HTTP: [service-flow.test.cjs](../backend/tests/service-flow.test.cjs). Luồng đã kiểm tra gồm tạo chi nhánh/nhân viên → đồng bộ Identity → đăng nhập staff → phân ca/chặn trùng → ghi công → task chung/riêng → Wi-Fi → duyệt đơn không đổi công → cập nhật salary → sinh phiếu → chặn chi chưa xác nhận → xác nhận → chi REST/SOAP và gửi lặp không trừ tiền lần hai → nội dung/thông báo theo chi nhánh.

Ngoài suite chính, một chuỗi nghiệp vụ mới chạy thẳng qua Gateway :4000 và PostgreSQL thật
(13/11/2026 05:56, tạo nhân viên QA mới để không reuse dữ liệu cũ): **23/23 PASS** —
nhân viên → tài khoản staff → phân ca → chặn key thừa → chấm công → Decimal trả về dạng
chuỗi → task cá nhân/chung → staff chỉ sửa `status`/`proofPhotoUrl` → sinh phiếu lương.

### 6.2. Các kiểm tra khác

- Prisma format/validate thành công cho cả năm schema.
- Prisma Client đã generate cho năm service trong quá trình tái cấu trúc.
- git diff --check không phát hiện lỗi whitespace; cảnh báo LF/CRLF không phải lỗi test.
- `.env` thật được gitignore. Đã đồng bộ secret nội bộ và bổ sung URL peer còn thiếu; báo cáo không chứa giá trị secret.
- Tên tiếng Việt được mã hóa khi truyền HTTP header; có regression test proxy và kiểm tra tên người duyệt trong luồng liên service.

### 6.3. Kiểm chứng bị chặn

Lệnh `npm.cmd run test:database` gọi [verify-database.cjs](../backend/scripts/verify-database.cjs). Script thiết kế tạo bản ghi thử và kiểm tra CHECK/unique/FK trong transaction rồi rollback, không seed dữ liệu.

Lượt chạy hiện tại dừng tại kết nối Identity với P1001, localhost:5433. Chưa chạy thành công các phép ghi/rollback trong script; không đánh dấu các DB còn lại đã qua. Các phép kiểm tra migrate status/diff gần nhất cũng không kết nối được localhost:5433–5437.

### 6.4. Web — kiểm tra tĩnh và tương tác trình duyệt thật

Kiểm tra tĩnh (13/11/2026 05:35, tại `frontend/`): `npx tsc --noEmit` 0 lỗi; `npx eslint .`
0 lỗi/cảnh báo; `npx vitest run` 17 file / **62 test qua / 0 fail**; `npm run build` thành
công, sinh 16 tuyến (`/login`, `/dashboard` + 14 trang con, `/icon.svg`).

Tương tác thật (13/11/2026 05:40): Next dev trên `http://localhost:3000`, Gateway
`:4000` và năm PostgreSQL thật, tài khoản `admin.test@hrm.local` qua giao diện (không gọi
API tắt):

- `POST /api/auth/login → 200`, chuyển vào `/dashboard/employees`; trang render bản ghi
  thật (`NV-00001…NV-00003`, lương `8.000.000 ₫ / tháng`, trạng thái `đang làm`).
- Cả **15/15** tuyến dashboard mở được, mọi request xuống Gateway trả **2xx**, không có
  text lỗi `Lỗi/Không tải được/Không có quyền`, không có `undefined`/`NaN`/`[object Object]`
  lọt ra màn hình.
- **Một lượt ghi thật qua UI**: biểu mẫu "Giao công việc" gửi `POST /api/tasks → 201`,
  `assignedTo` là UUID nhân sự, bảng cập nhật thành "Đã lưu công việc". Đây là bằng chứng
  contract `fields()` nghiêm ngặt (không khóa thừa) khớp với web.
- Số Decimal từ Prisma (vd `baseSalary:"375000"`) render đúng dạng `375.000 ₫`, xác nhận
  client đã parse chuỗi thay vì giả định số.

Lưu ý môi trường: truy cập dev server bằng `http://127.0.0.1:3000` bị Next 16 chặn
**403 trên chính file JS của nó** (`/_next/static/chunks/*`) vì host không nằm trong dev
origin — trình duyệt không hydrate được và form submit kiểu GET. Đây là cấu hình dev,
không phải lỗi sản phẩm; dùng `http://localhost:3000`.

### 6.5. Mobile — kiểm chứng bằng Flutter SDK

Máy **có** Flutter 3.47.0 / Dart 3.13.0 tại `D:\flutter` (không nằm trên PATH nên
`flutter`/`dart` trần báo not found; phải gọi `D:/flutter/bin/flutter.bat`). Số liệu
vòng 5, chạy 13/11/2026 ~05:55 trong `mobile/`:

| Lệnh | Kết quả đo được |
|---|---|
| `D:/flutter/bin/flutter.bat analyze` | `No issues found!` (đầu vòng 5 là 174 lỗi + 1 cảnh báo) |
| `D:/flutter/bin/flutter.bat test` | `+125 ~1`: 125 test qua, 1 skip, **0 fail** |

Hai nhóm từng đỏ đã xanh: `test/home_avatar_test.dart` (avatar quy về một nguồn
`UserScope`, bỏ biến `_avatarUrl` riêng ở `home.dart`) và
`test/missed_shift_checkout_test.dart` (fixture dùng token `missed`, sửa thành
`missed_checkout` đúng giá trị WorkService phát ra). Root-cause chính của phần lớn lỗi
biên dịch là `UserModel` bị cắt quá tay ba cột có thật trong API
(`phone`/`cccdFrontUrl`/`avatarUrl`), đã khôi phục thay vì sửa lẻ 70 điểm gọi.

Chưa chạy trên thiết bị vật lý: Wi-Fi thật, camera CCCD và push nền vẫn chỉ ở mức code
+ unit/widget test (xem §9).

### 6.6. Nghiệm thu DB thật (vòng 5, 13/11/2026 ~05:58)

Năm container PostgreSQL local (5433–5437) đang chạy và healthy trong lượt kiểm này.

| Lệnh | Kết quả đo được |
|---|---|
| `npm.cmd run test:database` | 5/5 service: `real PostgreSQL writes, constraints and rollback OK`, exit 0. Mọi bản ghi kiểm chứng nằm trong `$transaction` luôn bị `throw rollback`, không để lại dữ liệu. |
| `npm.cmd run prisma:check` | Exit 0. Ràng buộc còn nguyên trên DB thật: organization 2 CHECK, work 5 CHECK + 1 partial unique index, payroll 7 CHECK, integration 3 CHECK. Cảnh báo còn lại là **nợ drift** (17 ràng buộc chỉ biểu diễn được trong `migration.sql`, không có trong `schema.prisma`) — hệ quả cố hữu của Prisma, không phải mất ràng buộc. |
| `payroll-service``npm.cmd run test:db` | `tests 6 / pass 6 / fail 0` — idempotency lệnh chi trên DB thật, `after` hook chỉ xóa đúng dòng test tự tạo. |

Không dùng `migrate reset`, `db push` hay seed trong toàn bộ vòng 5.

## 7. Checklist bạn kiểm tra thủ công

Các ô dưới đây để người review xác nhận; không đánh dấu thay bằng kết quả unit test.

### 7.1. Hạ tầng và dữ liệu

- [ ] Bật năm PostgreSQL local trên cổng 5433–5437, kiểm tra DATABASE_URL từng service.
- [ ] Chạy status của từng service; migration redesign có trạng thái applied, xem cả lịch sử cũ nếu có cảnh báo.
- [ ] Chạy `npm.cmd run test:database` thành công cho cả năm DB.
- [ ] Khởi động sáu service với cấu hình DB thật, kiểm tra health và thao tác CRUD qua Gateway.
- [ ] Tạo tài khoản admin bằng seed tường minh hoặc quy trình được chọn; không trông chờ tài khoản/mật khẩu mặc định khi boot.

### 7.2. Quyền và phạm vi

- [ ] Staff bị từ chối đăng nhập Web; đăng nhập Mobile xem/sửa đúng dữ liệu cá nhân.
- [ ] Manager không truy cập chức năng quản trị hệ thống hoặc dữ liệu chi nhánh khác.
- [ ] Admin thao tác được trên Web/Mobile theo nghiệp vụ; nghiệm thu tương tác Web đã refactor và Mobile trên API mới.
- [ ] Bản tin/quy định/thông báo chi nhánh A không xuất hiện với nhân viên chi nhánh B; nội dung toàn thể hiển thị đúng.

### 7.3. Ca và chấm công

- [ ] Sai Wi-Fi: không mở màn chọn ca. Đổi sang Wi-Fi sai sau khi mở màn: BE vẫn từ chối ghi nhận.
- [ ] Phân lại cùng ca cho cùng nhân viên bị chặn; ca khác mẫu chồng giờ cũng bị chặn; ca liền nhau được phép.
- [ ] Nhân viên đã có ca không được chọn nhờ làm thay/đổi vào cùng ca.
- [ ] Check-in sớm tuân thủ cấu hình; đi muộn vẫn cho chấm và phạt đúng phần trăm.
- [ ] Checkout sớm có phạt; hết hạn checkout thì giữ thiếu giờ ra, không có lương, không cản ca mới.
- [ ] Bật chấm kép, đặt sớm/trễ 5 phút: hai ca nối 12:00 ghi cả giờ ra/vào bằng 12:00 khi bấm trong khoảng hợp lệ.
- [ ] Chấm kép không ghi trùng, không chỉ ghi một nửa và không làm ca sau có lương trước khi checkout.
- [ ] Admin/manager nhập giờ trên ca: lưu đủ giờ thì có công; duyệt đơn riêng lẻ không tự sinh giờ.

### 7.4. Công việc và lương

- [ ] Task riêng giao trực tiếp, không có bước nhận việc, không ảnh hưởng task riêng người khác.
- [ ] Task chung chỉ đúng nhóm ca nhìn thấy; một người hoàn thành thì mọi người thấy xong; ảnh minh chứng theo yêu cầu.
- [ ] Parttime 30.000đ × 5 giờ = 150.000đ trước thưởng/phạt; số phút lẻ không tính tiền từ workedHours đã làm tròn.
- [ ] Fulltime dùng số công thỏa thuận, mỗi ca hoàn thành một công; thiếu cặp vào–ra không cộng lương và không trừ thêm phạt ca đó.
- [ ] Đổi salary cập nhật phần chưa xác nhận; phiếu đã xác nhận giữ kết quả.
- [ ] Nhân viên/manager đúng phạm vi/admin xác nhận phiếu; không tự trừ số dư.
- [ ] Chi phiếu chưa xác nhận bị chặn; chi hợp lệ qua REST/SOAP trừ đúng một lần; gửi lại cùng key không chi lại.
- [ ] Thiếu số dư hoặc tổng tiền không khớp không làm phiếu thành paid và không trừ tiền.

## 8. Lệnh tái kiểm

Tại thư mục backend, PowerShell:

```powershell
npm.cmd test
npm.cmd run test:flow
```

Sau khi PostgreSQL đã chạy:

```powershell
npm.cmd run test:database
```

Kiểm tra lịch sử từng service, ví dụ Identity:

```powershell
Set-Location identity-service
npx.cmd prisma migrate status
```

Chỉ deploy khi cần áp migration ở môi trường đích đã kiểm tra:

```powershell
npm.cmd run prisma:deploy
```

Không dùng db:setup/seed/reset làm bước “sửa lỗi” khi DB có dữ liệu. Không lấy build/typecheck làm bằng chứng hoàn tất nghiệm thu giao diện tương tác trên DB thật.

## 9. Giới hạn và phần chưa nghiệm thu

| Điểm | Hiện trạng |
|---|---|
| DB thật | Đã kiểm chứng ở lượt này (13/11/2026 ~05:58, xem §6.6): ghi/ràng buộc/rollback qua `test:database` cho cả năm service, ràng buộc còn nguyên theo `prisma:check`, 6 test payroll DB qua trực tiếp. Nợ drift 17 ràng buộc chỉ-sql vẫn còn như mô tả ở §6.6. |
| UI và thiết bị thật | Web ĐÃ chạy tương tác trình duyệt thật với DB thật (13/11/2026 05:40, xem §6.4): đăng nhập + 15 trang dashboard + một lượt ghi thật `POST /api/tasks → 201`. Mobile mới dừng ở `flutter analyze` + `flutter test` trên máy, chưa chạy trên thiết bị vật lý; Wi-Fi thật và camera CCCD chưa nghiệm thu trên phần cứng. |
| Nhắc giờ | Tạo khi client lấy thông báo hoặc gọi nhắc giờ; chưa có scheduler/push nền khi ứng dụng không hoạt động. |
| Nhật ký Work → Integration | Best-effort; lưu công thành công không bị trả lỗi giả khi dịch vụ nhật ký gián đoạn, nhưng không bảo đảm log xuyên DB trong trường hợp đó. |
| Token truy cập đã cấp | Thu hồi phiên/refresh không biến mọi access token cũ thành vô hiệu ngay; Gateway kiểm chữ ký và hạn sống ngắn. |
| Hồ sơ → Identity | Đồng bộ HTTP best-effort; khi Identity tạm ngừng có cảnh báo, cần kiểm tra/đồng bộ lại, không có hàng đợi bền vững. |
| Phân trang/khối lượng lớn | Bộ test hiện không chứng minh hiệu năng với dữ liệu lớn; phạm vi vẫn là đồ án. |
| Coverage | Chưa đo lại coverage; 236 test qua không có nghĩa 100% nhánh code đã được bao phủ. |
| Nhánh/commit | Thay đổi nằm trong workspace; báo cáo này không khẳng định đã commit, merge hoặc triển khai môi trường khác. |

### 9.1. Điểm dừng có chủ đích (quyết định sản phẩm, không phải lỗi chưa sửa)

Bốn mục dưới đây **đội đã cố ý dừng** vì hợp đồng BE mới hoặc vì tránh over-engineering;
chủ sản phẩm cần xác nhận nếu muốn mở lại.

| Mục | Hiện trạng & lý do |
|---|---|
| Nhân viên (Staff) không còn chế độ "chỉ xem" và nút quét Wi-Fi chung | `GET /api/wifi-configs` phía Gateway chỉ vai trò quản trị; nhân viên lấy cấu hình qua luồng chấm công. Giữ màn hình quản trị cho Staff sẽ hiển thị dữ liệu bị chặn, nên UI Mobile đã bỏ hai đường đó theo contract. |
| Nút "kê khai sai lương" trên phiếu lương | `Request` chỉ nhận `type` ∈ {leave, advance, …} theo `docs/BUSINESS.md`; không có loại `other`. Nút cũ gửi type không tồn tại nên đã bỏ. Nếu nghiệp vụ cần khiếu nại lương, phải thêm type vào BE trước. |
| "Chấm công kép" không tới được từ giao diện | Hành vi **có từ trước** vòng 5, không do đợt này thay đổi; BE vẫn từ chối trùng ca bằng ràng buộc DB, còn đường vào UI thì cả Web lẫn Mobile đều không có ô nhập. |
| Xóa / đánh dấu tất cả thông báo đã đọc | BE không cấp route tương ứng (`integration-service` chỉ có danh sách + đánh dấu đã đọc từng cái). Client đã ẩn thao tác thay vì gọi API 404. |

Dữ liệu kiểm chứng (QA) tạo ra trong các lượt probe trước đây **được giữ nguyên**, không
xóa: người dùng đang ngủ và thao tác xóa hàng loạt là không đảo ngược. probe chủ động đều
chạy với `DATABASE_URL=""` hoặc tự rollback chính record nó tạo.

## 10. Cách chốt nghiệm thu

- [x] Có đặc tả nghiệp vụ và schema mới để đối chiếu.
- [x] Có code BE và migration tương ứng, đã bỏ các luồng cũ trái phạm vi.
- [x] Bộ test BE và HTTP liên service gần nhất qua 236/236 (có DB thật cho payroll; 6 test tự skip khi thiếu `DATABASE_URL`).
- [x] Cả năm schema hợp lệ; migration đã được áp dụng trên năm DB local ở lượt trước.
- [x] Hoàn thành kiểm chứng ghi/rollback và tái kiểm migration trên DB thật sau khi bật lại hạ tầng (xem §6.6).
- [ ] Người review hoàn thành checklist thủ công và xác nhận kết quả.
- [x] Tái cấu trúc Web theo API mới, kiểm tra production build/TypeScript/ESLint và HTTP smoke.
- [x] Chạy Vitest Web khi có đủ devDependencies và nghiệm thu giao diện tương tác cùng BE/DB thật (62 test + E2E trình duyệt, xem §6.4).
- [x] Cập nhật/kiểm thử Mobile: `flutter analyze` 0 issue, `flutter test` 125 pass / 1 skip / 0 fail (xem §6.5).

Không đánh dấu “toàn hệ thống hoàn tất” chỉ dựa vào những ô đã tick ở phần BE.

## 11. Đợt tái cấu trúc Web — 13/11/2026

Đã chia nhóm triển khai: tổ chức/nhân sự, ca/công việc, lương/ngân hàng và đơn/nội dung/Wi-Fi. Phần dùng chung gồm xác thực, DTO, phân quyền, thông báo và nhật ký được tích hợp tại frontend. Không sửa BE, Prisma, migration hoặc Mobile trong đợt này.

| Phần | Kết quả triển khai |
|---|---|
| Xác thực/phân quyền | Cookie HttpOnly, refresh đồng thời dùng chung một yêu cầu, gửi clientPlatform web; staff không vào Web. Phân biệt userId, employeeId và branchId; admin quản lý quyền/reset mật khẩu tại màn hình tài khoản. Menu cá nhân hỗ trợ đổi mật khẩu và xem/thu hồi phiên thiết bị khác bằng dữ liệu BE. |
| Tổ chức/nhân sự | UUID và mã nghiệp vụ riêng, danh mục vị trí, CCCD, tải ảnh đại diện/CCCD qua uploader cũ được tận dụng, một salary theo giờ/tháng và standardWorkDays; nghỉ việc giữ lịch sử. Loại bỏ role/slug và mô hình nhiều mức lương cũ trên Employee. |
| Ca/chấm công | Ca/nguyện vọng dùng ngày và templateId; kiểm tra ca đã phân, lỗi trùng/đè giờ cuối cùng vẫn do BE xác nhận. Quản lý nhập đủ giờ vào/ra ngay trên ca, có xử lý múi giờ Việt Nam và ca qua đêm. Cấu hình vào sớm, hạn checkout, phần trăm phạt và chấm công kép. |
| Công việc/đơn | Task cá nhân giao thẳng; task chung dùng một kết quả theo chi nhánh/ngày/ca mẫu. Đơn chỉ duyệt/từ chối, không gọi API sửa ca/công/lương. |
| Lương/ngân hàng | Xác nhận theo phạm vi; chỉ chọn phiếu đã xác nhận chưa chi. Tổng lệnh chi tính chính xác từ chuỗi tiền bằng BigInt, giữ idempotencyKey khi retry, SOAP Fault được trả lỗi. Giữ xuất Excel bằng ExcelJS. |
| Nội dung/thông báo | Bản tin/quy định theo chi nhánh/toàn thể, manager chỉ sửa nội dung chi nhánh mình; PDF gắn quy định. Gửi thông báo và lưu cài đặt nhắc; Topbar tải mỗi 60 giây, đánh dấu đọc từng bản ghi qua API thật. |
| Dọn code | Xóa luồng và component cũ không còn dùng, biểu đồ/thống kê giả, thao tác FE tự ghi/xóa nhật ký, hook thừa và dependency xlsx. Tận dụng Table/Form/Modal/Card/Button, upload ảnh và ExcelJS. Font Public Sans hiện có được lưu local kèm giấy phép. |

### Kiểm tra đã chạy

- `npm.cmd run typecheck`: qua. TypeScript production được tách khỏi cấu hình test; không bỏ qua lỗi build.
- `npm.cmd run lint`: qua, không lỗi hoặc cảnh báo.
- `npm.cmd run build`: qua, sinh đủ route Web; không cần kết nối Google Fonts.
- Chạy bản production tại cổng tạm 3100 và kiểm tra HTTP: 6 trường hợp qua (login, chặn khách chưa đăng nhập, chặn staff, token hết hạn, shell phục hồi bằng refresh cookie, font tiếng Việt local). Đây không phải kiểm thử đăng nhập thật hay tương tác trình duyệt.
- Các phép kiểm tra trực tiếp qua Node + TypeScript transpile: 46 assertion API/phân quyền/proxy, 17 trường hợp thời gian/payload công việc, 14 assertion lương/lệnh chi, 6 assertion quyền duyệt đơn và 8 assertion payload cài đặt nhắc đều qua. SOAP XML parser chưa được chạy trong các phép kiểm tra Node này.
- Đã cập nhật/thêm bộ test Vitest cho payload và các luồng quan trọng. **Chưa chạy được**: `npm.cmd test` dừng vì không có executable Vitest. Cài devDependencies bị lỗi mạng EACCES; thử cache offline thiếu tarball (ENOTCACHED). Không đánh dấu bộ test này đã qua.

### Còn nghiệm thu

- Cài đủ dependency từ lockfile rồi chạy `npm.cmd test` và `npm.cmd run typecheck:test`.
- Khi BE và PostgreSQL hoạt động, thử trên trình duyệt với admin/manager: hồ sơ → phân ca/chấm công → công việc/đơn → tạo/xác nhận phiếu → chi REST/SOAP → thông báo/PDF.
- Kiểm tra hình thức, bàn phím, lỗi API và thao tác nhiều tab trên trình duyệt. Chưa có bằng chứng nghiệm thu giao diện này trong đợt hiện tại.
