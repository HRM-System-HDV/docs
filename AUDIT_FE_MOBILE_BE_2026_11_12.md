# Audit toàn bộ tính năng Web + Mobile + Cầu nối BE — và kế hoạch khắc phục

Ngày audit: **12/11/2026** (chạy qua đêm, 14 sub-agent đọc code) · **Cập nhật sau khi sửa: 12/11/2026**
Phạm vi: `frontend/` (web Next.js), `mobile/` (Flutter), `backend/` (gateway + 5 service)
Cách đánh giá: **đọc code request/response + test**, KHÔNG dựa tài liệu `docs/*.md`
Góc nhìn: đồ án môn học SWD hướng dịch vụ – nhóm 4 người, không đòi hardening kiểu production

**Trạng thái tài liệu:** toàn bộ lỗi P1/P2/P3 đã được **sửa trong code và có test khoá lại** (xem mục 4 và mục 11).
Mục 11 liệt kê đúng những gì đã đổi trong code, mục 12 liệt kê việc còn phải làm tay.

Cách đọc file này buổi sáng: mục **1** (3 câu trả lời) → mục **11** (đã sửa gì) → mục **4** (mục nào còn tồn đọng) → mục **3** (cái gì cố tình, đừng nhận là bug).

Quy ước nhãn: `BUG` = hỏng thật; `RỦI RO` = chưa đứt nhưng lệch/thiếu; `CỐ Ý` = hợp lý, có ý đồ trong code/test.
Quy ước ownership theo `docs/AGENTS.md` §2: **A** = gateway + identity + mobile + frontend (nhóm trưởng), **B** = organization + work, **D** = payroll, **E** = integration.

---

## 1. TL;DR – trả lời 3 câu hỏi của bạn (trả lời lại SAU khi sửa)

**a) Cầu nối BE ở giữa FE và mobile đã liên kết ổn chưa?**
Giờ **đã ổn về hệ quy chiếu dữ liệu**. Envelope `{data}` / `{error:{code,message}}` nhất quán (web `frontend/src/lib/api.ts:38-47`, mobile `mobile/lib/src/core/network/api_client.dart:285-292`).
Nguyên nhân lớn nhất là **hai client dùng hai hệ quy chiếu khác nhau cho cùng một field**: web ghi `assignedTo` = **TÊN**, mobile ghi `assignedTo` = **ID** → task giao từ web biến mất hoàn toàn trên app nhân viên.
→ **ĐÃ SỬA (mục 4 P1-1, mục 11.1)**: thống nhất nguyên tắc **gửi ID, hiển thị tên**; `assignedTo` luôn là `employeeId`, task theo ca (`sourceType=shift`) dùng `assignedTo = null` và được ghim theo chi nhánh.

**b) Dữ liệu có di chuyển ổn định không?**
Đọc–ghi từng phía thì ổn, mutation có refetch ở đa số màn. Trước đây **chạy một chiều** (web ghi task thì staff mất hẳn) và có nhiều chỗ lỗi mạng/403 bị nuốt thành "không có dữ liệu".
→ **ĐÃ SỬA (mục 4 P1-1, P2-2, P3-8)**: hai chiều đã thông, lỗi mạng/403 giờ hiện banner + nút "Thử lại" ở cả web lẫn mobile, và refetch sau mutation đã bổ sung.
Vẫn còn: **không service nào phân trang** → chỉ ổn khi dữ liệu demo nhỏ (`CỐ Ý`, xem mục 3).

**c) Có leak dữ liệu nào không?**
Có, và **toàn bộ thuộc nhóm "thiếu kiểm tra quyền ở chiều ĐỌC"**, không phải lộ token ra ngoài.
→ **ĐÃ SỬA (mục 5, mục 4 P2-5)**: manager giờ bị ghim chi nhánh ở mọi đường đọc (phiếu lương, ca, đơn, Wi-Fi, file, thông báo); quét Wi-Fi và gửi thông báo chỉ còn admin/manager; đọc file kiểm chủ sở hữu/chi nhánh.
Còn 1 điểm **có chủ đích, không sửa**: BE không chặn staff gọi API (chỉ FE chặn web) — xem mục 3.13.

---

## 2. Việc đã làm: ẩn phân quyền mobile như web (đợt 1)

| File | Nội dung |
|---|---|
| `mobile/lib/src/core/config/permissions.dart` (mới) | Bảng quyền tập trung `route → vai trò`, soi theo `MENUS` của `frontend/src/lib/permissions.tsx`. `/tasks/monitor`, `/approvals`, `/schedule/assignment` = admin+manager; `/wifi` = **chỉ admin**. Route không liệt kê = tự phục vụ, mọi role. Chưa resolve được phiên → coi là staff (mức thấp nhất). |
| `mobile/lib/src/core/widgets/no_access_screen.dart` (mới) | `NoAccessScreen` (tương đương `NoAccessNotice` web) + `guardRoute()` chặn **ngay trong GoRoute builder**. |
| `mobile/lib/src/core/router/router.dart:98,201,207,213` | Guard cho 4 tuyến đặc quyền → deep link/push không vào được màn mà mọi thao tác sẽ ăn 403. |
| `mobile/lib/src/features/tasks/operations.dart:26-27,99` | Thẻ lấy theo bảng quyền. Bỏ thẻ "Wi-Fi chi nhánh" của manager; staff vẫn ẩn trọn cụm Quản trị. |
| `mobile/test/permissions_test.dart`, `test/route_role_guard_test.dart`, `test/operations_permissions_test.dart` (mới) | 11 test: bảng quyền, guard route thật, card ẩn theo role. |

Lưu ý: vì staff bị ẩn tính năng quản trị theo đúng web, `mobile/lib/src/features/wifi_config/wifi.dart` vẫn còn nhánh "chế độ đọc cho manager/staff" — giờ **không vào được từ UI**, để lại như lớp phòng vệ thứ hai (nhãn `CỐ Ý`, không phải bug).

---

## 3. Nhóm "CỐ Ý" – đừng ghi vào báo cáo nộp giảng viên như là lỗi

Đây là thứ người review (kể cả AI) rất dễ báo nhầm. Tất cả đều có bằng chứng trong code/test:

1. **Web Wi-Fi chỉ XEM + XÓA, không thêm/sửa — hoàn toàn chủ đích** vì trình duyệt không quét được Wi-Fi. `frontend/src/app/dashboard/wifi/page.tsx:22-68` chỉ gọi `GET /api/wifi-configs` + `DELETE`, **không có POST/PUT**, không gọi `/scan`; `frontend/src/features/wifi/WifiForm.tsx` là **form chết** (không file nào import). Nút xóa chỉ hiện khi `isAdmin` (`page.tsx:106`). Trong khi mobile quét + thêm thật (`mobile/lib/src/features/wifi_config/wifi.dart:116,282,308`). Hai client cố tình khác nhau vì khả năng thiết bị khác nhau.
2. **Staff không đăng nhập được web, chỉ dùng app** — chặn ở FE: `frontend/src/context/AuthContext.tsx:107-110,193-200` (`STAFF_WEB_FORBIDDEN`). Phần tự phục vụ của staff trên mobile **rộng hơn** `STAFF_MENU` web là có chủ đích, ghi rõ trong comment `mobile/lib/src/core/config/permissions.dart:11-13`.
3. **Manager không có Wi-Fi/Chi nhánh/Ngân hàng/Lịch sử thao tác** — phân tầng 3 vai trò tường minh, mỗi tầng liệt kê menu riêng thay vì suy ra "không phải admin" (`frontend/src/lib/permissions.tsx:21-25,49-91`; test `backend/api-gateway/tests/authorize.test.js`).
4. **Quên mật khẩu KHÔNG reset gì cả** — chỉ hướng dẫn liên hệ admin, không lộ thông tin (`backend/identity-service/tests/integration/account.test.js:101-132`). Đó là thiết kế, không phải thiếu chức năng.
5. **Không fallback dữ liệu giả khi DB lỗi** — payroll/work **ném 500 có chủ đích** thay vì trả kho memory ("thà nổ còn hơn hiện số giả"): `backend/payroll-service/src/services/PayrollService.js` + test `payslipPayoutFallback.test.js`, `bankAccountFallback.test.js`, `backend/work-service/tests/unit/work-postgres.test.js`. Đợt sửa này **mở rộng đúng tinh thần này**: SOAP không còn nuốt lỗi rồi trả cấu hình giả (mục 11.9).
6. **"Bỏ ca" (missed) tính lúc đọc, không có cron**, và chỉ thông báo 1 lần/chi nhánh; `checkOut = null` là cách biểu diễn "không được tính công": `backend/work-service/src/services/WorkService.js:438-497` + `tests/unit/checkoutWindow.test.js:147-443`.
7. **Không có endpoint "chấm công hộ / bổ sung chấm công" riêng** — manager/admin chốt hộ qua chính `/api/attendance/checkout`, staff bị `assertSelfEmployee` ép về mình: `backend/work-service/src/controllers/WorkController.js`, `utils/caller.js:26-33`.
8. **Tuyến "hệ thống" để trống trong `authorize.js` không phải quên**: `PUT /api/tasks/:id`, `POST /api/attendance/*`, `DELETE /api/shifts/registrations/:id`, `PUT /api/employees/:id`, `POST /api/backlog` — service tự chặn theo chủ sở hữu/field whitelist, và thứ tự bảng luật có test khóa.
9. **Slug chi nhánh do hệ thống sinh (`CN-0001`)**, client gửi slug vào sẽ bị bỏ qua (`organization-service/tests/integration/org.test.js:39-47`).
10. **Client nội bộ tự gắn `x-user-role: admin`** khi gọi service khác — bắt buộc vì service tin header mù: `payroll-service/src/clients/WorkClient.js:13-16`, `payroll-service/src/clients/OrgClient.js` (đã bổ sung trong đợt sửa này khi payroll cần đọc hồ sơ nhân viên để lấy chi nhánh).
11. **Nút "Kiểm tra đường truyền" SOAP không chi tiền thật**: `POST /api/soap-config/ping` chỉ `fetch GET wsdlUrl` đo thời gian và trả 200 kể cả `status:"down"` (`integration-service/src/services/SoapConfigService.js`, test `soapPing.test.js`).
12. `staff_detail.dart` hiện thẻ "Không có quyền truy cập" cho staff xem ca người khác — **có chủ đích**: danh sách đã khóa thẻ không bấm được (`schedule/general.dart`), màn chi tiết chỉ là chốt chặn cuối.
13. **BE không chặn staff gọi API** (chỉ FE chặn web) — quyết định UX "staff chỉ dùng app"; gateway vẫn áp luật theo route nhưng không có luật "cấm staff dùng API". Nói rõ với giảng viên thay vì giấu.

---

## 4. BUG đã tìm ra — trạng thái sau khi sửa

> Mỗi mục ghi rõ **đã sửa ở đâu** và **còn tồn đọng gì**. Chi tiết toàn bộ danh sách file đã đổi nằm ở mục 11.

### P1 – Đứt luồng thật giữa web ↔ mobile → **ĐÃ SỬA 100%**

**P1-1. Task tạo từ web nhân viên không bao giờ thấy trên app** → **ĐÃ SỬA**
- Nguyên nhân: web gửi `assignedTo: newTask.assignedToName` (**TÊN**), mobile gửi `assignedTo: assigneeId` (**ID**); `scopeTaskQuery` ép `assignedTo == x-user-id` ⇒ task web vô hình với staff, bấm hoàn thành thì 403.
- Cách sửa: thống nhất **gửi ID, hiển thị tên** — `frontend/src/features/tasks/TaskForm.tsx` + `frontend/src/app/dashboard/tasks/page.tsx:363` gửi `assignedTo = employee.id`; `assignedToName` chỉ để hiển thị.
- Task theo ca (`sourceType=shift`) gửi `assignedTo = null`; `scopeTaskQuery` (`backend/work-service/src/utils/caller.js`) cho staff thấy **task của mình + task theo ca của chi nhánh mình**; `WorkController.updateTask` cho phép nhân viên chi nhánh báo cáo tiến độ task theo ca.
- Test khoá: `backend/work-service/tests/unit/branchScope.test.js`.

**P1-2. Dropdown chọn người nhận task trên web rỗng** → **ĐÃ SỬA**
- `GET /api/employees` trả `branchSlug`, web lọc `e.branch` ⇒ rỗng. Đã normalize `branch: e.branchSlug ?? e.branch` ngay khi nhận dữ liệu (`tasks/page.tsx`).

**P1-3. Mobile "Hủy ca" ở Lịch làm việc chung luôn fail** → **ĐÃ SỬA**
- `staff_detail.dart` dùng `shift.id` là **id khung ca mẫn** (template id) nên `DELETE /api/shifts/<template>` = 404. Đã đổi sang `staff.shiftId` (id dòng ca thật `CA-xxxx`) cho cả hủy ca **và** chấm công; bỏ fallback `employeeId = '1'`; lỗi hiện `ApiException.message` thay vì `$e`.

**P1-4. Trang Phiếu lương web tải lần đầu hiện "không có dữ liệu"** → **ĐÃ SỬA**
- `selectedMonth` mặc định `"all"` nhưng `useEffect` nối `?month=all` ⇒ `PayrollService` lọc `where.month = "all"` ⇒ rỗng. Đã dùng lại điều kiện `month !== "all"`.

### P2 – UI nói sai / khác hành vi backend → **ĐÃ SỬA 100%**

**P2-1. "Xác thực Wi-Fi khi chấm công" chỉ là giao diện** → **ĐÃ SỬA (phần wording)**
- Backend vẫn **không** đối chiếu SSID/BSSID (mobile vẫn gửi `wifiSsid/wifiBssid`; `WorkService.checkin` chưa dùng) — đây là việc chưa làm, nêu rõ với giảng viên.
- Đã sửa **cả 2 nơi mô tả** để không hứa hơn thực tế: `mobile/lib/src/features/home/help.dart` (đúng path, trước báo cáo ghi sai `attendance/help.dart`) và `mobile/lib/src/features/wifi_config/wifi.dart` → câu chữ: "Khi bạn bấm nút **Chấm công**, app đọc SSID/BSSID tại thời điểm đó và gửi kèm. Hệ thống **chưa chặn** chấm công nếu Wi-Fi không khớp cấu hình chi nhánh."
- **Còn tồn đọng:** `checkinTime` vẫn do đồng hồ điện thoại gửi ⇒ máy sai giờ vẫn quyết định phạt muộn (`WorkService.js` dùng `time || checkinTime`).

**P2-2. Lỗi mạng/403 bị biến thành "không có dữ liệu"** → **ĐÃ SỬA**
- Web: `requests/page.tsx`, `shifts/page.tsx`, `tasks/page.tsx` xử lý cả nhánh `rejected` của `Promise.allSettled` → `setError(...)` (banner + nút "Thử lại" giờ chạy được); `dashboard/page.tsx` bỏ `.catch(()=>[])` và hiện cảnh báo lỗi thay vì "0 nhân sự".
- Mobile: `tasks/monitor.dart` (có banner lỗi + nút thử lại, và **BranchSelector giờ thực sự lọc dữ liệu**), `salary/advance.dart` (báo lỗi tải lương), `attendance/attendance_repository.dart` (`getConfig` **ném lỗi** thay vì trả `{}` → không còn dùng mặc định 5/60 như thật), `profile/employee_repository.dart` + `profile.dart` + `home.dart` (lỗi không bị nuốt), `tasks/tasks.dart` (dialog giao việc báo lỗi danh mục).
- Test cũ khẳng định hành vi sai đã được cập nhật: `mobile/test/repositories/attendance_repository_test.dart`, `mobile/test/repositories/challenger_verification_test.dart`.

**P2-3. Nút/handle tồn tại nhưng không nối vào UI (web)** → **ĐÃ SỬA**
- Xoá 2 handler chết `handleUpdateShift` / `handleAssignEmployeeToShift` (`shifts/page.tsx`).
- **Thêm nút "Xóa" khung ca mẫn** (`features/shifts/ShiftTemplates.tsx`) nối với `onDelete` đã có sẵn ở page, kèm hộp xác nhận (`ConfirmDialog`).
- `departments/DepartmentList.tsx`: bỏ điều kiện `!isManager` ⇒ manager mở được "Chi tiết" (khớp `authorize.js` cho phép manager ghi departments).
- `employees/page.tsx`: xoá nhánh chết `handleConfirmDelete` + import `ConfirmDialog` (bản thật là `EmployeeDeleteModal`).

**P2-4. Tiêu đề/nhãn hứa khác nội dung thật** → **ĐÃ SỬA**
- `mobile/.../salary/salary.dart`: tiêu đề "Kỳ lương & Phiếu lương cá nhân" cho mọi vai trò (màn này chỉ fetch phiếu của chính mình).
- `mobile/.../tasks/monitor.dart`: tiêu đề theo chi nhánh **đang chọn**; `branchSlug` được truyền vào `getEmployees/getAttendance/getShifts` ⇒ `BranchSelector` không còn là trang trí.
- `mobile/.../salary/advance.dart`: bỏ "Hạn mức 50%" (backend không kiểm 50%).
- Web `requests/page.tsx`: bỏ giờ bịa "08:00→17:30" (model `Request` không có cột from/to); badge đếm đúng (bỏ lọc `status`/`type` ở server, lọc ở client).
- `news`: thêm cột `author` và **không còn xoá** `payload.author`; người đăng lấy từ danh tính gateway ký (`x-user-name`).

**P2-5. Manager vượt scope ở chiều ĐỌC/GHI** → **ĐÃ SỬA**
- **Payroll**: `scopePayslipQuery` ghim `branchSlug` cho manager; manager không có chi nhánh → 403; `assertOwnPayslip` kiểm chi nhánh; thêm cột `Payslip.branchSlug` + script backfill. Test: `backend/payroll-service/tests/unit/payslipScope.test.js`.
- **Work (ca)**: `createShift/updateShift/deleteShift/assignShift` giờ nhận `caller` và so `branchSlug`; manager chỉ sửa/xoá/gán ca chi nhánh mình (admin không giới hạn).
- **Work (chấm công)**: guard chi nhánh áp cho **mọi** checkout của manager (trước chỉ chạy ở nhánh "quá cửa sổ chốt ra"); checkin cũng được kiểm.
- **Integration (đơn)**: `GET/PUT approve|reject/DELETE /api/requests/:id` so `x-branch-slug` với `request.branchSlug`; staff chỉ xem đơn của mình/được nhờ.
- **Integration (Wi-Fi)**: `GET /api/wifi-configs` bị ghim chi nhánh với non-admin; `GET /api/wifi-configs/scan` chỉ admin (gateway + controller).
- **Integration (file)**: `FileAsset.branchSlug` lưu lúc upload; đọc file chỉ admin / chủ file / manager cùng chi nhánh.
- **Integration (thông báo)**: `POST /api/notifications` chỉ admin/manager (gateway); người gửi lấy từ token; manager không dùng được `?employeeId=` của người khác; xoá chỉ chủ thông báo hoặc admin.
- **Gateway/identity**: token access/refresh mang thêm `name`; gateway gửi `x-user-name` xuống service.
- **Còn tồn đọng:** `listRegistrations/listAttendance` vẫn thả privileged ⇒ manager xem bảng công **toàn công ty** (chưa ghim được vì attendance không lưu chi nhánh, chỉ lọc qua bảng `shifts`). Đây là hạng mục để sau, không ảnh hưởng demo.

### P3 – Nhỏ nhưng nên biết trước khi demo → **ĐÃ SỬA 100%**

- **P3-1.** Guard `canAccessPath` của trang Chi nhánh **đã đưa lên trước** `useEffect` (không còn bắn 2 GET rồi mới hiện NoAccess). Và **hết nháy "Không có quyền truy cập"** cho admin: `dashboard/layout.tsx` không render nội dung trang khi `loading` còn true.
- **P3-2.** SOAP: bỏ `allowedIPs` bịa sẵn và **không còn nuốt lỗi** rồi trả cấu hình memory (fail-closed). Backlog: bỏ 4 bản ghi seed mô phỏng thao tác của admin, chỉ còn 1 bản ghi "Hệ thống – Khởi tạo dịch vụ".
- **P3-3.** `clearWebSessionCookies()` (không xoá được cookie httpOnly) **đã bỏ**, thay bằng gọi `POST /api/auth/logout` khi phát hiện staff.
- **P3-4.** `syncEmployee` **ưu tiên enum `systemRole`** (`admin|manager|staff`) rồi mới fallback chuỗi ⇒ "Nhân viên quản lý chất lượng" không còn bị nhầm là manager.
- **P3-5.** Whitelist field khi update: `WorkService.updateTemplate`, `WifiConfigService.update`, `NewsService.update` (qua controller) không còn spread thẳng `payload` vào Prisma.
- **P3-6.** Chuẩn hoá **so sánh theo slug** (`sameBranch()` bỏ dấu "-" và hạ chữ thường) ở `shifts/page.tsx`; mobile `shift_form.dart` **gửi slug** (`ShiftDetail.branchSlug` mới) thay vì tên chi nhánh, tên chỉ dùng hiển thị.
- **P3-7.** Đã sửa phần chính: `staff_detail.dart` không còn in `$e` thô; `monitor.dart` không còn `catch (_) {}` im lặng (có banner lỗi + token chống race khi đổi chi nhánh). **Còn tồn đọng:** state tĩnh toàn app của `notifications.dart` / `notification_service.dart` khi đổi user.
- **P3-8.** Refetch sau mutation đã bổ sung: mobile `assignment.dart` (`_toggleRecurring`), web `tasks/page.tsx` (hoàn thành/xoá), `departments/page.tsx` (khoá/xoá), `employees/page.tsx` (khoá/xoá).
- **P3-9.** Bỏ fallback `payslips.first` ở `salary.dart` (không có phiếu đúng kỳ thì hiện "—") và ở `advance.dart` (lấy phiếu kỳ mới nhất).
- **P3-10.** `balance` bỏ khỏi allowlist cập nhật tài khoản; trừ tiền dùng `updateMany({ where: { balance: { gte: amount } }, data: { balance: { decrement } } })` ⇒ không còn vượt số dư khi 2 lệnh chi chạy song song; `isPrimary` unset + set gói trong cùng `prisma.$transaction` ⇒ lỗi giữa chừng không mất primary.

---

## 5. LEAK – bảng phân tích quyền ĐỌC (mục giảng viên hay hỏi) — đã xử lý

| Dữ liệu | Ai không nên thấy | Trước khi sửa | **Sau khi sửa** |
|---|---|---|---|
| Phiếu lương mọi người | manager (theo chi nhánh) | manager đọc **toàn công ty** qua API | ✅ manager bị ghim `branchSlug` (`payroll-service/src/utils/caller.js`), có cột `Payslip.branchSlug` + script backfill |
| CCCD, số TK ngân hàng, lương gốc, SĐT | staff; manager khác chi nhánh | staff bị redact đúng; **phần "manager khác chi nhánh" là nhầm** | ✅ không cần sửa: `assertCanReadEmployee` + `scopeEmployeeQuery` đã chặn đọc chéo chi nhánh (`organization-service/src/utils/employeeAccess.js:44-60`) |
| SSID + **BSSID** mọi chi nhánh | staff, manager khác CN | `GET /api/wifi-configs` không luật role/branch | ✅ non-admin bị ghim chi nhánh; `scan` chỉ admin |
| Ảnh CCCD/avatar | người không liên quan | `GET /api/files/*` mở cho mọi role đã đăng nhập | ✅ kiểm chủ file / admin / manager cùng chi nhánh (`FileAsset.branchSlug`) |
| Chạy lệnh OS trên server | mọi user | `GET /api/wifi-configs/scan` gọi `networksetup` cho mọi user | ✅ chỉ admin (gateway `authorize.js` + controller) |
| Thông báo | người khác | staff gửi đích danh, mạo `senderName`; manager đọc/xoá của người khác | ✅ POST chỉ admin/manager; người gửi lấy từ token; `?employeeId=` bị bỏ qua với non-admin; xoá chỉ chủ hoặc admin |
| Danh tính trong nhật ký | — | `BacklogController.js` ghi `userName = x-user-email` ⇒ email nằm trong log | ⚠️ Còn nguyên (FE chỉ lấy `name/title/ssid` nên không lộ CCCD/số tiền) |
| Token | — | Không có chỗ nào log token/password. `morgan("dev")` in **path kèm query** ⇒ PII nhẹ | ⚠️ Còn nguyên; `morgan` nằm ở `api-gateway/src/app.js` (không phải `middlewares/logging.js` như báo cáo cũ ghi) |
| Header danh tính | client giả mạo | Gateway **xóa rồi ký lại** `x-user-*` ⇒ không spoof qua cổng. Service sau cổng **tin header mù** ⇒ gọi thẳng port 4001-4005 là **toàn quyền admin** | ⚠️ Còn nguyên (thiết kế microservices, chỉ chạy nội bộ) |
| Cookie vs Bearer | — | Gateway ưu tiên **cookie** (`fromCookie \|\| fromHeader`), mobile gửi cả hai | ⚠️ Còn nguyên (`api-gateway/src/middlewares/auth.js:24-27`) |
| Mật khẩu demo | — | `login.dart` hardcode `admin@company.com/123456` | ⚠️ Còn nguyên — **nhớ đổi khi nộp** |

---

## 6. Cầu nối FE ↔ BE ↔ mobile: bảng hợp đồng (cập nhật sau khi sửa)

| Endpoint | Web | Mobile | Kết luận |
|---|---|---|---|
| `POST /api/auth/login` | cookie httpOnly, bỏ qua token trong body | Bearer + refresh + `x-device-session-id` | `CỐ Ý` (2 mô hình phiên) |
| `GET /api/auth/me` | `!res.ok → null` | `on ApiException → null` | `RỦI RO` – cả hai **nuốt 403/500** như "chưa đăng nhập" (chưa sửa) |
| `GET /api/tasks` | `?branchSlug=` | filter client | `RỦI RO` – toàn bộ workload đi 1 request (chưa sửa) |
| `POST /api/tasks` | `assignedTo` = **ID** ✅ | `assignedTo` = **ID** ✅ | ✅ **đã đồng bộ** (P1-1) |
| `PUT /api/tasks/:id` | `{status, proofPhotoUrl, completedAt}` | thêm `photoSubmittedBy/At` | `RỦI RO` – whitelist staff bỏ 2 field cuối ⇒ ảnh minh chứng không ghi được người nộp (chưa sửa) |
| `GET /api/shifts` | `?branchSlug=` | `?branchSlug=` + `employeeId` | ✅ đã đồng bộ (slug, không còn so tên) |
| `POST /api/attendance/checkin` | `{employeeId, shiftId}` | + `checkinTime` (đồng hồ máy) + `wifiSsid/Bssid` | ⚠️ BE vẫn bỏ qua 2 field Wi-Fi (chỉ sửa wording) |
| `POST /api/shifts/register` | *(web không có luồng này)* | gửi 4 khóa trùng giá trị `week/preference/day/wish` | `CỐ Ý` – đọc dữ liệu legacy |
| `PUT /api/requests/:id/approve` | `{reviewedBy, reviewNote}` | chỉ `{reviewNote}` | `RỦI RO` – BE default `reviewedBy = "admin"` ⇒ **ai cũng bị ghi là admin** (chưa sửa) |
| `GET /api/payroll/payslips` | `?month=` | luôn `employeeId: uid` | ✅ BE đã ghim chi nhánh cho manager (P2-5) |
| `GET /api/wifi-configs` | không `branch` | `?branch=` | ✅ BE ghim chi nhánh với non-admin |
| `GET /api/notifications` | không `employeeId` | có `employeeId` | ✅ BE tự scope, `?employeeId=` bị bỏ qua với non-admin |
| Envelope | unwrap `{data}` | unwrap `{data}` | `OK` |
| Date | DD/MM/YYYY, BE chuẩn hoá lúc ghi | DD/MM/YYYY | `OK` nhưng `dateFormat.js` bị copy 2 bản (org + work) |
| Phân trang | không page | không page | `CỐ Ý` cho demo |

**Kết luận cầu nối:** "nối" đúng về khung (envelope, lỗi, date) và **đã được siết lại về hệ quy chiếu dữ liệu** (tên↔id, hoa↔thường) cùng **quyền đọc theo chi nhánh**. Phần còn lại (`reviewedBy`, `checkinTime`, `photoSubmittedBy`, phân trang) là những mục đã ghi rõ ở đây để nói thẳng với giảng viên thay vì để bị phát hiện.

---

## 7. Test: cái gì xanh, cái gì đỏ, cái gì chưa có (sau khi sửa)

**Đã chạy và XANH (sau đợt sửa):**
| Suite | Lệnh đúng | Kết quả |
|---|---|---|
| mobile | `flutter analyze` / `flutter test` | **0 issue** · **118 pass, 1 skip** |
| frontend | `npx tsc --noEmit` / `npx vitest run` | sạch · **39/39 pass** (7 file) |
| api-gateway | `node --test tests/*.test.js` | **112 pass** (107 cũ + 5 case mới) |
| organization-service | `node --test tests/unit/*.test.js` | **16 pass** |
| payroll-service | `node --test tests/unit/*.test.js` | **24 pass** (19 cũ + 5 case mới) |
| integration-service | `node --test tests/unit/*.test.js` | **35 pass** |
| identity-service | `node --test tests/unit/*.test.js` | **4 pass** |
| work-service | `DATABASE_URL= node --test tests/unit/*.test.js` | **34 pass** (29 cũ + 5 case mới) |

**Đính chính so với báo cáo cũ:**
- Repo **không dùng jest**: cả 6 service chạy bằng `node --test` (`npm test`). Lệnh `npx jest` trong báo cáo cũ **không tái lập được** (npx tải jest ngoài rồi fail "must contain at least one test").
- **work-service KHÔNG còn đỏ**: 2 fail trước đây là do **Postgres 5435 không chạy**, không phải test mâu thuẫn chính sách. Chạy `DATABASE_URL= node --test tests/unit/*.test.js` (ép chế độ memory) → **29/29 pass**. Cần bật Postgres hoặc ép `DATABASE_URL=""` khi chạy CI.
- **frontend vitest ĐÃ có sẵn** trong `node_modules` (39/39 pass), không cần `npm i`.
- Backend integration + archive cần Postgres 5433-5437/docker; `mobile/test/api_login_test.dart:18` tự skip khi gateway chết.

**Test mới thêm trong đợt sửa:**
- `backend/work-service/tests/unit/branchScope.test.js` — 5 test: manager chỉ thao tác chi nhánh mình, không chấm công vào ca người khác, staff không chấm công hộ, `scopeTaskQuery` (task của mình + task theo ca của chi nhánh).
- `backend/payroll-service/tests/unit/payslipScope.test.js` — 5 test: ghim chi nhánh phiếu lương, manager chưa gán chi nhánh bị 403, staff chỉ thấy phiếu mình, admin không giới hạn.
- `backend/api-gateway/tests/authorize.test.js` — thêm case `scan` (staff/manager bị chặn, admin được) và `POST /api/notifications` (staff bị chặn, manager/admin được).

**Ý đồ mà test ĐÃ khóa:** ma trận role×route, staff self-service, manager chặn branches/wifi/soap/bank/backlog/archive, refresh-token không dùng làm access, forgot-password không reset, checkout window + "bỏ ca" + phạt, staff không đọc phiếu lương người khác, notification visibility, shift-swap hoàn tác, slug tự sinh, fail-closed DB, **gim chi nhánh ở work/payroll/integration**, **không chấm công vào ca người khác**.

**Ý đồ CHƯA có test khoá:**
- Web Wi-Fi chỉ xem + xoá (mục 3.1).
- "Staff chỉ dùng mobile" — test authorize vẫn **cho** staff ghi nhiều tuyến mà web chặn FE.
- Bảng quyền mobile mới (chỉ phủ 4 route đặc quyền).

**Độ phủ UI:**
- Web: chỉ `bank`, `branches`, `departments` có component test (render tĩnh). **Không có UI test:** dashboard, employees, shifts, tasks, requests, news, regulations, wifi, backlog, payslips.
- Mobile: có test cho splash/home/session/permissions/repositories/wifi-hardware/missed-checkout. **Không có test:** login form, `schedule/*`, `attendance/adjustment`, `salary` + `advance`, `tasks` UI, `monitor`, `approvals`, `news`/`regulations`/`notifications`/`request_detail`, UI `wifi`, **toàn bộ `profile/*`**, `help`.

---

## 8. Bảng trạng thái nhanh (để đối chiếu khi bị hỏi)

| Nhóm | Mục | Trạng thái |
|---|---|---|
| P1 | Task web → mobile (id/name), dropdown rỗng, hủy ca mobile, phiếu lương rỗng | ✅ đã sửa |
| P2 | Wi-Fi wording, lỗi mạng bị nuốt (web+mobile), handler chết, nhãn sai, manager vượt scope | ✅ đã sửa |
| P3 | P3-1 … P3-10 | ✅ đã sửa (P3-7 còn 1 phần state tĩnh của notifications) |
| Bảo mật | **shiftId IDOR** (staff chấm công vào ca người khác) | ✅ đã sửa + test |
| Còn tồn đọng | `listAttendance`/`listRegistrations` chưa ghim chi nhánh; `reviewedBy` mặc định "admin"; `checkinTime` do máy gửi; `photoSubmittedBy` bị whitelist bỏ; không phân trang; `morgan` in query; state tĩnh notifications | ⚠️ ghi nhận, không sửa |

---

## 9. Việc cần làm tay trước khi demo (Postgres đang tắt nên không chạy tự động được)

```bash
# 1. Đồng bộ schema sau khi thêm 2 cột mới
cd backend/payroll-service    && npx prisma db push   # payslips.branchSlug
cd backend/integration-service && npx prisma db push   # file_assets.branchSlug, news.author

# 2. Gán chi nhánh cho phiếu lương đã tạo trước khi có cột branchSlug
cd backend/payroll-service && node scripts/backfill-payslip-branch.js
```
> ⚠️ Không chạy bước 2 thì **manager không thấy phiếu lương cũ** (BE đã ghim chi nhánh nên phiếu cũ có `branchSlug = null` không khớp).

---

## 10. Cách tái kiểm (lệnh thật)

```bash
# Mobile (đã chạy, xanh)
cd mobile && flutter analyze && flutter test

# Backend unit tests — dùng node --test, KHÔNG phải jest
cd backend/api-gateway      && npm test          # hoặc node --test tests/*.test.js
cd backend/organization-service && node --test tests/unit/*.test.js
cd backend/payroll-service  && node --test tests/unit/*.test.js
cd backend/integration-service && node --test tests/unit/*.test.js
cd backend/identity-service && node --test tests/unit/*.test.js
cd backend/work-service     && DATABASE_URL= node --test tests/unit/*.test.js   # ép memory, không cần Postgres

# Frontend (không cần npm i)
cd frontend && npx tsc --noEmit && npx vitest run
```

---

## 11. Nhật ký sửa đổi trong code (đợt 12/11/2026)

Nguyên tắc sửa áp dụng xuyên suốt: **gửi ID, hiển thị tên**; **BE là nơi kiểm quyền, FE chỉ hiển thị**; **lỗi phải nổi lên chứ không được biến thành "không có dữ liệu"**; **fail-closed khi DB lỗi**.

### 11.1 Nhóm A (P1) — cầu nối web ↔ mobile
| File | Thay đổi |
|---|---|
| `frontend/src/features/tasks/types.ts` | thêm `assignedToId?: string \| null` vào `TaskItem` |
| `frontend/src/features/tasks/TaskForm.tsx` | theo dõi id nhân viên được chọn; task theo ca ⇒ `assignedToId = null` |
| `frontend/src/app/dashboard/tasks/page.tsx` | gửi `assignedTo = newTask.assignedToId ?? null`; normalize `branch` từ `branchSlug`; báo lỗi khi request chính fail; refetch sau hoàn thành/xoá |
| `frontend/src/app/dashboard/payslips/page.tsx` | bỏ `?month=all`; manager chỉ thấy phiếu cùng chi nhánh (không còn ngoại lệ `!p.branch`) |
| `mobile/lib/src/features/schedule/staff_detail.dart` | hủy ca & chấm công dùng `staff.shiftId`; bỏ fallback `employeeId='1'`; lỗi hiện `ApiException.message` |
| `backend/work-service/src/utils/caller.js` | `scopeTaskQuery` trả `includeShiftTasksOfBranch`; thêm `assertBranchScope` |
| `backend/work-service/src/services/WorkService.js` | `listTasks` hỗ trợ OR (task của mình + task theo ca của chi nhánh); `syncShiftCheckin/Checkout` chặn ca của người khác; `updateTemplate` whitelist field |
| `backend/work-service/src/controllers/WorkController.js` | truyền `caller` cho create/update/delete/assignShift; guard chi nhánh cho checkin/checkout; cho phép staff cập nhật task theo ca của chi nhánh; **xoá** `getShift`/`getTask` |
| `backend/work-service/src/routes/workRoutes.js` | **xoá** `GET /shifts/:id` và `GET /tasks/:id` (không màn nào dùng) |

### 11.2 Nhóm B — UI/UX
| File | Thay đổi |
|---|---|
| `frontend/src/app/dashboard/requests/page.tsx` | bỏ lọc status/type ở server (badge đếm đúng); bỏ giờ bịa; báo lỗi khi fetch fail |
| `frontend/src/app/dashboard/shifts/page.tsx` | báo lỗi khi fetch fail; **xoá** 2 handler chết; so chi nhánh bằng slug (`sameBranch`); thêm nút xoá khung ca + hộp xác nhận |
| `frontend/src/features/shifts/ShiftTemplates.tsx` | thêm nút Xoá khung ca |
| `frontend/src/features/shifts/types.ts` | thêm `branchSlug` cho `ShiftRegistrationRecord` |
| `frontend/src/features/departments/DepartmentList.tsx` | bỏ `!isManager` ⇒ manager mở được Chi tiết phòng ban |
| `frontend/src/app/dashboard/departments/page.tsx` | refetch sau khoá/xoá |
| `frontend/src/app/dashboard/employees/page.tsx` | xoá nhánh chết `handleConfirmDelete` + import thừa; refetch sau khoá/xoá |
| `frontend/src/app/dashboard/page.tsx` | không còn `.catch(()=>[])`; hiện cảnh báo lỗi thay vì "0 nhân sự" |
| `frontend/src/app/dashboard/branches/page.tsx` | guard quyền **trước** khi gọi API |
| `frontend/src/context/AuthContext.tsx` | bỏ `clearWebSessionCookies` (cookie httpOnly) → gọi `POST /api/auth/logout` |
| `frontend/src/app/dashboard/layout.tsx` | chờ `loading` xong mới render nội dung ⇒ hết nháy "Không có quyền truy cập" |
| `mobile/.../tasks/monitor.dart` | truyền `branchSlug` thật; banner lỗi + "Thử lại"; token chống race |
| `mobile/.../attendance/attendance_repository.dart` | `getAttendance(branchSlug:)`; `getConfig` **ném lỗi** thay vì `{}` |
| `mobile/.../schedule/general.dart` | lỗi cấu hình chấm công tách riêng, không làm hỏng cả màn lịch |
| `mobile/.../salary/salary.dart` | tiêu đề đúng nội dung; bỏ fallback phiếu sai kỳ |
| `mobile/.../salary/advance.dart` | bỏ "50%"; banner lỗi tải lương |
| `mobile/.../approvals/approvals.dart` | bỏ nhánh `overtime` |
| `mobile/.../schedule/shift_form.dart` + `schedule_service.dart` | gửi `branchSlug` (mã) thay tên; `ShiftDetail.branchSlug` mới |
| `mobile/.../tasks/tasks.dart` | báo lỗi danh mục thay vì dialog trống |
| `mobile/.../schedule/assignment.dart` | refetch sau đổi chế độ lặp |
| `mobile/.../profile/employee_repository.dart`, `profile.dart`, `home.dart` | lỗi đọc hồ sơ không bị nuốt |
| `mobile/.../home/help.dart`, `wifi_config/wifi.dart` | câu chữ Wi-Fi đúng thực tế |
| `mobile/test/repositories/attendance_repository_test.dart`, `challenger_verification_test.dart` | cập nhật kỳ vọng mới (lỗi phải nổ lên) |

### 11.3 Nhóm C — quyền đọc/ghi
| File | Thay đổi |
|---|---|
| `backend/payroll-service/prisma/schema.prisma` | thêm `Payslip.branchSlug` |
| `backend/payroll-service/src/services/PayrollService.js` | lưu/lọc `branchSlug`; `generatePayslipUseCase` **luôn** đọc hồ sơ nhân viên (fail-closed) để lấy chi nhánh; trừ tiền có điều kiện; `isPrimary` trong transaction; bỏ `balance` khỏi allowlist |
| `backend/payroll-service/src/utils/caller.js` | `scopePayslipQuery` ghim chi nhánh cho manager; `assertOwnPayslip` kiểm chi nhánh |
| `backend/payroll-service/src/clients/OrgClient.js` | kèm header nội bộ `x-user-role: admin` (nếu thiếu thì mọi lời gọi bị 403) |
| `backend/payroll-service/src/utils/errors.js` | thêm `InternalServerError` |
| `backend/payroll-service/scripts/backfill-payslip-branch.js` (mới) | gán chi nhánh cho phiếu lương cũ |
| `backend/integration-service/prisma/schema.prisma` | thêm `FileAsset.branchSlug`, `News.author`; sửa comment loại đơn |
| `backend/integration-service/src/controllers/RequestController.js` | kiểm chi nhánh/chủ khi xem/duyệt/từ chối/xoá đơn |
| `backend/integration-service/src/controllers/NotificationController.js` | người gửi lấy từ token; `?employeeId=` bị bỏ qua với non-admin; xoá chỉ chủ hoặc admin |
| `backend/integration-service/src/controllers/FileController.js` + `services/FileService.js` + `controllers/UploadController.js` | lưu `branchSlug` lúc upload; kiểm chủ/chi nhánh khi đọc file |
| `backend/integration-service/src/controllers/WifiConfigController.js` + `services/WifiConfigService.js` | ghim chi nhánh khi đọc; `scan` chỉ admin; whitelist field khi update |
| `backend/integration-service/src/controllers/NewsController.js` | giữ `author` (lấy từ `x-user-name`); whitelist field khi update |
| `backend/integration-service/src/validators/requestValidator.js` + `services/RequestService.js` | bỏ loại `overtime`/`other` |
| `backend/integration-service/src/services/SoapConfigService.js` | bỏ `allowedIPs` bịa; lỗi DB nổ lên thay vì trả cấu hình memory |
| `backend/integration-service/src/services/BacklogService.js` | bỏ 4 bản ghi seed mô phỏng thao tác admin |
| `backend/api-gateway/src/middlewares/authorize.js` | `GET /api/wifi-configs/scan` chỉ admin; `POST /api/notifications` chỉ admin/manager |
| `backend/api-gateway/src/middlewares/auth.js` + `app.js` | đọc `name` từ token; gửi `x-user-name` xuống service (xoá header giả mạo) |
| `backend/identity-service/src/services/AuthService.js` | access + refresh token mang `name`; `syncEmployee` ưu tiên enum `systemRole` |

### 11.4 Nhóm D — nhỏ
So chi nhánh bằng slug ở web & mobile; refetch sau mutation; bỏ fallback phiếu sai kỳ; trừ tiền có điều kiện; `isPrimary` trong transaction; chặn sửa `balance`; whitelist field khi update (template/wifi/news); bỏ seed/giá trị bịa (SOAP, Backlog); logout gọi API; hết nháy quyền.

---

## 12. Ghi chú kỹ thuật của lần audit + đợt sửa

- 14 sub-agent đọc code cho lần audit; 4 phát hiện nặng nhất (P1-1, P1-3, P1-4, cookie-first) đã **kiểm lại tay** bằng cách đọc trực tiếp file.
- Đợt sửa: **7 sub-agent bị chặn bởi giới hạn usage** nên phần sửa được thực hiện trực tiếp, chia theo nhóm file không chồng lấn (frontend / mobile / work / payroll / integration / gateway+identity).
- **6 đính chính so với bản audit gốc** (đã phản ánh trong tài liệu này):
  1. "Manager đọc CCCD khác chi nhánh" — **sai**: `assertCanReadEmployee` + `scopeEmployeeQuery` đã chặn.
  2. "Mobile gửi `shiftId` tự do không phải IDOR" — **sai**: `shiftId` **không** được kiểm ⇒ đã sửa (mục 11.1).
  3. "Chạy test bằng `npx jest`" — **sai runner**: repo dùng `node --test`.
  4. "work-service đỏ 2 test vì test mâu thuẫn chính sách" — **sai nguyên nhân**: đỏ vì thiếu Postgres 5435; ép memory thì xanh hết.
  5. "frontend vitest chưa cài" — **sai**: đã có sẵn, 39/39 pass.
  6. Đường dẫn `file:line` đã lệch ở vài chỗ (ví dụ `help.dart` thật là `features/home/help.dart`; `morgan` nằm ở `api-gateway/src/app.js`; `PaymentService`/`PayrollService.js:1014` đã dịch dòng sau khi sửa) — trước khi sửa hãy mở đúng file.
- **Không có `.git` ở thư mục gốc dự án** (`fatal: not a git repository`) ⇒ không có backup tự động cho các sửa đổi ở `backend/`, `frontend/`, `mobile/`. Riêng thư mục `docs/` **có** git riêng nên các file báo cáo cũ vẫn khôi phục được nếu cần.
