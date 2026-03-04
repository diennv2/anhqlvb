# Tài liệu API – App Quản lý Văn bản (Đến/Đi) (CHI TIẾT – dùng được để làm app)

> Mục tiêu: Tài liệu API đủ chi tiết để FE/Mobile làm app thật:
> - Có headers, auth, paging/filter/sort
> - Có request/response schema + ví dụ
> - Có các endpoint “đúng đời thực”: upload/preview/download file, timeline, giao xử lý, bút phê, audit log
> - Có RBAC + độ mật (tách quyền metadata vs file), delegation, dispatch log, link incoming↔outgoing
>
> Base URL ví dụ: `https://api.example.com/api/v1`

---

## Mục lục
1. [Quy ước chung](#1-quy-ước-chung)
2. [Headers](#2-headers)
3. [Auth & Session](#3-auth--session)
4. [Chuẩn Response](#4-chuẩn-response)
5. [Phân trang/Lọc/Sắp xếp](#5-phân-tranglọcsắp-xếp)
6. [Chuẩn lỗi](#6-chuẩn-lỗi)
7. [Auth endpoints](#7-auth-endpoints)
8. [User/Role/Department/Position](#8-userroledepartmentposition)
9. [Delegations (Uỷ quyền xử lý)](#9-delegations-uỷ-quyền-xử-lý)
10. [Danh mục nghiệp vụ văn bản](#10-danh-mục-nghiệp-vụ-văn-bản)
11. [Incoming Documents – Văn bản đến](#11-incoming-documents--văn-bản-đến)
12. [Outgoing Documents – Văn bản đi](#12-outgoing-documents--văn-bản-đi)
13. [Recipients/Dispatch Log – Nơi nhận VB đi](#13-recipientsdispatch-log--nơi-nhận-vb-đi)
14. [Document Links – Liên kết VB đến/đi](#14-document-links--liên-kết-vb-đếnđi)
15. [Attachments (File) – Upload/Preview/Download](#15-attachments-file--uploadpreviewdownload)
16. [Assignments – Giao xử lý](#16-assignments--giao-xử-lý)
17. [Comments/Directives – Bút phê/Ý kiến](#17-commentsdirectives--bút-phêý-kiến)
18. [Timeline – Lịch sử xử lý](#18-timeline--lịch-sử-xử-lý)
19. [Search/Export](#19-searchexport)
20. [Notifications](#20-notifications)
21. [Audit Logs](#21-audit-logs)
22. [Phân quyền (RBAC + độ mật) – quy ước thực thi](#22-phân-quyền-rbac--độ-mật--quy-ước-thực-thi)
23. [Checklist MVP (đủ để làm app)](#23-checklist-mvp-đủ-để-làm-app)

---

## 1. Quy ước chung
- Format JSON: `snake_case`
- Time: ISO 8601 UTC (ví dụ `2026-03-03T10:15:30Z`)
- ID: integer tăng dần (DB). UI/FE nên dùng `doc_code` để hiển thị/định danh ổn định (không đoán ID).
- Độ mật/độ khẩn: lấy theo master data, FE dùng `id` + `code`.
- Paging: `page=1` default, `page_size=20` default, `page_size` max 200.

---

## 2. Headers
### 2.1 Request headers
| Header | Bắt buộc | Ví dụ | Mục đích |
|---|---:|---|---|
| `Authorization` | ✅ | `Bearer <token>` | Auth |
| `Accept` | ✅ | `application/json` | Nhận JSON |
| `Content-Type` | ✅ | `application/json` | Khi gửi JSON |
| `X-Request-Id` | ⛔ | `a1b2c3...` | Trace request |
| `X-Client-Platform` | ⛔ | `web/ios/android` | Debug |
| `X-App-Version` | ⛔ | `1.0.0` | Debug |

### 2.2 Response headers (khuyến nghị)
- `X-Request-Id`: echo lại
- `Cache-Control`: với master data có thể cache

---

## 3. Auth & Session
### 3.1 JWT
- `access_token`: dùng gọi API, TTL ngắn
- `refresh_token`: làm mới token, TTL dài

### 3.2 Quy tắc cho app
- FE lưu token (secure storage), tự refresh khi gặp:
  - HTTP 401 + `error=token_expired`
- Logout: xoá token, (tuỳ) gọi API revoke refresh token

---

## 4. Chuẩn Response
### 4.1 Response list (có paging)
```json
{
  "data": [],
  "meta": { "page": 1, "page_size": 20, "total": 0 },
  "request_id": "a1b2c3"
}
```

### 4.2 Response detail
```json
{
  "data": {},
  "request_id": "a1b2c3"
}
```

### 4.3 Response message
```json
{
  "message": "Thao tác thành công",
  "request_id": "a1b2c3"
}
```

---

## 5. Phân trang/Lọc/Sắp xếp
- `page`, `page_size`
- `search`
- `sort` (ví dụ `-created_at,received_date`)

---

## 6. Chuẩn lỗi
```json
{
  "error": "validation_error",
  "message": "Dữ liệu đầu vào không hợp lệ",
  "field": "document_name",
  "details": [
    { "field": "document_name", "message": "Required" }
  ],
  "request_id": "a1b2c3"
}
```

Mã lỗi chuẩn hoá (gợi ý):
- 401: `unauthorized`, `token_expired`
- 403: `forbidden`
- 404: `not_found`
- 409: `conflict`
- 422: `validation_error`, `business_rule_violation`

---

## 7. Auth endpoints

### 7.1 POST `/auth/login`
**Request**
```json
{ "username": "vanthu01", "password": "123456" }
```

**Response 200**
```json
{
  "access_token": "eyJhbGciOi...",
  "refresh_token": "eyJhbGciOi...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

### 7.2 POST `/auth/token/refresh`
**Request**
```json
{ "refresh_token": "eyJhbGciOi..." }
```

**Response 200**
```json
{
  "access_token": "new_access...",
  "refresh_token": "new_refresh...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

### 7.3 GET `/auth/whoami`
**Response 200**
```json
{
  "data": {
    "id": 12,
    "username": "vanthu01",
    "full_name": "Nguyễn Văn Thư",
    "department": { "id": 1, "code": "HC", "name": "Phòng Hành chính" },
    "role": { "id": 2, "code": "van_thu", "name": "Văn thư" },
    "permissions": [
      "incoming.create",
      "incoming.view",
      "incoming.update",
      "attachment.upload",
      "audit.view"
    ]
  }
}
```

### 7.4 POST `/auth/logout`
**Request**
```json
{ "refresh_token": "eyJhbGciOi..." }
```

**Response 200**
```json
{ "message": "Đã đăng xuất" }
```

---

## 8. User/Role/Department/Position
> CRUD tiêu chuẩn. App cần để: phân quyền, phân đơn vị, hiển thị người giao/nhận.

### 8.1 GET `/users`
Query: `page,page_size,search,department_id,role_id,status`

**Response 200**
```json
{
  "data": [
    {
      "id": 12,
      "username": "vanthu01",
      "full_name": "Nguyễn Văn Thư",
      "email": "vanthu01@gov.vn",
      "phone": "0912345678",
      "status": "active",
      "department": { "id": 1, "code": "HC", "name": "Phòng Hành chính" },
      "position": { "id": 1, "name": "Văn thư" },
      "role": { "id": 2, "code": "van_thu", "name": "Văn thư" }
    }
  ],
  "meta": { "page": 1, "page_size": 20, "total": 1 }
}
```

### 8.2 POST `/users`
**Request**
```json
{
  "username": "user01",
  "password": "123456",
  "full_name": "Nguyễn Văn A",
  "email": "user01@gov.vn",
  "phone": "0900000000",
  "department_id": 1,
  "position_id": 1,
  "role_id": 2,
  "status": "active",
  "must_change_password": true
}
```

**Response 201**
```json
{ "data": { "id": 99 } }
```

### 8.3 GET `/users/{id}`
**Response 200**
```json
{
  "data": {
    "id": 99,
    "username": "user01",
    "full_name": "Nguyễn Văn A",
    "email": "user01@gov.vn",
    "phone": "0900000000",
    "status": "active",
    "department": { "id": 1, "code": "HC", "name": "Phòng Hành chính" },
    "position": { "id": 1, "name": "Văn thư" },
    "role": { "id": 2, "code": "van_thu", "name": "Văn thư" }
  }
}
```

### 8.4 PATCH `/users/{id}`
**Request**
```json
{
  "full_name": "Nguyễn Văn A (cập nhật)",
  "phone": "0911111111",
  "department_id": 2,
  "position_id": 3,
  "role_id": 4,
  "status": "active"
}
```

**Response 200**
```json
{ "message": "Cập nhật thành công" }
```

### 8.5 DELETE `/users/{id}`
**Response 200**
```json
{ "message": "Đã xoá" }
```

### 8.6 PUT `/users/{id}/password`
**Request**
```json
{
  "new_password": "NewPass@123",
  "force_logout": true
}
```

**Response 200**
```json
{ "message": "Đổi mật khẩu thành công" }
```

### 8.7 GET `/users/{id}/audit-logs`
Query: `page,page_size,from,to,action`
> Shortcut cho audit-logs theo user.

---

### 8.8 Departments
- GET `/departments`
- POST `/departments`
- GET `/departments/{id}`
- PATCH `/departments/{id}`
- DELETE `/departments/{id}` (khuyến nghị: inactive)

**Department schema**
```json
{
  "id": 1,
  "code": "HC",
  "name": "Phòng Hành chính",
  "parent_id": null,
  "is_active": true
}
```

---

### 8.9 Positions
- GET `/positions`
- POST `/positions`
- PATCH `/positions/{id}`
- DELETE `/positions/{id}` (inactive)

---

### 8.10 Roles & Permissions
- GET `/roles`
- POST `/roles`
- PATCH `/roles/{id}`
- DELETE `/roles/{id}` (inactive)
- GET `/permissions`
- PUT `/roles/{id}/permissions`

**PUT role permissions – Request**
```json
{
  "permission_ids": [1, 2, 3, 10]
}
```

**Response 200**
```json
{ "message": "Cập nhật quyền thành công" }
```

---

## 9. Delegations (Uỷ quyền xử lý)
> Dùng khi người uỷ quyền vắng mặt. Delegation phải được áp vào rule RBAC/độ mật khi check quyền xem/xử lý.

### 9.1 GET `/delegations`
Query: `page,page_size,delegator_user_id,delegate_user_id,is_active`

### 9.2 POST `/delegations`
**Request**
```json
{
  "delegator_user_id": 12,
  "delegate_user_id": 33,
  "scope": {
    "modules": ["incoming", "outgoing"],
    "confidentiality_codes": ["public", "confidential", "secret"],
    "department_ids": [1, 2]
  },
  "start_at": "2026-03-01T00:00:00Z",
  "end_at": "2026-03-10T23:59:59Z"
}
```

**Response 201**
```json
{ "data": { "id": 5001 } }
```

### 9.3 POST `/delegations/{id}/revoke`
**Request**
```json
{ "reason": "Kết thúc uỷ quyền sớm" }
```

**Response 200**
```json
{ "message": "Đã thu hồi uỷ quyền" }
```

---

## 10. Danh mục nghi���p vụ văn bản (Master Data)

### 10.1 Document Types
- GET `/document-types`
- POST `/document-types`
- PATCH `/document-types/{id}`
- DELETE `/document-types/{id}` (thực tế nên chuyển inactive)

### 10.2 Urgency Levels
- GET `/urgency-levels`

### 10.3 Confidentiality Levels
- GET `/confidentiality-levels`

### 10.4 Organizations
- GET `/organizations`
- POST `/organizations`
- PATCH `/organizations/{id}` (khuyến nghị)
- DELETE `/organizations/{id}` (inactive)

### 10.5 Field Sectors
- GET `/field-sectors`

---

## 11. Incoming Documents – Văn bản đến

### 11.1 GET `/incoming-documents`
Query:
- paging: `page,page_size`
- search: `search`
- filter:
  - `status`
  - `confidentiality_level_id`
  - `urgency_level_id`
  - `sender_org_id`
  - `received_from, received_to`
  - `due_from, due_to`
  - `owner_department_id`
  - `assignee_user_id` (lọc “được giao cho tôi”)
- sort: `sort=-received_date`

### 11.2 POST `/incoming-documents`
**Request**
```json
{
  "incoming_number": "123",
  "symbol_number": "01/CV-ABC",
  "document_name": "Công văn về việc ...",
  "summary": "Trích yếu ...",
  "document_type_id": 1,
  "urgency_level_id": 2,
  "confidentiality_level_id": 2,
  "field_sector_id": 3,
  "sender_org_id": 9,
  "signer": "Nguyễn Văn A",
  "issued_date": "2026-02-28",
  "received_date": "2026-03-01",
  "received_channel": "paper",
  "due_date": "2026-03-10",
  "owner_department_id": 1
}
```

### 11.3 GET `/incoming-documents/{id}`

### 11.4 PATCH `/incoming-documents/{id}`
> Không cho sửa nếu đã archived (hoặc theo policy).

### 11.5 PATCH `/incoming-documents/{id}/status`
**Request**
```json
{ "status": "processing", "reason": "Bắt đầu xử lý" }
```

---

## 12. Outgoing Documents – Văn bản đi

### 12.1 GET `/outgoing-documents`
Query: giống incoming + `publish_from,publish_to`

### 12.2 POST `/outgoing-documents`
Tạo nháp văn bản đi.

### 12.3 GET `/outgoing-documents/{id}` (BỔ SUNG)
**Response 200**
```json
{
  "data": {
    "id": 201,
    "doc_code": "OUT-2026-000201",
    "outgoing_number": "45",
    "symbol_number": "02/QD-XYZ",
    "document_name": "Quyết định về việc ...",
    "summary": "Trích yếu ...",
    "status": "draft",
    "publish_date": null,
    "confidentiality": { "id": 1, "code": "public", "name": "Bình thường" },
    "urgency": { "id": 1, "code": "normal", "name": "Thường" },
    "attachments": [],
    "created_by": { "id": 12, "full_name": "Nguyễn Văn Thư" },
    "created_at": "2026-03-01T08:00:00Z",
    "updated_at": "2026-03-01T08:05:00Z"
  }
}
```

### 12.4 PATCH `/outgoing-documents/{id}` (BỔ SUNG)
**Request**
```json
{
  "summary": "Trích yếu cập nhật",
  "urgency_level_id": 2,
  "confidentiality_level_id": 2
}
```

**Response 200**
```json
{ "message": "Cập nhật thành công" }
```

### 12.5 Versions
- POST `/outgoing-documents/{id}/versions`
- GET `/outgoing-documents/{id}/versions`
- PATCH `/outgoing-documents/{id}/versions/{version_id}`

### 12.6 Publish/Recall
- POST `/outgoing-documents/{id}/publish`
- POST `/outgoing-documents/{id}/recall`

---

## 13. Recipients/Dispatch Log – Nơi nhận VB đi (BỔ SUNG)

### 13.1 GET `/outgoing-documents/{id}/recipients`
Query: `page,page_size,status,method`

### 13.2 PATCH `/outgoing-recipients/{id}`
Cập nhật trạng thái gửi (dùng bởi hệ thống tích hợp / admin).
**Request**
```json
{ "status": "sent", "sent_at": "2026-03-02T10:00:00Z", "note": "Gửi email thành công" }
```

### 13.3 POST `/outgoing-recipients/{id}/retry`
Retry gửi (nếu method là email/portal).
**Request**
```json
{ "reason": "Retry do lỗi SMTP" }
```

---

## 14. Document Links – Liên kết VB đến/đi (BỔ SUNG)

### 14.1 POST `/document-links`
**Request**
```json
{
  "incoming_document_id": 101,
  "outgoing_document_id": 201,
  "link_type": "reply"
}
```

**Response 201**
```json
{ "data": { "id": 3001 } }
```

### 14.2 GET `/incoming-documents/{id}/links`
### 14.3 GET `/outgoing-documents/{id}/links`
### 14.4 DELETE `/document-links/{id}`
**Response 200**
```json
{ "message": "Đã xoá liên kết" }
```

---

## 15. Attachments (File) – Upload/Preview/Download

### 15.1 POST `/incoming-documents/{id}/attachments`
multipart/form-data:
- `file` (binary)
- `is_main` (boolean)
- `note` (string, optional)

### 15.2 POST `/outgoing-documents/{id}/attachments`
multipart/form-data tương tự.

### 15.3 GET `/attachments/{id}/download`
- Backend kiểm tra quyền theo: role + department + confidentiality + download policy
- Nếu bị chặn download: trả 403
```json
{ "error": "forbidden", "message": "Không có quyền tải file", "request_id": "..." }
```

### 15.4 GET `/attachments/{id}/preview`
Trả signed URL ngắn hạn.
Nếu bị chặn preview: 403.

### 15.5 DELETE `/attachments/{id}`
Soft delete + audit.

---

## 16. Assignments – Giao xử lý
- POST `/assignments`
- GET `/assignments`
- PATCH `/assignments/{id}`
- PATCH `/assignments/{id}/complete`
- POST `/assignments/{id}/revoke`

---

## 17. Comments/Directives – Bút phê/Ý kiến

### 17.1 POST `/comments`
### 17.2 GET `/comments`
### 17.3 PATCH `/comments/{id}`
**Request**
```json
{ "content": "Nội dung cập nhật", "edit_reason": "Sửa lỗi chính tả" }
```

**Response 200**
```json
{ "message": "Cập nhật thành công" }
```

---

## 18. Timeline – Lịch sử xử lý
- GET `/incoming-documents/{id}/timeline`
- GET `/outgoing-documents/{id}/timeline`

---

## 19. Search/Export
- GET `/documents/search`
- GET `/documents/export?format=excel|pdf`

---

## 20. Notifications

### 20.1 GET `/notifications`
Query: `unread=true|false`

**Response**
```json
{
  "data": [
    {
      "id": 90001,
      "type": "assignment",
      "content": "Bạn được giao xử lý INC-2026-000101",
      "read": false,
      "created_at": "2026-03-01T09:00:00Z",
      "ref": { "document_kind": "incoming", "document_id": 101 }
    }
  ]
}
```

### 20.2 PATCH `/notifications/{id}/read`
### 20.3 PATCH `/notifications/read-all`

---

## 21. Audit Logs
- GET `/audit-logs`
- GET `/incoming-documents/{id}/audit-logs`
- GET `/outgoing-documents/{id}/audit-logs`

---

## 22. Phân quyền (RBAC + độ mật) – quy ước thực thi

### 22.1 Permission codes (gợi ý)
- Incoming:
  - `incoming.create`, `incoming.view`, `incoming.update`, `incoming.change_status`
- Outgoing:
  - `outgoing.create`, `outgoing.view`, `outgoing.update`, `outgoing.publish`, `outgoing.recall`
- Assignment:
  - `assignment.create`, `assignment.update`, `assignment.revoke`
- Comment:
  - `comment.create`, `comment.update`
- Attachment:
  - `attachment.upload`, `attachment.download`, `attachment.preview`, `attachment.delete`
- Audit:
  - `audit.view`
- System:
  - `user.manage`, `role.manage`, `masterdata.manage` (khuyến nghị thêm)

### 22.2 Rule độ mật (thực tế)
- Khi văn bản có `confidentiality_level >= secret`:
  - chỉ user thuộc đúng đơn vị/được giao/được uỷ quyền (delegation) mới xem
  - tách quyền “xem metadata” và “xem/tải file”
  - hạn chế download/print theo policy
  - bắt buộc ghi audit khi view/preview/download/print/export

---

## 23. Checklist MVP (đủ để làm app)
✅ Auth: login/refresh/whoami/logout  
✅ Master data: document-types, confidentiality-levels, urgency-levels, organizations, field-sectors  
✅ Incoming: list/create/detail/update/change_status/upload/timeline  
✅ Outgoing: list/create/detail/update/version/publish/recall/upload/timeline  
✅ Assignment: create/list/update/complete/revoke  
✅ Comment: create/list/update  
✅ Attachment: upload/preview/download/delete  
✅ Search + Export  
✅ Notifications: list/mark read/read-all  
✅ Audit logs: list/by-document  

---
