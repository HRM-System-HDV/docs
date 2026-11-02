# Work_flow_PTPMDV_E.md — Thành viên E — Môn Phát triển phần mềm hướng dịch vụ

> File này dành cho AI agent. Đọc hết trước khi sửa code hoặc tạo file. Nguồn: `PTPMDV_PhanCong.docx`.
> Mục có nhãn **(suy ra)** hoặc **(chưa quy định)** không có trong tài liệu phân công gốc. Không coi đó là yêu cầu chắc chắn; hỏi lại khi cần.
> Không đổi cổng, envelope, tên trường hay cấu trúc thư mục chung.

---

## 0. Tóm tắt nhanh

| Mục | Nội dung |
|---|---|
| Môn | Phát triển phần mềm hướng dịch vụ |
| Thành viên nhóm | A (nhóm trưởng), B, D, E |
| Phần E sở hữu | `backend/integration-service` (4005) |
| Chức năng chính | SOAP ngân hàng, yêu cầu, thông báo, bảng tin, nội quy, Wi-Fi |

---

## 1. Phạm vi: được sửa và không được sửa

### Được sửa (thuộc E)
- `backend/integration-service/` (cổng 4005): SOAP ngân hàng, yêu cầu, thông báo, bảng tin, nội quy, Wi-Fi

### Không sửa (thuộc thành viên khác)
| Thư mục | Cổng | Chủ sở hữu | Chức năng chính |
|---|---|---|---|
| `backend/api-gateway/`, `backend/identity-service/`, `mobile/`, `docker-compose.yml` | 4000, 4001 | A | Gateway, xác thực, mobile, compose |
| `backend/organization-service/` | 4002 | B | Danh mục tổ chức và nhân sự |
| `backend/work-service/` | 4003 | B | Ca, chấm công, tác vụ |
| `backend/payroll-service/` | 4004 | D | Lương, lệnh chi, phiếu lương |

Nếu cần thay đổi ở phần của người khác (thiếu endpoint, sai schema, sai envelope), không tự sửa. Ghi lại vấn đề, nêu rõ service, endpoint, kỳ vọng và thực tế, rồi báo cho E để trao đổi với chủ phần đó.

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
- Tạo nhánh theo mẫu `feat/<service>-<tên>` (ví dụ `feat/integration-soap`, `feat/integration-news`).
- Xong thì tạo PR về `dev`.
- **Không commit thẳng vào `dev` hoặc `main`.**
- Mỗi PR nên gọn theo một service hoặc một chức năng.

---

## 4. Danh sách việc của E

### Task 1 — integration-service (cổng 4005) — ✅ XONG (nhánh `feat/integration-requests-soap`)
- [x] **SOAP:** `GET /soap/payroll?wsdl` (WSDL hợp lệ) và `POST /soap/payroll` (nhận `PayoutRequest`, trả `PayoutResponse`, lỗi trả `soap:Fault`). SOAP phải tạo được lệnh chi.
- [x] **Requests (yêu cầu nội bộ):**
  - `GET /api/requests?employeeId=&branchSlug=&status=&type=` — danh sách yêu cầu.
  - `GET /api/requests/:id` — chi tiết một yêu cầu (phục vụ màn `request_detail` trên mobile) **(suy ra)**.
  - `POST /api/requests` — tạo yêu cầu mới. `type` chấp nhận: `leave`, `overtime`, `advance`, `shift_swap`, `work_supplement`, `other`.
  - `PUT /api/requests/:id/approve` — duyệt (tự sinh thông báo).
  - `PUT /api/requests/:id/reject` — từ chối (tự sinh thông báo).
  - `DELETE /api/requests/:id` — xóa, chỉ khi `status = pending`.
  - **Lưu ý `shift_swap`:** payload cần thêm `sourceShiftId` và `targetShiftId` để xác định ca đổi. Khi duyệt, gọi `PUT /api/shifts/:id` của B (work-service 4003) để cập nhật phân công **(suy ra)**. Xác nhận với B trước.
  - **Lưu ý `work_supplement` (bổ sung công):** có thể cần bổ sung vào `attendance` tương ứng khi được duyệt **(suy ra)**. Xác nhận với B trước.
- [x] **Notifications (thông báo):**
  - `GET /api/notifications?employeeId=&branchSlug=` — gồm cả broadcast (`targetEmployeeId = null`).
  - `POST /api/notifications` — gửi thông báo (admin/manager).
  - `PUT /api/notifications/:id/read` — đánh dấu đã đọc.
  - `DELETE /api/notifications/:id`.
- [x] **Bảng tin (News/Announcement):** (Đã hoàn thành trên nhánh `feat/integration-service`)
  - `GET /api/news`, `GET /api/news/:id`.
  - `POST /api/news`, `PUT /api/news/:id`, `DELETE /api/news/:id` — chỉ `admin`/`manager`.
- [x] **Nội quy (Regulations):** (Đã hoàn thành trên nhánh `feat/integration-service`)
  - `GET /api/regulations`, `GET /api/regulations/:id`.
  - `POST /api/regulations`, `PUT /api/regulations/:id`, `DELETE /api/regulations/:id`.
- [x] **Wi-Fi chấm công:** (Đã hoàn thành trên nhánh `feat/integration-service`)
  - `GET /api/wifi-configs?branch=`.
  - `POST /api/wifi-configs`, `PUT /api/wifi-configs/:id`, `DELETE /api/wifi-configs/:id`.
- [x] `GET /health` (Đã hoàn thành trên nhánh `feat/integration-service`).

---

## 5. Schema dữ liệu liên quan (phải khớp — SSOT tại `A_Work_Flow.md §6`)

Không tự đổi tên trường. Mọi thay đổi phải cập nhật `A_Work_Flow.md §6` trước.

- **Khuôn SOAP:** `POST /soap/payroll` nhận `PayoutRequest (idempotencyKey, debitAccount, content, totalAmount, beneficiaryCount)`, trả `PayoutResponse (transactionId (TXN-xxxxxx), bankReference (BANK-xxxxxxxx), status)`. Lỗi trả `soap:Fault` với `faultcode = soap:Client` và `faultstring` mô tả lỗi (ví dụ: Số dư không đủ).

- **Yêu cầu nội bộ (Request):** `id, type, employeeId, branchSlug, title, content, attachmentUrl, status (pending/approved/rejected), reviewedBy, reviewNote, createdAt, updatedAt`
  > `type` hợp lệ: `leave` (nghỉ phép), `overtime` (tăng ca), `advance` (tạm ứng lương), `shift_swap` (đổi ca), `work_supplement` (bổ sung công), `other`.
  > `shift_swap` cần thêm `sourceShiftId` và `targetShiftId` trong payload.

- **Thông báo (Notification):** `id, targetEmployeeId (null = toàn hệ thống), branchSlug, title, body, isRead, createdAt`

- **Bảng tin (Announcement/News):** `id, title, summary, content, date, tag, tagTone (danger/warning/success/primary/gray), pinned (bool)`

- **Nội quy (Regulation):** `id, code, title, category, summary, content, status (hiệu lực/dự thảo/hết hiệu lực), scope (Toàn công ty hoặc slug chi nhánh), effectiveDate (DD-MM-YYYY), expiryDate, createdAt, updatedAt, version, pinned, attachments (số file đính kèm)`

- **Wi-Fi chấm công:** `id, ssid, bssid, branch (slug chi nhánh), status (hoạt động/vô hiệu hóa)`

Ghi chú: `createdAt`, `updatedAt`, `date` dùng ISO 8601 hoặc `DD-MM-YYYY` — hỏi A để thống nhất với mobile.

---

## 6. Phụ thuộc và phối hợp

- Mọi request từ web và mobile đi qua `api-gateway` (4000) của A. Client không gọi thẳng vào service của bạn.
- Đăng nhập và phiên `hrm-session` do `identity-service` (4001) của A xử lý.
- **D (payroll-service, 4004):** SOAP của bạn tạo lệnh chi và trả `transactionId` (`TXN-xxxxxx`), `bankReference` (`BANK-xxxxxxxx`). Cách tạo là gọi REST payout của D qua HTTP **(suy ra)**, timeout tối đa 5000ms, không import chéo mã nguồn. `idempotencyKey` của `PayoutRequest` cần được chuyển tiếp để tính idempotent nhất quán với REST.
- **D:** khi D trả lỗi hết số dư (422), chuyển thành `soap:Fault` với `faultcode = soap:Client` và `faultstring` mô tả (ví dụ "Số dư không đủ").
- **B (work-service, 4003):** khi duyệt `shift_swap`, gọi HTTP đến `PUT /api/shifts/:id` của B để cập nhật phân công ca. Khi duyệt `work_supplement`, có thể cần gọi `attendance` của B. Xác nhận với B trước.
- **A:** demo SOAP end-to-end trên web và mobile của A gọi `/soap/payroll` của bạn qua gateway, cần chuyển tiếp nguyên vẹn XML và `soap:Fault`. Màn hình `notifications`, `leave_request`, `approvals`, `salary_advance` trên mobile của A gọi requests và notifications của bạn.
- **Liên động yêu cầu và thông báo:** duyệt hoặc từ chối yêu cầu tự sinh thông báo. Cả hai nằm trong cùng service của bạn, nhưng phải đúng liên động vì đây là điểm nghiệm thu.

---

## 7. Mốc kiểm tra

| Mốc | Việc của E |
|---|---|
| 1 | Xong requests, notifications, SOAP |
| 2 | Xong bảng tin, nội quy, Wi-Fi (✅ Đã xong trên nhánh `feat/integration-service`) |

Mốc đầu tiên của A là gateway + identity. Cần bám sát để chạy được qua gateway khi tích hợp.

---

## 8. Nghiệm thu (Definition of Done)

- WSDL hợp lệ.
- SOAP tạo được lệnh chi.
- Yêu cầu và thông báo liên động đúng.
- CRUD bảng tin, nội quy, Wi-Fi đúng envelope.

### Checklist trước khi mở PR
- [ ] Đúng cấu trúc thư mục service (mục 2.1).
- [ ] Response đúng envelope `{data}` hoặc `{error: {code, message}}`.
- [ ] Có `GET /health`.
- [ ] Có `.env.example` (mặc định trỏ Postgres local của chính service; Supabase để dạng comment, mỗi service 1 link riêng).
- [ ] `GET /soap/payroll?wsdl` hợp lệ; lỗi trả `soap:Fault` với `faultcode = soap:Client`.
- [ ] Duyệt/từ chối yêu cầu tự sinh thông báo; chỉ xóa yêu cầu khi `pending`.
- [ ] Tạo/sửa/xóa bảng tin chỉ cho `admin`/`manager`.
- [ ] Gọi liên service qua HTTP, timeout tối đa 5000ms, không import chéo mã nguồn.
- [ ] Nhánh đặt tên `feat/<service>-<tên>`, PR về `dev`.
- [ ] Không sửa file thuộc phần của người khác.

---

## 9. Điểm chưa quy định

Các mục sau không có trong tài liệu phân công. Không tự quyết định rồi coi như đã chốt. Hỏi A (nhóm trưởng) hoặc ghi rõ giả định trong PR.

- **Cơ sở dữ liệu và ORM (đã chốt 5 DB trên 5 link Supabase riêng):** E sở hữu DB riêng `hrm_integration` (4005) với Prisma schema + migration + `DATABASE_URL` riêng trỏ link Supabase riêng. Lớp lưu trữ nằm trong `src/services/` gọi trực tiếp Prisma (Mẫu A — không có tầng `repositories` riêng hay `src/infrastructure`). Cấm join xuyên cơ sở dữ liệu; SOAP tạo lệnh chi thì gọi HTTP `POST /api/payroll/payouts` của D, duyệt `shift_swap`/`work_supplement` thì gọi HTTP sang work-service của B.
- **Cách service nhận danh tính người dùng (vai trò `admin`/`manager`/`staff`):** tài liệu chỉ nêu middleware `hrm-session` gắn `req.user` ở identity-service. Cách các service khác lấy được danh tính và vai trò chưa được quy định. Hỏi A trước khi làm phần phân quyền.
- **Đường dẫn chính xác của requests, notifications và thao tác approve/reject/read:** tài liệu PTPMDV chỉ nêu phương thức HTTP, không nêu đường dẫn. **Phụ trách: E** đề xuất, A duyệt khi cấu hình gateway (A §5) **(suy ra)**.
- **Cấu trúc WSDL:** không được nêu chi tiết, chỉ yêu cầu WSDL hợp lệ.
- **Cách SOAP tạo lệnh chi:** không nêu rõ gọi payroll-service bằng cách nào, xem mục 6.
- **Ai được duyệt hoặc từ chối yêu cầu:** không được nêu. **Phụ trách: A** chốt trong ma trận phân quyền (A Task 5) **(suy ra)**.
- **Định dạng body của `/health`:** tài liệu PTPMDV không nêu, xem mục 2.3.

---

## 10. Quy tắc làm việc cho agent

1. Đọc mục 1 trước. Chỉ sửa trong phạm vi của E.
2. Không tự bịa endpoint, tên trường hay hành vi. Nếu tài liệu không nêu, dùng nhãn **(suy ra)** và hỏi lại.
3. Bám đúng envelope, tên trường và định dạng ngày ở mục 5.
4. Không commit thẳng vào `dev` hoặc `main`. Mỗi thay đổi đi qua nhánh `feat/...` và PR về `dev`.
5. Không import chéo mã nguồn giữa các service. Muốn dùng dữ liệu của service khác thì gọi qua HTTP.
6. Sau mỗi thay đổi, kiểm tra `GET /health` và chạy test của service (`tests/`).
7. Khi báo kết quả, nói rõ đã chạy lệnh nào và kết quả ra sao. Không báo "pass" khi chưa chạy.
8. Ưu tiên theo thứ tự: requests, notifications, SOAP (mốc 1) trước, rồi bảng tin, nội quy, Wi-Fi (mốc 2).
