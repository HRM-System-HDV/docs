# AUDIT BE VÀ CSDL — HRM Enterprise

> Cập nhật: 13/11/2026. Phạm vi: Gateway + 5 service BE và 5 DB PostgreSQL của đồ án SOA.
> Căn cứ nghiệp vụ: [BUSINESS.md](BUSINESS.md). Căn cứ cấu trúc dữ liệu: năm file Prisma trong backend.
> Báo cáo phân biệt code đã sửa, test đã chạy và kiểm chứng DB còn thiếu; không đồng nhất test memory/Prisma mock với test trên PostgreSQL thật.

## 1. Kết luận hiện tại

| Hạng mục | Trạng thái và bằng chứng |
|---|---|
| Tái cấu trúc 5 service và Gateway | Đã sửa code theo nghiệp vụ mới; xem mục 3. |
| Thiết kế CSDL | 25 bảng nghiệp vụ, UUID, Decimal, ngày có kiểu; cả 5 schema qua Prisma validate. |
| Migration | Có migration `20261112230000_hrm_uuid_redesign` cho mỗi service; đã chạy migrate deploy thành công trên cả 5 DB local trong lượt triển khai trước. |
| Kiểm thử BE | Lần chạy gần nhất: `npm.cmd test` trong backend, 187/187 test qua, không có test fail hoặc skip. |
| Kiểm thử HTTP xuyên service | Đã qua, chạy sáu tiến trình HTTP độc lập với dữ liệu memory cô lập, có luồng REST và SOAP thật giữa các service. |
| Kiểm chứng ghi/rollback trên DB thật | Chưa hoàn tất. Lượt chạy `npm.cmd run test:database` bị lỗi P1001 tại localhost:5433; kiểm tra status/diff cũng không kết nối được các cổng 5433–5437. |
| Web và Mobile | Không sửa trong đợt BE. Chưa nghiệm thu giao diện hoặc sự tương thích với API mới. |
| Bàn giao | Có thể review code/test; chưa kết luận toàn hệ thống đã chạy hoàn chỉnh trên DB thật và thiết bị thật. |

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
- Unique nhân viên/ngày/ca mẫu chống phân trùng; Attendance.shiftId unique bảo đảm một dòng chấm công mỗi ca.
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
| Integration | Mẫu đơn không tự sửa chức năng khác; kiểm tra đổi ca; thông báo fanout/phạm vi/nhắc giờ; nội dung chi nhánh/toàn thể; Wi-Fi cho Work; file/CCCD/PDF và nhật ký; SOAP cần payslipIds. | [IntegrationService](../backend/integration-service/src/services/IntegrationService.js), [SoapService](../backend/integration-service/src/services/SoapService.js), [FileService](../backend/integration-service/src/services/FileService.js), [app](../backend/integration-service/src/app.js) |

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

Cả năm service có thư mục `prisma/migrations/20261112230000_hrm_uuid_redesign/`.

Trước khi áp dụng, đã đọc cấu trúc và đếm dữ liệu trên năm DB local: tất cả bảng nghiệp vụ đều có 0 dòng, cấu trúc cũ dùng ID số nguyên. Sau đó đã chạy migrate deploy lần lượt; cả năm lệnh đều báo áp dụng thành công.

Migration thực hiện trong transaction từng DB:

1. Khóa/kiểm tra các bảng nghiệp vụ cũ. Nếu xuất hiện dữ liệu thì báo lỗi và dừng.
2. Thay cấu trúc bảng trống sang UUID và quan hệ mới; loại bảng archive cũ.
3. Tạo hàm pad_code/sequence ở service cần sinh mã.
4. Tạo unique, FK, CHECK và partial unique index theo thiết kế.
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

Chưa xác nhận lại drift/history hoặc kết quả test ghi/rollback trên DB thật sau lượt deploy: Postgres không kết nối được ở lượt tiếp tục. Migration được tạo cho DB trống đã kiểm tra, không phải công cụ chuyển dữ liệu cũ sang UUID trên DB có dữ liệu.

## 6. Bằng chứng kiểm thử đã chạy

### 6.1. Bộ test BE gần nhất

Lệnh: `npm.cmd test` tại `backend/`. Runner chạy từng service rồi test HTTP xuyên service.

| Bộ test | Qua | Fail | Phạm vi bằng chứng |
|---|---:|---:|---|
| Gateway | 120 | 0 | JWT/platform, quyền route, chống header mạo danh, proxy và SOAP XML. |
| Identity | 9 | 0 | Phiên/refresh, staff Mobile, admin tài khoản, sync và Prisma mock. |
| Organization | 12 | 0 | CRUD, salary/ngày/FK, phạm vi, nghỉ việc và lỗi Prisma. |
| Work | 15 | 0 | Wi-Fi, trùng/chồng giờ, kép/rollback mock, qua đêm, hạn checkout, manual và task. |
| Payroll | 9 | 0 | Tính lương chính xác, xác nhận/freeze, số dư, idempotency và transaction mock. |
| Integration | 21 | 0 | Form-only, quyền/phạm vi, fanout, file/PDF, SOAP và Prisma mock. |
| HTTP xuyên service | 1 | 0 | Một kịch bản gồm nhiều bước thực hiện qua sáu tiến trình độc lập. |
| Tổng | **187** | **0** | Không có test skip; không phải số đo coverage hoặc bằng chứng DB thật. |

Nguồn test HTTP: [service-flow.test.cjs](../backend/tests/service-flow.test.cjs). Luồng đã kiểm tra gồm tạo chi nhánh/nhân viên → đồng bộ Identity → đăng nhập staff → phân ca/chặn trùng → ghi công → task chung/riêng → Wi-Fi → duyệt đơn không đổi công → cập nhật salary → sinh phiếu → chặn chi chưa xác nhận → xác nhận → chi REST/SOAP và gửi lặp không trừ tiền lần hai → nội dung/thông báo theo chi nhánh.

### 6.2. Các kiểm tra khác

- Prisma format/validate thành công cho cả năm schema.
- Prisma Client đã generate cho năm service trong quá trình tái cấu trúc.
- git diff --check không phát hiện lỗi whitespace; cảnh báo LF/CRLF không phải lỗi test.
- `.env` thật được gitignore. Đã đồng bộ secret nội bộ và bổ sung URL peer còn thiếu; báo cáo không chứa giá trị secret.
- Tên tiếng Việt được mã hóa khi truyền HTTP header; có regression test proxy và kiểm tra tên người duyệt trong luồng liên service.

### 6.3. Kiểm chứng bị chặn

Lệnh `npm.cmd run test:database` gọi [verify-database.cjs](../backend/scripts/verify-database.cjs). Script thiết kế tạo bản ghi thử và kiểm tra CHECK/unique/FK trong transaction rồi rollback, không seed dữ liệu.

Lượt chạy hiện tại dừng tại kết nối Identity với P1001, localhost:5433. Chưa chạy thành công các phép ghi/rollback trong script; không đánh dấu các DB còn lại đã qua. Các phép kiểm tra migrate status/diff gần nhất cũng không kết nối được localhost:5433–5437.

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
- [ ] Admin thao tác được trên Web/Mobile theo nghiệp vụ; chưa coi UI cũ đã tương thích API mới.
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

Không dùng db:setup/seed/reset làm bước “sửa lỗi” khi DB có dữ liệu. Không chạy lại test Web/Mobile rồi kết luận UI mới tương thích khi chưa cập nhật client.

## 9. Giới hạn và phần chưa nghiệm thu

| Điểm | Hiện trạng |
|---|---|
| DB thật | Migration đã báo applied; lượt kiểm chứng ghi/rollback và tái kiểm drift chưa hoàn tất vì mất kết nối local. |
| UI và thiết bị thật | Chưa sửa/test Web/Mobile; chưa kiểm chứng thao tác Wi-Fi trên thiết bị thực tế. |
| Nhắc giờ | Tạo khi client lấy thông báo hoặc gọi nhắc giờ; chưa có scheduler/push nền khi ứng dụng không hoạt động. |
| Nhật ký Work → Integration | Best-effort; lưu công thành công không bị trả lỗi giả khi dịch vụ nhật ký gián đoạn, nhưng không bảo đảm log xuyên DB trong trường hợp đó. |
| Token truy cập đã cấp | Thu hồi phiên/refresh không biến mọi access token cũ thành vô hiệu ngay; Gateway kiểm chữ ký và hạn sống ngắn. |
| Hồ sơ → Identity | Đồng bộ HTTP best-effort; khi Identity tạm ngừng có cảnh báo, cần kiểm tra/đồng bộ lại, không có hàng đợi bền vững. |
| Phân trang/khối lượng lớn | Bộ test hiện không chứng minh hiệu năng với dữ liệu lớn; phạm vi vẫn là đồ án. |
| Coverage | Chưa đo lại coverage; 187 test qua không có nghĩa 100% nhánh code đã được bao phủ. |
| Nhánh/commit | Thay đổi nằm trong workspace; báo cáo này không khẳng định đã commit, merge hoặc triển khai môi trường khác. |

## 10. Cách chốt nghiệm thu

- [x] Có đặc tả nghiệp vụ và schema mới để đối chiếu.
- [x] Có code BE và migration tương ứng, đã bỏ các luồng cũ trái phạm vi.
- [x] Bộ test BE và HTTP liên service gần nhất qua 187/187.
- [x] Cả năm schema hợp lệ; migration đã được áp dụng trên năm DB local ở lượt trước.
- [ ] Hoàn thành kiểm chứng ghi/rollback và tái kiểm migration trên DB thật sau khi bật lại hạ tầng.
- [ ] Người review hoàn thành checklist thủ công và xác nhận kết quả.
- [ ] Cập nhật/kiểm thử Web/Mobile trong đợt riêng nếu tiếp tục triển khai toàn hệ thống.

Không đánh dấu “toàn hệ thống hoàn tất” chỉ dựa vào những ô đã tick ở phần BE.
