# Phân tích trường dữ liệu (Data Dictionary) – App Quản lý Văn bản (Đến/Đi)

> Mục tiêu: liệt kê **trường dữ liệu cần có** để triển khai app quản lý văn bản.
>
> Nguyên tắc:
> - Tách rõ **Văn bản đến** và **Văn bản đi**
> - Có **độ mật/độ khẩn**, phân quyền theo vai trò/đơn vị
> - Có **luân chuyển xử lý + bút phê + timeline**
> - Có **audit log** cho hành vi xem/tải/in/chuyển

---

## 0) Quy ước dữ liệu dùng chung (khuyến nghị)

### 0.1 Soft delete / audit columns
Khuyến nghị mọi bảng nghiệp vụ quan trọng có các cột sau (tuỳ hệ DB):
- `created_at` (datetime, not null)
- `updated_at` (datetime, not null)
- `created_by` (FK -> users.id, nullable tuỳ bảng)
- `updated_by` (FK -> users.id, nullable tuỳ bảng)
- `deleted_at` (datetime, null) **hoặc** `deleted` (boolean, default false)
- `deleted_by` (FK -> users.id, null) (nếu cần truy vết xóa)
- `row_version` (int/bigint, default 1) (tuỳ chọn) để optimistic locking

### 0.2 Chuẩn enum
- Tránh enum “cứng” ở DB nếu hệ thống hay đổi; có thể dùng varchar + check constraint.
- Nếu dùng enum, cần thống nhất với API docs.

---

## 1) Nhóm danh mục dùng chung (Master Data)

### 1.1 DocumentType (Loại văn bản)
- id (PK, int)
- code (varchar(50), unique, not null) — ví dụ: `CV`, `TB`, `QD`
- name (nvarchar(200), not null)
- description (nvarchar(500), null)
- is_active (boolean, default true)
- created_at, updated_at

**Ràng buộc**
- Không xoá khi đã được dùng → chỉ cho `is_active=false`

**Index**
- unique(code)
- name

---

### 1.2 UrgencyLevel (Độ khẩn)
- id (PK)
- code (varchar(30), unique) — `normal`, `urgent`, `express`
- name (nvarchar(100))
- level (int) — số càng nhỏ càng khẩn (tuỳ)
- is_active (boolean, default true)

---

### 1.3 ConfidentialityLevel (Độ mật)
- id (PK)
- code (varchar(30), unique) — `public`, `confidential`, `secret`, `top_secret`
- name (nvarchar(100)) — Bình thường/Mật/Tối mật/Tuyệt mật
- level (int) — để so sánh cấp độ
- restrict_download (boolean, default false) — nếu true: chặn tải xuống (tuỳ chính sách)
- restrict_print (boolean, default false) — nếu true: chặn in (tuỳ)
- watermark_required (boolean, default false) — nếu true: bắt buộc watermark
- is_active (boolean, default true)

**Index**
- unique(code)
- level

---

### 1.4 FieldSector (Lĩnh vực)
- id (PK)
- code (varchar(50), unique)
- name (nvarchar(200))
- is_active (boolean)

---

### 1.5 Organization (Cơ quan gửi/nhận)
- id (PK)
- code (varchar(50), unique, null) — nếu cần chuẩn hoá
- name (nvarchar(255), not null)
- address (nvarchar(255), null)
- phone (varchar(30), null)
- email (varchar(100), null)
- is_active (boolean, default true)
- created_at, updated_at

**Index**
- name
- code

---

## 2) Nhóm tổ chức – người dùng – phân quyền

### 2.1 Department/OrgUnit (Đơn vị/Phòng ban)
- id (PK)
- code (varchar(30), unique, not null)
- name (nvarchar(200), not null)
- parent_id (FK -> departments.id, nullable)
- is_active (boolean, default true)
- created_at, updated_at

**Index**
- unique(code)
- parent_id

---

### 2.2 Position (Chức vụ)
- id (PK)
- name (nvarchar(200), not null)
- level (int, null) — để sắp xếp
- is_active (boolean, default true)
- created_at, updated_at

---

### 2.3 Role / Permission
**Role**
- id (PK)
- code (varchar(50), unique) — `van_thu`, `lanh_dao`, `truong_phong`, `chuyen_vien`, `luu_tru`, `admin`
- name (nvarchar(200))
- description (nvarchar(500), null)
- is_active (boolean, default true)
- created_at, updated_at

**Permission**
- id (PK)
- code (varchar(100), unique) — ví dụ: `incoming.view`, `attachment.download`, `audit.view`
- description (nvarchar(500), null)
- is_active (boolean, default true)
- created_at, updated_at

**RolePermission**
- role_id (FK)
- permission_id (FK)
- unique(role_id, permission_id)

**Index**
- permission(code)
- role(code)

---

### 2.4 User (Người dùng)
- id (PK)
- username (varchar(50), unique, not null)
- password_hash (varchar(255), not null)
- full_name (nvarchar(200), not null)
- email (varchar(150), unique, null)
- phone (varchar(30), null)
- avatar_url (varchar(255), null)

- department_id (FK -> departments.id, not null)
- position_id (FK -> positions.id, null)
- role_id (FK -> roles.id, not null)

- status (enum: active, locked, inactive) default active
- last_login_at (datetime, null)

- two_fa_enabled (boolean, default false)
- login_retry_count (int, default 0)
- must_change_password (boolean, default false)

- created_at, updated_at

**Index**
- username unique
- department_id
- role_id
- status

---

### 2.5 Delegation (Uỷ quyền xử lý khi vắng mặt)
- id (PK)
- delegator_user_id (FK -> users.id) — người uỷ quyền
- delegate_user_id (FK -> users.id) — người được uỷ quyền
- scope (json/text) — phạm vi: module nào, độ mật nào, department nào…
- start_at (datetime)
- end_at (datetime)
- is_active (boolean, default true)
- created_at

**Ràng buộc**
- end_at > start_at
- (khuyến nghị) Không cho uỷ quyền trùng thời gian quá nhiều bản ghi cho cùng delegator

**Index**
- delegator_user_id
- delegate_user_id
- start_at, end_at
- is_active

---

## 3) Văn bản đến (Incoming Document)

### 3.1 IncomingDocument
**Nhóm định danh & thông tin gốc**
- id (PK)
- doc_code (varchar(50), unique, not null) — mã nội bộ (UUID/ULID), để API/UI dùng ổn định
- incoming_number (varchar(50), null) — số đến (nếu có)
- symbol_number (varchar(100), null) — số/ký hiệu trên văn bản
- document_name (nvarchar(500), not null)
- summary (nvarchar(2000), null)

**Nhóm phân loại**
- document_type_id (FK -> document_types.id, null)
- urgency_level_id (FK -> urgency_levels.id, null)
- confidentiality_level_id (FK -> confidentiality_levels.id, not null)
- field_sector_id (FK -> field_sectors.id, null)

**Nhóm nguồn gửi**
- sender_org_id (FK -> organizations.id, null)
- signer (nvarchar(200), null)
- position_signer (nvarchar(200), null)
- issued_date (date, null)
- received_date (date, null)
- received_channel (enum: paper, email, portal, other) null

**Nhóm xử lý**
- status (enum):
  - draft
  - received
  - submitted
  - assigned
  - processing
  - completed
  - archived
  - cancelled
- due_date (date, null)
- owner_department_id (FK -> departments.id, null)
- created_by (FK -> users.id, not null)
- created_at, updated_at

**Nhóm bảo mật (policy snapshot)**
> Có thể lấy từ ConfidentialityLevel, nhưng khuyến nghị lưu snapshot tại document để áp chính sách theo thời điểm.
- allow_download (boolean, default true)
- allow_print (boolean, default true)
- watermark_required (boolean, default false)

**Index gợi ý**
- symbol_number
- incoming_number
- received_date
- due_date
- sender_org_id
- owner_department_id
- confidentiality_level_id
- status

---

### 3.2 IncomingDocumentAttachment (File đính kèm văn bản đến)
- id (PK)
- incoming_document_id (FK -> incoming_documents.id, on delete cascade)
- file_name (nvarchar(255), not null)
- file_path (varchar(500), not null)
- file_type (varchar(50), null) — pdf, docx, jpg…
- file_size (bigint, null)
- sha256 (varchar(64), null) — khuyến nghị
- is_main (boolean, default false)
- note (nvarchar(500), null)
- uploaded_by (FK -> users.id)
- uploaded_at (datetime)
- deleted (boolean, default false)

**Index**
- incoming_document_id
- is_main
- deleted

---

## 4) Văn bản đi (Outgoing Document)

### 4.1 OutgoingDocument
**Nhóm định danh**
- id (PK)
- doc_code (varchar(50), unique, not null)
- outgoing_number (varchar(50), null)
- symbol_number (varchar(100), null)

**Nội dung**
- document_name (nvarchar(500), not null)
- summary (nvarchar(2000), null)

**Phân loại**
- document_type_id (FK)
- urgency_level_id (FK)
- confidentiality_level_id (FK, not null)
- field_sector_id (FK, null)

**Thông tin ban hành**
- signer (nvarchar(200), null)
- issued_date (date, null)
- publish_date (date, null)
- status (enum):
  - draft
  - in_review
  - approved
  - signed
  - published
  - recalled
  - archived
- created_by (FK -> users.id)
- created_at, updated_at

**Index**
- outgoing_number
- symbol_number
- publish_date
- confidentiality_level_id
- status

---

### 4.2 OutgoingDocumentVersion (Phiên bản/dự thảo)
- id (PK)
- outgoing_document_id (FK -> outgoing_documents.id, on delete cascade)
- version_no (int, not null)
- title (nvarchar(500), null)
- content_text (text, null)
- created_by (FK -> users.id)
- created_at
- is_current (boolean, default false)

**Ràng buộc**
- unique(outgoing_document_id, version_no)

**Index**
- outgoing_document_id
- is_current

---

### 4.3 OutgoingDocumentAttachment
- id (PK)
- outgoing_document_id (FK)
- file_name (nvarchar(255), not null)
- file_path (varchar(500), not null)
- file_type (varchar(50), null)
- file_size (bigint, null)
- sha256 (varchar(64), null)
- is_main (boolean, default false)
- note (nvarchar(500), null)
- uploaded_by (FK -> users.id)
- uploaded_at (datetime)
- deleted (boolean, default false)

**Index**
- outgoing_document_id
- is_main
- deleted

---

### 4.4 OutgoingRecipient (Nơi nhận / lịch sử gửi)
- id (PK)
- outgoing_document_id (FK)
- recipient_org_id (FK -> organizations.id, nullable)
- recipient_name (nvarchar(255), null) — nếu không thuộc danh mục
- method (enum: paper, email, portal, other)
- sent_at (datetime, null)
- status (enum: pending, sent, failed, confirmed) default pending
- note (nvarchar(500), null)
- created_at

**Index**
- outgoing_document_id
- recipient_org_id
- status
- sent_at

---

### 4.5 Link Incoming ↔ Outgoing (Trả lời/Căn cứ)
- id (PK)
- incoming_document_id (FK -> incoming_documents.id)
- outgoing_document_id (FK -> outgoing_documents.id)
- link_type (enum: reply, reference, related)
- created_at

**Ràng buộc**
- unique(incoming_document_id, outgoing_document_id, link_type)

**Index**
- incoming_document_id
- outgoing_document_id
- link_type

---

## 5) Giao xử lý – bút phê – timeline (dùng chung)

### 5.1 Assignment (Giao xử lý)
- id (PK)
- document_kind (enum: incoming, outgoing)
- document_id (int)
- assigned_by (FK -> users.id)
- assignee_department_id (FK -> departments.id, null)
- assignee_user_id (FK -> users.id, null)
- assign_role (enum: primary, cooperate, notify)
- directive (nvarchar(2000), null)
- assigned_at (datetime)
- due_date (date, null)
- status (enum: new, accepted, processing, completed, rejected, revoked)
- completed_at (datetime, null)
- revoked_at (datetime, null)
- revoke_reason (nvarchar(500), null)

**Ràng buộc**
- assignee_department_id hoặc assignee_user_id phải có ít nhất 1
- revoked => revoke_reason not null

**Index**
- (document_kind, document_id)
- assignee_user_id
- assignee_department_id
- due_date
- status

---

### 5.2 Comment/Directive (Bút phê/ý kiến)
- id (PK)
- document_kind (incoming/outgoing)
- document_id (int)
- author_user_id (FK -> users.id)
- comment_type (enum: directive, comment, note)
- content (nvarchar(4000) or text, not null)
- created_at
- edited_at (datetime, null)
- edit_reason (nvarchar(255), null)
- deleted (boolean, default false) (tuỳ policy)

**Khuyến nghị**
- Với directive quan trọng: không cho xoá/sửa hoặc lưu version.

**Index**
- (document_kind, document_id)
- author_user_id
- created_at

---

## 5.3 (BỔ SUNG) DocumentTimelineEvent (Timeline nghiệp vụ – khuyến nghị)
> Tách khỏi AuditLog để timeline “đúng nghiệp vụ” và query nhanh.

- id (PK)
- document_kind (incoming/outgoing)
- document_id (int)
- event (varchar(50), not null)  
  Ví dụ: `created`, `metadata_updated`, `status_changed`, `assignment_created`, `assignment_completed`,
  `comment_created`, `comment_updated`, `attachment_uploaded`, `attachment_deleted`, `published`, `recalled`
- note (nvarchar(2000), null) — mô tả ngắn để hiển thị
- actor_user_id (FK -> users.id, null) — ai thực hiện (null nếu system)
- metadata_json (json/text, null) — before/after hoặc thông tin bổ sung
- created_at (datetime)

**Index**
- (document_kind, document_id), created_at
- actor_user_id, created_at
- event, created_at

---

## 5.4 (BỔ SUNG) DocumentStatusHistory (lịch sử đổi trạng thái – tuỳ chọn)
> Nếu muốn audit/trace state machine rõ hơn timeline.

- id (PK)
- document_kind (incoming/outgoing)
- document_id (int)
- from_status (varchar(30), null)
- to_status (varchar(30), not null)
- changed_by (FK -> users.id, null)
- change_reason (nvarchar(500), null)
- created_at (datetime)

**Index**
- (document_kind, document_id), created_at

---

## 6) Audit log (Nhật ký hệ thống – bắt buộc)

### 6.1 AuditLog
- id (PK)
- user_id (FK -> users.id, null) — null nếu system
- action (varchar(50), not null) — `login`, `view_document`, `preview_file`, `download_file`, `print`, `export`, ...
- document_kind (incoming/outgoing/null)
- document_id (int, null)
- attachment_id (int, null)
- entity_type (varchar(50), null)
- entity_id (int, null)
- result (enum: success, denied, failed) default success
- ip_address (varchar(45), null)
- user_agent (varchar(300), null)
- metadata_json (json/text, null)
- created_at (datetime)

**Index**
- user_id, created_at
- action, created_at
- (document_kind, document_id), created_at
- attachment_id, created_at

**Khuyến nghị**
- Partition theo tháng/quý nếu dữ liệu lớn.
- Append-only: không cho update/delete.

---

## 7) Notifications (Thông báo)

### 7.1 (BỔ SUNG) Notification
- id (PK)
- user_id (FK -> users.id, not null) — người nhận
- type (varchar(30), not null) — `assignment`, `deadline`, `document`, `system`...
- content (nvarchar(500), not null)
- ref_kind (varchar(30), null) — `incoming`, `outgoing`, `assignment`...
- ref_id (int, null)
- read_at (datetime, null)
- created_at (datetime, not null)

**Index**
- user_id, created_at
- user_id, read_at

### 7.2 (BỔ SUNG) NotificationDevice (push token) (tuỳ chọn)
- id (PK)
- user_id (FK -> users.id)
- platform (enum: ios, android, web)
- device_token (varchar(255))
- is_active (boolean, default true)
- last_seen_at (datetime, null)
- created_at

**Index**
- user_id
- device_token unique

---

## 8) Search/Export (tuỳ chọn nâng cao)

### 8.1 (BỔ SUNG) SavedFilter (lưu bộ lọc)
- id (PK)
- user_id (FK -> users.id)
- name (nvarchar(100))
- kind (enum: incoming, outgoing, all)
- filter_json (json/text) — lưu query/search/sort
- created_at, updated_at
- is_active (boolean, default true)

**Index**
- user_id
- kind

---

## 9) Gợi ý validate tối thiểu
- Chuẩn hoá khoảng trắng, cấm ký tự nguy hiểm trong số/ký hiệu.
- File:
  - giới hạn dung lượng,
  - whitelist MIME/type,
  - tính sha256,
  - (tuỳ chọn) virus scan.
- `doc_code`: ưu tiên UUID/ULID để không đoán được ID.

---
