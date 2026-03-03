## Mục đích
Chi tiết hóa database schema theo chuẩn nghiệp vụ quản lý công việc, phân chia từng bảng/thực thể, thuộc tính (data type, default, validate), khóa ngoại (foreign key), chỉ mục (index), thiết lập logic ràng buộc dữ liệu cùng mối quan hệ đầy đủ giữa các thực thể và gợi ý migration.

---

## 1. Sơ đồ tổng thể thực thể (Textual ERD overview)
Các nhóm thực thể chính:
  - USERs (người dùng), phân vai ROLE, thuộc phòng ban DEPARTMENT, nắm giữ POSITION
  - Công việc (TASK), phân loại (TASK_CATEGORY), trạng thái (TASK_STATUS), mức ưu tiên (TASK_PRIORITY), log, gán nhiều assignee
  - Hồ sơ đính kèm (TASK_FILE), nhật ký (TASK_LOG), nhắc việc (NOTIFICATION), quy trình nghiệp vụ (WORKFLOW, WORKFLOW_STEP)
  - Quan hệ n-n: USER-ROLE, TASK-ASSIGNEE, ROLE-PERMISSION
  - Chỉnh sửa, chuyển giao, báo cáo, chuyển vị trí đều log chi tiết vào CHANGE_LOG

---

## 2. Mô tả bảng, trường dữ liệu và ràng buộc
### 2.1 USERS
  - **id** (PK, int, auto_increment)
  - username (varchar(50), unique, not null)
  - password_hash (varchar(128), not null)
  - full_name (nvarchar(100), not null)
  - email (varchar(100), unique, not null)
  - phone (varchar(20))
  - status (enum: active, locked, inactive), default 'active'
  - avatar_url (varchar(255))
  - created_at (timestamp, default current)
  - updated_at (timestamp)
  - department_id (FK to DEPARTMENT.id, cascade delete/set null)
  - position_id (FK to POSITION.id)
  - role_id (FK to ROLE.id)
  - last_login (timestamp)
  - two_fa_enabled (boolean, default false)
  - login_retry_count (int, default 0)
  - must_change_password (boolean, default false)

**Indexes**: username (unique), email (unique), department_id

### 2.2 ROLES
  - id (PK, int, auto_increment)
  - name (varchar(50), unique, not null)
  - description (text)
**Sample**: Admin, Quản lý, Nhân viên, Trưởng phòng

### 2.3 PERMISSIONS
  - id (PK), code (varchar(50), unique), description (text)

### 2.4 ROLE_PERMISSION
  - id (PK)
  - role_id (FK, not null)
  - permission_id (FK, not null)

UNIQUE (role_id, permission_id)

### 2.5 DEPARTMENTS
  - id (PK)
  - name (varchar(100), not null, unique)
  - parent_id (FK to DEPARTMENTS.id, null if phòng ban gốc)
  - code (varchar(20), unique)
  - description (text)
  - created_at / updated_at (timestamp)

### 2.6 POSITIONS
  - id (PK)
  - name (varchar(50), not null)
  - level (int, for sort)

---

### 2.7 TASK_CATEGORIES
  - id (PK)
  - name (varchar(100), not null, unique)
  - description (text)
  - is_active (boolean, default true)

### 2.8 TASK_PRIORITIES
  - id (PK)
  - name (varchar(50), not null)
  - level (int)
  - is_default (boolean)

### 2.9 TASK_STATUSES
  - id (PK)
  - name (varchar(50), not null)
  - code (varchar(20), unique)
  - is_closed_state (boolean, default false)

---

### 2.10 TASKS
  - id (PK)
  - title (varchar(250), not null)
  - description (text)
  - category_id (FK to TASK_CATEGORIES.id)
  - priority_id (FK to TASK_PRIORITIES.id)
  - status_id (FK to TASK_STATUSES.id)
  - creator_id (FK to USERS.id)
  - created_at (timestamp, default current)
  - updated_at (timestamp)
  - department_id (FK to DEPARTMENTS.id)
  - start_date (date)
  - due_date (date)
  - completed_date (date)
  - progress (tinyint 0-100)
  - confidential (boolean, default false)
  - parent_task_id (FK to TASKS.id, null nếu không phải task con)
  - must_confirm (boolean, default false, yêu cầu xác nhận hoàn thành)
  - last_comment_id (FK to TASK_LOG.id, null)
  - workflow_id (FK to WORKFLOW.id, null)

**INDEXES**: department_id, category_id, priority_id, status_id, due_date, parent_task_id

---

### 2.11 TASK_ASSIGNEE
  - id (PK)
  - task_id (FK to TASKS.id, on delete cascade)
  - user_id (FK to USERS.id)
  - assigned_at (timestamp)
  - active (boolean, default true)
  - is_primary (boolean, default false – phân biệt người gán chính)

  UNIQUE (task_id, user_id)

---

### 2.12 TASK_FILE
  - id (PK)
  - task_id (FK to TASKS.id, on delete cascade)
  - file_path (varchar(300), not null)
  - file_name (varchar(200))
  - uploaded_by (FK to USERS.id)
  - uploaded_at (timestamp)
  - file_type (varchar(20))
  - file_size (int) (bytes)
  - deleted (boolean, default false)

---

### 2.13 TASK_LOG
  - id (PK)
  - task_id (FK to TASKS.id)
  - user_id (FK to USERS.id, null)
  - action (varchar(50): create, update, comment, status_change, file_upload...)
  - content (text)
  - created_at (timestamp)
  - is_system (boolean, default false)

---

### 2.14 WORKFLOW
  - id (PK)
  - name (varchar(100), unique, not null)
  - description (text)
  - is_active (boolean, default true)

### 2.15 WORKFLOW_STEP
  - id (PK)
  - workflow_id (FK to WORKFLOW.id)
  - name (varchar(50))
  - order_index (int)
  - is_approval (boolean)
  - auto_advance (boolean, default false)
  - condition_json (text)
  - role_id (FK to ROLES.id, null, nếu bước này chỉ người vai trò cụ thể được duyệt)

### 2.16 TASK_WORKFLOW
  - id (PK)
  - task_id (FK to TASKS.id)
  - workflow_id (FK to WORKFLOW.id)
  - current_step_id (FK to WORKFLOW_STEP.id)
  - status (enum: running, finished, rejected, revoked)
  - started_at (timestamp)
  - finished_at (timestamp, null)

### 2.17 TASK_WORKFLOW_STEP_LOG
  - id (PK)
  - task_workflow_id (FK to TASK_WORKFLOW.id)
  - step_id (FK to WORKFLOW_STEP.id)
  - user_id (FK to USERS.id)
  - started_at (timestamp)
  - finished_at (timestamp)
  - action (enum: approve, reject, comment, skip)
  - comments (text)
  - approved (boolean, null nếu chưa duyệt)
  - is_revert_step (boolean, default false)

---

### 2.18 NOTIFICATION
  - id (PK)
  - user_id (FK to USERS.id)
  - content (text, not null)
  - type (enum: system, task, file, workflow, message, custom)
  - read_status (boolean, default false)
  - sent_at (timestamp)
  - task_id (FK to TASKS.id, null)

---

### 2.19 REPORT_TEMPLATE
  - id (PK)
  - name (varchar(100))
  - description (text)
  - config_json (text)
  - created_by (FK to USERS.id)
  - created_at (timestamp)

### 2.20 CHANGE_LOG
  - id (PK)
  - entity (varchar(50): USER, TASK, TASK_FILE, DEPARTMENT...)
  - entity_id (int)
  - user_id (FK to USERS.id, null)
  - action (varchar(30): insert, update, delete, login, permission_change, transfer)
  - content (text hoặc json trước-sau)
  - created_at (timestamp)

### 2.21 UNIT_TRANSFER
  - id (PK)
  - user_id (FK to USERS.id)
  - from_department_id (FK to DEPARTMENTS.id)
  - to_department_id (FK to DEPARTMENTS.id)
  - action_by (FK to USERS.id)
  - transfer_type (enum: assign, remove, restore)
  - transfer_at (timestamp)
  - undo_at (timestamp, null)

---

## 3. Quan hệ/Relationships tóm tắt
  - Một USERS thuộc một DEPARTMENT, một POSITION, một/nhiều ROLE
  - TASK: n-1 DEPARTMENT, 1-n LOG, 1-n FILE, n-n ASSIGNEE
  - WORKFLOW cho phép nhiều TASK, mỗi TASK chạy nhiều bước (WORKFLOW_STEP, TASK_WORKFLOW_STEP_LOG)
  - NOTIFICATION ràng với USERS/TASK
  - CHANGE_LOG audit mọi thực thể
  - UNIT_TRANSFER lưu lịch sử chuyển phòng/ban

---

## 4. Ràng buộc integrity/validation gợi ý:
  - ON DELETE CASCADE: TASK→ASSIGNEE, LOG, FILE; USER→TASK_ASSIGNEE
  - ON UPDATE CASCADE: cho các quan hệ FK không phải khóa tự nhiên
  - ENUM, CHECK constraint cho status, type, action fields
  - UNIQUE cho mã phòng ban, username, email, tên category, v.v.

## 5. Index & tối ưu hóa
  - Bắt buộc: INDEX/FULLTEXT cho TASK.title, TASK.description, TASK.status_id, ASSIGNEE, due_date
  - Phổ biến: Index trên department_id, user_id, created_at trên các bảng bận rộn
  - Ngoài ra: Index cho mọi trường liên kết khoá ngoại thường truy cập

## 6. Mẫu seed data
  - ROLE: [{Admin}, {Lãnh đạo}, {Trưởng phòng}, {Chuyên viên}, ...]
  - POSITION: [{GD}, {PGD}, {TP}, {Phó TP}, {NV}]
  - STATUS: [{Mới tạo}, {Đang xử lý}, {Chờ duyệt}, {Hoàn thành}, {Từ chối}, ...]
  - PRIORITY: [{Cao}, {Trung bình}, {Thấp}]
  - CATEGORIES: [{Văn bản đi}, {Văn bản đến}, {Công việc đột xuất}, ...]
  - Sample DEPARTMENT/PT ban mẫu

## 7. Migration scripts (gợi ý)
- Script tạo từng bảng với PRIMARY KEY, FOREIGN KEY, chỉ mục, DEFAULT value
- Script ADD UNIQUE, ON DELETE CASCADE/SET NULL cho các quan hệ
- Chuẩn hóa seed database đi kèm cho local/test

---
