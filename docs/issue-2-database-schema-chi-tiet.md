# Phân tích trường dữ liệu (Data Dictionary) – App Quản lý Văn bản (Đến/Đi)

> Mục tiêu: liệt kê **trường dữ liệu cần có** để triển khai app quản lý văn bản.
>
> Nguyên tắc:
> - Tách rõ **Văn bản đến** và **Văn bản đi**
> - Có **độ mật/độ khẩn**, phân quyền theo vai trò/đơn vị
> - Có **luân chuyển xử lý + bút phê + timeline**
> - Có **audit log** cho hành vi xem/tải/in/chuyển

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

---

### 2.2 Position (Chức vụ)
- id (PK)
- name (nvarchar(200), not null)
- level (int, null) — để sắp xếp
- is_active (boolean, default true)

---

### 2.3 Role / Permission
**Role**
- id (PK)
- code (varchar(50), unique) — `van_thu`, `lanh_dao`, `truong_phong`, `chuyen_vien`, `luu_tru`, `admin`
- name (nvarchar(200))
- description (nvarchar(500), null)

**Permission**
- id (PK)
- code (varchar(100), unique) — ví dụ: `incoming.view`, `incoming.assign`, `outgoing.publish`, `audit.view`
- description (nvarchar(500), null)

**RolePermission**
- role_id (FK)
- permission_id (FK)
- unique(role_id, permission_id)

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

---

### 2.5 Delegation (Uỷ quyền xử lý khi vắng mặt)
- id (PK)
- delegator_user_id (FK -> users.id) — người uỷ quyền
- delegate_user_id (FK -> users.id) — người được uỷ quyền
- scope (json/text) — phạm vi: loại văn bản nào, độ mật nào, module nào…
- start_at (datetime)
- end_at (datetime)
- is_active (boolean, default true)
- created_at

**Ràng buộc**
- end_at > start_at
- không cho uỷ quyền vòng (A→B và B→A) (tuỳ mức chặt)

---

## 3) Văn bản đến (Incoming Document)

### 3.1 IncomingDocument
**Nhóm định danh & thông tin gốc**
- id (PK)
- doc_code (varchar(50), unique, not null) — mã nội bộ (UUID/chuỗi), để API/UI dùng ổn định
- incoming_number (varchar(50), null) — số đến (nếu có)
- symbol_number (varchar(100), null) — số/ký hiệu trên văn bản
- document_name (nvarchar(500), not null) — tiêu đề/tên văn bản (tuỳ bạn)
- summary (nvarchar(2000), null) — trích yếu

**Nhóm phân loại**
- document_type_id (FK -> document_types.id, null)
- urgency_level_id (FK -> urgency_levels.id, null)
- confidentiality_level_id (FK -> confidentiality_levels.id, not null)
- field_sector_id (FK -> field_sectors.id, null)

**Nhóm nguồn gửi**
- sender_org_id (FK -> organizations.id, null)
- signer (nvarchar(200), null) — người ký
- position_signer (nvarchar(200), null) — chức vụ người ký (tuỳ)
- issued_date (date, null) — ngày ban hành trên văn bản
- received_date (date, null) — ngày đến/tiếp nhận
- received_channel (enum: paper, email, portal, other) null

**Nhóm xử lý**
- status (enum):
  - draft (mới tạo nháp)
  - received (đã tiếp nhận)
  - submitted (đã trình)
  - assigned (đã giao xử lý)
  - processing (đang xử lý)
  - completed (đã hoàn thành xử lý)
  - archived (đã lưu trữ)
  - cancelled (huỷ/không xử lý)
- due_date (date, null) — hạn xử lý
- owner_department_id (FK -> departments.id, null) — đơn vị chủ quản/đầu mối
- created_by (FK -> users.id, not null)
- created_at, updated_at

**Nhóm bảo mật**
- allow_download (boolean, default true) — tuỳ chính sách theo độ mật
- allow_print (boolean, default true)
- watermark_required (boolean, default false)

**Index gợi ý**
- symbol_number
- incoming_number
- received_date
- due_date
- sender_org_id
- confidentiality_level_id
- status

**Ràng buộc gợi ý**
- Khi `confidentiality_level` cao → ép `allow_download=false` hoặc yêu cầu watermark (tuỳ quy định)

---

### 3.2 IncomingDocumentAttachment (File đính kèm văn bản đến)
- id (PK)
- incoming_document_id (FK -> incoming_documents.id, on delete cascade)
- file_name (nvarchar(255), not null)
- file_path (varchar(500), not null) — đường dẫn lưu trữ
- file_type (varchar(50), null) — pdf, docx, jpg…
- file_size (bigint, null)
- sha256 (varchar(64), null) — kiểm tra toàn vẹn (khuyến nghị)
- is_main (boolean, default false) — file chính
- uploaded_by (FK -> users.id)
- uploaded_at (datetime)
- deleted (boolean, default false)

**Index**
- incoming_document_id
- is_main

---

## 4) Văn bản đi (Outgoing Document)

### 4.1 OutgoingDocument
**Nhóm định danh**
- id (PK)
- doc_code (varchar(50), unique, not null)
- outgoing_number (varchar(50), null) — số đi (nếu có)
- symbol_number (varchar(100), null) — số/ký hiệu

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
- issued_date (date, null) — ngày ký/ngày ban hành
- publish_date (date, null) — ngày phát hành/gửi đi
- status (enum):
  - draft
  - in_review (đang trình duyệt)
  - approved (đã duyệt)
  - signed (đã ký/đã ký số)
  - published (đã phát hành)
  - recalled (thu hồi)
  - archived (lưu trữ)
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
- version_no (int, not null) — 1,2,3…
- title (nvarchar(500), null)
- content_text (text, null) — nếu lưu nội dung; hoặc chỉ lưu file
- created_by (FK -> users.id)
- created_at
- is_current (boolean, default false)

**Ràng buộc**
- unique(outgoing_document_id, version_no)
- chỉ 1 bản `is_current=true` (tuỳ)

---

### 4.3 OutgoingDocumentAttachment
- id (PK)
- outgoing_document_id (FK)
- file_name, file_path, file_type, file_size, sha256
- is_main (boolean)
- uploaded_by, uploaded_at
- deleted (boolean)

---

### 4.4 OutgoingRecipient (Nơi nhận / lịch sử gửi)
- id (PK)
- outgoing_document_id (FK)
- recipient_org_id (FK -> organizations.id, nullable) — nơi nhận là cơ quan
- recipient_name (nvarchar(255), null) — nếu gửi cá nhân/đơn vị không nằm trong danh mục
- method (enum: paper, email, portal, other)
- sent_at (datetime, null)
- status (enum: pending, sent, failed, confirmed) default pending
- note (nvarchar(500), null)

**Index**
- outgoing_document_id
- recipient_org_id

---

### 4.5 Link Incoming ↔ Outgoing (Trả lời/Căn cứ)
- id (PK)
- incoming_document_id (FK -> incoming_documents.id)
- outgoing_document_id (FK -> outgoing_documents.id)
- link_type (enum: reply, reference, related) — trả lời/căn cứ/liên quan
- created_at

**Ràng buộc**
- unique(incoming_document_id, outgoing_document_id, link_type)

---

## 5) Giao xử lý – bút phê – timeline (dùng chung cho VB đến và/hoặc VB đi)

### 5.1 Assignment (Giao xử lý)
> Có 2 cách:
> - Cách A: tách riêng IncomingAssignment / OutgoingAssignment
> - Cách B: 1 bảng dùng chung, có `document_kind` + `document_id`
>
> Ở đây mô tả theo cách B (dùng chung) để app dễ mở rộng.

- id (PK)
- document_kind (enum: incoming, outgoing)
- document_id (int) — id văn bản tương ứng
- assigned_by (FK -> users.id) — ai giao
- assignee_department_id (FK -> departments.id, null) — giao cho đơn vị
- assignee_user_id (FK -> users.id, null) — hoặc giao cho cá nhân
- assign_role (enum: primary, cooperate, notify) — chủ trì/phối hợp/để biết
- directive (nvarchar(2000), null) — nội dung giao việc/chỉ đạo
- assigned_at (datetime)
- due_date (date, null)
- status (enum: new, accepted, processing, completed, rejected, revoked)
- completed_at (datetime, null)
- revoked_at (datetime, null)
- revoke_reason (nvarchar(500), null)

**Ràng buộc**
- Phải có ít nhất 1 trong 2: assignee_department_id hoặc assignee_user_id
- Khi revoked → bắt buộc revoke_reason

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
- comment_type (enum: directive, comment, note) — bút phê/ý kiến/ghi chú
- content (nvarchar(4000) or text, not null)
- created_at
- edited_at (datetime, null)
- edit_reason (nvarchar(255), null)

**Khuyến nghị**
- Với “bút phê” quan trọng: hạn chế sửa/xoá; nếu cho sửa phải lưu version.

---

## 6) Audit log (Nhật ký hệ thống – bắt buộc)

### 6.1 AuditLog
- id (PK)
- user_id (FK -> users.id, null) — null nếu system
- action (varchar(50), not null) — `login`, `view_document`, `download_file`, `print`, `assign`, `revoke_assign`, `update_metadata`, `publish_outgoing`, ...
- document_kind (incoming/outgoing/null)
- document_id (int, null)
- attachment_id (int, null)
- entity_type (varchar(50), null) — nếu log cho danh mục/user…
- entity_id (int, null)
- result (enum: success, denied, failed) default success
- ip_address (varchar(45), null)
- user_agent (varchar(300), null)
- metadata_json (json/text, null) — chi tiết trước/sau, lý do, payload rút gọn
- created_at (datetime)

**Index**
- user_id, created_at
- action, created_at
- (document_kind, document_id), created_at

**Rủi ro/điểm cần chú ý**
- Log sẽ rất lớn → cần retention/partition/index tốt.
- Không cho sửa/xoá log (append-only) hoặc chỉ super admin theo quy trình.

---

## 7) Gợi ý kiểu dữ liệu & validate tối thiểu
- Các trường số/ký hiệu: giới hạn độ dài, loại bỏ ký tự nguy hiểm, chuẩn hoá khoảng trắng
- File: giới hạn dung lượng; whitelist file_type; hash sha256 để kiểm tra toàn vẹn
- `doc_code`: nên dùng UUID/ULID để API ổn định, tránh đoán ID
- `confidentiality_level_id`: bắt buộc, vì mọi văn bản phải có mức độ bảo mật

---

## 8) Mapping nhanh sang API (để làm app)
- Incoming:
  - Tạo/sửa văn bản đến + upload file + trình bút phê + giao xử lý + xem timeline + audit
- Outgoing:
  - Tạo/sửa văn bản đi + version + trình duyệt + phát hành + nơi nhận + liên kết incoming + audit
- Search/export:
  - Filter theo các trường index mạnh: symbol_number, incoming_number/outgoing_number, org, date, status, confidentiality, urgency, assignee

---
