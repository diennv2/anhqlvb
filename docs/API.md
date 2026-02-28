# Tài liệu API – Hệ thống Quản lý Công việc Hải Phòng

> Phiên bản: 1.0.0 | OpenAPI 3.0.3

---

## Mục lục

1. [Thông tin chung](#1-thông-tin-chung)
2. [Xác thực (Authentication)](#2-xác-thực-authentication)
3. [Phân trang, Lọc, Sắp xếp](#3-phân-trang-lọc-sắp-xếp)
4. [Định dạng lỗi](#4-định-dạng-lỗi)
5. [Auth – Đăng nhập & Mật khẩu](#5-auth--đăng-nhập--mật-khẩu)
6. [Users – Quản lý người dùng](#6-users--quản-lý-người-dùng)
7. [Roles – Vai trò](#7-roles--vai-trò)
8. [Permissions – Quyền hạn](#8-permissions--quyền-hạn)
9. [Departments – Phòng ban](#9-departments--phòng-ban)
10. [Positions – Chức vụ](#10-positions--chức-vụ)
11. [TaskCategories – Loại công việc](#11-taskcategories--loại-công-việc)
12. [TaskPriorities – Mức ưu tiên](#12-taskpriorities--mức-ưu-tiên)
13. [TaskStatuses – Trạng thái công việc](#13-taskstatuses--trạng-thái-công-việc)
14. [Tasks – Công việc](#14-tasks--công-việc)
15. [TaskAssignees – Người thực hiện](#15-taskassignees--người-thực-hiện)
16. [Workflows – Quy trình phê duyệt](#16-workflows--quy-trình-phê-duyệt)
17. [Notifications – Thông báo](#17-notifications--thông-báo)
18. [Logs – Nhật ký hệ thống](#18-logs--nhật-ký-hệ-thống)
19. [Reports – Báo cáo](#19-reports--báo-cáo)

---

## 1. Thông tin chung

| Mục            | Giá trị                                 |
|----------------|-----------------------------------------|
| Base URL       | `https://api.example.com/api/v1`        |
| Giao thức      | HTTPS                                   |
| Định dạng      | JSON (`application/json`)               |
| Mã hóa ký tự  | UTF-8                                   |
| Phiên bản API  | v1                                      |

### Headers bắt buộc

```
Content-Type: application/json
Authorization: Bearer <access_token>
```

---

## 2. Xác thực (Authentication)

API sử dụng **JWT Bearer Token**. Mọi endpoint (trừ các endpoint xác thực) đều yêu cầu header:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Luồng xác thực

```
1. POST /auth/login          → nhận access_token + refresh_token
2. Dùng access_token cho các request (thường hết hạn sau 15-60 phút)
3. Khi access_token hết hạn: POST /auth/token/refresh → nhận access_token mới
4. Khi refresh_token hết hạn: đăng nhập lại
```

---

## 3. Phân trang, Lọc, Sắp xếp

### Phân trang

Các endpoint trả về danh sách hỗ trợ phân trang qua query parameters:

| Parameter  | Mô tả                     | Mặc định | Giới hạn   |
|------------|---------------------------|----------|------------|
| `page`     | Số trang (bắt đầu từ 1)   | 1        | ≥ 1        |
| `pageSize` | Số bản ghi mỗi trang      | 20       | 1 – 200    |

**Cấu trúc response phân trang:**

```json
{
  "data": [...],
  "meta": {
    "page": 1,
    "pageSize": 20,
    "total": 150
  }
}
```

### Lọc (Filter)

Mỗi endpoint có các filter riêng. Các filter phổ biến:

| Parameter    | Mô tả                              |
|--------------|------------------------------------|
| `search`     | Tìm kiếm theo từ khóa              |
| `department` | Lọc theo ID phòng ban              |
| `status`     | Lọc theo ID trạng thái công việc   |
| `assignee`   | Lọc theo ID người được gán         |
| `priority`   | Lọc theo ID mức ưu tiên            |
| `from`       | Lọc từ ngày (định dạng YYYY-MM-DD) |
| `to`         | Lọc đến ngày (định dạng YYYY-MM-DD)|
| `unread`     | Lọc thông báo chưa đọc (boolean)   |

### Sắp xếp (Sort)

Dùng query parameter `sort`:

- Tăng dần: `sort=created_at`
- Giảm dần: `sort=-created_at`
- Nhiều trường: `sort=-created_at,title`

---

## 4. Định dạng lỗi

Tất cả lỗi trả về cùng cấu trúc:

```json
{
  "error": "validation_error",
  "message": "Dữ liệu đầu vào không hợp lệ",
  "field": "title"
}
```

| Trường    | Mô tả                                          |
|-----------|------------------------------------------------|
| `error`   | Mã lỗi (snake_case)                            |
| `message` | Thông báo lỗi mô tả cho người dùng             |
| `field`   | Tên trường gây lỗi (nếu là lỗi validation)    |

### Các HTTP status code

| Code | Ý nghĩa                                        |
|------|------------------------------------------------|
| 200  | Thành công                                     |
| 201  | Tạo mới thành công                             |
| 204  | Thành công, không có nội dung trả về           |
| 400  | Dữ liệu đầu vào không hợp lệ                  |
| 401  | Chưa xác thực hoặc token hết hạn               |
| 403  | Không có quyền thực hiện                       |
| 404  | Không tìm thấy tài nguyên                      |
| 409  | Xung đột dữ liệu (ví dụ: trùng username)       |
| 422  | Dữ liệu không thể xử lý (lỗi logic nghiệp vụ) |
| 500  | Lỗi máy chủ nội bộ                            |

---

## 5. Auth – Đăng nhập & Mật khẩu

### Endpoints

| Method | Path                  | Mô tả                     |
|--------|-----------------------|---------------------------|
| POST   | `/auth/login`         | Đăng nhập                 |
| POST   | `/auth/token/refresh` | Làm mới access token      |
| POST   | `/auth/request-reset` | Yêu cầu reset mật khẩu   |
| POST   | `/auth/reset-password`| Đặt lại mật khẩu          |
| GET    | `/auth/whoami`        | Lấy thông tin user hiện tại |

---

### POST `/auth/login`

**Mô tả:** Đăng nhập bằng username và password, nhận JWT tokens.

**Không cần xác thực.**

**Request body:**

```json
{
  "username": "nguyenvana",
  "password": "123456"
}
```

| Trường     | Kiểu   | Bắt buộc | Mô tả          |
|------------|--------|----------|----------------|
| `username` | string | ✅       | Tên đăng nhập (3-50 ký tự) |
| `password` | string | ✅       | Mật khẩu (tối thiểu 6 ký tự) |

**Response 200:**

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer"
}
```

**curl:**

```bash
curl -X POST https://api.example.com/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "nguyenvana", "password": "123456"}'
```

---

### POST `/auth/token/refresh`

**Mô tả:** Dùng refresh_token để lấy access_token mới.

**Request body:**

```json
{
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response 200:** Giống như `/auth/login`.

**curl:**

```bash
curl -X POST https://api.example.com/api/v1/auth/token/refresh \
  -H "Content-Type: application/json" \
  -d '{"refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}'
```

---

### POST `/auth/request-reset`

**Mô tả:** Gửi email chứa link đặt lại mật khẩu.

**Request body:**

```json
{
  "email": "nguyenvana@haiphong.gov.vn"
}
```

**Response 200:**

```json
{ "message": "Thao tác thành công" }
```

---

### POST `/auth/reset-password`

**Mô tả:** Đặt mật khẩu mới bằng token từ email.

**Request body:**

```json
{
  "token": "abc123",
  "new_password": "newpass123"
}
```

---

### GET `/auth/whoami`

**Mô tả:** Trả về thông tin user đang đăng nhập (từ JWT token).

**Response 200:**

```json
{
  "id": 1,
  "username": "nguyenvana",
  "email": "nguyenvana@haiphong.gov.vn",
  "full_name": "Nguyễn Văn A",
  "phone": "0912345678",
  "department": "Phòng Hành chính",
  "position": "Trưởng phòng",
  "role": "Manager",
  "status": "active"
}
```

**curl:**

```bash
curl https://api.example.com/api/v1/auth/whoami \
  -H "Authorization: Bearer <access_token>"
```

---

## 6. Users – Quản lý người dùng

### Endpoints

| Method | Path                     | Mô tả                        |
|--------|--------------------------|------------------------------|
| GET    | `/users`                 | Danh sách người dùng         |
| POST   | `/users`                 | Tạo người dùng mới           |
| GET    | `/users/{id}`            | Chi tiết người dùng          |
| PATCH  | `/users/{id}`            | Cập nhật người dùng          |
| DELETE | `/users/{id}`            | Xóa người dùng               |
| PUT    | `/users/{id}/password`   | Đổi mật khẩu người dùng     |
| GET    | `/users/{id}/logs`       | Lịch sử thao tác người dùng |

---

### GET `/users`

**Mô tả:** Lấy danh sách người dùng với filter và phân trang.

**Query parameters:**

| Parameter    | Kiểu    | Mô tả                     |
|--------------|---------|---------------------------|
| `page`       | integer | Số trang (mặc định: 1)    |
| `pageSize`   | integer | Số bản ghi (mặc định: 20) |
| `search`     | string  | Tìm kiếm theo từ khóa     |
| `department` | integer | Lọc theo ID phòng ban     |
| `role`       | integer | Lọc theo ID vai trò       |

**Response 200:**

```json
{
  "data": [
    {
      "id": 1,
      "username": "nguyenvana",
      "full_name": "Nguyễn Văn A",
      "email": "nguyenvana@hp.gov.vn",
      "department": "Phòng HC",
      "position": "Trưởng phòng",
      "role": "Manager",
      "status": "active"
    }
  ],
  "meta": {
    "page": 1,
    "pageSize": 20,
    "total": 150
  }
}
```

**curl:**

```bash
curl "https://api.example.com/api/v1/users?page=1&pageSize=20&search=nguyen" \
  -H "Authorization: Bearer <access_token>"
```

---

### POST `/users`

**Mô tả:** Tạo tài khoản người dùng mới.

**Request body:**

```json
{
  "username": "tranthib",
  "email": "tranthib@hp.gov.vn",
  "full_name": "Trần Thị B",
  "password": "123456",
  "phone": "0912345679",
  "department_id": 2,
  "position_id": 3,
  "role_id": 2
}
```

| Trường          | Kiểu    | Bắt buộc | Mô tả                              |
|-----------------|---------|----------|------------------------------------|
| `username`      | string  | ✅       | Tên đăng nhập (chữ thường, số, _)  |
| `email`         | string  | ✅       | Địa chỉ email hợp lệ               |
| `full_name`     | string  | ✅       | Họ tên đầy đủ                      |
| `password`      | string  | ✅       | Mật khẩu (tối thiểu 6 ký tự)      |
| `phone`         | string  |          | Số điện thoại (9-11 chữ số)        |
| `department_id` | integer | ✅       | ID phòng ban                       |
| `position_id`   | integer | ✅       | ID chức vụ                         |
| `role_id`       | integer | ✅       | ID vai trò                         |

**Response 201:** Trả về đối tượng `User`.

**curl:**

```bash
curl -X POST https://api.example.com/api/v1/users \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "tranthib",
    "email": "tranthib@hp.gov.vn",
    "full_name": "Trần Thị B",
    "password": "123456",
    "department_id": 2,
    "position_id": 3,
    "role_id": 2
  }'
```

---

### GET `/users/{id}`

**Mô tả:** Lấy thông tin chi tiết một người dùng.

**Path parameter:** `id` – ID người dùng (integer ≥ 1)

**Response 200:** Trả về đối tượng `User`.

**curl:**

```bash
curl https://api.example.com/api/v1/users/1 \
  -H "Authorization: Bearer <access_token>"
```

---

### PATCH `/users/{id}`

**Mô tả:** Cập nhật thông tin người dùng (chỉ các trường cần thay đổi).

**Request body (ví dụ):**

```json
{
  "full_name": "Trần Thị Bình",
  "department_id": 3
}
```

**Response 200:** Trả về đối tượng `User` sau cập nhật.

---

### DELETE `/users/{id}`

**Mô tả:** Xóa người dùng.

**Response 204:** Không có nội dung.

---

### PUT `/users/{id}/password`

**Mô tả:** Đặt lại mật khẩu cho người dùng.

**Request body:**

```json
{
  "new_password": "newpass123"
}
```

---

### GET `/users/{id}/logs`

**Mô tả:** Lấy lịch sử thao tác của người dùng.

**Response 200:** Mảng các đối tượng `TaskLog`.

---

## 7. Roles – Vai trò

### Endpoints

| Method | Path          | Mô tả                |
|--------|---------------|----------------------|
| GET    | `/roles`      | Danh sách vai trò    |
| POST   | `/roles`      | Tạo vai trò mới      |
| GET    | `/roles/{id}` | Chi tiết vai trò     |
| PATCH  | `/roles/{id}` | Cập nhật vai trò     |
| DELETE | `/roles/{id}` | Xóa vai trò          |

### Cấu trúc Role

```json
{
  "id": 1,
  "name": "Admin",
  "description": "Administrator"
}
```

### POST `/roles`

**Request body:**

```json
{
  "name": "Staff",
  "description": "Nhân viên"
}
```

**curl:**

```bash
curl -X POST https://api.example.com/api/v1/roles \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{"name": "Staff", "description": "Nhân viên"}'
```

---

## 8. Permissions – Quyền hạn

### Endpoints

| Method | Path            | Mô tả                  |
|--------|-----------------|------------------------|
| GET    | `/permissions`  | Danh sách quyền hạn    |

### GET `/permissions`

**Response 200:**

```json
[
  { "id": 1, "code": "task.create", "description": "Tạo công việc" },
  { "id": 2, "code": "task.delete", "description": "Xóa công việc" }
]
```

**curl:**

```bash
curl https://api.example.com/api/v1/permissions \
  -H "Authorization: Bearer <access_token>"
```

---

## 9. Departments – Phòng ban

### Endpoints

| Method | Path                 | Mô tả                  |
|--------|----------------------|------------------------|
| GET    | `/departments`       | Danh sách phòng ban    |
| POST   | `/departments`       | Tạo phòng ban mới      |
| GET    | `/departments/{id}`  | Chi tiết phòng ban     |
| PATCH  | `/departments/{id}`  | Cập nhật phòng ban     |
| DELETE | `/departments/{id}`  | Xóa phòng ban          |

### Cấu trúc Department

```json
{
  "id": 1,
  "name": "Phòng Hành chính",
  "code": "HC",
  "parent_id": null,
  "description": "Phòng Hành chính tổng hợp",
  "created_at": "2026-01-01T00:00:00Z",
  "updated_at": "2026-01-01T00:00:00Z"
}
```

### POST `/departments`

**Request body:**

```json
{
  "name": "Phòng Kế hoạch",
  "code": "KH",
  "parent_id": null,
  "description": "Phòng Kế hoạch"
}
```

| Trường        | Kiểu    | Bắt buộc | Mô tả                                  |
|---------------|---------|----------|----------------------------------------|
| `name`        | string  | ✅       | Tên phòng ban (2-200 ký tự)            |
| `code`        | string  | ✅       | Mã phòng ban (chữ hoa, số, _ ; 1-20 ký tự) |
| `parent_id`   | integer |          | ID phòng ban cha (null nếu cấp cao nhất)|
| `description` | string  |          | Mô tả                                  |

**curl:**

```bash
curl -X POST https://api.example.com/api/v1/departments \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{"name": "Phòng Kế hoạch", "code": "KH"}'
```

---

## 10. Positions – Chức vụ

### Endpoints

| Method | Path               | Mô tả                |
|--------|--------------------|----------------------|
| GET    | `/positions`       | Danh sách chức vụ    |
| POST   | `/positions`       | Tạo chức vụ mới      |
| GET    | `/positions/{id}`  | Chi tiết chức vụ     |
| PATCH  | `/positions/{id}`  | Cập nhật chức vụ     |
| DELETE | `/positions/{id}`  | Xóa chức vụ          |

### Cấu trúc Position

```json
{
  "id": 1,
  "name": "Trưởng phòng",
  "level": 5
}
```

### POST `/positions`

**Request body:**

```json
{
  "name": "Phó phòng",
  "level": 4
}
```

| Trường  | Kiểu    | Bắt buộc | Mô tả                  |
|---------|---------|----------|------------------------|
| `name`  | string  | ✅       | Tên chức vụ (2-100 ký tự) |
| `level` | integer | ✅       | Cấp bậc (1-10, cao nhất là 10) |

---

## 11. TaskCategories – Loại công việc

### Endpoints

| Method | Path                      | Mô tả                       |
|--------|---------------------------|-----------------------------|
| GET    | `/task-categories`        | Danh sách loại công việc    |
| POST   | `/task-categories`        | Tạo loại công việc mới      |
| GET    | `/task-categories/{id}`   | Chi tiết loại công việc     |
| PATCH  | `/task-categories/{id}`   | Cập nhật loại công việc     |
| DELETE | `/task-categories/{id}`   | Xóa loại công việc          |

### POST `/task-categories`

**Request body:**

```json
{
  "name": "Văn bản đến",
  "description": "Loại công việc văn bản đến",
  "is_active": true
}
```

---

## 12. TaskPriorities – Mức ưu tiên

### Endpoints

| Method | Path                       | Mô tả                    |
|--------|----------------------------|--------------------------|
| GET    | `/task-priorities`         | Danh sách mức ưu tiên    |
| POST   | `/task-priorities`         | Tạo mức ưu tiên mới      |
| GET    | `/task-priorities/{id}`    | Chi tiết mức ưu tiên     |
| PATCH  | `/task-priorities/{id}`    | Cập nhật mức ưu tiên     |
| DELETE | `/task-priorities/{id}`    | Xóa mức ưu tiên          |

### POST `/task-priorities`

**Request body:**

```json
{
  "name": "Trung bình",
  "level": 2,
  "is_default": true
}
```

| Trường       | Kiểu    | Bắt buộc | Mô tả                              |
|--------------|---------|----------|------------------------------------|
| `name`       | string  | ✅       | Tên mức ưu tiên                    |
| `level`      | integer | ✅       | Cấp độ ưu tiên (1 = cao nhất)     |
| `is_default` | boolean |          | Có phải mức ưu tiên mặc định không |

---

## 13. TaskStatuses – Trạng thái công việc

### Endpoints

| Method | Path                      | Mô tả                         |
|--------|---------------------------|-------------------------------|
| GET    | `/task-statuses`          | Danh sách trạng thái          |
| POST   | `/task-statuses`          | Tạo trạng thái mới            |
| GET    | `/task-statuses/{id}`     | Chi tiết trạng thái           |
| PATCH  | `/task-statuses/{id}`     | Cập nhật trạng thái           |
| DELETE | `/task-statuses/{id}`     | Xóa trạng thái                |

### POST `/task-statuses`

**Request body:**

```json
{
  "name": "Đang xử lý",
  "code": "processing",
  "is_closed_state": false
}
```

| Trường            | Kiểu    | Bắt buộc | Mô tả                              |
|-------------------|---------|----------|------------------------------------|
| `name`            | string  | ✅       | Tên trạng thái                     |
| `code`            | string  | ✅       | Mã trạng thái (chữ thường, số, _)  |
| `is_closed_state` | boolean |          | Đây có phải trạng thái đóng không  |

---

## 14. Tasks – Công việc

### Endpoints

| Method | Path                       | Mô tả                           |
|--------|----------------------------|---------------------------------|
| GET    | `/tasks`                   | Danh sách công việc             |
| POST   | `/tasks`                   | Tạo công việc mới               |
| GET    | `/tasks/{id}`              | Chi tiết công việc              |
| PATCH  | `/tasks/{id}`              | Cập nhật công việc              |
| DELETE | `/tasks/{id}`              | Xóa công việc                   |
| POST   | `/tasks/import`            | Import từ Excel                 |
| POST   | `/tasks/{id}/assign`       | Gán người thực hiện (thay thế)  |
| POST   | `/tasks/{id}/subtask`      | Tạo công việc con               |
| PATCH  | `/tasks/{id}/status`       | Đổi trạng thái                  |
| GET    | `/tasks/{id}/files`        | Danh sách file đính kèm         |
| POST   | `/tasks/{id}/files`        | Upload file đính kèm            |
| DELETE | `/tasks/{id}/files/{file_id}` | Xóa file đính kèm           |
| GET    | `/tasks/{id}/logs`         | Lịch sử công việc               |
| GET    | `/tasks/{id}/comments`     | Danh sách comment               |
| POST   | `/tasks/{id}/comments`     | Thêm comment                    |

### Cấu trúc Task

```json
{
  "id": 123,
  "title": "Báo cáo tài chính Q1",
  "description": "Cần hoàn thành báo cáo...",
  "category_id": 2,
  "priority_id": 1,
  "status_id": 2,
  "creator_id": 5,
  "department_id": 3,
  "parent_task_id": null,
  "start_date": "2026-02-10",
  "due_date": "2026-03-10",
  "completed_date": null,
  "progress": 50,
  "confidential": false,
  "must_confirm": true,
  "workflow_id": null,
  "created_at": "2026-02-01T09:00:00Z",
  "updated_at": "2026-02-15T14:30:00Z"
}
```

---

### GET `/tasks`

**Query parameters:**

| Parameter    | Kiểu    | Mô tả                          |
|--------------|---------|--------------------------------|
| `page`       | integer | Số trang                       |
| `pageSize`   | integer | Số bản ghi mỗi trang           |
| `search`     | string  | Tìm kiếm theo tiêu đề          |
| `sort`       | string  | Sắp xếp (ví dụ: `-created_at`) |
| `status`     | integer | Lọc theo trạng thái            |
| `department` | integer | Lọc theo phòng ban             |
| `assignee`   | integer | Lọc theo người được gán        |
| `priority`   | integer | Lọc theo mức ưu tiên           |

**curl:**

```bash
curl "https://api.example.com/api/v1/tasks?page=1&pageSize=20&status=2&sort=-due_date" \
  -H "Authorization: Bearer <access_token>"
```

---

### POST `/tasks`

**Mô tả:** Tạo công việc mới.

**Request body:**

```json
{
  "title": "Báo cáo tài chính Q1",
  "description": "Cần hoàn thành báo cáo tổng kết...",
  "category_id": 2,
  "priority_id": 1,
  "status_id": 1,
  "department_id": 3,
  "assignees": [5, 8],
  "start_date": "2026-02-10",
  "due_date": "2026-03-10",
  "confidential": false,
  "must_confirm": true
}
```

| Trường          | Kiểu    | Bắt buộc | Mô tả                             |
|-----------------|---------|----------|-----------------------------------|
| `title`         | string  | ✅       | Tiêu đề (2-500 ký tự)             |
| `description`   | string  |          | Mô tả chi tiết                    |
| `category_id`   | integer | ✅       | ID loại công việc                 |
| `priority_id`   | integer | ✅       | ID mức ưu tiên                    |
| `status_id`     | integer | ✅       | ID trạng thái ban đầu             |
| `department_id` | integer | ✅       | ID phòng ban phụ trách            |
| `assignees`     | array   |          | Mảng ID người thực hiện           |
| `start_date`    | string  | ✅       | Ngày bắt đầu (YYYY-MM-DD)        |
| `due_date`      | string  | ✅       | Hạn chót (YYYY-MM-DD)            |
| `progress`      | integer |          | Tiến độ 0-100% (mặc định: 0)     |
| `confidential`  | boolean |          | Công việc mật (mặc định: false)   |
| `must_confirm`  | boolean |          | Cần phê duyệt (mặc định: false)   |
| `workflow_id`   | integer |          | ID quy trình phê duyệt            |

**curl:**

```bash
curl -X POST https://api.example.com/api/v1/tasks \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Báo cáo tài chính Q1",
    "category_id": 2,
    "priority_id": 1,
    "status_id": 1,
    "department_id": 3,
    "assignees": [5, 8],
    "start_date": "2026-02-10",
    "due_date": "2026-03-10"
  }'
```

---

### PATCH `/tasks/{id}/status`

**Mô tả:** Thay đổi trạng thái và tiến độ công việc.

**Request body:**

```json
{
  "status_id": 3,
  "progress": 80
}
```

| Trường      | Kiểu    | Bắt buộc | Mô tả                 |
|-------------|---------|----------|-----------------------|
| `status_id` | integer | ✅       | ID trạng thái mới     |
| `progress`  | integer |          | Tiến độ mới (0-100)   |

**curl:**

```bash
curl -X PATCH https://api.example.com/api/v1/tasks/123/status \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{"status_id": 3, "progress": 80}'
```

---

### POST `/tasks/{id}/assign`

**Mô tả:** Gán nhiều người thực hiện cùng lúc (thay thế toàn bộ danh sách cũ).

**Request body:**

```json
{
  "assignees": [5, 8, 12]
}
```

---

### POST `/tasks/import`

**Mô tả:** Import danh sách công việc từ file Excel.

**Request:** Multipart form-data với field `file`.

**Response 201:**

```json
{
  "imported": 25,
  "failed": 2
}
```

**curl:**

```bash
curl -X POST https://api.example.com/api/v1/tasks/import \
  -H "Authorization: Bearer <access_token>" \
  -F "file=@tasks.xlsx"
```

---

### POST `/tasks/{id}/files`

**Mô tả:** Upload file đính kèm cho công việc.

**curl:**

```bash
curl -X POST https://api.example.com/api/v1/tasks/123/files \
  -H "Authorization: Bearer <access_token>" \
  -F "file=@BaoCao_Q1.xlsx"
```

---

### POST `/tasks/{id}/comments`

**Request body:**

```json
{
  "content": "Đã hoàn thành phần thu thập số liệu"
}
```

---

## 15. TaskAssignees – Người thực hiện

### Endpoints

| Method | Path                               | Mô tả                          |
|--------|------------------------------------|--------------------------------|
| GET    | `/tasks/{id}/assignees`            | Danh sách người được gán       |
| POST   | `/tasks/{id}/assignees`            | Thêm người thực hiện           |
| PATCH  | `/tasks/{id}/assignees/{assign_id}`| Cập nhật thông tin người thực hiện |
| DELETE | `/tasks/{id}/assignees/{assign_id}`| Gỡ người thực hiện             |

### Cấu trúc Assignee

```json
{
  "id": 1,
  "user_id": 8,
  "user_name": "tranthib",
  "is_primary": false,
  "assigned_at": "2026-02-01T09:00:00Z"
}
```

### POST `/tasks/{id}/assignees`

**Request body:**

```json
{
  "user_id": 8,
  "is_primary": false
}
```

**curl:**

```bash
curl -X POST https://api.example.com/api/v1/tasks/123/assignees \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{"user_id": 8, "is_primary": false}'
```

---

## 16. Workflows – Quy trình phê duyệt

### Endpoints

| Method | Path                            | Mô tả                          |
|--------|---------------------------------|--------------------------------|
| GET    | `/workflows`                    | Danh sách workflow             |
| POST   | `/workflows`                    | Tạo workflow mới               |
| GET    | `/workflows/{id}`               | Chi tiết workflow              |
| PATCH  | `/workflows/{id}`               | Cập nhật workflow              |
| DELETE | `/workflows/{id}`               | Xóa workflow                   |
| GET    | `/workflow-steps`               | Danh sách bước workflow        |
| POST   | `/workflow-steps`               | Tạo bước workflow mới          |
| PATCH  | `/workflow-steps/{id}`          | Cập nhật bước workflow         |
| DELETE | `/workflow-steps/{id}`          | Xóa bước workflow              |
| POST   | `/tasks/{id}/approve`           | Phê duyệt / từ chối công việc  |
| GET    | `/tasks/{id}/workflow-logs`     | Lịch sử phê duyệt              |

---

### POST `/workflows`

**Request body:**

```json
{
  "name": "Quy trình phê duyệt văn bản",
  "description": "3 bước: Trưởng phòng → Phó giám đốc → Giám đốc",
  "is_active": true
}
```

---

### GET `/workflow-steps`

**Query parameters:**

| Parameter     | Kiểu    | Mô tả               |
|---------------|---------|---------------------|
| `workflow_id` | integer | Lọc theo workflow   |

---

### POST `/workflow-steps`

**Request body:**

```json
{
  "workflow_id": 1,
  "name": "Trưởng phòng duyệt",
  "order_index": 1,
  "is_approval": true,
  "auto_advance": false,
  "role_id": 3
}
```

| Trường         | Kiểu    | Bắt buộc | Mô tả                            |
|----------------|---------|----------|----------------------------------|
| `workflow_id`  | integer | ✅       | ID workflow                      |
| `name`         | string  | ✅       | Tên bước                         |
| `order_index`  | integer | ✅       | Thứ tự bước (bắt đầu từ 1)       |
| `is_approval`  | boolean |          | Bước này có yêu cầu phê duyệt không |
| `auto_advance` | boolean |          | Tự động chuyển bước tiếp theo    |
| `role_id`      | integer |          | ID vai trò được phép phê duyệt  |

---

### POST `/tasks/{id}/approve`

**Mô tả:** Người có thẩm quyền phê duyệt hoặc từ chối công việc trong quy trình.

**Request body:**

```json
{
  "action": "approve",
  "comments": "Đồng ý, nội dung đã đầy đủ"
}
```

| Trường     | Kiểu   | Bắt buộc | Mô tả                        |
|------------|--------|----------|------------------------------|
| `action`   | string | ✅       | `approve` hoặc `reject`      |
| `comments` | string |          | Ghi chú phê duyệt (tối đa 2000 ký tự) |

**curl:**

```bash
curl -X POST https://api.example.com/api/v1/tasks/123/approve \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{"action": "approve", "comments": "Đồng ý"}'
```

---

### GET `/tasks/{id}/workflow-logs`

**Response 200:**

```json
[
  {
    "step_name": "Trưởng phòng duyệt",
    "user_name": "nguyenvana",
    "action": "approve",
    "comments": "Đồng ý",
    "created_at": "2026-02-15T10:30:00Z"
  }
]
```

---

## 17. Notifications – Thông báo

### Endpoints

| Method | Path                         | Mô tả                             |
|--------|------------------------------|-----------------------------------|
| GET    | `/notifications`             | Danh sách thông báo               |
| POST   | `/notifications`             | Tạo và gửi thông báo              |
| PATCH  | `/notifications/{id}/read`   | Đánh dấu thông báo đã đọc         |
| GET    | `/reminder`                  | Danh sách nhắc việc               |
| PATCH  | `/reminder/{id}/done`        | Đánh dấu nhắc việc hoàn thành    |

---

### GET `/notifications`

**Query parameters:**

| Parameter | Kiểu    | Mô tả                        |
|-----------|---------|------------------------------|
| `unread`  | boolean | Lọc chỉ thông báo chưa đọc  |

**Response 200:**

```json
[
  {
    "id": 1,
    "user_id": 5,
    "content": "Task #123 hết hạn hôm nay",
    "type": "task",
    "read_status": false,
    "task_id": 123,
    "sent_at": "2026-03-10T08:00:00Z"
  }
]
```

**curl:**

```bash
curl "https://api.example.com/api/v1/notifications?unread=true" \
  -H "Authorization: Bearer <access_token>"
```

---

### POST `/notifications`

**Request body:**

```json
{
  "user_id": 5,
  "content": "Task #123 đã được gán cho bạn",
  "type": "task",
  "task_id": 123
}
```

---

## 18. Logs – Nhật ký hệ thống

### Endpoints

| Method | Path         | Mô tả                   |
|--------|--------------|-------------------------|
| GET    | `/logs`      | Danh sách nhật ký       |
| GET    | `/logs/{id}` | Chi tiết log entry      |

### GET `/logs`

**Query parameters:**

| Parameter | Kiểu    | Mô tả                     |
|-----------|---------|---------------------------|
| `user`    | integer | Lọc theo ID người dùng    |
| `entity`  | string  | Lọc theo loại đối tượng   |
| `action`  | string  | Lọc theo hành động        |
| `from`    | string  | Từ ngày (YYYY-MM-DD)      |
| `to`      | string  | Đến ngày (YYYY-MM-DD)     |

**Response 200:**

```json
[
  {
    "id": 1,
    "task_id": 123,
    "user_id": 5,
    "action": "status_change",
    "content": "Chuyển trạng thái từ 'Mới' sang 'Đang xử lý'",
    "is_system": false,
    "created_at": "2026-02-15T14:30:00Z"
  }
]
```

**curl:**

```bash
curl "https://api.example.com/api/v1/logs?from=2026-01-01&to=2026-03-31" \
  -H "Authorization: Bearer <access_token>"
```

---

## 19. Reports – Báo cáo

### Endpoints

| Method | Path                             | Mô tả                              |
|--------|----------------------------------|------------------------------------|
| GET    | `/reports/tasks-summary`         | Tổng hợp công việc                 |
| GET    | `/reports/progress-by-department`| Tiến độ theo phòng ban             |
| GET    | `/reports/chart-progress`        | Dữ liệu biểu đồ                    |
| GET    | `/reports/export`                | Xuất báo cáo ra file               |

---

### GET `/reports/tasks-summary`

**Query parameters:**

| Parameter  | Kiểu   | Mô tả                                         |
|------------|--------|-----------------------------------------------|
| `group_by` | string | Nhóm theo: `department`, `status`, `user`    |
| `from`     | string | Từ ngày (YYYY-MM-DD)                          |
| `to`       | string | Đến ngày (YYYY-MM-DD)                         |

**Response 200:**

```json
{
  "data": [
    {
      "group": "Phòng Tài chính",
      "total": 25,
      "completed": 10,
      "in_progress": 12,
      "overdue": 3
    }
  ]
}
```

**curl:**

```bash
curl "https://api.example.com/api/v1/reports/tasks-summary?group_by=department&from=2026-01-01&to=2026-03-31" \
  -H "Authorization: Bearer <access_token>"
```

---

### GET `/reports/progress-by-department`

**Response 200:**

```json
[
  {
    "department": "Phòng Tài chính",
    "total_tasks": 30,
    "completion_rate": 66.7
  }
]
```

---

### GET `/reports/chart-progress`

**Query parameters:**

| Parameter | Kiểu   | Mô tả                           |
|-----------|--------|---------------------------------|
| `type`    | string | Loại biểu đồ: `pie`, `bar`, `line` |
| `from`    | string | Từ ngày                         |
| `to`      | string | Đến ngày                        |

**Response 200:**

```json
{
  "labels": ["Mới tạo", "Đang xử lý", "Hoàn thành"],
  "data": [10, 25, 15]
}
```

---

### GET `/reports/export`

**Query parameters:**

| Parameter | Kiểu   | Mô tả                          |
|-----------|--------|--------------------------------|
| `type`    | string | Định dạng: `excel` hoặc `pdf`  |
| `report`  | string | Loại báo cáo                   |
| `from`    | string | Từ ngày                        |
| `to`      | string | Đến ngày                       |

**Response 200:** File nhị phân (application/octet-stream).

**curl:**

```bash
curl "https://api.example.com/api/v1/reports/export?type=excel&from=2026-01-01&to=2026-03-31" \
  -H "Authorization: Bearer <access_token>" \
  --output report.xlsx
```

---

## Ví dụ tích hợp hoàn chỉnh

### Tạo và theo dõi công việc

```bash
# 1. Đăng nhập
TOKEN=$(curl -s -X POST https://api.example.com/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "nguyenvana", "password": "123456"}' \
  | jq -r '.access_token')

# 2. Tạo công việc
TASK_ID=$(curl -s -X POST https://api.example.com/api/v1/tasks \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Báo cáo Q1",
    "category_id": 1,
    "priority_id": 1,
    "status_id": 1,
    "department_id": 2,
    "start_date": "2026-02-10",
    "due_date": "2026-03-31"
  }' | jq -r '.id')

# 3. Gán người thực hiện
curl -X POST https://api.example.com/api/v1/tasks/$TASK_ID/assignees \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"user_id": 8, "is_primary": true}'

# 4. Upload file đính kèm
curl -X POST https://api.example.com/api/v1/tasks/$TASK_ID/files \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@BaoCao_Q1.xlsx"

# 5. Cập nhật tiến độ
curl -X PATCH https://api.example.com/api/v1/tasks/$TASK_ID/status \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status_id": 2, "progress": 50}'

# 6. Thêm comment
curl -X POST https://api.example.com/api/v1/tasks/$TASK_ID/comments \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content": "Đã hoàn thành 50% phần thu thập số liệu"}'
```
