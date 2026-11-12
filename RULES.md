# RULES.md — Bộ Quy Tắc Bất Biến Của Dự Án HRM Enterprise

> Toàn bộ quy tắc trong file này là bắt buộc và không được phép phá vỡ.
> Mọi code, API, schema và pull request vi phạm quy tắc đều phải bị từ chối.
> File này được viết hoàn toàn bằng tiếng Việt.

---

## 1. Quy Tắc Đơn Khách Hàng (Single-Tenant)

1. Đây là hệ thống nội bộ của một doanh nghiệp duy nhất, triển khai on-premises.
2. **Nghiêm cấm tuyệt đối**: không được thêm `tenantId`, `tenantSlug` hay bất kỳ khái niệm đa khách hàng nào vào schema CSDL, route URL, payload request hay logic nghiệp vụ.
3. Mọi đề xuất thêm đa khách hàng phải bị bác bỏ ngay, kể cả khi có lý do mở rộng trong tương lai.

## 2. Quy Tắc Ngôn Ngữ Giao Diện

1. Toàn bộ nhãn, thông báo, placeholder và cảnh báo hiển thị cho người dùng trên Web và Mobile **bắt buộc 100% tiếng Việt**.
2. Cấm để sót tiếng Anh trong giao diện người dùng cuối (cho phép tiếng Anh trong log kỹ thuật, tên biến, comment code).
3. Thông điệp trả về trong envelope API (`message`) phải là tiếng Việt chuẩn hóa doanh nghiệp.

## 3. Quy Tắc Git Và Nhánh

1. Nhánh làm việc chuẩn là `dev`; nhánh sản phẩm là `main`; remote là `PTPMDV`.
2. **Cấm commit trực tiếp vào `main` hoặc `dev`.**
3. Luôn tách nhánh từ `dev` theo mẫu `feat/<ten-service>-<ten-nguoi>` hoặc `fix/<ten-service>-<ten-loi>` (ví dụ `feat/payroll-payouts`, `fix/work-attendance-penalty`).
4. Mỗi pull request chỉ gọn trong một service hoặc một chức năng, sau đó merge về `dev`.
5. Trước khi hoàn thành phải chạy `git pull origin dev` để hòa giải xung đột và kiểm thử lại.

## 4. Quy Tắc Bảo Mật Môi Trường

1. File `.env` thật luôn phải nằm trong `.gitignore`, không bao giờ commit.
2. Chỉ được commit `.env.example` với giá trị mẫu đã khử nhạy cảm.
3. Không hardcode `JWT_SECRET`, mật khẩu, chuỗi kết nối CSDL trong mã nguồn.
4. Cookie phiên chuẩn là `hrm-session` (HttpOnly). Khi dùng cookie qua Gateway phải cấu hình CORS cho phép credentials và chỉ định origin cụ thể, cấm dùng `*`.

## 5. Quy Tắc Envelope HTTP Chuẩn

Mọi controller trên mọi service phải trả đúng một trong hai khuôn sau.

### 5.1. Thành công (HTTP 200, 201)

```json
{
  "data": {
    "id": "e-1001",
    "name": "Nguyễn Văn A",
    "role": "staff"
  },
  "message": "Thao tác thành công"
}
```

### 5.2. Thất bại (HTTP 400, 401, 403, 404, 409, 422, 500)

```json
{
  "error": {
    "code": "EMPLOYEE_NOT_FOUND",
    "message": "Không tìm thấy thông tin nhân sự trên hệ thống",
    "details": null
  }
}
```

### 5.3. Bảng mã lỗi chuẩn

| Mã lỗi | HTTP | Ý nghĩa và khi nào dùng |
|---|---|---|
| `VALIDATION_ERROR` | 400 | Payload không qua kiểm tra schema |
| `UNAUTHORIZED` | 401 | Thiếu hoặc sai token / phiên đăng nhập |
| `FORBIDDEN` | 403 | Đã đăng nhập nhưng không đủ quyền |
| `NOT_FOUND` | 404 | Tài nguyên không tồn tại |
| `DUPLICATE_RESOURCE` | 409 | Trùng lặp (email, cặp employeeId + month, idempotencyKey đã xử lý theo nghĩa khác) |
| `INSUFFICIENT_FUNDS` | 422 | Số dư tài khoản công ty không đủ để chi |
| `INTERNAL_SERVER_ERROR` | 500 | Lỗi hệ thống không lường trước |

Ngoại lệ duy nhất: tuyến `POST /soap/payroll` nhận và trả XML, lỗi trả `soap:Fault` với `faultcode = soap:Client`. Gateway phải chuyển tiếp nguyên vẹn body, header và status SOAP, không được bọc lại thành `{data}` hay `{error}`.

## 6. Quy Tắc Giao Tiếp Giữa Các Service Và Database-Per-Service (Đã Chốt 5 DB)

1. Quyết định đã chốt: **5 database Postgres riêng trên 5 link Supabase riêng**, mỗi service nghiệp vụ 1 DB: `hrm_identity` (4001), `hrm_organization` (4002), `hrm_work` (4003), `hrm_payroll` (4004), `hrm_integration` (4005). Gateway (4000) stateless, không có DB.
2. Gọi liên service **chỉ qua HTTP REST hoặc gRPC**, đặt adapter trong `src/clients/`.
2. **Timeout tối đa 5000ms** cho mọi cuộc gọi HTTP giữa các service.
3. **Cấm import code trực tiếp từ service khác**, ví dụ:

```javascript
// SAI — nghiêm cấm
const { calculateTax } = require("../../payroll-service/src/services/tax");

// ĐÚNG — gọi qua client adapter
const payrollClient = require("../clients/PayrollClient");
const taxResult = await payrollClient.getTaxCalculation(payload);
```

4. Không được để lộ `role` hay `branchSlug` trên URL route của frontend.
5. **Cấm join xuyên cơ sở dữ liệu, cấm foreign key xuyên cơ sở dữ liệu, cấm đọc bảng của service khác trực tiếp.** Tham chiếu chéo lưu UUID (`employeeId`, `branchId`, `shiftId`); cần chi tiết thì gọi HTTP. Payroll lấy `salary`, `salaryType`, `standardWorkDays` từ Organization và chấm công/phạt từ Work; Integration chuyển yêu cầu chi lương SOAP tới Payroll. Không dùng slug hoặc tên làm khóa liên kết.
6. Mỗi service có `prisma/schema.prisma`, migration, seed và `DATABASE_URL` riêng trỏ về link Supabase riêng của mình. Cấm dùng chung schema Prisma hay trỏ 2 service về cùng 1 link DB.

## 7. Quy Tắc Nghiệp Vụ Cốt Lõi

- Admin toàn quyền trên Web và Mobile; manager bị chặn chức năng quản trị hệ thống; staff chỉ đăng nhập Mobile và không được truy cập chức năng admin/manager. BE phải kiểm tra quyền và phạm vi dữ liệu.
- Mọi đơn đề nghị chỉ là mẫu đơn; duyệt/chấp nhận không tự sửa ca, chấm công, lương hoặc thanh toán. Thực hiện nội dung đơn là thao tác riêng ở chức năng tương ứng.
- Task chung theo chi nhánh/ngày/ca mẫu có kết quả chung: một người xong thì cả ca thấy xong. Task riêng có kết quả riêng cho từng người được giao.
- Task riêng gán trực tiếp cho nhân viên, không có bước nhận/chấp nhận việc.
- Thông báo, bảng tin và quy định áp dụng theo một chi nhánh hoặc toàn thể; BE thực thi phạm vi, không chỉ lọc ở giao diện.

### 7.1. Chấm công, lương tích lũy và phạt theo phần trăm

- Nghiệp vụ chi tiết đã chốt tại [BUSINESS.md](BUSINESS.md), dùng để tái cấu trúc BE/FE; không lấy hành vi code cũ thay thế đặc tả này.
- Khi nhân viên bấm Chấm công, kiểm tra Wi-Fi theo chi nhánh ngay trước màn chọn ca; sai Wi-Fi thì báo lỗi và không mở danh sách ca. BE kiểm tra lại điều kiện khi ghi nhận. Không thêm thuộc tính kiểm tra Wi-Fi cho từng ca.
- Đúng Wi-Fi mới được mở các ca trong ngày trên Mobile. `Shift` không lưu `branchId`; không dùng chi nhánh ca để thực hiện bước kiểm tra này.
- Unique `(employeeId, workDate, templateId)` ngăn phân trùng ca. Không cho nhân viên đã có ca được chọn làm thay/đổi vào hoặc được thêm lại ca đó; BE kiểm tra trên đơn và thao tác phân lịch thực tế, kể cả chồng giờ giữa các ca khác mẫu.
- Mỗi ca chỉ một bản ghi chấm công, tối đa một lần vào và một lần ra. Chỉ đủ check-in và checkout hợp lệ mới có lương ca.
- Có cấu hình chấm công kép cho hai ca đã phân liền nhau: trong khoảng sớm/trễ riêng được cấu hình, một thao tác chủ động ghi checkout ca trước và check-in ca sau đúng mốc nối. Ghi cả hai trong một transaction, kiểm tra Wi-Fi/quyền/hạn checkout; không ghi đè lượt đã chấm, không phạt tại mốc nối đã chuẩn hóa. Mỗi ca vẫn chỉ một cặp vào–ra.
- Giữ `earlyCheckinMinutes` cho check-in sớm thông thường; `doubleCheckinEarlyMinutes`/`doubleCheckinLateMinutes` là khoảng riêng cho chấm công kép, bật/tắt bằng `allowDoubleCheckin`.
- Chỉ ca đã được phân mới có thể tính công/lương. Ca không đủ check-in/checkout chỉ không được cộng lương, không trừ thêm tiền phạt ca đó vào các ca khác.
- Ví dụ ca bắt đầu 17:00, sang 17:01 chưa check-in thì hiển thị Quên chấm công; vẫn được check-in muộn khi ca hợp lệ và chịu phạt phần trăm theo cấu hình. Ân hạn mặc định 0 phút. Nhãn này không khóa check-in; việc đổi ca vẫn bị cấm khi đã tới giờ bắt đầu.
- Hạn checkout bằng thời điểm kết thúc ca áp dụng từ ca mẫu cộng `checkoutWindowMinutes`. Từ hạn này, nhân viên không được checkout ca cũ; nút trở về Chấm công cho ca khác. Giữ giờ vào, không tự điền giờ ra, không cộng lương ca thiếu checkout.
- Đơn bổ sung chỉ gửi/duyệt đề nghị. Admin/quản lý vào chính ca hiện có để chọn và lưu giờ vào/ra; khi đủ hai giờ hợp lệ, ca được tính công và lương như bình thường. Không tạo phân hệ/bảng chấm công hộ riêng; duyệt đơn không tự sửa chấm công.
- `Employee` chỉ lưu một mức tiền `salary`: đơn giá giờ nếu `salaryType = hourly`, lương tháng thỏa thuận nếu `monthly`.
- Parttime: lương ca = `salary × giờ thực tế` của ca hợp lệ.
- Fulltime: người quản lý nhập `standardWorkDays > 0` là số công/tháng thỏa thuận; một ca hoàn thành bằng một công. Lương ca = `salary / standardWorkDays`. Không cố định 26 ngày hoặc tự lấy nguyên lương tháng khi thiếu công.
- Phạt theo phần trăm: parttime áp dụng trên tiền ca trước cộng/trừ; fulltime áp dụng trên `salary / standardWorkDays`. Không giới hạn số lần phạt, không trừ tiền trực tiếp lúc checkout.
- `Payslip.baseSalary` là tổng lương tích lũy hợp lệ trong kỳ, không phải mức lương thỏa thuận trên hồ sơ.
- Đổi mức lương cập nhật ngay phần tích lũy chưa xác nhận theo mức mới; phiếu đã xác nhận giữ bản chụp.
- Công thức cuối kỳ: `netSalary = baseSalary + bonus - totalPenalty`.
- Tạm ứng chỉ gửi/duyệt đơn, không tự tạo lệnh chi hoặc trừ lương.
- Đổi ca không giới hạn số lần, có hai chế độ `manager_approval` và `mutual_acceptance`.
- Chỉ đổi ca chưa check-in và chưa tới giờ bắt đầu; kiểm tra khi gửi và khi xử lý đơn, cho cả ca nguồn/đích liên quan.
- Các công thức/luồng trên được BE thực hiện. Comment Prisma chỉ ghi chú; thay đổi tài liệu không đồng nghĩa đã sửa code hoặc chạy migration.

### 7.2. Lệnh chi và idempotency

- Phiếu lương được nhân viên sở hữu, manager đúng phạm vi hoặc admin xác nhận; lưu người và thời điểm xác nhận.
- Chỉ chi tiền cho phiếu đã xác nhận và chưa thanh toán, áp dụng cho REST lẫn SOAP. Xác nhận không tự tạo lệnh chi hoặc trừ số dư.

- Mọi thao tác chi tiền bắt buộc có `idempotencyKey`.
- Gửi trùng `idempotencyKey` phải trả lại bản ghi cũ kèm `{ "deduped": true }`, không được trừ tiền lần hai.
- `Payout.id` là UUID; mã nghiệp vụ `bankReference` có tiền tố `GD-` theo schema Prisma mới.
- Có mô phỏng số dư tài khoản nguồn; lệnh chi thành công trừ đúng số tiền, gửi lại cùng lệnh không trừ thêm.
- Hết số dư trả HTTP `422` với mã `INSUFFICIENT_FUNDS`; phía SOAP chuyển thành `soap:Fault` mô tả tiếng Việt (ví dụ "Số dư không đủ").

### 7.3. Đồng bộ schema

- Các thực thể lõi (Employee, Branch, Shift, BankAccount, Payslip, Request) phải đồng bộ tuyệt đối giữa backend, `frontend/src/types` và `mobile/lib/src/core/models/`.
- Không tự đổi tên trường. Mọi thay đổi schema phải cập nhật tài liệu SSOT trước rồi mới sửa code.
- Ngày nghiệp vụ lưu kiểu ngày, mốc chấm công lưu timestamp có múi giờ; `Payslip.month` và kỳ truy vấn dùng `YYYY-MM`. Định dạng hiển thị ngày không thay kiểu lưu trữ DB.

## 8. Quy Tắc Cấu Trúc Và Chất Lượng Bắt Buộc

1. Mỗi backend service phải có đủ (Flat 3-Tier Mẫu A): `src/config/`, `src/controllers/`, `src/routes/`, `src/middlewares/`, `src/services/` (gọi trực tiếp Prisma), `src/validators/` (nếu có), `src/clients/` (nếu có), `src/utils/`, `tests/`, `.env.example`, `package.json`, `server.js`. Cấm `src/api/`, `src/domain/`, `src/infrastructure/`, tầng `repositories` riêng.
2. Mỗi service (kể cả Gateway) phải có `GET /health` đúng khuôn `{ "status": "ok", "service": "...", "time": "..." }`.
3. `src/utils/errors.js` không được import Express, driver CSDL, ORM hay thư viện HTTP.
4. Controller không được chứa nghiệp vụ hay gọi trực tiếp Prisma; mọi nghiệp vụ và truy xuất Prisma nằm trong `src/services/`.
5. Cổng kiểm tra chất lượng:
   - Backend: service khởi động sạch bằng `cd backend/<ten-service> && npm install && npm run dev`; test của service phải qua.
   - Web: `npx tsc --noEmit` 0 lỗi; `npm run build` thành công.
   - Mobile: `flutter analyze` 0 vấn đề; `flutter test` phải qua.
6. Không sửa file thuộc service của thành viên khác khi chưa được giao. Nếu thiếu endpoint hay sai schema, ghi lại service, endpoint, kỳ vọng và thực tế rồi báo cho chủ service.
