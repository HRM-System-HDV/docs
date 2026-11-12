# Audit toàn bộ tính năng Web + Mobile + Cầu nối BE

Ngày audit: 12/11/2026 (chạy qua đêm, 14 sub-agent đọc code)
Phạm vi: `frontend/` (web Next.js), `mobile/` (Flutter), `backend/` (gateway + 5 service)
Cách đánh giá: **đọc code request/response + test**, KHÔNG dựa tài liệu `docs/*.md`
Góc nhìn: đồ án môn học SWD hướng dịch vụ – nhóm 4 người, không đòi hardening kiểu production

Cách đọc file này buổi sáng: mục **1** (3 câu trả lời) → mục **4** (bug nên sửa trước khi bảo vệ) → mục **3** (cái gì cố tình, đừng nhận là bug).

Quy ước nhãn: `BUG` = hỏng thật; `RỦI RO` = chưa đứt nhưng lệch/thiếu; `CỐ Ý` = hợp lý, có ý đồ trong code/test.
Quy ước ownership theo `docs/AGENTS.md` §2: **A** = gateway + identity + mobile + frontend (nhóm trưởng), **B** = organization + work, **D** = payroll, **E** = integration.

---

## 1. TL;DR – trả lời 3 câu hỏi của bạn

**a) Cầu nối BE ở giữa FE và mobile đã liên kết ổn chưa?**
Chưa hẳn. Envelope `{data}` / `{error:{code,message}}` thì **nhất quán** (web `frontend/src/lib/api.ts:38-47`, mobile `mobile/lib/src/core/network/api_client.dart:285-292`) — phần "nối" cơ bản dùng được. Nhưng hai client đang dùng **hai hệ quy chiếu khác nhau cho cùng một field**, nặng nhất là `assignedTo` (web ghi **TÊN người**, mobile ghi **ID**) khiến task giao từ web **biến mất hoàn toàn trên app nhân viên**. Chi tiết ở mục 7.

**b) Dữ liệu có di chuyển ổn định không?**
Đọc–ghi từng phía thì ổn (mutation có refetch ở đa số màn), nhưng **chạy một chiều**: mobile ghi thì web đọc tạm được, web ghi task thì staff mất hẳn. Thêm 3 điểm làm dữ liệu "trông như rỗng/không có" dù API lỗi (mục 4, nhóm P2), và **không service nào phân trang** → chỉ ổn khi dữ liệu demo nhỏ.

**c) Có leak dữ liệu nào không?**
Có, và **toàn bộ thuộc nhóm "thiếu kiểm tra quyền ở chiều ĐỌC"**, không phải lộ token ra ngoài: manager đọc phiếu lương **toàn công ty** (BE không ghim chi nhánh, web chỉ ẩn bằng filter client), manager thấy CCCD/số tài khoản/lương của mọi chi nhánh, `GET /api/wifi-configs` + `/scan` mở cho **mọi** role đã đăng nhập (staff nhận hết SSID/BSSID mọi chi nhánh), `GET /api/files/*` không kiểm chủ sở hữu (ảnh CCCD lộ nếu biết UUID), `POST /api/notifications` không có luật role (staff bắn thông báo đích danh tới bất kỳ ai, mạo `senderName`). Chi tiết ở mục 6.

---

## 2. Việc đã làm tối nay (ẩn phân quyền mobile như web)

Đã xong, đã test:

| File | Nội dung |
|---|---|
| `mobile/lib/src/core/config/permissions.dart` (mới) | Bảng quyền tập trung `route → vai trò`, soi theo `MENUS` của `frontend/src/lib/permissions.tsx`. `/tasks/monitor`, `/approvals`, `/schedule/assignment` = admin+manager; `/wifi` = **chỉ admin**. Route không liệt kê = tự phục vụ, mọi role. Chưa resolve được phiên → coi là staff (mức thấp nhất). |
| `mobile/lib/src/core/widgets/no_access_screen.dart` (mới) | `NoAccessScreen` (tương đương `NoAccessNotice` web) + `guardRoute()` chặn **ngay trong GoRoute builder**. |
| `mobile/lib/src/core/router/router.dart:98,201,207,213` | Guard cho 4 tuyến đặc quyền → deep link/push không vào được màn mà mọi thao tác sẽ ăn 403. |
| `mobile/lib/src/features/tasks/operations.dart:26-27,99` | Thẻ lấy theo bảng quyền. Bỏ thẻ "Wi-Fi chi nhánh" của manager; staff vẫn ẩn trọn cụm Quản trị. |
| `mobile/test/permissions_test.dart`, `test/route_role_guard_test.dart`, `test/operations_permissions_test.dart` (mới) | 11 test: bảng quyền, guard route thật, card ẩn theo role. |

Kết quả: `flutter analyze` sạch (0 issue), `flutter test` **118 pass, 1 skip**.
Lưu ý: vì staff bị ẩn tính năng quản trị theo đúng web, `mobile/lib/src/features/wifi_config/wifi.dart` vẫn còn nhánh "chế độ đọc cho manager/staff" — giờ **không vào được từ UI**, tôi để lại như lớp phòng vệ thứ hai (nhãn `CỐ Ý`, không phải bug).

---

## 3. Nhóm "CỐ Ý" – đừng ghi vào báo cáo nộp giảng viên như là lỗi

Đây là thứ người review (kể cả AI) rất dễ báo nhầm. Tất cả đều có bằng chứng trong code/test:

1. **Web Wi-Fi chỉ XEM + XÓA, không thêm/sửa — hoàn toàn chủ đích** vì trình duyệt không quét được Wi-Fi. `frontend/src/app/dashboard/wifi/page.tsx:22-68` chỉ gọi `GET /api/wifi-configs` + `DELETE`, **không có POST/PUT**, không gọi `/scan`; `frontend/src/features/wifi/WifiForm.tsx` là **form chết** (không file nào import). Nút xóa chỉ hiện khi `isAdmin` (`page.tsx:106`). Trong khi mobile quét + thêm thật (`mobile/lib/src/features/wifi_config/wifi.dart:116,282,308`). Hai client cố tình khác nhau vì khả năng thiết bị khác nhau.
   *(Chưa có test nào encode ý đồ này — chỉ có comment. Nên thêm 1 test nếu muốn chắc.)*
2. **Staff không đăng nhập được web, chỉ dùng app** — chặn ở FE: `frontend/src/context/AuthContext.tsx:107-110,193-200` (`clearWebSessionCookies` + `STAFF_WEB_FORBIDDEN`). Phần tự phục vụ của staff trên mobile **rộng hơn** `STAFF_MENU` web là có chủ đích, ghi rõ trong comment `mobile/lib/src/core/config/permissions.dart:11-13`.
3. **Manager không có Wi-Fi/Chi nhánh/Ngân hàng/Lịch sử thao tác** — phân tầng 3 vai trò tường minh, mỗi tầng liệt kê menu riêng thay vì suy ra "không phải admin" (`frontend/src/lib/permissions.tsx:21-25,49-91`; test `backend/api-gateway/tests/authorize.test.js:46,91-127`).
4. **Quên mật khẩu KHÔNG reset gì cả** — chỉ hướng dẫn liên hệ admin, không lộ thông tin (`backend/identity-service/tests/integration/account.test.js:101-132`). Đó là thiết kế, không phải thiếu chức năng.
5. **Không fallback dữ liệu giả khi DB lỗi** — payroll/work **ném 500 có chủ đích** thay vì trả kho memory ("thà nổ còn hơn hiện số giả"): `backend/payroll-service/src/services/PayrollService.js:60-66,199-204` + test `payslipPayoutFallback.test.js`, `bankAccountFallback.test.js`, `backend/work-service/tests/unit/work-postgres.test.js:39-139`.
   ⚠️ Nhưng còn 2 chỗ **vẫn** hiện số mẫu như số thật → xem mục 4, P3-2.
6. **"Bỏ ca" (missed) tính lúc đọc, không có cron**, và chỉ thông báo 1 lần/chi nhánh; `checkOut = null` là cách biểu diễn "không được tính công": `backend/work-service/src/services/WorkService.js:438-497` + `tests/unit/checkoutWindow.test.js:147-443`.
7. **Không có endpoint "chấm công hộ / bổ sung chấm công" riêng** — manager/admin chốt hộ qua chính `/api/attendance/checkout`, staff bị `assertSelfEmployee` ép về mình: `backend/work-service/src/controllers/WorkController.js:134-148`, `utils/caller.js:26-33`.
8. **Tuyến "hệ thống" để trống trong `authorize.js` không phải quên**: `PUT /api/tasks/:id`, `POST /api/attendance/*`, `DELETE /api/shifts/registrations/:id`, `PUT /api/employees/:id`, `POST /api/backlog` — service tự chặn theo chủ sở hữu/field whitelist, và thứ tự bảng luật có test khóa (`api-gateway/tests/authorize.test.js:60-64,122-127`).
9. **Slug chi nhánh do hệ thống sinh (`CN-0001`)**, client gửi slug vào sẽ bị bỏ qua (`organization-service/tests/integration/org.test.js:39-47`).
10. **`WorkClient` phía payroll tự gắn `x-user-role: admin` khi gọi nội bộ** — nếu bỏ thì phiếu lương âm thầm mất phần phạt (`payroll-service/src/clients/WorkClient.js:13-16`).
11. **Nút "Kiểm tra đường truyền" SOAP không chi tiền thật**: `POST /api/soap-config/ping` chỉ `fetch GET wsdlUrl` đo thời gian và trả 200 kể cả `status:"down"` (`integration-service/src/services/SoapConfigService.js:78-120`, test `soapPing.test.js`).
12. `staff_detail.dart:170` (mobile) hiện thẻ "Không có quyền truy cập" cho staff xem ca người khác — **có chủ đích**: danh sách đã khóa thẻ không bấm được (`schedule/general.dart:773-786`), màn chi tiết chỉ là chốt chặn cuối.

---

## 4. BUG nên sửa — xếp theo ưu tiên cho buổi bảo vệ

### P1 – Đứt luồng thật giữa web ↔ mobile (A sửa được ngay)

**P1-1. Task tạo từ web nhân viên không bao giờ thấy trên app** *(đã kiểm chứng tay)*
- Web: `assignedTo: newTask.assignedToName` → lưu **TÊN** (`frontend/src/app/dashboard/tasks/page.tsx:342`).
- Mobile: `assignedTo: assigneeId` → lưu **ID** (`mobile/lib/src/features/tasks/task_repository.dart:152`).
- BE scope staff: `scopeTaskQuery` ép `assignedTo == x-user-id` (`backend/work-service/src/utils/caller.js:41-43`) → khớp theo chuỗi chính xác ⇒ task từ web **vô hình** với staff; nếu staff bấm hoàn thành thì 403 (`WorkController.js:214-219`).
- Hệ quả khi demo: "giao việc trên web → mở app không thấy" là lỗi bị phát hiện ngay.
- Sửa: web gửi `employeeId` (id) thay vì tên; `assignedToName` để hiển thị. **A**, ~30 phút.
- Liên quan: task loại `shift` có `assignedTo = null` ⇒ **không nhân viên nào thấy được** (`WorkService.js:863`).

**P1-2. Dropdown chọn người nhận task trên web rỗng**
- `GET /api/employees` trả field `branchSlug`, **không có** `branch`, nhưng web lọc `e.branch` (`frontend/src/app/dashboard/tasks/page.tsx:471-473`) → dropdown trống với admin/manager đang lọc chi nhánh. (Trang nhân sự đọc đúng: `employees/page.tsx:30-33`.)

**P1-3. Mobile "Hủy ca" ở Lịch làm việc chung luôn fail** *(đã kiểm chứng tay phần id)*
- `staff_detail.dart:430` gọi `deleteShift(shift.id)` trong khi `shift.id` của `GeneralShiftModel` được dựng từ **template id** (`schedule/general.dart:296-306`) → `DELETE /api/shifts/<TL-…>` không tồn tại ca ⇒ 404, snackbar báo "Hủy ca thất bại".
- Cùng màn hình còn fallback nguy hiểm `employeeId = '1'` khi id rỗng (`staff_detail.dart:450`) và in lỗi thô `Text('... $e')` → user thấy `ApiException(FORBIDDEN): …` (`staff_detail.dart:443,466`).

**P1-4. Trang Phiếu lương web tải lần đầu hiện "không có dữ liệu"** *(đã kiểm chứng tay)*
- `selectedMonth` mặc định `"all"` (`frontend/src/app/dashboard/payslips/page.tsx:54`), còn `useEffect` chỉ kiểm truthy rồi nối `?month=all` (`:157`) → `PayrollService.js:318-324` lọc `where.month = "all"` → rỗng.
- Mâu thuẫn ngay trong 1 file: `fetchData` (`:120-126`) xử lý `"all"` đúng. Nút "Thử lại" làm bảng hiện dữ liệu ⇒ càng dễ tưởng DB lỗi.
- Sửa: dùng lại điều kiện `month !== "all"` ở `:157`.

### P2 – UI nói sai/khác hành vi backend (dễ bị hỏi khi demo)

**P2-1. "Xác thực Wi-Fi khi chấm công" chỉ là giao diện**
- Mobile gửi `wifiSsid`, `wifiBssid` (`attendance/attendance_repository.dart:16-22`) nhưng `WorkService.checkin` **bỏ qua toàn bộ** 2 trường (`backend/work-service/src/services/WorkService.js:689-712`; validator chỉ bắt `employeeId` `workValidator.js:19-23`).
- Check-in còn tin `checkinTime` do **đồng hồ điện thoại** gửi → máy sai giờ quyết định phạt muộn (`WorkService.js:692`).
- `help.dart:16-18` (mobile) mô tả điều BE không làm. → hoặc BE đối chiếu, hoặc sửa wording + nói rõ với giảng viên là "đã thiết kế field, chưa bật kiểm tra".

**P2-2. Lỗi mạng/403 bị biến thành "không có dữ liệu" ở nhiều trang web**
- Kiểu lỗi chung: `Promise.allSettled(...)` rồi bỏ qua nhánh lỗi ⇒ `catch` gán `error` **không bao giờ chạy**, banner "Thử lại" thành code chết. Gặp tại:
  `frontend/src/app/dashboard/requests/page.tsx:160,213-215,472`; `shifts/page.tsx:455-466,516-531`; `tasks/page.tsx:165-167,217-219`; `dashboard/page.tsx:37-40` (`.catch(()=>[])` → "0 nhân sự" im lặng).
- Trên mobile: nuốt lỗi im lặng ở `tasks/monitor.dart:109-110`, `salary/advance.dart:74`, `attendance/attendance_repository.dart:43-53` (`getConfig` trả `{}` với mọi lỗi → grace period/about-to-close mặc định 5/60), `profile/employee_repository.dart:29`, `tasks/tasks.dart:73-87` (dialog giao việc trống).

**P2-3. Nút/handle tồn tại nhưng không nối vào UI (web)**
- `shifts/page.tsx:670` `handleUpdateShift` và `:687` `handleAssignEmployeeToShift` **không được gọi từ đâu cả** → "sửa ca / gán người vào ca" không có trên web dù BE cho phép (chỉ có Xóa, `ShiftList.tsx:196`). Nút xóa template (`page.tsx:979`) cũng không có trong `ShiftTemplates.tsx:37-47`.
- `departments/DepartmentList.tsx:91` dùng `!isManager` làm **manager không mở được Chi tiết** ⇒ không sửa/khóa/xóa phòng ban được, trái `authorize.js:23` (manager được ghi departments) — đúng loại lỗi "ẩn nhầm theo phân quyền" bạn vừa yêu cầu sửa trên mobile.
- `employees/page.tsx:548-562` `handleConfirmDelete` + `ConfirmDialog` là nhánh chết (bản thật ở `EmployeeDeleteModal.tsx:636`).

**P2-4. Tiêu đề/nhãn hứa khác nội dung thật**
- `mobile/lib/src/features/salary/salary.dart:250-254`: tiêu đề manager "Bảng công & Lương chi nhánh" nhưng body vẫn lọc `employeeId = uid của mình` (`:96`) → manager chỉ thấy phiếu của chính mình.
- `mobile/lib/src/features/tasks/monitor.dart:130` "Nhân sự chi nhánh" nhưng attendance trả **toàn công ty** (xem P2-5), `BranchSelector` trên AppBar (`:141`) không ảnh hưởng fetch gì cả → thuần trang trí.
- `mobile/lib/src/features/salary/advance.dart:296` ghi "Hạn mức được ứng tối đa (50%)" nhưng **backend không kiểm 50%** (tạm ứng chỉ là `POST /api/requests` type `advance`, không đụng payroll) → chỉ là hint client-side (`advance.dart:159`).
- Web `requests/page.tsx:339-352` đếm badge trên mảng **đã lọc server theo status** → "Đã duyệt (0)" sai khi đang ở tab khác; `:53-54` bịa giờ "08:00→17:30" trong khi model Request không có cột from/to (`integration-service/prisma/schema.prisma:13-28`); dropdown loại đơn thiếu `overtime`/`other` (`:533-537`).
- `news` BE bỏ field `author` (`integration-service/src/controllers/NewsController.js:50`) → "Đăng bởi" biến mất sau refresh (cả web lẫn mobile).

**P2-5. Manager vượt_scope ở chiều ĐỌC/ĐỌC-ROI (nhiều nơi cùng mô típ)**
- `backend/payroll-service/src/utils/caller.js:19-23`: `scopePayslipQuery` **trả nguyên query cho manager** (có đọc `x-branch-slug` ở dòng 10 rồi bỏ trống) → manager xem phiếu lương toàn công ty; web chỉ ẩn bằng filter client (`frontend/src/app/dashboard/payslips/page.tsx:236-240`) và `!p.branch` giữ lại phiếu ngoài chi nhánh.
- `backend/work-service`: `createShift/updateShift/deleteShift/assignShift` **không nhận caller** (`WorkController.js:24-64`) → manager chi nhánh A sửa/xóa/gán ca chi nhánh B; id tuần tự `CA-0001` dễ dò. Chỉ `checkout` có guard chi nhánh (`WorkService.js:537-547`), `checkin` thì không.
- `backend/integration-service`: `GET/PUT/DELETE /api/requests/:id` không đối chiếu `x-branch-slug` (`RequestController.js:52-70,106-145,171-189`), `branchSlug` chỉ là bộ lọc → manager đọc/duyệt/xóa đơn chi nhánh khác theo id `YC-0001`.
- `GET /api/shifts/:id`, `/api/tasks/:id` không đối chiếu chủ/chi nhánh (`WorkController.js:244-262`) → đọc chéo theo id.
- `listRegistrations/listAttendance` thả privileged (`caller.js:36-52`) → manager nhận bảng công toàn công ty (gốc của P2-4).
- Mobile gửi `employeeId`/`shiftId` tự do **không phải IDOR** vì BE ghi đè theo header cho staff (`employeeAccess.js:44-52`, `caller.js:36-39`) — điểm này làm đúng.

### P3 – Nhỏ nhưng nên biết trước khi demo

- **P3-1.** `frontend/src/app/dashboard/branches/page.tsx:175`: guard `canAccessPath` đặt **sau** `useEffect` (`:69-98`) → manager/staff vẫn bắn 2 GET rồi mới hiện NoAccess (bank/backlog làm đúng: không fetch). Thêm: `AuthContext.tsx:54-63` role placeholder = `"staff"` khi `loading` ⇒ **admin thấy nháy "Không có quyền truy cập"** rồi mới vào trang.
- **P3-2.** "Số mẫu hiện như số thật" còn 2 chỗ: `SoapConfigService.js:29-38,66` `catch(_){}` rơi về `memoryConfig` với **allowedIPs bịa sẵn** (`:11-27`); `BacklogService.js:13-110` seed 5 bản ghi "Trần Minh Tuấn (Admin)" — sau `clear()` + restart sẽ seed lại, nhìn như nhật ký thật. (Trái với tinh thần fail-closed ở mục 3.5.)
- **P3-3.** `clearWebSessionCookies()` (`AuthContext.tsx:89-93`) **không có tác dụng**: `hrm-session`/`hrm-refresh` là `httpOnly` (`identity-service/src/controllers/AuthController.js:41-49`) → JS xóa không được; phải gọi `POST /api/auth/logout`. Kết hợp với việc BE không chặn staff ở `/api/auth/me` ⇒ staff vẫn còn cookie hợp lệ, và **BE hoàn toàn không chặn staff dùng API** (chỉ FE chặn) → curl/Postman vẫn gọi được.
- **P3-4.** `syncEmployee` map vai trò bằng chuỗi: `rawRole.includes("quản lý")` ⇒ chức danh "Nhân viên quản lý chất lượng" thành **manager** (`identity-service/src/services/AuthService.js:588-595`). Organization lưu 2 field (`role` = chức danh tự do, `systemRole` = enum) nên đây là đường biên dễ trượt.
- **P3-5.** `WorkService.js:1014` `updateTemplate` spread thẳng `payload` vào Prisma (nơi khác đều whitelist) ⇒ field lạ = 500; tương tự `WifiConfigController.js:67` (PUT) và `NewsController.js:80-83`.
- **P3-6.** Web `shifts/page.tsx:560-568` so **tên** chi nhánh với **slug** (`resolveBranchDisplay:41-46`) → manager có thể thấy danh sách rỗng. Mobile `shift_form.dart:358` + `schedule_service.dart:291` gửi `branchSlug` = **tên chi nhánh**. Hoa/thường `branchSlug` cũng là chỗ dễ trượt (`WorkService.js:184`).
- **P3-7.** Mobile `staff_detail.dart:443` (in `$e` thô), `monitor.dart:110` `check (_) {}`, `notifications.dart:17,132` + `notification_service.dart:127` dùng state tĩnh toàn app → **race khi đổi user** (fetch đang bay đè danh sách mới, chỉ clear `_knownIds`).
- **P3-8.** Refetch thiếu sau mutation: `assignment.dart:416-440` (`_toggleRecurring`), `tasks/page.tsx:327,361,605-620` (web), `shifts/page.tsx:671`, `departments` lock/delete local-only, `employees/page.tsx:533,553` (đếm staff BE đã tính lại nhưng UI stale).
- **P3-9.** `advance.dart:68` + `salary.dart:87`: không khớp tháng thì fallback `_apiPayslips.first` → hiện **sai kỳ** dưới nhãn kỳ đang chọn.
- **P3-10.** `PayrollService.js:246-260` read-then-update `balance` không lock ⇒ 2 lệnh chi song song có thể vượt số dư (chỉ admin chạm được). `:70` cho phép sửa `balance` tùy ý; `:77-80` unset mọi `isPrimary` trước khi update, nếu lỗi thì mất cả primary.

---

## 5. LEAK – bảng phân tích quyền ĐỌC (mục giảng viên hay hỏi)

| Dữ liệu | Ai không nên thấy | Thực tế | Bằng chứng |
|---|---|---|---|
| Phiếu lương mọi người | manager (theo chi nhánh) | manager đọc **toàn công ty** qua API; web/mobile chỉ lọc client | `payroll-service/src/utils/caller.js:19-23`; `frontend/.../payslips/page.tsx:236-240` |
| CCCD, số TK ngân hàng, lương gốc, SĐT, ngày sinh | staff; manager khác chi nhánh | staff: bị redact **đúng** (`SENSITIVE_FIELDS`); manager: `canSeeSensitive` với **mọi đồng nghiệp**, kể cả khác chi nhánh | `organization-service/src/utils/employeeAccess.js:7-12,29-42` |
| SSID + **BSSID** mọi chi nhánh | staff, manager khác CN | `GET /api/wifi-configs` **không có luật role/branch** → home mobile lấy hết | `integration-service/src/controllers/WifiConfigController.js:7`; `api-gateway/src/middlewares/authorize.js:37` (chỉ chặn WRITE); `mobile/.../home.dart:136` |
| Ảnh CCCD/avatar | người không liên quan | `GET /api/files/*` mở cho **mọi** role đã đăng nhập, không kiểm chủ sở hữu → lộ nếu biết UUID (UUID là barrier duy nhất) | `integration-service/src/controllers/FileController.js:10`; `authorize.js:56` (chặn POST/DELETE, không chặn GET) |
| Chạy lệnh OS trên server | mọi user | `GET /api/wifi-configs/scan` gọi `networksetup` (macOS) cho **bất kỳ** ai đăng nhập | `WifiConfigController.js:97-116` |
| Thông báo | người khác | `POST /api/notifications` **không luật role** → staff gửi đích danh tới bất kỳ ai, mạo `senderName/senderPhone`; `GET ?employeeId=` → manager đọc/xóa thông báo người khác. Visibility (staff không thấy bản ghi phạm-vi-chi-nhánh) là **chủ đích và có test** | `NotificationController.js:42-52,93-118`; `NotificationService.js:11-31`; test `notificationVisibility.test.js:39` |
| Danh tính trong nhật ký | — | `BacklogController.js:20` ghi `userName = x-user-email` ⇒ **email nằm trong log**; FE chỉ lấy `name/title/ssid` nên không lộ CCCD/số tiền | `frontend/src/lib/backlog.ts:79-139` |
| Token | — | Không có chỗ nào log token/password. `morgan("dev")` in **path kèm query** (`?employeeId=`, `?branchSlug=`) → PII nhẹ | `api-gateway/src/middlewares/logging.js`; `identity-service/src/utils/logger.js` (gần như code chết) |
| Header danh tính | client giả mạo | Gateway **xóa rồi ký lại** `x-user-*` ⇒ không spoof qua cổng (`api-gateway/src/app.js:44-55`). Nhưng 4 service sau cổng **tin header mù** → gọi thẳng port 4001-4005 (app chạy trên host, docker-compose chỉ expose Postgres) là **toàn quyền admin** | `organization-service/src/utils/employeeAccess.js:14-21`; `work|payroll-service/src/utils/caller.js:8-10`; `backend/docker-compose.yml:15-91` |
| Cookie vs Bearer | — | Gateway ưu tiên **cookie** (`fromCookie \|\| fromHeader`), mobile gửi **cả hai** (`api_client.dart:168-181`) ⇒ Bearer mới bị cookie cũ ghi đè. Refresh có cập nhật cookie (`:277`) nên thường tự khỏi, nhưng GET thì không capture → lệch hạn. | `api-gateway/src/middlewares/auth.js:24-27` |
| Mật khẩu demo | — | `login.dart:19-22` hardcode `admin@company.com/123456`; seed cùng mật khẩu (`AuthService.js:38-45`). Bình thường với đồ án, nhưng nhớ đổi khi nộp. | |

---

## 6. Cầu nối FE ↔ BE ↔ mobile: bảng hợp đồng

| Endpoint | Web | Mobile | Kết luận |
|---|---|---|---|
| `POST /api/auth/login` | cookie httpOnly, **bỏ qua** token trong body (`AuthContext.tsx:190`) | Bearer + refresh + `x-device-session-id` (`auth_repository.dart:55`) | `CỐ Ý` (2 mô hình phiên), xem leak cookie-first ở mục 5 |
| `GET /api/auth/me` | `!res.ok → null` (`AuthContext.tsx:103`) | `on ApiException → null` (`auth_repository.dart:136`) | `RỦI RO` – cả hai **nuốt 403/500** như "chưa đăng nhập" |
| `GET /api/tasks` | `?branchSlug=` (`tasks/page.tsx:134`) | **không query** (`tasks.dart:102`), filter client | `RỦI RO` – toàn bộ workload đổ về 1 request |
| `POST /api/tasks` | `assignedTo` = **TÊN** (`:342`) | `assignedTo` = **ID** (`task_repository.dart:152`) | **BUG P1-1** |
| `PUT /api/tasks/:id` | `{status, proofPhotoUrl, completedAt}` (`:295`) | thêm `photoSubmittedBy/At` (`:166`) | `RỦI RO` – whitelist staff bỏ 2 field cuối (`WorkController.js:222`) ⇒ ảnh minh chứng không ghi được người nộp |
| `GET /api/shifts` | `?branchSlug=UPPER` chỉ khi manager (`shifts/page.tsx:416-424`) | luôn gửi + `employeeId` (`shift_repository.dart:79`) | `RỦI RO` – so sánh hoa/thường (`WorkService.js:184`); mobile gửi `shift.id` = id khung `st-N` vs web gửi `CA-xxxx` (`general.dart:302` vs `ScheduleOverview.tsx:189-194`) → BE rơi vào fallback "ca của employeeId hôm nay" |
| `POST /api/attendance/checkin` | `{employeeId, shiftId}` (`shifts/page.tsx:710`) | + `checkinTime` (đồng hồ máy) + `wifiSsid/Bssid` (`attendance_repository.dart:16`) | `BUG P2-1` – BE không đọc 2 field Wi-Fi |
| `POST /api/shifts/register` | *(web không có luồng này)* | gửi 4 khóa trùng giá trị `week/preference/day/wish` (`shift_repository.dart:118-123`), BE chỉ đọc `week\|\|preference` (`WorkService.js:315`) | `CỐ Ý` – đọc dữ liệu legacy |
| `PUT /api/requests/:id/approve` | `{reviewedBy, reviewNote}` (`requests/page.tsx:363`) | chỉ `{reviewNote}` (`approval_repository.dart:20`) | `RỦI RO` – BE default `reviewedBy = "admin"` (`RequestService.js:247`) ⇒ **ai cũng bị ghi là admin** |
| `GET /api/payroll/payslips` | `?month=`, **không** `employeeId` (`payslips/page.tsx:126`) | luôn `employeeId: uid` (`salary.dart:96`) | **BUG P2-4/P3-9** + leak manager (mục 5) |
| `GET /api/wifi-configs` | không `branch` (`wifi/page.tsx:34`) | `?branch=UPPER` (`wifi_config_repository.dart:46`) | `RỦI RO` – web nhận **mọi** chi nhánh, không lọc BE |
| `GET /api/notifications` | không `employeeId` (`Topbar.tsx:42`) | có `employeeId` (`notification_repository.dart:10`) | `CỐ Ý` – BE tự scope, vô hại |
| Envelope | unwrap `{data}` (`api.ts:38`) | unwrap `{data}` (`api_client.dart:288`) | `OK` – chỉ mobile unwrap trùng 3 nơi (`attendance_repository.dart:45`) |
| Date | DD/MM/YYYY khi ghi, BE chuẩn hóa lúc ghi và **lưu chuỗi** (`OrgService.js:721-723`, `WorkService.js:43-45`) | mobile gửi DD/MM/YYYY (`date_format.dart:12`) | `OK` nhưng `dateFormat.js` bị **copy 2 bản giống hệt** (org + work) → dễ lệch về sau |
| Phân trang | không page | không page | `CỐ Ý` cho demo; `take:500` cứng ở backlog (`BacklogService.js:121`) |

**Kết luận cầu nối:** "nối" đúng về khung (envelope, lỗi, date, envelope unwrap) nhưng **sai về hệ quy chiếu dữ liệu** (tên↔id, hoa↔thường, `reviewedBy`, `checkinTime`) và **lỏng về quyền đọc** — đây chính là 2 loại lỗi nên chủ động nói trong buổi bảo vệ thay để giảng viên tìm ra.

---

## 7. Test: cái gì xanh, cái gì đỏ, cái gì chưa có

**Đã chạy và XANH:** api-gateway `107/107` · organization unit `16` · payroll unit `19` · integration-service unit `35` · identity unit `4` · mobile `flutter test` **118 pass / 1 skip** · `flutter analyze` **0 issue**.
**ĐỎ:** work-service unit `2 fail` — `backend/work-service/tests/unit/work.test.js:16-28` vẫn giả định **memory fallback** trong khi code hiện tại **ném 500** khi DB lỗi ⇒ test tự mâu thuẫn với chính sách mới (`checkoutWindow.test.js:11`, `work-postgres.test.js:39-139`). **Việc của B.**
**KHÔNG chạy được (môi trường, không phải code):** backend integration + archive (cần Postgres 5433-5437 / docker), `frontend` vitest (chưa cài trong `node_modules`), `mobile/test/api_login_test.dart:18` tự skip khi gateway chết.
**Mâu thuẫn phụ:** `checkoutWindow.test.js:351-372` gọi `WorkService.checkin` thật trước khi stub ⇒ phụ thuộc Postgres dev; integration tests dùng seed chung `admin/123456` (`account.test.js:28`).

**Ý đồ mà test ĐÃ khóa:** ma trận role×route (`authorize.test.js:34-160`), staff self-service, manager chặn branches/wifi/soap/bank/backlog/archive, refresh-token không dùng làm access (`auth.test.js:65`), forgot-password không reset (`account.test.js:101-132`), checkout window + "bỏ ca" + phạt (`checkoutWindow.test.js:147-443`, `latePenalty.test.js`), staff không đọc phiếu lương người khác (`integration/payroll.test.js:101-107`), notification visibility, shift-swap hoàn tác, slug tự sinh, fail-closed DB.
**Ý đồ CHƯA có test nào khóa** (chỉ nằm trong comment):
- Web Wi-Fi chỉ xem + xóa (mục 3.1).
- "Staff chỉ dùng mobile" — test authorize vẫn **cho** staff ghi nhiều tuyến mà web chặn FE.
- Bảng quyền mobile mới (`mobile/test/permissions_test.dart` + `route_role_guard_test.dart`) — tối nay tôi đã thêm, nhưng chỉ phủ 4 route.

**Độ phủ UI (để bạn cân nhắc khi bị hỏi "test ở đâu"):**
- Web: chỉ `bank`, `branches`, `departments` có component test (render tĩnh). **Không có UI test:** dashboard, employees, shifts, tasks, requests, news, regulations, wifi, backlog, payslips.
- Mobile: có test cho splash/home/session/permissions/repositories/wifi-hardware/missed-checkout. **Không có test:** login form, `schedule/*` (general, registration, assignment, shift_detail, shift_form), `attendance/adjustment`, `salary` + `advance`, `tasks` UI, `monitor`, `approvals`, `news`/`regulations`/`notifications`/`request_detail`, UI `wifi`, **toàn bộ `profile/*`**, `help`.

---

## 8. Đề xuất thứ tự làm trước khi bảo vệ (chỉ phần của A)

1. **P1-1 + P1-2** (task `assignedTo` id + dropdown `branchSlug`) – ~30–45 phút, sửa được ngay lỗi demo "giao việc không thấy trên app".
2. **P1-4** (`month=all` phiếu lương) – 1 dòng.
3. **P1-3** (mobile hủy ca dùng template id) + bỏ `employeeId = '1'` + in message `ApiException.message` thay `$e` – ~30 phút.
4. **P2-2** (banner lỗi thật thay "không có dữ liệu") ở `requests`, `shifts`, `tasks`, `dashboard` + `monitor.dart`/`advance.dart` mobile – ~45 phút, đây là lỗi gây hiểu lầm nhiều nhất khi demo có sự cố mạng.
5. **P3-1** (guard web đặt trước `useEffect`; hết nháy "Không có quyền" cho admin) – ~20 phút.
6. **P2-1** (Wi-Fi check-in): quyết định 1 trong 2 — BE đối chiếu SSID/BSSID (thật) **hoặc** sửa wording `help.dart` + nói rõ "field đã chuẩn bị, kiểm tra ở tầng BE chưa bật". Đừng để 2 tài liệu/2 UI nói khác nhau.
7. Siết quyền đọc nếu còn thời gian: `scopePayslipQuery` ghim `branchSlug` cho manager (D), luật gateway cho `POST /api/notifications` + `GET /api/wifi-configs/scan` (E/A), `GET /api/files/:id` kiểm chủ (E).
8. Nhắc B sửa `work.test.js` (đỏ) và `DepartmentList.tsx:91`/`handleUpdateShift` (P2-3) — theo `docs/AGENTS.md` §2.3 thì **A không tự sửa service của B**, chỉ ghi lại (đã ghi ở mục 4).
9. Nếu kịp: thêm 2 test khóa ý đồ ở mục 7 (web Wi-Fi view+delete; bảng quyền mobile cho `/salary`, `/tasks`) để phần "test" không bị hỏi.

**Nên nói thẳng với giảng viên (biến điểm yếu thành thiết kế có ý đồ):** memory store cho một số domain + id tuần tự, SOAP ghi nợ trên "ngân hàng mô phỏng" trong DB nội bộ, không phân trang, `forgot-password` no-op, chặn staff ở web là quyết định UX (staff chỉ dùng app), FE là cửa chặn staff chứ BE chưa chặn.

---

## 9. Cách tái kiểm (lệnh thật)

```bash
# Mobile (đã chạy, xanh)
cd mobile && flutter analyze && flutter test

# Backend unit tests (xanh, trừ work-service)
cd backend/api-gateway && npx jest
cd backend/organization-service && npx jest tests/unit
cd backend/work-service && npx jest tests/unit            # 2 fail - xem P3/mục 7
cd backend/payroll-service && npx jest tests/unit
cd backend/integration-service && npx jest tests/unit
cd backend/identity-service && npx jest tests/unit

# Backend integration + frontend: CẦN Postgres (5433-5437) + service chạy + `npm i` cho frontend (vitest chưa có trong node_modules)
cd frontend && npx tsc --noEmit
```

## 10. Ghi chú kỹ thuật của lần audit này

- 14 sub-agent đọc code, mỗi agent 1 cụm tính năng; 4 phát hiện nặng nhất (P1-1, P1-3, P1-4, cookie-first) **tôi đã kiểm lại tay** bằng cách đọc trực tiếp file. Các đường dẫn `file:line` còn lại do agent trích — trước khi sửa, mở đúng dòng để xác nhận.
- Không sửa file nào ngoài `mobile/` (4 file tính năng + 3 file test) và file báo cáo này.
- Không có `.git` trong thư mục dự án (`fatal: not a git repository`) nên **không có bản backup tự động** cho các sửa đổi tối nay.
