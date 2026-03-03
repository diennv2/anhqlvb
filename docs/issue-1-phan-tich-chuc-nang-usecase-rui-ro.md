## 1. Giới thiệu
Phân tích chuyên sâu hệ thống **quản lý văn bản** (văn bản đến / văn bản đi / văn bản nội bộ) theo mô-đun, bám sát thực tiễn nghiệp vụ văn thư – lưu trữ và luân chuyển xử lý trong cơ quan nhà nước. Tài liệu phục vụ kiến trúc hóa hệ thống quy mô lớn, vận hành chuẩn hóa, dễ bảo trì và mở rộng.

---

## 2. Phân chia modules, chức năng chính, chức năng con (sub-function), use-case mẫu, rủi ro – thách thức

### 2.1. Module: Quản trị người dùng & phân quyền
#### Danh mục/Thực thể liên quan:
- Người dùng (User)
- Nhóm quyền (Roles/Permission group)
- Phòng ban/đơn vị (Department/OrgUnit)
- Chức vụ (Position)

#### Chức năng chính:
- Đăng nhập/đăng xuất
- Quản lý người dùng (CRUD)
- Phân quyền theo vai trò nghiệp vụ (Văn thư, Lãnh đạo, Trưởng phòng, Chuyên viên, Lưu trữ...)
- Quản lý phòng ban/chức vụ
- Xem nhật ký, lịch sử truy cập/xem/tải văn bản

#### Chức năng con (Sub-function):
1. Xác thực tăng cường (2FA/OTP) cho tài khoản nhạy cảm
2. Khoá tài khoản khi đăng nhập sai nhiều lần
3. Cơ chế phân quyền theo độ mật (mật/tối mật/tuyệt mật)
4. Cơ chế uỷ quyền xử lý khi vắng mặt
5. Export danh sách user và lịch sử truy cập

#### Use-case chính:
- Văn thư tạo tài khoản cán bộ mới, gán phòng ban/chức vụ, gán quyền xem văn bản theo đơn vị.
- Lãnh đạo có quyền bút phê/giao xử lý văn bản đến.
- Trưởng phòng phân công cán bộ xử lý văn bản được giao.

#### Rủi ro/thách thức:
- Lộ văn bản mật do phân quyền sai; tải xuống trái phép.
- Thiếu audit hành vi xem/tải/in/chuyển văn bản.

---

### 2.2. Module: Danh mục nghiệp vụ văn bản
#### Danh mục/Thực thể:
- Loại văn bản (DocumentType)
- Độ khẩn (UrgencyLevel)
- Độ mật (ConfidentialityLevel)
- Cơ quan gửi/nhận (Organization)
- Sổ văn bản (Register) theo năm/đơn vị
- Lĩnh vực (Field/Sector)

#### Chức năng chính và chức năng con:
1. CRUD danh mục (active/inactive)
2. Ràng buộc không cho xoá danh mục đang được sử dụng
3. Chuẩn hoá mã/định danh danh mục để phục vụ báo cáo và tra cứu

#### Use-case:
- Văn thư cấu hình sổ văn bản đến năm 2026 và quy tắc đánh số.
- Quản trị viên cập nhật danh mục độ khẩn/độ mật.

#### Rủi ro/thách thức:
- Danh mục không chuẩn gây sai thống kê; khó tra cứu.

---

### 2.3. Module: Quản lý văn bản đến (Incoming)
#### Thực thể:
- Văn bản đến (Incoming Document)
- Tệp scan/đính kèm

#### Chức năng chính/chức năng con:
1. Tiếp nhận văn bản đến: nhập metadata (số/ký hiệu, trích yếu, cơ quan gửi, ngày đến...)
2. Vào sổ & cấp số đến (chống trùng theo sổ/năm)
3. Đính kèm file scan, phân loại tài liệu
4. Trình lãnh đạo bút phê và giao xử lý
5. Theo dõi hạn xử lý, cảnh báo quá hạn
6. Lưu trữ và kết thúc hồ sơ

#### Use-case:
- Văn thư nhập văn bản đến, scan đính kèm, trình lãnh đạo.
- Lãnh đạo bút phê: giao phòng A chủ trì, phòng B phối hợp, deadline.

#### Rủi ro/thách thức:
- Trùng số đến; metadata sai làm giảm giá trị pháp lý.
- File scan bị thất lạc/không đồng bộ phiên bản.

---

### 2.4. Module: Quản lý văn bản đi (Outgoing)
#### Thực thể:
- Văn bản đi (Outgoing Document)
- Dự thảo/phiên bản

#### Chức năng chính/chức năng con:
1. Soạn thảo hoặc upload dự thảo
2. Trình duyệt nội bộ; (tuỳ chọn) ký số
3. Cấp số đi theo sổ/năm/đơn vị
4. Phát hành: nơi nhận, ghi nhận lịch sử gửi
5. Liên kết văn bản đi với văn bản đến (trả lời/căn cứ)

#### Rủi ro/thách thức:
- Cấp số sai gây trùng/nhảy số.
- Phát hành nhầm nơi nhận; rò rỉ văn bản mật.

---

### 2.5. Module: Luân chuyển xử lý, bút phê & giao xử lý
#### Thực thể:
- Assignment (giao xử lý)
- Comment (bút phê/ý kiến)
- Timeline xử lý

#### Chức năng con:
1. Giao xử lý theo vai trò (chủ trì/phối hợp/nhận để biết)
2. Điều chỉnh/thu hồi giao xử lý (có log)
3. Chuỗi bút phê nhiều cấp
4. Theo dõi trạng thái xử lý theo từng người/đơn vị

#### Rủi ro/thách thức:
- Tranh chấp trách nhiệm nếu log không rõ ràng.
- Workflow quá phức tạp gây khó dùng.

---

### 2.6. Module: Tra cứu – tìm kiếm – kết xuất
#### Chức năng con:
1. Tra cứu theo số đến/số đi/số ký hiệu, trích yếu, cơ quan, người ký, ngày...
2. Lọc theo trạng thái, hạn xử lý, độ mật/khẩn
3. Export danh sách văn bản và báo cáo

#### Rủi ro/thách thức:
- Fulltext/tìm kiếm chậm khi dữ liệu lớn; cần index.
- Rò rỉ dữ liệu nếu filter không kiểm soát theo quyền.

---

### 2.7. Module: Thông báo – nhắc việc
#### Chức năng con:
1. Nhắc hạn xử lý văn bản đến
2. Thông báo văn bản mới được giao
3. Đồng bộ thông báo web/mobile/email

#### Rủi ro/thách thức:
- Spam thông báo làm bỏ sót cảnh báo quan trọng.

---

### 2.8. Module: Lịch sử, nhật ký, audit
#### Chức năng con:
1. Log xem/tải/in/chuyển văn bản
2. Timeline xử lý văn bản
3. Giao diện diff thay đổi metadata

#### Rủi ro/thách thức:
- Log lớn, khó lưu trữ và truy vấn.
- Sửa log làm mất tính toàn vẹn nếu không có cơ chế immutable.

---

## 3. Tổng kết
- Phân rã module theo đúng nghiệp vụ quản lý văn bản.
- Sẵn sàng cho bước tiếp theo: thiết kế database schema, API và UI luân chuyển văn bản.

---
*Góp ý/bổ sung vui lòng comment để hoàn thiện trước khi bước sang bước Database Schema!*