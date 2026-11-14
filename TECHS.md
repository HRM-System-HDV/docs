# TECHS.md — Danh Mục Công Nghệ Và Kiến Trúc Hệ Thống HRM Enterprise

> Tài liệu này là nguồn chân lý duy nhất (SSOT) về công nghệ và kiến trúc của dự án.
> Toàn bộ nội dung được viết bằng tiếng Việt.
> Mọi AI Agent và lập trình viên phải đọc hết file này trước khi viết code hoặc tạo service mới.

---

## 1. Tổng Quan Hệ Thống

| Mục | Nội dung |
|---|---|
| Tên hệ thống | HRM Enterprise — Quản trị nhân sự, phân ca, chấm công Wi-Fi, chi lương nội bộ |
| Kiểu kiến trúc | Monorepo 3 tầng, On-Premises, đơn khách hàng (Single-Tenant) |
| Môn học | Phát triển phần mềm theo hướng dịch vụ (PTPMDV) |
| Nhánh làm việc | `dev` (nhánh sản phẩm: `main`, remote: `PTPMDV`) |

Ba tầng chính trong monorepo:

- `frontend/` — Cổng quản trị Web (Web Portal).
- `mobile/` — Ứng dụng di động cho nhân viên và quản lý.
- `backend/` — Nhiều service độc lập theo kiến trúc hướng dịch vụ (SOA).

---

## 2. Bảng Tổng Hợp Công Nghệ Chính

| Thành phần / Tầng | Công nghệ chính | Thư viện và công cụ bổ trợ | Vai trò trong hệ thống |
|---|---|---|---|
| **Web Portal** | **Next.js 16 (App Router)** | React 19, Tailwind CSS v4, Font Awesome | Cổng quản trị cho ban giám đốc, nhân sự, quản lý chi nhánh, kế toán |
| **Backend** | **Express (Node.js >= 18)** | Express 4.x, Flat 3-Tier Architecture (Mẫu A), REST JSON + SOAP XML | Cung cấp các service nghiệp vụ độc lập |
| **Xác thực** | **JWT thuần (không qua bên thứ ba)** | Access Token + Refresh Token, cookie `hrm-session` | Xác thực phi trạng thái, phân quyền theo vai trò (RBAC) |
| **Cơ sở dữ liệu** | **5 Postgres riêng (database-per-service)** | RDBMS, chuẩn ACID, hỗ trợ transaction, JSONB | Mỗi service 1 DB riêng (`hrm_identity`, `hrm_organization`, `hrm_work`, `hrm_payroll`, `hrm_integration`); Gateway không DB; cấm dùng chung |
| **ORM** | **Prisma (mỗi service 1 schema riêng)** | Prisma Client, Prisma Migrate, truy vấn type-safe | Mỗi service khai báo `schema.prisma` + migration + seed riêng, cấm schema tập trung dùng chung |
| **Mobile** | **Flutter 3.x** | Dart 3.x, Material 3, `go_router`, Google Fonts (Public Sans) | App iOS: chấm công, xem ca, gửi đơn, tra cứu lương |

---

## 3. Kiến Trúc Backend (Mỗi Service Là Một Gói Node Độc Lập)

### 3.1. Cấu Trúc Thư Mục Chuẩn (Bắt Buộc 100%) — Flat 3-Tier Architecture (Mẫu A)

Mọi service trong `backend/<ten-service>/` phải tuân thủ đúng cấu trúc phẳng dưới đây (luồng chuẩn `Route -> Controller -> Service -> Prisma`):

```text
backend/<ten-service>/
├── prisma/                       # Giữ nguyên: schema.prisma + migration + seed riêng
├── src/
│   ├── config/                   # Đọc và kiểm tra biến môi trường, kết nối CSDL
│   │   ├── database.js           # Kết nối Postgres (Prisma) hoặc fallback in-memory
│   │   └── index.js              # Đọc env, có giá trị mặc định
│   ├── clients/                  # (Nếu có) Client HTTP gọi sang service khác, timeout 5000ms
│   ├── controllers/              # Nhận req/res, gọi Service, trả JSON (không chứa nghiệp vụ)
│   ├── middlewares/              # Xác thực, phân quyền, kiểm tra dữ liệu, xử lý lỗi
│   ├── routes/                   # Khai báo route Express, gắn HTTP verb với controller
│   ├── services/                 # Chứa toàn bộ Business Logic + gọi trực tiếp Prisma (không qua repository)
│   ├── validators/               # (Nếu có) Schema kiểm tra đầu vào (Joi hoặc Zod)
│   ├── utils/                    # Hàm thuần túy: errors.js, logger.js, jwt.js, constants...
│   │   ├── errors.js             # Lớp lỗi nghiệp vụ (NotFoundError, ValidationError...)
│   │   ├── logger.js             # Logger tập trung (Winston hoặc Pino / console)
│   │   └── jwt.js                # (Nếu có) Helper JWT
│   └── app.js                    # Khởi tạo Express, CORS, parser, gắn route, health check
├── tests/
│   ├── unit/                     # Kiểm thử validator và service
│   └── integration/              # Kiểm thử API đầu cuối
├── .env.example                  # Mẫu biến môi trường (bắt buộc)
├── package.json                  # Phụ thuộc và script riêng của service (bắt buộc)
└── server.js                     # Điểm vào: nạp env, nối CSDL, lắng nghe cổng (bắt buộc)
```

*(Riêng `api-gateway`: chỉ cần `src/config/` + `src/middlewares/`, stateless không DB, không có `services/`/`prisma/`).*

> Cấm tuyệt đối các thư mục cũ: `config/` ở root (phải nằm trong `src/config/`), `src/api/`, `src/domain/`, `src/infrastructure/`, `src/domain/entities/`, `src/infrastructure/database/repositories/`.

### 3.2. Vai Trò Từng Lớp

1. **Môi trường Node độc lập (`package.json`)**:
   - Mỗi service là một gói Node tự chủ, tự `npm install` riêng.
   - Mọi lệnh chạy trong thư mục service: `cd backend/<ten-service> && npm install && npm run dev`.
   - Cấm tham chiếu package của service khác bằng đường dẫn tương đối (`../../service-khac/node_modules`).

2. **Điểm vào (`server.js`) và khởi tạo Express (`src/app.js`)**:
   - `server.js` nằm ở gốc service, nạp `.env`, nối CSDL, lắng nghe cổng, xuất đối tượng `app` cho kiểm thử.
   - `src/app.js` cấu hình Express, middleware toàn cục và gắn route.
   - Bắt buộc có route `GET /health` trả về:

   ```json
   {
     "status": "ok",
     "service": "<ten-service>",
     "time": "2026-09-17T08:00:00.000Z"
   }
   ```

4. **Tầng trình bày (`src/controllers/`, `src/routes/`, `src/middlewares/`, `src/validators/`)**:
   - `controllers/` chỉ nhận `(req, res, next)`, trích xuất tham số, gọi Service, trả JSON chuẩn. Cấm viết nghiệp vụ, SQL hay sửa dữ liệu trực tiếp ngoài Service.
   - `middlewares/` gồm `auth.js` (kiểm tra Bearer Token hoặc cookie `hrm-session`, gắn `req.user`), `validation.js` (kiểm tra body/query/params theo `validators/`), `errorHandler.js` (ánh xạ lỗi từ `utils/errors.js` sang mã HTTP).
   - `routes/` gắn động từ HTTP với controller, ví dụ `router.post("/", controller.create)`.

4. **Tầng nghiệp vụ (`src/services/` — gộp Use Case + Repository)**:
   - Mỗi file là một Service theo miền (`WorkService.js`, `PayoutService.js`...), chứa toàn bộ business logic và **gọi trực tiếp Prisma**, không qua tầng `repositories`.
   - Không dùng class Entity / Value-Object map 1-1 với Prisma. Nếu cần ẩn trường nhạy cảm (như `password`) hay format DTO thì viết helper function ngay trong Service hoặc `src/utils/`.
   - Lỗi nghiệp vụ ném từ `src/utils/errors.js` (`EmployeeNotFoundError`, `InsufficientBalanceError`...).

6. **Hạ tầng tinh gọn (`src/config/`, `src/clients/`, `src/utils/`)**:
   - `src/config/database.js` quản lý kết nối Prisma + fallback in-memory; `src/config/index.js` đọc env.
   - `src/clients/` gọi HTTP sang service khác hoặc cổng ngân hàng, bắt buộc timeout tối đa 5000ms và có chính sách thử lại.
   - `src/utils/logger.js` cấu hình log thống nhất.

### 3.3. Phân Bổ Cổng Và Bảng Đăng Ký Service

Quy tắc phân bổ cổng để chạy đồng thời trên máy local mà không xung đột:

- `3000` — Web Portal (`frontend/`).
- `4000` — API Gateway (`backend/api-gateway/`).
- `4001–4099` — Các service backend (`backend/<ten-service>/`).

Khi tạo service mới phải chọn cổng còn trống trong dải `4001+`, khai báo trong `.env.example` và đăng ký vào bảng dưới đây (trong file này).

| Tên Service | Thư mục | Cổng mặc định | Database riêng | Trách nhiệm chính |
|---|---|---|---|---|
| **Web Portal** | `frontend/` | `3000` | Không có (gọi API) | Cổng quản trị Next.js 16 |
| **API Gateway** | `backend/api-gateway/` | `4000` | Không có (stateless, chỉ proxy) | Proxy tập trung về 4001–4005, CORS credentials, timeout 5000ms |
| **Identity Service** | `backend/identity-service/` | `4001` | `hrm_identity` | JWT + cookie hrm-session, phân quyền admin/manager/staff (Postgres + Prisma) |
| **Organization Service** | `backend/organization-service/` | `4002` | `hrm_organization` | Danh mục tổ chức và nhân sự (chủ sở hữu: B) |
| **Work Service** | `backend/work-service/` | `4003` | `hrm_work` | Ca, chấm công, tác vụ (chủ sở hữu: B) |
| **Payroll Service** | `backend/payroll-service/` | `4004` | `hrm_payroll` | Lương, lệnh chi idempotent, phiếu lương (chủ sở hữu: D) |
| **Integration Service** | `backend/integration-service/` | `4005` | `hrm_integration` | SOAP ngân hàng, yêu cầu, thông báo, tin tức, nội quy, Wi-Fi, lưu trữ file nội bộ, kho lưu trữ dữ liệu cũ (chủ sở hữu: E) |

> Quyết định đã chốt: **5 database riêng (database-per-service)**. Gateway (4000) stateless nên không có DB. Cấm dùng chung 1 DB cho nhiều service.

Mẫu `.env.example` tối thiểu cho mỗi service:

```env
PORT=4001
NODE_ENV=development
SERVICE_NAME=identity-service
DATABASE_URL=postgresql://user:password@localhost:5432/hrm_identity
JWT_SECRET=enterprise_jwt_secret_key
```

> Mỗi service thay tên DB trong `DATABASE_URL` theo DB của mình (`hrm_identity` / `hrm_organization` / `hrm_work` / `hrm_payroll` / `hrm_integration`). Cấm trỏ 2 service về cùng 1 DB.

---

## 4. Kiến Trúc Frontend (`frontend/`, Next.js 16 App Router)

- **Framework**: Next.js 16 (App Router), React 19, Tailwind CSS v4, Font Awesome.
- **Thư mục mã nguồn**: toàn bộ nằm dưới `src/`, bí danh `@/*` trỏ tới `./src/*`.
- **Định tuyến**: `src/app/` chứa route và các file chuẩn `loading.tsx`, `error.tsx`, `not-found.tsx`, `layout.tsx`. Cây route: `/` chuyển về `/login`; `/dashboard/*` gồm employees, departments, branches, shifts, tasks, requests, payslips, bank, news, regulations, wifi.
- **Tổ chức tính năng**: module phẳng đặt dưới `src/features/<ten-mien>/` (ví dụ `src/features/employees/`). Không lồng sâu kiểu `hr/...`. Component dùng chung đặt dưới `src/components/ui/`, component bố cục đặt dưới `src/components/layout/`.
- **Component client**: mọi component có state, hook hoặc sự kiện trình duyệt phải có `"use client"` ở dòng đầu tiên.
- **Xác thực và phân quyền**: qua `AuthContext` và cookie phiên `hrm-session`. Route bảo vệ bởi `src/middleware.ts`. Không đưa `role` hay `branchSlug` lên URL. Menu điều hướng sinh động bằng `buildMenuItems(role)` trong `src/lib/permissions.tsx`.
- **Tiền tệ**: bắt buộc dùng `formatVND()` từ `src/lib/utils.ts`.
- **Kiểm tra chất lượng**: `npx tsc --noEmit` phải 0 lỗi; `npm run build` phải thành công.

---

## 5. Kiến Trúc Mobile (`mobile/`, Flutter 3.x)

- **Công nghệ**: Flutter 3.x, Dart 3.x, `go_router`, Material 3.
- **Bố cục thư mục**:

```text
lib/src/
├── app.dart                        # Điểm vào HRMApp, dùng MaterialApp.router
├── core/
│   ├── constants/                  # Hằng số và đường dẫn tài nguyên
│   ├── models/                     # Model dùng chung (UserModel, ShiftModel...)
│   ├── router/router.dart          # Cấu hình go_router duy nhất (SSOT)
│   ├── theme/                      # AppColors (#8E1B2F), AppTheme, AppTypography (Public Sans)
│   └── utils/                      # Định dạng ngày, tiền tệ
└── features/<ten_tinh_nang>/
    ├── data/                       # Repository và nguồn API / mock
    └── presentation/               # Màn hình và widget
```

- **Điều hướng**: `core/router/router.dart` là cấu hình router duy nhất (`/`, `/login`, `/main` kèm `extra: UserModel`). Luôn dùng `context.go()` hoặc `context.pushReplacement()`, không bao giờ gọi `Navigator` trực tiếp.
- **Giao diện**: màu thương hiệu `AppColors.primary` là `#8E1B2F`; chữ Public Sans qua `AppTypography`; độ trong suốt màu luôn dùng `.withValues(alpha: ...)`, cấm dùng `.withOpacity()`; cấm hardcode mã hex trong widget, luôn tham chiếu `AppColors`.
- **Kiểm tra chất lượng**: `flutter analyze` 0 vấn đề; `flutter test` phải qua.

---

## 6. Xác Thực, Cơ Sở Dữ Liệu Và Mẫu Kiến Trúc Áp Dụng

### 6.1. Xác thực JWT thuần

- Payload mã hóa gồm `userId`, `username`, `role`, `branchId` (nếu có).
- Cơ chế token kép: Access Token ngắn hạn + Refresh Token để làm mới phiên.
- Xác thực phi trạng thái: Gateway và service nội bộ kiểm tra chữ ký bằng `JWT_SECRET` mà không cần truy vấn CSDL liên tục.
- Web lưu phiên bằng cookie an toàn `hrm-session`.

### 6.2. Cơ sở dữ liệu: 5 Postgres riêng (database-per-service, đã chốt)

- Quyết định đã chốt: mỗi service nghiệp vụ sở hữu **1 database Postgres riêng**, tổng **5 DB**: `hrm_identity` (4001), `hrm_organization` (4002), `hrm_work` (4003), `hrm_payroll` (4004), `hrm_integration` (4005). Gateway (4000) stateless, không có DB.
- Triển khai: **5 link Supabase riêng** (mỗi service 1 project/link Supabase với `DATABASE_URL` riêng trong `.env.example` của chính mình). Môi trường local: `docker-compose.yml` gốc dựng **5 container Postgres riêng biệt** — 5 server/node/volume/port độc lập (`5433`–`5437`) để dev/test, cách ly failure domain thật; app service chạy ngoài Docker bằng `npm run dev`. Gateway (4000) stateless, không có DB.
- Tuân thủ ACID trong phạm vi từng DB, hỗ trợ transaction, khóa bi quan / lạc quan khi giải ngân lương, index phong phú, JSONB cho cấu hình linh hoạt.
- Prisma: mỗi service có `prisma/schema.prisma` + migration + seed riêng, truy vấn type-safe. Cấm dùng chung schema Prisma giữa các service.
- Cấm tuyệt đối: join xuyên cơ sở dữ liệu, foreign key xuyên cơ sở dữ liệu, đọc bảng của service khác trực tiếp. Tham chiếu chéo chỉ lưu dạng chuỗi (`employeeId`, `branchSlug`, `shiftId`), cần chi tiết thì gọi HTTP qua `external-clients/` (timeout 5000ms).
- Single-tenant: cả 5 DB đều không có `tenantId`.
- Bảng sở hữu chi tiết:
  - `hrm_identity`: `users`, `refresh_tokens`, `device_sessions`.
  - `hrm_organization`: `branches`, `departments`, `employees`.
  - `hrm_work`: `shifts`, `attendances`, `attendance_configs`, `tasks`, `archived_tasks`.
  - `hrm_payroll`: `bank_accounts`, `payouts`, `payslips`.
  - `hrm_integration`: `requests`, `notifications`, `news`, `regulations`, `wifi_configs`, `file_assets`, `archived_requests`, `archived_notifications`.

### 6.2.1. Lưu trữ file nội bộ và kho lưu trữ dữ liệu cũ (on-premise)

- Ảnh upload lưu trên ổ đĩa máy chủ trong `STORAGE_DIR` (integration-service), đăng ký trong bảng `file_assets`; phục vụ qua `GET /api/files/*` sau xác thực Gateway (cookie `hrm-session` cho Web, Bearer cho Mobile). Không dùng dịch vụ đám mây bên thứ ba.
- Env cấu hình: `STORAGE_DIR` + `PUBLIC_BASE_URL` + `MAX_UPLOAD_BYTES` (integration-service); `INTEGRATION_SERVICE_URL` (work-service, organization-service — gọi nội bộ xóa ảnh khi xóa bản ghi, best-effort timeout 5000ms).
- Archive: `/api/archive/work/*` (nhiệm vụ hoàn thành) và `/api/archive/integration/*` (đơn đã xử lý, thông báo đã đọc) — chỉ admin; chuyển batch vào bảng `archived_*` trong transaction, xem lại và khôi phục được. `Backlog` không bao giờ archive.
- Yêu cầu triển khai: Web Portal và Gateway phải **cùng site** (cookie `SameSite=Lax` mới gửi kèm request ảnh `<img>`), ví dụ `hrm.company.local` + `api.hrm.company.local` sau một nginx; `PUBLIC_BASE_URL` là địa chỉ Gateway cả web lẫn mobile truy cập được. Backup = Postgres dump + thư mục `STORAGE_DIR` đi cùng nhau; nếu container hóa integration-service phải gắn named volume cho `STORAGE_DIR`.

### 6.3. Các mẫu kiến trúc trong môn PTPMDV

- **IPC (giao tiếp giữa các service)**: gọi qua `src/clients/` bằng HTTP REST, timeout tối đa 5000ms; cấm import code trực tiếp giữa các service; `integration-service` dùng SOAP XML mô phỏng cổng ngân hàng.
- **SAGA (giao dịch phân tán)**: SAGA điều phối cho luồng chi lương `payroll-service` → `integration-service` (cổng SOAP) → phát sinh giao dịch bù trừ nếu thất bại; mỗi bước ghi log `PENDING → SUCCESS / COMPENSATED`.
- **DDD và Event Sourcing**: tách domain rõ theo từng service (mỗi service sở hữu `src/services/` + schema Prisma riêng); sự kiện quan trọng (checkout ca, duyệt đơn, tạo phiếu lương) thiết kế dạng event để tái hiện trạng thái. Không dùng `domain/entities/`, `domain/value-objects/` hay tầng `repositories` riêng.
- **CQRS và API Composition**: Gateway tổng hợp dữ liệu nhiều service thành một response cho frontend; tách luồng đọc (GET) và luồng ghi (POST/PUT/DELETE) ở controller.
- **API Gateway và BFF**: Gateway (cổng 4000) là điểm vào duy nhất, làm xác thực JWT, giới hạn tần suất, định tuyến, không chứa nghiệp vụ; Web nhận payload đầy đủ cho quản trị, Mobile nhận payload gọn cho băng thông di động.
- **Triển khai production-ready**: `docker-compose.yml` gốc dựng hạ tầng DB local (**5 Postgres riêng**, mỗi service 1 container/volume/port); production trỏ `DATABASE_URL` về 5 link Supabase riêng; phân phiên bản API bằng tiền tố `/api/v1/`; mỗi service mở `GET /health` cho giám sát.
