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
5. **Cấm join xuyên cơ sở dữ liệu, cấm foreign key xuyên cơ sở dữ liệu, cấm đọc bảng của service khác trực tiếp.** Tham chiếu chéo chỉ lưu dạng chuỗi (`employeeId`, `branchSlug`, `shiftId`); cần chi tiết thì gọi HTTP. Ví dụ payroll cần `penaltyAmount` thì gọi `GET /api/attendance?employeeId=&month=` của work-service, cần `baseSalary` thì gọi `GET /api/employees/:id` của organization-service; integration tạo lệnh chi SOAP thì gọi `POST /api/payroll/payouts` của payroll-service.
6. Mỗi service có `prisma/schema.prisma`, migration, seed và `DATABASE_URL` riêng trỏ về link Supabase riêng của mình. Cấm dùng chung schema Prisma hay trỏ 2 service về cùng 1 link DB.

## 7. Quy Tắc Nghiệp Vụ Cốt Lõi

### 7.1. Lương cứng và phạt chấm công

- Nhân sự hưởng lương cứng hằng tháng (`baseSalary`).
- Checkout ca không cộng/trừ lương theo giờ. Checkout chỉ ghi nhận khoản phạt nếu vi phạm (đi trễ, về sớm, nghỉ không phép).
- Công thức chuẩn duy nhất: `netSalary = baseSalary + bonus - totalPenalty`.
- Mọi tài liệu, code và kiểm thử phải dùng đúng một công thức này. Phát hiện bản lệch phải báo ngay cho nhóm trưởng để chốt.

### 7.2. Lệnh chi và idempotency

- Mọi thao tác chi tiền bắt buộc có `idempotencyKey`.
- Gửi trùng `idempotencyKey` phải trả lại bản ghi cũ kèm `{ "deduped": true }`, không được trừ tiền lần hai.
- Mã giao dịch nội bộ có tiền tố `TXN-`, mã đối soát ngân hàng có tiền tố `BANK-`.
- Hết số dư trả HTTP `422` với mã `INSUFFICIENT_FUNDS`; phía SOAP chuyển thành `soap:Fault` mô tả tiếng Việt (ví dụ "Số dư không đủ").

### 7.3. Đồng bộ schema

- Các thực thể lõi (Employee, Branch, Shift, BankAccount, Payslip, Request) phải đồng bộ tuyệt đối giữa backend, `frontend/src/types` và `mobile/lib/src/core/models/`.
- Không tự đổi tên trường. Mọi thay đổi schema phải cập nhật tài liệu SSOT trước rồi mới sửa code.
- Quy ước ngày tháng: `date` và `joinDate` dùng `DD-MM-YYYY`; `Payslip.month` dùng `MM-YYYY`; query attendance dùng `month=YYYY-MM`.

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
