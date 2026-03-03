# Tài liệu API – App Quản lý Văn bản (Đến/Đi) (CHI TIẾT – dùng được để làm app)

> Mục tiêu: Tài liệu API đủ chi tiết để FE/Mobile làm app thật:
> - Có headers, auth, paging/filter/sort
> - Có request/response schema + ví dụ
> - Có các endpoint “đúng đời thực”: upload/preview/download file, timeline, giao xử lý, bút phê, audit log
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
9. [Danh mục nghiệp vụ văn bản](#9-danh-mục-nghiệp-vụ-văn-bản)
10. [Incoming Documents – Văn bản đến](#10-incoming-documents--văn-bản-đến)
11. [Outgoing Documents – Văn bản đi](#11-outgoing-documents--văn-bản-đi)
12. [Attachments (File) – Upload/Preview/Download](#12-attachments-file--uploadpreviewdownload)
13. [Assignments – Giao xử lý](#13-assignments--giao-xử-lý)
14. [Comments/Directives – Bút phê/Ý kiến](#14-commentsdirectives--bút-phêý-kiến)
15. [Timeline – Lịch sử xử lý](#15-timeline--lịch-sử-xử-lý)
16. [Search/Export](#16-searchexport)
17. [Notifications](#17-notifications)
18. [Audit Logs](#18-audit-logs)
19. [Phân quyền (RBAC + độ mật) – quy ước thực thi](#19-phân-quyền-rbac--độ-mật--quy-ước-thực-thi)
20. [Checklist MVP (đủ để làm app)](#20-checklist-mvp-đủ-để-làm-app)

---

## 1. Quy ước chung
- Format JSON: `snake_case` (nhất quán với backend phổ biến)
- Time: ISO 8601 UTC (ví dụ `2026-03-03T10:15:30Z`)
- ID: integer tăng dần (DB), nhưng FE nên dùng `doc_code` để hiển thị/định danh ổn định (không đoán ID).
- Độ mật/độ khẩn: lấy theo master data, FE dùng id + code.

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
- `Cache-Control`: với master data có thể cache (tuỳ)

---

## 3. Auth & Session
### 3.1 JWT
- `access_token`: dùng gọi API, TTL ngắn
- `refresh_token`: làm mới token, TTL dài

### 3.2 Quy tắc thực tế cho app
- FE lưu token (secure storage), tự refresh khi gặp 401 token_expired
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
  "data": { },
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
- `page` (default 1)
- `page_size` (default 20, max 200)
- `search` (chuỗi tìm kiếm)
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
Mã lỗi gợi ý:
- `unauthorized`, `forbidden`, `not_found`
- `validation_error`, `conflict`
- `business_rule_violation` (422)

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

### 7.4 POST `/auth/logout` (khuyến nghị)
Dùng để revoke refresh token (nếu backend quản lý token)
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
      "department": { "id": 1, "name": "Phòng Hành chính" },
      "role": { "id": 2, "code": "van_thu", "name": "Văn thư" }
    }
  ],
  "meta": { "page": 1, "page_size": 20, "total": 1 }
}
```

### 8.2 POST `/users`
### 8.3 GET `/users/{id}`
### 8.4 PATCH `/users/{id}`
### 8.5 DELETE `/users/{id}`
### 8.6 PUT `/users/{id}/password`
### 8.7 GET `/users/{id}/audit-logs` (thay cho logs kiểu task)

---

## 9. Danh mục nghiệp vụ văn bản (Master Data)

### 9.1 Document Types
- GET `/document-types`
- POST `/document-types`
- PATCH `/document-types/{id}`
- DELETE `/document-types/{id}` (thực tế nên chuyển thành inactive)

**Response GET**
```json
{
  "data": [
    { "id": 1, "code": "CV", "name": "Công văn", "is_active": true },
    { "id": 2, "code": "QD", "name": "Quyết định", "is_active": true }
  ]
}
```

### 9.2 Urgency Levels
- GET `/urgency-levels`

### 9.3 Confidentiality Levels
- GET `/confidentiality-levels`

### 9.4 Organizations
- GET `/organizations`
- POST `/organizations`

### 9.5 Field Sectors
- GET `/field-sectors`

> App cần cache master data để load nhanh màn “Nhập văn bản”.

---

## 10. Incoming Documents – Văn bản đến

### 10.1 GET `/incoming-documents`
**Query thực tế cho app**
- paging: `page,page_size`
- search: `search` (tìm theo số/ký hiệu, trích yếu)
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

**Response 200**
```json
{
  "data": [
    {
      "id": 101,
      "doc_code": "INC-2026-000101",
      "incoming_number": "123",
      "symbol_number": "01/CV-ABC",
      "document_name": "Công văn về việc ...",
      "summary": "Trích yếu ...",
      "status": "assigned",
      "received_date": "2026-03-01",
      "due_date": "2026-03-10",
      "sender_org": { "id": 9, "name": "Sở X" },
      "confidentiality": { "id": 2, "code": "confidential", "name": "Mật" },
      "urgency": { "id": 2, "code": "urgent", "name": "Khẩn" },
      "has_attachments": true,
      "last_activity_at": "2026-03-01T09:05:00Z"
    }
  ],
  "meta": { "page": 1, "page_size": 20, "total": 1 }
}
```

---

### 10.2 POST `/incoming-documents`
Tạo mới văn bản đến

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
  "due_date": "2026-03-10",
  "owner_department_id": 1
}
```

**Response 201**
```json
{
  "data": {
    "id": 101,
    "doc_code": "INC-2026-000101",
    "status": "received",
    "created_at": "2026-03-01T08:00:00Z"
  }
}
```

---

### 10.3 GET `/incoming-documents/{id}`
**Response 200**
```json
{
  "data": {
    "id": 101,
    "doc_code": "INC-2026-000101",
    "incoming_number": "123",
    "symbol_number": "01/CV-ABC",
    "document_name": "Công văn về việc ...",
    "summary": "Trích yếu ...",
    "status": "assigned",
    "received_date": "2026-03-01",
    "due_date": "2026-03-10",
    "sender_org": { "id": 9, "name": "Sở X" },
    "confidentiality": { "id": 2, "code": "confidential", "name": "Mật" },
    "urgency": { "id": 2, "code": "urgent", "name": "Khẩn" },
    "field_sector": { "id": 3, "name": "Tài chính" },
    "attachments": [
      {
        "id": 501,
        "file_name": "scan.pdf",
        "file_type": "pdf",
        "file_size": 1024000,
        "is_main": true,
        "created_at": "2026-03-01T08:10:00Z"
      }
    ],
    "created_by": { "id": 12, "full_name": "Nguyễn Văn Thư" },
    "created_at": "2026-03-01T08:00:00Z",
    "updated_at": "2026-03-02T09:00:00Z"
  }
}
```

---

### 10.4 PATCH `/incoming-documents/{id}`
Cập nhật metadata (không cho sửa nếu đã “lưu trữ” hoặc theo policy)

**Request**
```json
{
  "summary": "Trích yếu cập nhật",
  "due_date": "2026-03-12",
  "urgency_level_id": 1
}
```

**Response 200**
```json
{ "message": "Cập nhật thành công" }
```

---

### 10.5 PATCH `/incoming-documents/{id}/status`
Đổi trạng thái vòng đời (tuỳ nghiệp vụ)
**Request**
```json
{ "status": "processing" }
```

---

## 11. Outgoing Documents – Văn bản đi

### 11.1 GET `/outgoing-documents`
Query: giống incoming, thêm `publish_from,publish_to`

### 11.2 POST `/outgoing-documents`
Tạo nháp văn bản đi

**Request**
```json
{
  "outgoing_number": "45",
  "symbol_number": "02/QD-XYZ",
  "document_name": "Quyết định về việc ...",
  "summary": "Trích yếu ...",
  "document_type_id": 2,
  "urgency_level_id": 1,
  "confidentiality_level_id": 1,
  "field_sector_id": 2
}
```

**Response 201**
```json
{ "data": { "id": 201, "doc_code": "OUT-2026-000201", "status": "draft" } }
```

### 11.3 POST `/outgoing-documents/{id}/versions`
Tạo phiên bản dự thảo
**Request**
```json
{ "version_no": 1, "title": "Dự thảo lần 1", "content_text": "..." }
```
**Response 201**
```json
{ "data": { "id": 901, "version_no": 1, "is_current": true } }
```

### 11.4 GET `/outgoing-documents/{id}/versions`
Danh sách version

### 11.5 PATCH `/outgoing-documents/{id}/versions/{version_id}`
Cập nhật nội dung version (tuỳ policy)

### 11.6 POST `/outgoing-documents/{id}/publish`
Phát hành
**Request**
```json
{
  "publish_date": "2026-03-02",
  "recipients": [
    { "recipient_org_id": 9, "method": "paper" },
    { "recipient_name": "Đơn vị ngoài danh mục", "method": "email" }
  ]
}
```
**Response 200**
```json
{ "message": "Phát hành thành công" }
```

### 11.7 POST `/outgoing-documents/{id}/recall`
Thu hồi văn bản đi
**Request**
```json
{ "reason": "Phát hành nhầm nơi nhận" }
```

---

## 12. Attachments (File) – Upload/Preview/Download
> Đây là phần app cần nhất ngoài list/detail: upload + preview + download có kiểm soát.

### 12.1 POST `/incoming-documents/{id}/attachments`
multipart/form-data:
- `file` (binary)
- `is_main` (boolean)
- `note` (string, optional)

**Response 201**
```json
{
  "data": {
    "id": 501,
    "file_name": "scan.pdf",
    "file_type": "pdf",
    "file_size": 1024000,
    "is_main": true,
    "created_at": "2026-03-01T08:10:00Z"
  }
}
```

### 12.2 POST `/outgoing-documents/{id}/attachments`
tương tự

### 12.3 GET `/attachments/{id}/download`
- Backend kiểm tra quyền theo: role + department + confidentiality + download policy
- Trả file stream

**Response 200**
- `Content-Type: application/octet-stream`

### 12.4 GET `/attachments/{id}/preview`
Trả preview URL ngắn hạn (signed URL) để app mở PDF/image nhanh

**Response 200**
```json
{
  "data": {
    "preview_url": "https://cdn.example.com/signed/....",
    "expires_at": "2026-03-03T10:20:00Z"
  }
}
```

### 12.5 DELETE `/attachments/{id}`
Xoá mềm (deleted=true), bắt buộc quyền + log

---

## 13. Assignments – Giao xử lý

### 13.1 POST `/assignments`
**Request**
```json
{
  "document_kind": "incoming",
  "document_id": 101,
  "assign_role": "primary",
  "assignee_department_id": 3,
  "assignee_user_id": null,
  "directive": "Phòng A chủ trì xử lý, báo cáo kết quả",
  "due_date": "2026-03-10"
}
```

**Response 201**
```json
{
  "data": {
    "id": 7001,
    "status": "new",
    "assigned_at": "2026-03-01T09:00:00Z"
  }
}
```

### 13.2 GET `/assignments`
Dùng cho “Văn bản được giao cho tôi/đơn vị tôi”
**Query**
- `assignee_user_id`
- `assignee_department_id`
- `status`
- `document_kind`
- `due_from,due_to`

### 13.3 PATCH `/assignments/{id}`
Cập nhật trạng thái xử lý
**Request**
```json
{ "status": "processing" }
```

### 13.4 PATCH `/assignments/{id}/complete`
Hoàn thành + ghi kết quả
**Request**
```json
{
  "result_note": "Đã xử lý xong",
  "result_attachment_ids": [801, 802]
}
```

### 13.5 POST `/assignments/{id}/revoke`
Thu hồi giao xử lý
**Request**
```json
{ "reason": "Giao nhầm đơn vị" }
```

---

## 14. Comments/Directives – Bút phê/Ý kiến

### 14.1 POST `/comments`
**Request**
```json
{
  "document_kind": "incoming",
  "document_id": 101,
  "comment_type": "directive",
  "content": "Chuyển phòng A xử lý, báo cáo trước 10/03"
}
```

**Response 201**
```json
{ "data": { "id": 8001, "created_at": "2026-03-01T09:05:00Z" } }
```

### 14.2 GET `/comments`
**Query**
- `document_kind`
- `document_id`

### 14.3 PATCH `/comments/{id}` (tuỳ policy)
Nếu cho sửa, bắt buộc lưu `edit_reason`

---

## 15. Timeline – Lịch sử xử lý
> Timeline giúp app hiển thị “ai làm gì” theo thời gian (khác audit log là hành vi kỹ thuật view/download).

### 15.1 GET `/incoming-documents/{id}/timeline`
### 15.2 GET `/outgoing-documents/{id}/timeline`

**Response 200**
```json
{
  "data": [
    {
      "event": "created",
      "at": "2026-03-01T08:00:00Z",
      "by": { "id": 12, "full_name": "Nguyễn Văn Thư" },
      "note": "Tiếp nhận văn bản"
    },
    {
      "event": "comment_created",
      "at": "2026-03-01T09:05:00Z",
      "by": { "id": 2, "full_name": "Lãnh đạo" },
      "note": "Chuyển phòng A xử lý..."
    },
    {
      "event": "assignment_created",
      "at": "2026-03-01T09:00:00Z",
      "by": { "id": 2, "full_name": "Lãnh đạo" },
      "note": "Giao phòng A chủ trì, hạn 10/03"
    }
  ]
}
```

---

## 16. Search/Export

### 16.1 GET `/documents/search`
**Query**
- `kind=incoming|outgoing|all`
- `search`
- `symbol_number`
- `incoming_number`
- `outgoing_number`
- `org_id`
- `signer`
- `status`
- `confidentiality_level_id`
- `urgency_level_id`
- `from,to`
- `page,page_size,sort`

### 16.2 GET `/documents/export`
**Query**
- giống `/documents/search`
- `format=excel|pdf`

---

## 17. Notifications

### 17.1 GET `/notifications`
Query: `unread=true|false`
**Response**
```json
{
  "data": [
    {
      "id": 90001,
      "type": "assignment",
      "content": "B���n được giao xử lý INC-2026-000101",
      "read": false,
      "created_at": "2026-03-01T09:00:00Z",
      "ref": { "document_kind": "incoming", "document_id": 101 }
    }
  ]
}
```

### 17.2 PATCH `/notifications/{id}/read`
### 17.3 PATCH `/notifications/read-all` (khuyến nghị)

---

## 18. Audit Logs
> Audit là bắt buộc để truy vết “xem/tải/in/chuyển”.

### 18.1 GET `/audit-logs`
**Query**
- `user_id`
- `action`
- `document_kind`
- `document_id`
- `attachment_id`
- `from,to`
- `page,page_size`

**Response**
```json
{
  "data": [
    {
      "id": 600001,
      "user": { "id": 12, "full_name": "Nguyễn Văn Thư" },
      "action": "download_file",
      "document_kind": "incoming",
      "document_id": 101,
      "attachment_id": 501,
      "ip_address": "203.0.113.5",
      "created_at": "2026-03-01T10:00:00Z",
      "result": "success"
    }
  ],
  "meta": { "page": 1, "page_size": 20, "total": 1 }
}
```

### 18.2 GET `/incoming-documents/{id}/audit-logs`
### 18.3 GET `/outgoing-documents/{id}/audit-logs`

---

## 19. Phân quyền (RBAC + độ mật) – quy ước thực thi
### 19.1 Permission codes (gợi ý)
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

### 19.2 Rule độ mật (thực tế)
- Khi văn bản có `confidentiality_level >= secret`:
  - chỉ user thuộc đúng đơn vị/được giao/được uỷ quyền mới xem
  - hạn chế download/print theo policy
  - bắt buộc ghi audit khi view/download/print

---

## 20. Checklist MVP (đủ để làm app)
✅ Auth: login/refresh/whoami/logout  
✅ Master data: document-types, confidentiality-levels, urgency-levels, organizations, field-sectors  
✅ Incoming: list/create/detail/update/change_status/upload/timeline  
✅ Outgoing: list/create/detail/update/version/publish/recall/upload/timeline  
✅ Assignment: create/list/update/complete/revoke  
✅ Comment: create/list  
✅ Attachment: preview/download/delete  
✅ Search + Export  
✅ Notifications: list/mark read/read-all  
✅ Audit logs: list/by-document  

---
