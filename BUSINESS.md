# ĐẶC TẢ NGHIỆP VỤ HỆ THỐNG QUẢN LÝ NHÂN SỰ

> HRM Enterprise — Đồ án môn Phát triển phần mềm hướng dịch vụ (SOA).
> Cập nhật ngày 14/11/2026 theo nghiệp vụ đã thống nhất và thiết kế Prisma mới.
> Đây là đặc tả nghiệp vụ; trạng thái triển khai và cách kiểm tra BE được ghi tại [backend/README.md](../backend/README.md). Migration thiết kế mới đã được tạo và áp dụng trên năm DB local trong đợt tái cấu trúc.
> Web và Mobile đã được làm lại theo thiết kế mới. Các mô tả nút/màn hình dưới đây xác định nghiệp vụ mà API và giao diện phải hỗ trợ.

## Cách dùng tài liệu

- Nghiệp vụ thực hiện theo tài liệu này; tên trường, kiểu dữ liệu và quan hệ thực hiện theo năm schema Prisma tại mục 11.1.
- Các mô tả lương cứng, phạt tiền cố định, khóa check-in ngay khi đi muộn, tự checkout hoặc archive trong tài liệu/code cũ không còn là yêu cầu của thiết kế mới. Riêng đơn đổi ca: duyệt/chấp nhận là lệnh đổi lịch thật, như mục 7.3.
- Tái sử dụng code phù hợp; sửa hoặc loại bỏ code cũ trái nghiệp vụ, không giữ thêm một luồng xử lý song song cho cùng chức năng.
- Không tự mở rộng thêm bảng, phân hệ hoặc quy trình ngoài phạm vi đồ án để giải quyết một thao tác đã được chức năng hiện có hỗ trợ.

## Mục lục

1. [Phạm vi và vai trò](#1-phạm-vi-và-vai-trò)
2. [Ranh giới dịch vụ và dữ liệu](#2-ranh-giới-dịch-vụ-và-dữ-liệu)
3. [Tài khoản và hồ sơ nhân sự](#3-tài-khoản-và-hồ-sơ-nhân-sự)
4. [Ca mẫu, phân ca và đăng ký ca](#4-ca-mẫu-phân-ca-và-đăng-ký-ca)
5. [Chấm công và kiểm tra Wi-Fi](#5-chấm-công-và-kiểm-tra-wi-fi)
6. [Lương tích lũy và phạt](#6-lương-tích-lũy-và-phạt)
7. [Đơn từ và đổi ca](#7-đơn-từ-và-đổi-ca)
8. [Phiếu lương, số dư và chi lương SOAP](#8-phiếu-lương-số-dư-và-chi-lương-soap)
9. [Công việc, thông báo và nội dung nội bộ](#9-công-việc-thông-báo-và-nội-dung-nội-bộ)
10. [Tệp và nhật ký](#10-tệp-và-nhật-ký)
11. [Quy ước dữ liệu và trách nhiệm triển khai](#11-quy-ước-dữ-liệu-và-trách-nhiệm-triển-khai)
12. [Tình huống nghiệm thu nghiệp vụ](#12-tình-huống-nghiệm-thu-nghiệp-vụ)

## 1. Phạm vi và vai trò

Hệ thống phục vụ một doanh nghiệp có nhiều chi nhánh. Phạm vi đồ án gồm quản lý nhân sự, phân ca, chấm công theo Wi-Fi, giao việc, đơn từ, lương tích lũy, phiếu lương, mô phỏng thanh toán qua REST/SOAP, thông báo, bảng tin và quy định.

Không thiết kế đa khách hàng; không có `tenantId`. Không bổ sung độ phức tạp của hệ thống nhân sự hoặc ngân hàng thực tế nếu không phục vụ yêu cầu đồ án.

| Vai trò hệ thống | Phạm vi nghiệp vụ |
|---|---|
| `admin` | Toàn quyền trên cả Web và Mobile. |
| `manager` | Dùng Web/Mobile cho nghiệp vụ quản lý trong phạm vi được giao; bị chặn các chức năng quản trị hệ thống. |
| `staff` | Chỉ đăng nhập Mobile, sử dụng nghiệp vụ cá nhân; không được truy cập chức năng của admin hoặc manager. |

Vai trò hệ thống khác chức danh công việc. `Position` mô tả vị trí nhân sự; `User.systemRole` phục vụ phân quyền.

- Staff không được đăng nhập Web, không chỉ bị ẩn menu sau đăng nhập.
- Các nhóm quản trị hệ thống như quản lý tài khoản/vai trò và cấu hình toàn hệ thống thuộc admin; manager không có quyền quản trị hệ thống. Riêng cấu hình chấm công áp dụng theo chi nhánh: manager được xem/sửa cấu hình của chi nhánh mình, admin quản lý cấu hình mặc định và mọi chi nhánh.
- Manager và staff chỉ đọc/thao tác dữ liệu trong phạm vi được cấp. BE kiểm tra quyền cho từng thao tác, không chỉ dựa vào việc FE ẩn nút.

## 2. Ranh giới dịch vụ và dữ liệu

| Thành phần | Cổng | DB sở hữu | Trách nhiệm |
|---|---|---|---|
| API Gateway | 4000 | Không có | Xác thực/định tuyến theo kiến trúc chung, chuyển tiếp REST và SOAP. |
| Identity | 4001 | `hrm_identity` | `User`, `DeviceSession`, `RefreshToken`; xác thực và vai trò hệ thống. |
| Organization | 4002 | `hrm_organization` | `Branch`, `Department`, `Position`, `Employee`; hồ sơ và thỏa thuận lương. |
| Work | 4003 | `hrm_work` | `ShiftTemplate`, `Shift`, `ShiftRegistration`, `Attendance`, `AttendanceConfig`, `Task`. |
| Payroll | 4004 | `hrm_payroll` | `BankAccount`, `Payout`, `Payslip`; tổng hợp lương và mô phỏng chi tiền. |
| Integration | 4005 | `hrm_integration` | `Request`, `Notification`, `NotificationSetting`, `SoapGatewayConfig`, `News`, `Regulation`, `WifiConfig`, `Backlog`, `FileAsset`. |

- Mỗi service chỉ đọc/ghi DB của mình. Trao đổi dữ liệu qua client HTTP nội bộ, timeout tối đa 5000ms.
- Không JOIN, tạo FK hoặc truy cập trực tiếp DB xuyên service; không import mã nguồn của service khác.
- Tham chiếu xuyên service dùng UUID do service sở hữu cấp. Quan hệ cùng DB được khai báo bằng relation/FK khi thích hợp.
- Web và Mobile truy cập qua Gateway. SOAP giữ định dạng XML, không bọc bằng envelope JSON.
- Không có chức năng archive/khôi phục theo đợt trong thiết kế đã chốt; không có bảng `ArchivedRecord`.

## 3. Tài khoản và hồ sơ nhân sự

### 3.1. Tài khoản, phiên và thiết bị

- Identity quản lý email đăng nhập, mật khẩu băm, vai trò `admin/manager/staff`, phiên thiết bị và refresh token.
- `User.employeeId` liên kết tài khoản với hồ sơ nhân viên, có thể trống khi chưa gắn hồ sơ.
- Email được trim và chuẩn hóa chữ thường trước khi ghi ở Identity và Organization.
- `User.systemRole` là nguồn quản lý vai trò; không lưu thêm vai trò hệ thống trên `Employee`.
- `User.branchId` là bản sao phạm vi chi nhánh từ Organization, cần cập nhật khi chuyển chi nhánh; không phải nơi sở hữu cơ cấu tổ chức.
- Có nhận diện thiết bị và vị trí đăng nhập: `deviceName`, `deviceType`, `osInfo`, `location`, `ipAddress`, `lastSeenAt`.
- Vị trí đăng nhập phục vụ thông tin phiên thiết bị, khác điều kiện Wi-Fi dùng để chấm công.
- Thiết bị hiện tại được xác định từ phiên đang gửi yêu cầu; không lưu cờ `isCurrent`.
- Refresh token có thời hạn và khả năng thu hồi; nếu gắn thiết bị thì token và thiết bị phải cùng người dùng.
- Access token và refresh token đều hạn 7 ngày (chốt 12/11/2026). Thu hồi khi đăng xuất, đổi mật khẩu, đổi vai trò/chi nhánh hoặc chặn thiết bị chỉ cắt refresh token và phiên thiết bị; access token đã cấp vẫn còn hiệu lực tới hết hạn vì Gateway chỉ kiểm chữ ký. Chủ sở hữu đã chấp nhận rủi ro này.
- Làm mới phiên là một lần dùng: refresh token vừa dùng bị đánh dấu đã thu hồi, không dùng lại; nền tảng của phiên không đổi được khi làm mới.

### 3.2. Tổ chức và hồ sơ

- Quản lý chi nhánh, phòng ban, vị trí và người phụ trách bằng ID, không dùng tên làm khóa liên kết.
- Trạng thái danh mục là enum phía BE: chi nhánh `hoạt động`/`vô hiệu hóa`; phòng ban và vị trí `hoạt động`/`tạm dừng`. Giá trị khác bị 400.
- Cột "Nhân sự" trên chi nhánh và phòng ban chỉ đếm hồ sơ `đang làm`; riêng vị trí đếm mọi hồ sơ đang gắn vị trí (kể cả đã nghỉ việc) vì xóa vị trí sẽ SetNull chức danh của đúng số hồ sơ đó.
- Hồ sơ nhân viên gồm thông tin cá nhân, liên hệ, địa chỉ, ngày vào/nghỉ việc, ảnh đại diện, cơ cấu tổ chức, thông tin lương và tài khoản ngân hàng.
- Có quản lý hồ sơ CCCD: số CCCD, ngày cấp, nơi cấp, ảnh mặt trước và mặt sau.
- Nhân viên nghỉ việc được đổi trạng thái và ghi `resignedAt`, không xóa cứng hồ sơ để giữ lịch sử.
- Hồ sơ `nghỉ việc` không được phân ca mới, đăng ký ca mới hay chấm công mới; vẫn được xem lịch sử, sửa giờ trên ca cũ theo mục 5.4 và dọn ca đã phân.
- BE kiểm tra vị trí thuộc phòng ban phù hợp, người quản lý và phạm vi tổ chức hợp lệ.
- Quản lý chỉ thao tác nhân sự chi nhánh mình và không đổi được email đăng nhập của nhân viên; email do admin quản lý. Nhân viên tự sửa được thông tin cá nhân, liên hệ, địa chỉ, CCCD, ảnh và tài khoản ngân hàng của mình; không tự sửa lương, chi nhánh, phòng ban, chức danh hay trạng thái làm việc.

### 3.3. Thỏa thuận lương khi nhận nhân viên

| Thuộc tính của `Employee` | Ý nghĩa |
|---|---|
| `salaryType` | `hourly` cho parttime; `monthly` cho fulltime. |
| `salary` | Chỉ một mức tiền: tiền một giờ nếu `hourly`, mức lương tháng thỏa thuận nếu `monthly`. |
| `standardWorkDays` | Số công/tháng thỏa thuận do người quản lý nhập, chỉ dùng cho `monthly`; một ca hoàn thành bằng một công. |

- Fulltime bắt buộc có `standardWorkDays > 0`; parttime để trường này là `null`.
- Không mặc định số công là 26; không suy ra từ số ngày của tháng hoặc lịch được phân.
- Không tách thêm `hourlySalary` hay mức lương tháng thứ hai trên `Employee`.
- Không mặc định lương bằng 0 hoặc tự chọn kiểu lương khi thiếu đầu vào.

## 4. Ca mẫu, phân ca và đăng ký ca

- `ShiftTemplate` là danh mục ca mẫu có mã, tên, giờ bắt đầu và giờ kết thúc.
- `Shift` là một ca cụ thể, vào một ngày cụ thể, được phân cho một nhân viên. Một mẫu ca dùng được cho nhiều ca được phân.
- Không phân trùng cùng `(employeeId, workDate, templateId)`; UNIQUE của DB là UNIQUE PARTIAL (`WHERE status <> 'đã hủy'`) nên vẫn bảo vệ khi hai yêu cầu đến đồng thời mà không chặn ca đã hủy. BE còn kiểm tra các ca khác mẫu nhưng trùng giờ.
- Hai ca liên tiếp 07:00–12:00 và 12:00–17:00 không được coi là chồng giờ. Ca kết thúc đúng lúc ca sau bắt đầu được phép xếp cho cùng nhân viên và là trường hợp dùng chấm công kép.
- Ca được phân bắt buộc có ca mẫu, ngày và giờ bắt đầu/kết thúc. `Shift` lưu bản chụp `branchId` của nhân viên lúc phân ca và BE chặn phân ca sang chi nhánh khác; nhân viên sau này chuyển chi nhánh vẫn giữ các ca đã phân, ca đó vẫn hiện trong lịch cá nhân của họ. Kiểm tra Wi-Fi dùng chi nhánh trong hồ sơ nhân viên, không dùng chi nhánh chụp trên ca.
- Khi phân ca, chụp giờ của ca mẫu vào `scheduledStart`/`scheduledEnd`; không để việc sửa ca mẫu làm thay đổi lịch đã phân.
- Ca qua đêm có thời điểm kết thúc ở ngày hôm sau. BE dựng thời điểm đầy đủ từ ngày làm và giờ ca.
- Trạng thái ca thể hiện vòng đời của ca, không thay thế kết quả chấm công: chỉ `đang làm` và `đã hủy`.
- Hủy ca và sửa ca (đổi người, ngày, ca mẫu) chỉ áp dụng cho ca chưa tới giờ bắt đầu và chưa có lượt chấm công nào. Ca đã bắt đầu hoặc đã có lịch sử chấm công thì không xóa được; muốn bỏ ca tương lai thì hủy ca.
- Ca `đã hủy` không chấm công được, không được cộng lương, không tính vào kiểm tra trùng/chồng giờ — hủy xong vẫn phân lại được đúng ca mẫu đó trong ngày — và không hiển thị cho cả ca khi là nguồn của công việc chung.
- Ca mẫu đang được ca phân, đăng ký ca hoặc công việc tham chiếu thì không xóa được.
- `ShiftRegistration` là nguyện vọng của một nhân viên cho một ca mẫu vào một ngày cụ thể; ngày và ca mẫu bắt buộc có.
- Không cho đăng ký trùng cùng `(employeeId, registerDate, templateId)`.
- Đăng ký ca khác phân ca: bản đăng ký không tự tạo ra công hoặc lương.
- Nhân viên chỉ làm và tích lũy lương trên ca đã được phân; không tự tạo công ngoài lịch được xếp.
- Không cần cờ ca lặp `isRecurring`; không lưu riêng khóa tuần vì tuần suy ra từ ngày.

## 5. Chấm công và kiểm tra Wi-Fi

### 5.1. Kiểm tra Wi-Fi ngay khi bấm nút

Luồng người dùng bắt buộc:

1. Nhân viên dùng nút **Chấm công** từ màn hình chính; nút chỉ cho mở màn chọn ca sau khi kiểm tra Wi-Fi thành công.
2. Kiểm tra Wi-Fi đang kết nối với cấu hình Wi-Fi hợp lệ của nhân viên; không dựa vào thuộc tính chi nhánh trên từng ca.
3. Nếu Wi-Fi không phù hợp: không cho mở màn chọn ca, báo lỗi ngay tại bước này.
4. Nếu Wi-Fi phù hợp: mở các ca được phân cho nhân viên trong ngày, chọn ca rồi chấm công.

- Không chờ người dùng chọn ca, mở chi tiết rồi bấm chấm công lần nữa mới báo sai Wi-Fi.
- Danh sách Wi-Fi được quản lý tập trung bằng `WifiConfig` (`ssid`, `bssid`, `branchId`, `status`).
- Không thêm `checkWifi`, SSID, BSSID hoặc cờ kiểm tra Wi-Fi vào từng `Shift`/`ShiftTemplate`.
- Work lấy cấu hình Wi-Fi cần thiết từ Integration qua HTTP, không đọc DB Integration trực tiếp.
- FE kiểm tra sớm để điều khiển luồng; BE vẫn phải kiểm tra điều kiện Wi-Fi khi xử lý check-in/checkout, không chỉ dựa vào việc FE đã mở màn hình.
- Nếu Wi-Fi thay đổi sau bước kiểm tra ban đầu, yêu cầu ghi nhận chấm công vẫn phải đáp ứng điều kiện tại thời điểm xử lý.

### 5.2. Một ca, một lần vào và một lần ra

- Một `Shift` có tối đa một `Attendance`, được bảo đảm bằng unique `Attendance.shiftId`.
- Một bản ghi chấm công luôn thuộc một ca; `employeeId` và ngày phải phù hợp ca đó.
- Chỉ check-in một lần và checkout một lần; không hỗ trợ ra giữa ca rồi vào lại.
- Không tự tạo giờ check-in khi chỉ nhận checkout.
- Không cho checkout trước check-in hoặc sửa checkout thành lần ra thứ hai.
- Giới hạn một lần vào–ra áp dụng cho từng ca, không giới hạn một ca làm trong cả ngày. Hai ca liền nhau vẫn có hai bản ghi chấm công riêng.
- BE kiểm tra danh tính, quyền đối với ca, Wi-Fi, thời gian cho phép và trạng thái ca trước khi ghi nhận.
- Quy tắc thời gian/Wi-Fi trên áp dụng cho nhân viên tự chấm công; thao tác nhập giờ trên ca của admin/quản lý được mô tả riêng ở mục 5.4.

#### Chấm công kép giữa hai ca liền nhau

- Cấu hình `allowDoubleCheckin` cho phép một thao tác chấm công đồng thời checkout ca trước và check-in ca sau của cùng nhân viên.
- Hai ca phải đã được phân, thời điểm kết thúc ca trước trùng thời điểm bắt đầu ca sau. Ca trước đã có check-in nhưng chưa checkout; ca sau chưa có check-in.
- Khoảng cho phép tính từ mốc nối hai ca: sớm tối đa `doubleCheckinEarlyMinutes`, trễ tối đa `doubleCheckinLateMinutes`. Đây là khoảng riêng của chấm công kép, khác `earlyCheckinMinutes` của check-in thông thường.
- Ví dụ ca sáng 07:00–12:00, ca trưa 12:00–17:00; cấu hình kép sớm 5 phút, trễ 5 phút thì thao tác trong khoảng 11:55–12:05 được ghi **checkout ca sáng = 12:00, check-in ca trưa = 12:00**.
- Dùng mốc đã chuẩn hóa để tính công/lương; không phạt về sớm ca trước hoặc đi muộn ca sau chỉ vì giờ bấm trong khoảng kép hợp lệ. Các vi phạm khác của hai ca vẫn được xét bình thường.
- Nhân viên phải chủ động bấm và đáp ứng kiểm tra Wi-Fi/quyền; hệ thống không tự checkout/check-in khi đến mốc nối ca.
- BE ghi cả hai bản ghi trong một transaction, không ghi đè lượt đã chấm và không chỉ ghi thành công một nửa. Giờ ra chuẩn hóa vẫn phải không trước giờ vào của ca trước.
- Hạn checkout ca trước vẫn có hiệu lực; chấm công kép không mở lại ca đã hết hạn. Ngoài khoảng kép thì chỉ xử lý thao tác thông thường nếu còn hợp lệ, không tự làm tròn giờ hoặc tạo check-in ca sau.
- Ca sau mới được check-in, chưa được tính lương cho tới khi có checkout hợp lệ. Mỗi ca vẫn chỉ có một cặp vào–ra.
- Mặc định tính năng tắt, hai khoảng bằng 0; admin bật và nhập khoảng phù hợp. Ví dụ 5 phút trên chỉ minh họa.

#### Trạng thái quên chấm công và check-in muộn

- Ca bắt đầu lúc 17:00 mà sang 17:01 vẫn chưa check-in thì được hiển thị là **Quên chấm công**. Đây là trạng thái nhắc việc, không khóa quyền check-in muộn của ca.
- Nhân viên vẫn được check-in muộn nếu ca và điều kiện chấm công còn hợp lệ; ghi nhận đi muộn và áp dụng `latePenaltyPct`. Sau khi checkout hợp lệ, ca được cộng lương và trừ phạt theo quy tắc tại mục 6.
- Không nhầm trạng thái này với hạn checkout: hết cửa sổ checkout thì không được tự checkout ca cũ nữa.
- Ca quên chấm công không đủ điều kiện đổi ca; nhân viên có thể gửi đơn để admin/quản lý xem và nhập giờ trên ca theo mục 5.4.
- Giữ cấu hình ân hạn/phạt đi muộn; `gracePeriodMinutes` mặc định 0 để ví dụ 17:01 check-in cho ca bắt đầu 17:00 bị phạt. Nếu quản lý cấu hình ân hạn khác 0 cho chi nhánh thì BE xét phạt theo cấu hình của chi nhánh nhân viên đó; nhãn Quên chấm công không tự tạo khoản phạt khi chưa có chấm công hợp lệ.

### 5.3. Hạn checkout và hành vi nút

```text
Hạn checkout = thời điểm kết thúc ca được áp dụng từ ca mẫu
               + checkoutWindowMinutes
```

Giờ kết thúc đã được áp dụng nằm ở `Shift.scheduledEnd`. Ca qua đêm phải cộng đúng ngày kết thúc; không so giờ đơn thuần trong cùng ngày.

Ví dụ: ca kết thúc lúc **17:00**, `checkoutWindowMinutes = 60`:

| Thời điểm/trạng thái | Hành vi |
|---|---|
| Đã check-in, chưa checkout và chưa tới 18:00 | Có thể checkout ca đó nếu các điều kiện còn lại hợp lệ. |
| Từ 18:00, ca vẫn thiếu checkout | Không còn được checkout ca cũ; nút trở về **Chấm công** để dùng cho ca khác hợp lệ. |
| Ca cũ đã hết hạn | Giữ nguyên check-in và `checkOut = null`; không có lương ca đó. |

- Thiếu checkout của ca cũ không được chặn check-in một ca mới hợp lệ.
- Trở về nút Chấm công không cho phép check-in lại chính ca cũ.
- Không tự checkout, tự điền giờ kết thúc, xóa check-in hoặc cộng lương khi hết hạn.
- Trạng thái nút là kết quả BE/FE xác định từ ca, thời gian và chấm công; không thêm cột DB để lưu nhãn nút. Work trả kèm `attendanceState` (`not_started`, `missed_checkin`, `working`, `missed_checkout`, `completed`) và cờ `canCheckout` cho từng ca; hết hạn checkout thì `canCheckout` phải là `false`.

### 5.4. Admin/quản lý nhập giờ trên ca sau khi nhận đơn

- Nhân viên gửi đơn đề nghị; admin hoặc quản lý có thẩm quyền xem và xử lý đơn.
- Người có quyền vào chính ca làm việc hiện có, sử dụng thao tác chấm công của ca để chọn giờ check-in và giờ checkout.
- Khi lưu, cập nhật bản ghi `Attendance` duy nhất của ca hoặc tạo bản ghi đó nếu chưa có; không tạo thêm lượt chấm công cho cùng ca.
- Giờ ra phải không trước giờ vào; nhân viên và ngày của chấm công phải khớp ca, bao gồm ca qua đêm.
- Sau khi đã lưu đủ hai giờ hợp lệ, ca được tính là hoàn thành chấm công và được đưa vào lương tích lũy, dù nhân viên tự chấm hay admin/quản lý nhập giờ.
- Ca thiếu checkout do hết hạn vẫn không có lương cho tới khi được người có quyền thực sự nhập đủ giờ trên ca. Hết hạn checkout chặn nhân viên tự checkout, không ngăn admin/quản lý thực hiện thao tác xử lý này.
- Duyệt đơn không tự tạo checkout; gửi đơn, duyệt đơn và lưu giờ trên ca là các bước khác nhau.
- Ghi nhật ký người thực hiện thao tác bằng `Backlog`; không tạo bảng, cờ trên ca hay một phân hệ “chấm công hộ” riêng.

### 5.5. Cấu hình còn sử dụng

Cấu hình chấm công lưu theo chi nhánh (`AttendanceConfig.branchId`): row có `branchId` áp dụng riêng cho chi nhánh đó; row `branchId = NULL` là cấu hình mặc định toàn hệ thống. Khi chấm công, BE xét theo thứ tự: row chi nhánh của nhân viên → row mặc định → giá trị mặc định trong code. Admin sửa được cấu hình mặc định và mọi chi nhánh; manager chỉ xem/sửa cấu hình chi nhánh mình (BE chặn request khác chi nhánh); staff chỉ đọc cấu hình chi nhánh mình.

| Trường trong `AttendanceConfig` | Vai trò |
|---|---|
| `gracePeriodMinutes` | Khoảng ân hạn khi xét đi muộn, mặc định 0 phút. |
| `earlyCheckinMinutes` | Khoảng cho phép check-in sớm. |
| `allowDoubleCheckin` | Bật/tắt chấm công kép giữa hai ca liền nhau; mặc định tắt. |
| `doubleCheckinEarlyMinutes` | Số phút cho phép chấm kép sớm trước mốc nối, mặc định 0. |
| `doubleCheckinLateMinutes` | Số phút cho phép chấm kép trễ sau mốc nối, mặc định 0. |
| `checkoutWindowMinutes` | Số phút sau giờ kết thúc ca còn được checkout. |
| `latePenaltyPct` | Tỷ lệ phạt đi muộn. |
| `earlyLeavePenaltyPct` | Tỷ lệ phạt về sớm. |
| `shiftSwapMode` | Quản lý duyệt hoặc nhân viên đích chấp nhận đổi ca. |
| `requireReasonSwap` | Cho biết chi nhánh có bắt giải trình khi đổi ca hay không; lý do được để trống (chốt 14/11/2026) nên BE không chặn đơn thiếu lý do. |

Không giới hạn số lần phạt/đổi ca mỗi tháng. Không còn cấu hình phạt tiền cố định, tự checkout, nhiều lượt vào–ra hoặc trừ tiền trực tiếp khi checkout.

- `earlyCheckinMinutes` mặc định 15 phút theo Prisma; đây là thời gian cho phép thao tác, không phải khoảng ân hạn đi muộn.
- `gracePeriodMinutes` mặc định 0; `checkoutWindowMinutes` mặc định 60. Hai giá trị này có mục đích riêng, không dùng thay cửa sổ sớm/trễ của chấm công kép.
- Cấu hình không làm phát sinh thao tác chấm công tự động. Một lần bấm chấm công kép hợp lệ chỉ hoàn tất ca trước và mở chấm công ca sau.

## 6. Lương tích lũy và phạt

### 6.1. Điều kiện được cộng lương

Chỉ ca có đủ **check-in và checkout hợp lệ** mới được cộng vào lương tích lũy. Check-in đơn lẻ không có lương. Nghỉ/vắng hoặc hết hạn mà chưa checkout không được cộng lương của ca.

Ca không được tính lương do thiếu check-in/checkout chỉ bị loại khỏi tích lũy, không trừ thêm tiền phạt của ca đó vào lương các ca khác.

Duyệt đơn bổ sung công không làm ca thiếu checkout trở thành ca được nhận lương. Admin/quản lý phải thực sự nhập và lưu đủ giờ trên ca theo mục 5.4; khi đó ca mới được tính lương.

### 6.2. Parttime — `hourly`

```text
Giờ thực tế của ca = (checkOut - checkIn) quy đổi ra giờ thập phân
Lương ca trước cộng/trừ = salary × giờ thực tế của ca
```

Ví dụ: `salary = 30.000đ/giờ`, ca hoàn thành 5,0 giờ thì tích lũy **150.000đ** trước thưởng/phạt. 5,5 giờ là 5 giờ 30 phút.

### 6.3. Fulltime — `monthly`

```text
Lương một công = salary / standardWorkDays
Một ca hoàn thành = một công
Lương tích lũy = tổng lương của các ca hoàn thành hợp lệ
```

Ví dụ: lương thỏa thuận **7.800.000đ**, **26 công/tháng** thì mỗi ca hoàn thành tích lũy **300.000đ**. Hoàn thành 20 ca thì tích lũy **6.000.000đ** trước thưởng/phạt.

Không tự lấy nguyên mức lương thỏa thuận làm tiền thực nhận. Không chia lương cho 26 cố định hoặc cho số ngày lịch của tháng; dùng số công đã thỏa thuận của nhân viên.

### 6.4. Phạt theo phần trăm

| Kiểu lương | Cơ sở tiền để áp dụng tỷ lệ phạt |
|---|---|
| Parttime | Tiền ca trước cộng/trừ: `salary × giờ thực tế của ca`. |
| Fulltime | Tiền một công: `salary / standardWorkDays`. |

```text
Tiền phạt đi muộn = cơ sở tiền × latePenaltyPct / 100
Tiền phạt về sớm = cơ sở tiền × earlyLeavePenaltyPct / 100
```

- Đi muộn được đối chiếu với giờ bắt đầu của ca và `gracePeriodMinutes`.
- Về sớm được đối chiếu với giờ kết thúc của ca, không dùng mốc 17:00 cố định cho mọi ca.
- Nếu có cả hai vi phạm, cộng hai khoản phạt; không tính lặp cùng một vi phạm.
- Không giới hạn số lần phạt trong tháng và không thay tỷ lệ bằng tiền phạt cố định.
- `Attendance.penaltyAmount` lưu số tiền phạt đã quy đổi; `penaltyNote` giải thích khoản phạt. Kết quả lượt chấm công là `present`, `late`, `early_leave` hoặc `late_early_leave`, do BE ghi lúc chấm công, không có giá trị mặc định.
- Không trừ số dư hoặc thực hiện thanh toán lúc checkout. Tiền phạt được tổng hợp khi tính lương cuối tháng.
- Admin được nhập trực tiếp tổng phạt của một phiếu lương chưa xác nhận (`Payslip.penaltyOverride`) kèm lý do (`adjustmentReason`, tối đa 2000 ký tự). Giá trị nhập tay thay thế toàn bộ tiền phạt suy ra từ chấm công cho tới khi được xóa về `null`; khi đó phiếu quay lại tính phạt theo chấm công. Đây là kênh duy nhất được sửa số phạt, chỉ áp dụng lúc phiếu còn chưa xác nhận và phải có lý do.

### 6.5. Tổng hợp cuối tháng

```text
baseSalary = tổng lương tích lũy hợp lệ trong kỳ, trước thưởng/phạt
netSalary = baseSalary + bonus - totalPenalty
```

- `Employee.salary` là mức thỏa thuận; `Payslip.baseSalary` là kết quả tích lũy, hai giá trị có ý nghĩa khác nhau.
- Không thêm cột lương tích lũy trên `Employee`; BE tổng hợp từ chấm công và lưu kết quả khi lập phiếu lương.
- `Payslip.salaryRate` chụp mức `Employee.salary`, `salaryType` chụp kiểu lương; `workedHours` dùng cho parttime.
- Khi đổi mức lương, phần lương tích lũy chưa xác nhận được tính lại theo mức mới ở lần cập nhật gần nhất: khi admin lưu điều thưởng/phạt và khi xác nhận phiếu. Đọc phiếu luôn trả bản chụp đã lưu, không tính lại lúc đọc; không cần quản lý nhiều mức lương theo ngày hiệu lực trong phạm vi này.
- `Payslip.workedHours` là tổng giờ thực tế của kỳ (chỉ dùng cho `hourly`), `Payslip.workedShifts` là số ca hoàn thành được tính công; cả hai là ảnh chụp để hiển thị và đối soát.
- Các khoản tiền dùng Decimal VNĐ; BE xử lý quy đổi giờ, tỷ lệ và làm tròn, không dùng sai số Float để cộng tiền.
- Tính tiền theo thời lượng gốc từ timestamp, làm tròn tiền ở kết quả cuối; `workedHours` trên phiếu là số giờ hiển thị đã làm tròn, không lấy giá trị này làm nguồn tính tiền.
- Phiếu đã xác nhận giữ kết quả đã được đồng ý, không tự thay đổi theo mức lương hoặc cấu hình mới.

### 6.6. Phân biệt tiền tích lũy, xác nhận và thanh toán

| Thao tác | Kết quả nghiệp vụ | Có trừ số dư tài khoản nguồn không? |
|---|---|---|
| Check-in một ca | Ghi giờ vào; chưa đủ điều kiện cộng lương. | Không. |
| Checkout hợp lệ | Ca đủ công, được đưa vào tổng tích lũy và tính phạt nếu có. | Không. |
| Chấm công kép hợp lệ | Chốt công ca trước; ca sau mới chỉ có giờ vào. | Không. |
| Admin/quản lý lưu đủ giờ trên ca | Ca được tính công như ca hoàn thành thông thường. | Không. |
| Gửi/duyệt đơn nghỉ phép, tạm ứng, bổ sung công | Ghi đề nghị và kết quả xử lý; không tự thay đổi công hoặc lương. | Không. |
| Duyệt hoặc chấp nhận đơn đổi ca | Lịch phân ca được đổi theo đề nghị (mục 7.3); không ghi/chấm công/thanh toán thay. | Không. |
| Xác nhận phiếu lương | Ghi người/thời điểm xác nhận, phiếu đủ điều kiện chi. | Không. |
| Chi thành công phiếu đã xác nhận | Ghi lệnh chi, liên kết phiếu và chuyển sang đã thanh toán. | Có, đúng một lần cho lệnh chi. |

## 7. Đơn từ và đổi ca

### 7.1. Các loại đơn

| `Request.type` | Nội dung và dữ liệu nghiệp vụ |
|---|---|
| `leave` | Xin nghỉ theo ngày, có `leaveStartDate`, `leaveEndDate`. |
| `advance` | Xin tạm ứng, có `advanceAmount > 0`. |
| `shift_swap` | Xin đổi ca/nhờ làm thay, có ca nguồn và nhân viên đích; ca đích khi đổi hai ca. |
| `work_supplement` | Đơn đề nghị bổ sung công, có ca liên quan và giờ đề nghị. |

- Trạng thái chung: `pending`, `approved`, `rejected`.
- Lưu người xử lý, thời điểm xử lý và ghi chú; lấy danh tính từ phiên đã xác thực.
- Chỉ điền các trường nghiệp vụ phù hợp loại đơn; kiểm tra khoảng ngày, giờ và số tiền hợp lệ. Trường của loại đơn khác bị từ chối bằng 400, không âm thầm bỏ qua.
- Đơn chỉ nhìn thấy bởi người gửi, người được nhờ, quản lý đúng chi nhánh chụp trên đơn và admin. Tài khoản chưa gắn hồ sơ nhân sự hoặc quản lý chưa được gán chi nhánh thì danh sách đơn rỗng, không phải lỗi.
- Chỉ admin hoặc quản lý đúng chi nhánh xóa đơn, và chỉ khi đơn còn `pending`.
- Có thể thông báo kết quả cho người liên quan; thông báo không tự thực hiện nghiệp vụ được đề nghị.

### 7.2. Đơn nghỉ phép, tạm ứng và bổ sung công chỉ là mẫu đơn

- Tạm ứng chỉ gửi và duyệt đơn; không tự tạo lệnh chi, không tự trừ số tiền ứng khỏi lương cuối tháng.
- Đơn bổ sung công lưu đề nghị và kết quả xử lý; không tự điền check-in/checkout hay cộng lương khi duyệt. Sau khi xem đơn, admin/quản lý vào ca và nhập giờ theo mục 5.4.
- Không diễn giải nút Duyệt đơn của ba loại này thành thao tác tự checkout hay tự thanh toán thay nhân viên.
- Quy tắc này áp dụng cho đơn nghỉ phép, tạm ứng và bổ sung công: gửi/duyệt chỉ xử lý mẫu đơn, không tự nghỉ làm, không tự sửa chấm công, lương hoặc thanh toán.
- Nếu cần thực hiện nội dung của ba loại đơn đó, người có quyền vào chức năng tương ứng và thao tác riêng.
- Ngoại lệ duy nhất là đơn đổi ca: chấp thuận (hoặc người được nhờ đồng ý) là lệnh đổi lịch phân công, thực hiện theo mục 7.3. Đổi ca không chấm công hộ, không cộng lương và không thanh toán.

### 7.3. Hai chế độ đổi ca

| Chế độ | Người chấp thuận |
|---|---|
| `manager_approval` | Quản lý có thẩm quyền duyệt hoặc từ chối. |
| `mutual_acceptance` | Nhân viên được nhờ đổi/làm thay chấp nhận hoặc từ chối; người tạo đơn đã đồng ý khi gửi. |

- Cấu hình nằm ở `AttendanceConfig.shiftSwapMode`; chụp vào `Request.shiftSwapMode` khi tạo đơn để thay cấu hình không đổi cách xử lý đơn đang chờ.
- Không giới hạn số lần đổi ca mỗi tháng.
- Chỉ đổi ca còn **chưa chấm công và chưa tới giờ bắt đầu**; kiểm tra lại điều kiện khi gửi đơn và khi chấp thuận/cập nhật lịch. Áp dụng cho cả ca nguồn và ca đích nếu đổi hai ca.
- Không đổi ca đã có check-in, đã đến giờ bắt đầu hoặc đã chuyển sang Quên chấm công, dù đơn đã được gửi trước đó.
- Không cho chọn/gửi đề nghị nhờ làm thay hoặc đổi vào ca mà nhân viên đích đã có cùng ngày/ca mẫu. Khi phân hoặc cập nhật ca thực tế cũng phải chặn trùng, không chỉ kiểm tra trên mẫu đơn.
- Khi đơn được chấp thuận, Integration gọi `POST /api/shifts/swap` (chỉ nội bộ, gateway chặn client ngoài) để Work đổi lịch; Integration không đọc/ghi DB của Work.
- Hai ca trong cùng đơn được **hoán đổi người cho nhau**: ca nguồn sang nhân viên đích và ca đích sang người gửi đơn. Khi đơn chỉ có ca nguồn và không có ca đích (nhờ làm thay), ca nguồn được chuyển sang nhân viên đích.
- Duyệt và đồng ý dùng cùng một bộ điều kiện đã kiểm tra lúc gửi: ca còn chưa tới giờ bắt đầu, chưa chấm công, ca đích hợp lệ và nhân viên đích không trùng ca đã có.
- Thứ tự ghi bắt buộc: kiểm tra điều kiện, đổi lịch ở Work **trước**, rồi mới ghi `approved` lên đơn. Nếu đổi lịch thất bại thì đơn vẫn `pending`, không sinh thông báo kết quả; người duyệt sửa đề nghị rồi duyệt lại. Không được ghi `approved` trước rồi mới đổi lịch.
- Rejected/decline không chạm vào lịch. Trạng thái `approved` của đơn đi trước ngày đổi ca là bình thường; người có quyền phải đổi lịch lại ở lần duyệt sau nếu ca đã chấm công hoặc đã tới giờ.
- Trạng thái triển khai 14/11/2026: đã có code — `WorkService.swapShiftEmployees` (endpoint nội bộ `POST /api/shifts/swap`) hoán đổi hai chiều hoặc chuyển một chiều khi nhờ làm thay, còn `IntegrationService.reviewRequest` gọi Work trước rồi mới ghi `approved`. Đơn `mutual_acceptance` do nhân viên đích phản hồi nên lệnh đổi lịch chạy bằng token nội bộ, không bằng vai trò người bấm nút.

## 8. Phiếu lương, số dư và chi lương SOAP

### 8.1. Phiếu lương

- Mỗi nhân viên chỉ có một phiếu cho một kỳ `YYYY-MM`, unique `(employeeId, month)`.
- Payroll lấy hồ sơ lương từ Organization và dữ liệu chấm công từ Work qua HTTP, tổng hợp theo mục 6.
- Phiếu lưu bản chụp kết quả, thưởng, tổng phạt, thực lĩnh, kiểu lương, đơn giá, số giờ và số ca hoàn thành khi phù hợp.
- Chỉ admin lập phiếu và điều chỉnh thưởng/phạt trên phiếu chưa xác nhận; mỗi nhân viên mỗi kỳ chỉ lập được một phiếu.
- Đọc phiếu theo phạm vi: nhân viên chỉ phiếu của mình, manager chỉ phiếu của chi nhánh được chụp trên phiếu, admin toàn hệ thống.
- Nhân viên xác nhận phiếu của mình trên Mobile; manager trong phạm vi được giao hoặc admin cũng được xác nhận phiếu của nhân viên.
- Một lần xác nhận của một trong các chủ thể có quyền là đủ; không yêu cầu cả nhân viên lẫn quản lý/admin cùng xác nhận một phiếu.
- Lưu người xác nhận (`confirmedBy`) và thời điểm xác nhận (`confirmedAt`). Xác nhận phiếu không đồng nghĩa đã trả tiền.
- Luồng trạng thái: **Chưa xác nhận → Đã xác nhận → Đã thanh toán**. Phiếu đã xác nhận giữ kết quả, không tự đổi theo hồ sơ lương mới.
- Người dùng chỉ được chuyển phiếu sang `confirmed`; `paid` do lệnh chi thành công ghi, không có thao tác tự đặt `paid` và không có bước hủy chi.
- Chỉ phiếu đã xác nhận, chưa được thanh toán mới đủ điều kiện đưa vào lệnh chi; BE kiểm tra trước khi chi tiền qua cả REST và SOAP.
- Một phiếu được trả đủ trong một lệnh chi; chưa có thanh toán từng phần.
- Phiếu đã xác nhận có thực lĩnh bằng 0 vẫn được chi: lệnh chi 0đ không trừ số dư nhưng đóng phiếu sang `đã thanh toán`, vì luồng trạng thái không có bước hủy. Thực lĩnh âm không được chi.
- Liên kết `payoutId` cho phép truy vết lệnh chi; không xóa lệnh chi đang được phiếu lương tham chiếu.

### 8.2. Tài khoản nguồn và số dư mô phỏng

- Có mô phỏng số dư bằng `BankAccount.balance` và trừ tiền tài khoản nguồn khi chi lương thành công.
- Phân biệt tài khoản nguồn của doanh nghiệp với tài khoản nhận lương trên `Employee`.
- Tài khoản nguồn lưu ngân hàng, chủ tài khoản, `bankBranch` (ngân hàng/phòng giao dịch nơi mở tài khoản, text tự do) và `branchId` (chi nhánh nội bộ sở hữu tài khoản, UUID của Organization).
- Có chọn tài khoản nguồn chính; tối đa một tài khoản chính toàn hệ thống, BE reset tài khoản chính cũ trong cùng transaction.
- Chỉ admin quản lý tài khoản nguồn, xem danh sách tài khoản và lệnh chi; manager không có quyền này.
- Cập nhật tài khoản chỉ sửa được tên chủ, ngân hàng, `bankBranch`, chi nhánh sở hữu và cờ tài khoản chính; không sửa được số tài khoản hay số dư qua bản ghi cấu hình. Tài khoản đã có lệnh chi thì không xóa được.
- Tổng tiền và số người hưởng của lệnh chi phải phù hợp tập phiếu lương được trả; số người hưởng đếm theo nhân viên, nhiều kỳ của cùng một người tính một lần. Không được gửi một lệnh chi chứa cùng một phiếu hai lần.
- Thiếu số dư thì từ chối chi, không tạo kết quả thanh toán thành công giả.

### 8.3. Chống chi trùng và SOAP

- Mỗi lệnh chi có `idempotencyKey` duy nhất. Gửi lại cùng lệnh trả kết quả cũ, có thể kèm `deduped: true`; không trừ tiền lần hai.
- BE phải bảo đảm kiểm tra số dư, trừ tiền và ghi nhận lệnh chi nhất quán trong DB Payroll.
- `Payout.id` là UUID; mã nghiệp vụ `bankReference` có dạng `GD-...` theo schema mới.
- Integration cung cấp SOAP chi lương và WSDL, chuyển yêu cầu tới Payroll qua HTTP.
- SOAP trả XML hoặc `soap:Fault`; Gateway không bọc XML bằng envelope JSON.
- REST thiếu số dư dùng `INSUFFICIENT_FUNDS`, HTTP 422; SOAP biểu diễn lỗi bằng Fault tiếng Việt.
- Thân SOAP bị từ chối trước khi chạm Payroll và trả `soap:Fault` kèm HTTP 400 khi: thiếu `idempotencyKey`, `debitAccount` hoặc `totalAmount`; `payslipId` không phải UUID hoặc danh sách rỗng; tổng tiền âm; có DTD hoặc thực thể ngoài. Không có đường nào trả 500 cho lỗi đầu vào.
- Không kết nối được Payroll hoặc Payroll từ chối thì SOAP trả Fault với HTTP 502 kèm thông điệp gốc, không trả `timeout: true` giả thành công.
- `POST /soap/payroll` tạo lệnh trừ tiền thật nên chỉ admin qua Gateway; `GET /soap/payroll?wsdl` để mở cho client ngân hàng.
- Cấu hình gateway SOAP có kiểm tra kết nối (`ping`): gọi WSDL, ghi trạng thái `healthy`/`degraded`/`down`, thời điểm kiểm tra và thời gian phản hồi; đọc/ghi cấu hình và `ping` chỉ admin.
- Giữ cấu hình SOAP: endpoint/WSDL, tên dịch vụ, chế độ bảo mật, trạng thái, lần kiểm tra, thời gian phản hồi và hạn chứng thư theo schema. Port nằm trong URL khi cần, không có trường port/channel riêng.

## 9. Công việc, thông báo và nội dung nội bộ

### 9.1. Công việc

- Quản lý tiêu đề, mô tả, người nhận, người giao, chi nhánh, hạn, ưu tiên và trạng thái công việc.
- Công việc chung gắn chi nhánh, ngày làm và ca mẫu. Nhân viên vào các ca được phân có cùng bộ thông tin này đều thấy công việc chung đó.
- Công việc chung chỉ có một kết quả/trạng thái: một người hoàn thành thì những người khác cũng thấy đã xong. Người nộp minh chứng được ghi nhận trên cùng task.
- Công việc riêng gán thẳng cho nhân viên, hiển thị ngay cho người đó; không có bước nhận/chấp nhận công việc. Mỗi người có task và kết quả hoàn thành riêng; một người xong không làm task riêng của người khác hoàn thành.
- Giao cùng một nội dung riêng cho nhiều người thì từng người có bản ghi riêng; không dùng một task chung có kết quả chung để thay cho công việc cá nhân.
- Task chung để `assignedTo = null` và có `branchId`, `workDate`, `templateId`; task riêng có `assignedTo`. BE kiểm tra đủ thông tin và điều kiện hiển thị theo ca.
- Có yêu cầu ảnh minh chứng, URL ảnh, người nộp và thời điểm nộp. Khi `requirePhoto` bật, nhân viên phải có ảnh mới được hoàn thành; admin/manager được duyệt đặc cách hoàn thành khi chưa có ảnh (UI web có bước xác nhận).
- Trạng thái công việc là `pending`, `in_progress`, `done`; tạo task mới không được đặt sẵn `done`. Hạn công việc là mốc thời gian có múi giờ (`dueDate`), client cũ gửi dạng ngày thì BE hiểu là nửa đêm giờ Việt Nam.
- Nhân viên được giao chỉ sửa được trạng thái và ảnh minh chứng của task mình; không sửa tiêu đề, hạn, người giao hay độ ưu tiên. Task đã `done` thì người không đặc quyền không sửa lại được.
- Manager và admin được đổi mọi trường của task trong phạm vi; manager không giao việc sang chi nhánh khác.
- Xóa task kèm theo xóa tệp minh chứng đã lưu ở Integration.
- Các tên chụp để hiển thị không dùng làm khóa hoặc kiểm tra quyền.
- Không lưu `sourceType` hay email người nhận riêng; xác định người giao/ca bằng ID.

### 9.2. Thông báo và nhắc giờ

- Có thông báo và nhắc check-in/checkout theo các mốc người dùng bật trong `NotificationSetting`.
- Giữ các lựa chọn nhắc trước ca, trước checkout, nhắc thiếu checkout, âm thanh/rung và cảnh báo theo schema.
- Nhắc giờ không tự thực hiện chấm công và không kéo dài hạn checkout.
- Mỗi `Notification` có một người nhận; gửi nhiều người tạo nhiều dòng để `isRead` độc lập. Một lượt gửi nhiều người được ghi nguyên tâm: lỗi giữa chừng không để lại danh sách người nhận dở dang.
- Gửi thông báo cho người khác là thao tác quản lý (manager/admin); nhân viên không tự bắn thông báo và không mạo được danh tính người gửi.
- Nhắc chấm công chỉ sinh khi BE xác định được trạng thái chấm công của ca; không xác định được thì không nhắc, và mỗi mốc chỉ nhắc một lần cho một ca.
- Nội dung thông báo có phạm vi một chi nhánh hoặc toàn thể; BE chỉ tạo dòng người nhận thuộc phạm vi đó, không gửi lẫn chi nhánh. Trạng thái đọc vẫn riêng từng người.
- `requestId` liên kết thật với đơn trong cùng DB; `requestType` hỗ trợ điều hướng, `metadata` chứa chi tiết hiển thị.
- Người gửi được xác định bằng `senderId`; không lưu thêm vai trò người gửi trên thông báo.

### 9.3. Bảng tin, quy định và Wi-Fi

- Có bảng tin, ghim bài, tác giả, ngày đăng và phạm vi toàn công ty/chi nhánh.
- Có quản lý quy định: mã, loại, nội dung, trạng thái, phạm vi, ngày hiệu lực/hết hạn, phiên bản, ghim và tệp đính kèm.
- Thông báo, bảng tin và quy định đều phải có phạm vi **một chi nhánh hoặc toàn thể**. Người thuộc chi nhánh được xem nội dung chi nhánh đó và nội dung toàn thể; admin có toàn quyền.
- Thông báo đã tạo cho người nhận vẫn chỉ người nhận đó được đọc/đánh dấu đọc theo quyền; phạm vi toàn thể không biến một dòng thông báo thành dòng dùng chung cho mọi người.
- News/Notification dùng `branchId`, Regulation dùng `scopeBranchId`; null biểu thị phạm vi toàn thể. Phạm vi không chỉ là bộ lọc giao diện mà phải được BE áp dụng khi trả dữ liệu.
- Tệp quy định là bản ghi `FileAsset` có relation; số tệp được đếm từ dữ liệu, không lưu cột đếm thủ công.
- Danh mục Wi-Fi theo chi nhánh được dùng cho bước kiểm tra chấm công tại mục 5.1, không chỉ để hiển thị cấu hình.

## 10. Tệp và nhật ký

- Tệp upload được quản lý tại Integration bằng `FileAsset`, lưu trên ổ đĩa theo cấu hình lưu trữ nội bộ.
- Đăng ký khóa tệp, đường dẫn, MIME, dung lượng, người tải, chi nhánh và quan hệ với quy định khi có.
- Ảnh CCCD, ảnh đại diện, minh chứng và tài liệu được tham chiếu bằng URL/ID phù hợp; không cần lưu kích thước ảnh trong DB.
- `Backlog` ghi người thao tác, vai trò tại thời điểm thao tác, hành động, module, loại và ID đối tượng, tên hiển thị, thời điểm.
- Nhãn hành động được hiển thị từ mã `action`; không có cột `actionLabel` riêng.
- Không thêm `updatedAt`; giữ các mốc nghiệp vụ cụ thể như `createdAt`, `lastSeenAt`, `reviewedAt`, `completedAt`, `issuedAt`.

## 11. Quy ước dữ liệu và trách nhiệm triển khai

### 11.1. Nguồn thiết kế dữ liệu

Các file dưới đây là nguồn định nghĩa bảng, trường, kiểu dữ liệu và relation; không sao chép cả schema vào tài liệu để tránh lệch phiên bản:

- [Identity schema](../backend/identity-service/prisma/schema.prisma)
- [Organization schema](../backend/organization-service/prisma/schema.prisma)
- [Work schema](../backend/work-service/prisma/schema.prisma)
- [Payroll schema](../backend/payroll-service/prisma/schema.prisma)
- [Integration schema](../backend/integration-service/prisma/schema.prisma)

UUID là khóa kỹ thuật; mã `branchCode`, `departmentCode`, `positionCode`, `employeeCode`, `templateCode`, `regulationCode`, `bankReference` là mã nghiệp vụ. Không dùng slug hoặc tên làm khóa liên kết.

Tiền dùng `Decimal`; ngày nghiệp vụ dùng `@db.Date`; giờ check-in/out dùng `Timestamptz`; giờ ca mẫu là `HH:mm`; kỳ lương là `YYYY-MM`. Định dạng ngày trên giao diện không thay kiểu lưu trữ trong DB.

### 11.2. Phân chia trách nhiệm

| Thành phần | Trách nhiệm |
|---|---|
| Schema/DB | Lưu dữ liệu, khóa, unique, FK và các CHECK/index được triển khai sau. |
| BE | Phân quyền, kiểm tra Wi-Fi/thời gian, quy trình đơn, tính giờ, tích lũy lương, quy đổi phạt và thanh toán. |
| FE/Mobile | Kiểm tra Wi-Fi ngay khi bấm nút, hiển thị nút/ca/lương/thông báo dựa trên dữ liệu hợp lệ từ BE. |

Comment Prisma là ghi chú thiết kế, không chạy công thức và không tự tạo logic BE.

### 11.3. Phần phải thực hiện khi tái cấu trúc

- Cập nhật hợp đồng API và model Web/Mobile theo schema mới; không lấy payload/seed cũ làm yêu cầu nghiệp vụ.
- Các `Work_Flow` cũ ghi nhận bản triển khai trước; phần hợp đồng và tiến độ tái cấu trúc cần được cập nhật khi thực hiện từng service, không coi checkbox cũ là đã xong thiết kế mới.
- Migration `20261112230000_hrm_uuid_redesign` của từng service tạo các CHECK/index ngoài khả năng biểu diễn của Prisma. Migration giữ lịch sử và dừng nếu bảng nghiệp vụ có dữ liệu; không dùng reset.
- Sau migration redesign còn các migration bổ sung đã áp dụng, mỗi cái là một quyết định nghiệp vụ: Work có `20261113120000_shift_partial_unique_cancelled` (UNIQUE PARTIAL bỏ ca đã hủy), `20261113170000_shift_branch_contract` (thêm `Shift.branchId`, bỏ cột `note`), `20261113180000_task_deadline_timestamp` (`Task.dueDate` thành Timestamptz), `20261114020000_attendance_config_per_branch` (cấu hình theo chi nhánh); Payroll có `20261113100000_payout_allow_zero_amount`, `20261113150000_payslip_manual_penalty`, `20261113190000_payslip_worked_shifts`, `20261114030000_bank_account_opening_branch`.
- **Hàm/sequence sinh mã nghiệp vụ hiện chỉ có ở 2/5 service**: Organization (`branch_code_seq`, `employee_code_seq`) và Payroll (`payout_ref_seq` → mã `GD-`). Work (`templateCode`), Integration (`regulationCode`) và Identity **không có sequence**; các mã này do caller gửi lên và tầng service kiểm tra trùng ở mã ứng dụng, đồng thời DB vẫn có unique index tương ứng. Nếu sau này muốn sinh mã ở DB cho các service còn lại thì phải bổ sung sequence **và** đổi hợp đồng API theo `BUSINESS.md` §11.1 — không tự ý đổi một phía.
- **Nợ drift ràng buộc**: Prisma không biểu diễn được `CHECK` và partial unique index trong `schema.prisma`, nên các ràng buộc đó **chỉ tồn tại trong file `migration.sql`**. Chỉ dùng `prisma migrate deploy` với file migration đã viết sẵn; **cấm `prisma migrate dev`/`migrate diff`** trên DB có dữ liệu vì sẽ sinh migration `DROP` toàn bộ ràng buộc. Kiểm tra trước khi deploy bằng `npm.cmd run prisma:check` tại `backend/`.
- Các CHECK gồm điều kiện số công, kiểu lương/loại đơn, số tiền, thứ tự thời gian, xác nhận phiếu và giới hạn tỷ lệ.
- Chỉ `SoapGatewayConfig` còn CHECK buộc `isSingleton = true`, kết hợp unique để bảo đảm tối đa một dòng. `AttendanceConfig` đã chuyển sang một row cho mỗi chi nhánh, row mặc định có `branchId = NULL`; Postgres coi NULL là phân biệt trong unique index nên "chỉ một row mặc định" do WorkService bảo đảm khi ghi, không phải do DB.
- Tài khoản nguồn có partial unique index để bảo đảm tối đa một tài khoản chính.
- Nội dung SQL nằm trong migration; comment Prisma chỉ mô tả nghiệp vụ. Kiểm chứng DB thật yêu cầu Postgres đang chạy, tách biệt với test memory/HTTP.
- `Payslip.status` dùng enum; tên giá trị trong Prisma là `pending_confirmation`, `confirmed`, `paid`, ánh xạ xuống DB thành các trạng thái tiếng Việt tương ứng.
- Không đánh dấu hoàn tất tái cấu trúc chỉ vì schema qua kiểm tra cú pháp.

### 11.4. Trách nhiệm nghiệp vụ khi sửa từng service

| Thành phần | Phần cần bảo đảm theo đặc tả |
|---|---|
| Gateway | Quyền Web/Mobile và nhóm route theo vai trò; định tuyến đúng; giữ nguyên SOAP XML. |
| Identity | Tài khoản, vai trò, liên kết hồ sơ, phiên/thiết bị và hạn token; không mang nghiệp vụ hồ sơ/lương vào xác thực. |
| Organization | Danh mục và hồ sơ, CCCD, chi nhánh nhân viên, một mức salary và số công fulltime; cung cấp dữ liệu qua HTTP. |
| Work | Phân/đăng ký ca không trùng, kiểm tra Wi-Fi trước danh sách ca, chấm công thường/kép/nhập giờ theo quyền, công việc chung/riêng và dữ liệu công để Payroll tổng hợp. |
| Payroll | Tích lũy lương theo công hợp lệ, phạt phần trăm và đường ghi đè phạt thủ công có lý do, phiếu và xác nhận, số dư mô phỏng, lệnh chi đủ điều kiện kể cả phiếu 0đ và chống trùng. |
| Integration | Đơn nghỉ phép/tạm ứng/bổ sung công chỉ là mẫu đơn; duyệt đơn đổi ca phải đổi lịch qua Work trước khi ghi `approved`; thông báo/nhắc giờ, nội dung theo phạm vi, Wi-Fi, tệp, nhật ký và SOAP chuyển yêu cầu chi tới Payroll. |

Đây là phân chia trách nhiệm để triển khai, không bổ sung endpoint hoặc thay đổi hợp đồng API ngầm. Các service cần thống nhất hợp đồng HTTP trước khi nối luồng; không giữ đồng thời các tên trường/logic cũ và mới chỉ để tránh sửa client nội bộ.

## 12. Tình huống nghiệm thu nghiệp vụ

Các tình huống sau là yêu cầu kiểm chứng khi triển khai, chưa phải báo cáo test đã chạy:

| Tình huống | Kết quả cần đạt |
|---|---|
| Bấm Chấm công khi khác Wi-Fi hợp lệ | Báo lỗi ngay, không mở màn chọn ca. |
| Đổi sang Wi-Fi không hợp lệ sau khi đã mở màn ca | BE từ chối ghi nhận chấm công. |
| Check-in/checkout lặp trong cùng ca | Bị từ chối; không tạo dòng hoặc cộng lương hai lần. |
| Hai ca 07:00–12:00 và 12:00–17:00, bật kép sớm/trễ 5 phút; bấm 11:58 hoặc 12:03 | Nếu các điều kiện hợp lệ, ghi ra ca trước và vào ca sau cùng lúc 12:00; không phạt tại mốc nối. |
| Chấm công kép khi ca sau chưa được phân, đã check-in hoặc ca trước đã hết hạn checkout | Không thực hiện kép, không ghi đè hay chỉ ghi một nửa thao tác. |
| Sau chấm công kép nhưng ca sau chưa checkout | Chỉ ca trước đủ cặp vào–ra được tích lũy lương; ca sau chưa có lương. |
| Check-in thông thường sớm hơn mức `earlyCheckinMinutes` cho phép | Từ chối check-in sớm; không nhầm với khoảng riêng của chấm công kép. |
| Phân thêm, nhờ làm thay hoặc đổi vào ca nhân viên đã có cùng ngày/ca mẫu | Bị từ chối, không tạo ca/công trùng. |
| Phân ca với `branchId` khác chi nhánh trong hồ sơ nhân viên | Bị từ chối; ca được lưu với bản chụp chi nhánh của nhân viên. |
| Hủy một ca rồi phân lại đúng ca mẫu đó trong ngày | Thành công; ca đã hủy không tính vào chống trùng. |
| Ca kết thúc 17:00, cửa sổ 60 phút; checkout lúc 17:30 | Có thể checkout nếu các điều kiện khác hợp lệ. |
| Cùng ca nhưng từ 18:00 vẫn thiếu checkout | Không cho checkout ca cũ, nút về Chấm công, giữ `checkOut = null`, lương ca bằng 0. |
| Ca cũ hết hạn thiếu checkout, nhân viên có ca mới hợp lệ | Cho chấm ca mới; không tự hoàn tất ca cũ. |
| Parttime 30.000đ/giờ, hoàn thành 5 giờ | Tích lũy 150.000đ trước thưởng/phạt; phạt 10% tương ứng 15.000đ. |
| Fulltime 7.800.000đ, thỏa thuận 26 công, hoàn thành một ca | Tích lũy 300.000đ trước thưởng/phạt; phạt 10% tương ứng 30.000đ. |
| Chỉ check-in, không checkout | Không cộng lương ca. |
| Ca không có lương vì thiếu chấm công | Không trừ thêm tiền phạt ca đó vào lương các ca khác. |
| Ca bắt đầu 17:00, sang 17:01 chưa check-in | Hiển thị Quên chấm công; vẫn cho check-in muộn nếu hợp lệ, nhưng không được đổi ca. |
| Ca bắt đầu 17:00, check-in lúc 17:01, ân hạn 0 phút | Ghi nhận đi muộn và phạt theo `latePenaltyPct`; đủ checkout mới tính lương ca. |
| Checkout trước giờ kết thúc ca | Áp dụng `earlyLeavePenaltyPct`; nếu đồng thời đi muộn thì cộng hai khoản phạt. |
| Gửi đơn đổi ca trước giờ bắt đầu nhưng xử lý sau giờ bắt đầu | Từ chối đổi ca vì không còn đủ điều kiện. |
| Thay đổi mức lương khi kỳ chưa xác nhận | Phiếu chưa xác nhận được tính lại theo mức mới khi lưu điều chỉnh hoặc khi xác nhận; đọc phiếu không tự tính lại. |
| Đơn bổ sung công được duyệt | Lưu kết quả đơn, không tự ghi checkout/cộng lương. |
| Admin/quản lý vào ca, chọn và lưu đủ giờ vào/ra hợp lệ | Ca được tính hoàn thành và có lương tương ứng; vẫn chỉ một Attendance, có nhật ký người thao tác. |
| Đơn tạm ứng được duyệt | Không tự trừ lương cuối tháng hoặc tạo lệnh chi. |
| Duyệt/chấp nhận đơn đổi ca ở mỗi chế độ | Quản lý duyệt hoặc nhân viên đích đồng ý thì lịch đổi theo đề nghị; đổi lịch thất bại thì đơn còn `pending` và không có thông báo. |
| Staff đăng nhập Web hoặc gọi chức năng quản trị | Bị từ chối; staff chỉ đăng nhập Mobile và dùng nghiệp vụ cá nhân. |
| Manager gọi chức năng quản trị hệ thống | Bị từ chối; admin dùng được trên Web và Mobile. |
| Nhân viên/manager đúng phạm vi/admin xác nhận phiếu | Ghi người và thời điểm xác nhận; chưa trừ tiền tài khoản. |
| Chi lương cho phiếu chưa xác nhận | Bị từ chối ở cả REST và SOAP. |
| Một người hoàn thành task chung của ca | Những người cùng ca thấy đã hoàn thành, dùng chung một kết quả. |
| Một người hoàn thành task riêng | Không làm hoàn thành task riêng của người khác. |
| Giao task cá nhân | Gán trực tiếp và hiển thị cho nhân viên, không đợi người đó nhận việc. |
| Xem thông báo, bản tin, quy định | Chỉ thấy phạm vi chi nhánh được phép và toàn thể, trừ admin có toàn quyền. |
| Thông báo gửi nhiều người, một người đọc | Chỉ dòng của người đó chuyển trạng thái đã đọc. |
| Chi lương cho phiếu đã xác nhận có thực lĩnh 0đ | Lệnh chi 0đ không trừ số dư, phiếu chuyển `đã thanh toán`; thực lĩnh âm bị từ chối. |
| Admin nhập tổng phạt thủ công kèm lý do trên phiếu chưa xác nhận | Phiếu dùng số phạt nhập tay; xóa về null thì phạt tính lại từ chấm công. |
| SOAP thiếu `idempotencyKey`/`debitAccount`/`totalAmount` | `soap:Fault` kèm HTTP 400, không tạo lệnh chi, không trả 500. |
| Chi lương đủ số dư | Trừ đúng số tiền trong tài khoản mô phỏng, ghi lệnh chi và liên kết phiếu. |
| Gửi lại cùng lệnh chi/idempotency key | Trả kết quả cũ, không trừ tiền lần hai. |
| Chi lương thiếu số dư | Từ chối; REST trả lỗi tương ứng, SOAP trả Fault. |
