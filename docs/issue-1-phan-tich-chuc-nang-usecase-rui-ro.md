# Phân tích hệ thống – App Quản lý Văn bản (Đến/Đi) (Chuẩn hoá)

> Mục tiêu: Xây dựng **app quản lý văn bản** (văn bản đến/văn bản đi) phục vụ nghiệp vụ văn thư – lưu trữ và luân chuyển xử lý:
> - Quản lý vòng đời, giao xử lý, bút phê/ý kiến, theo dõi hạn.
> - Tra cứu – kết xuất báo cáo.
> - Thông báo/nhắc việc.
> - Kiểm soát truy cập theo vai trò/đơn vị và **độ mật/độ khẩn**.
> - Bảo đảm truy vết (audit) đầy đủ: **ai xem, ai preview/tải, ai in, ai chuyển**, ai bút phê, ai giao xử lý.

---

## 0. Phạm vi & nguyên tắc triển khai

### 0.1. Phạm vi MVP (đủ để làm app chạy thật)
1) Auth: login/refresh/whoami/logout  
2) Master data: loại VB, độ khẩn, độ mật, cơ quan, lĩnh vực  
3) Incoming: list/create/detail/update/change status/upload/timeline/audit-logs  
4) Outgoing: list/create/detail/update/version/publish/recall/upload/timeline/audit-logs  
5) Assignment: create/list/update/complete/revoke  
6) Comment/directive: create/list (và update nếu policy cho phép)  
7) Attachment: upload/preview/download/delete  
8) Search/export  
9) Notifications: list/mark read/read-all  
10) Audit logs: list/by-document  

### 0.2. Phạm vi production (khuyến nghị)
- CRUD phòng ban/chức vụ/role/permission + mapping endpoint ↔ permission.
- Delegation (uỷ quyền xử lý khi vắng mặt).
- Watermark/restrict download/print theo độ mật.
- Dispatch log cho văn bản đi (trạng thái gửi: pending/sent/failed/confirmed) + retry.
- Log bất biến (append-only), retention/partition cho audit log lớn.

### 0.3. Nguyên tắc “độ mật”
- Tách quyền:
  - **Xem metadata** (tiêu đề, trích yếu, số/ký hiệu, ngày…)
  - **Xem/preview/tải file** (đính kèm)
- Văn bản có độ mật cao phải:
  - hạn chế đối tượng được xem (theo đơn vị/assignment/delegation),
  - có thể hạn chế download/print,
  - bắt buộc audit khi view/preview/download/print.

---

## 1. Giới thiệu
Hệ thống/app quản lý văn bản phục vụ nghiệp vụ văn thư – lưu trữ và luân chuyển xử lý văn bản trong cơ quan/đơn vị:
- Quản lý vòng đời văn bản: tiếp nhận/soạn thảo → trình duyệt/bút phê → giao xử lý → theo dõi tiến độ/hạn → hoàn thành → lưu trữ.
- Kiểm soát truy cập theo vai trò, đơn vị và **độ mật/độ khẩn**.
- Bảo đảm truy vết:
  - **Audit log**: hành vi truy cập kỹ thuật (xem, preview, tải, in, export…).
  - **Timeline**: lịch sử nghiệp vụ (ai giao ai, ai bút phê, ai hoàn thành, đổi hạn, đổi trạng thái…).

---

## 2. Phân chia modules, chức năng chính, chức năng con (sub-function), use-case mẫu, rủi ro – thách thức

## 2.1. Module: Quản trị người dùng & phân quyền

### Danh mục/Thực thể liên quan
- Người dùng (User)
- Role/Permission (RBAC)
- Phòng ban/đơn vị (Department/OrgUnit)
- Chức vụ (Position)
- Uỷ quyền xử lý khi vắng mặt (Delegation/Acting)
- Nhật ký truy cập & nhật ký thao tác (AuditLog)

### Chức năng chính
- Đăng nhập/đăng xuất.
- Quản lý người dùng (CRUD).
- Quản lý phòng ban/chức vụ.
- Quản trị role/permission và gán quyền.
- Xem audit logs (theo user/action/văn bản/file/khoảng thời gian).

### Chức năng con (Sub-function)
1) Xác thực tăng cường (2FA/OTP) cho tài khoản nhạy cảm (văn thư, lãnh đạo, admin) *(production)*.  
2) Khoá tài khoản khi đăng nhập sai nhiều lần; cơ chế mở khoá theo quy trình *(production)*.  
3) Cơ chế phân quyền theo độ mật:
   - Bình thường / Mật / Tối mật / Tuyệt mật
   - Tách quyền **xem metadata** và **xem/preview/tải file**
4) Uỷ quyền xử lý khi vắng mặt (Delegation):
   - Có thời hạn (từ ngày… đến ngày…)
   - Có phạm vi (chỉ incoming / chỉ outgoing / chỉ độ mật X / chỉ đơn vị Y…)
   - Có log đầy đủ (ai uỷ quyền cho ai, lúc nào, phạm vi gì)
5) Export danh sách user và lịch sử audit (nếu được phép).
6) (Tuỳ chọn) Watermark khi xem/tải văn bản mật; hạn chế download/print theo cấp độ mật.

### Use-case chính
- Văn thư tạo tài khoản cán bộ mới, gán phòng ban/chức vụ, gán quyền theo vai trò.
- Lãnh đạo bút phê/giao xử lý văn bản đến.
- Trưởng phòng phân công cán bộ xử lý văn bản được giao.
- Lưu trữ tra cứu/trích xuất văn bản theo thẩm quyền.

### Rủi ro/thách thức
- Lộ văn bản mật do phân quyền sai; tải xuống trái phép.
- Thiếu audit xem/preview/tải/in/export → khó truy trách nhiệm.
- Uỷ quyền sai phạm vi → vượt quyền/lộ dữ liệu.

---

## 2.2. Module: Danh mục nghiệp vụ văn bản (Master Data)

### Danh mục/Thực thể
- Loại văn bản (DocumentType)
- Độ khẩn (UrgencyLevel)
- Độ mật (ConfidentialityLevel)
- Cơ quan gửi/nhận (Organization)
- Lĩnh vực (Field/Sector)

### Chức năng chính và chức năng con (Sub-function)
1) CRUD danh mục (cho phép active/inactive).  
2) Ràng buộc không cho xoá danh mục đang được sử dụng (chỉ cho inactive).  
3) Chuẩn hoá mã/định danh danh mục để phục vụ báo cáo và tra cứu.  
4) Import/Export danh mục *(tuỳ chọn)*.  
5) Cache master data phía app để load nhanh màn nhập liệu.

### Use-case
- Quản trị viên cập nhật danh mục độ khẩn/độ mật.
- Văn thư cập nhật danh mục cơ quan gửi/nhận và loại văn bản.

### Rủi ro/thách thức
- Danh mục không chuẩn → sai thống kê, khó tra cứu, khó chuẩn hoá dữ liệu.

---

## 2.3. Module: Quản lý văn bản đến (Incoming)

### Thực thể
- IncomingDocument
- Attachment (file scan/đính kèm)
- Assignment
- Comment/Directive
- Timeline

### Chức năng chính/chức năng con (Sub-function)
1) Tiếp nhận văn bản đến: nhập metadata
   - Số/ký hiệu, trích yếu, cơ quan gửi
   - Ngày đến / ngày văn bản, người ký
   - Loại văn bản, độ khẩn, độ mật, lĩnh vực
   - Đơn vị chủ quản/đầu mối
   - (Khuyến nghị) Kênh tiếp nhận: giấy/email/cổng điện tử...
2) Đính kèm file scan:
   - file chính / phụ lục
   - preview nhanh, download theo policy độ mật
3) Trình lãnh đạo bút phê và giao xử lý:
   - bút phê có thể nhiều cấp
   - giao xử lý: chủ trì/phối hợp/để biết + hạn
4) Theo dõi hạn xử lý, cảnh báo quá hạn:
   - nhắc hạn cho cá nhân/đơn vị được giao
5) Quản lý trạng thái vòng đời (khuyến nghị chuẩn hoá state machine):
   - draft → received → submitted → assigned → processing → completed → archived
   - có thể có cancelled (huỷ/không xử lý) theo policy
6) Lưu trữ và kết thúc hồ sơ:
   - khoá sửa metadata theo policy
   - vẫn cho phép tra cứu theo quyền

### Use-case
- Văn thư nhập văn bản đến, scan đính kèm, trình lãnh đạo.
- Lãnh đạo bút phê: giao phòng A chủ trì, phòng B phối hợp, deadline.
- Trưởng phòng phân công chuyên viên, theo dõi kết quả và thời hạn.

### Rủi ro/thách thức
- Metadata sai → giảm giá trị pháp lý; khó tra cứu về sau.
- File scan thất lạc/không đồng bộ, dung lượng lớn gây chậm.
- Văn bản mật bị lộ qua preview/download ngoài kiểm soát.

---

## 2.4. Module: Quản lý văn bản đi (Outgoing)

### Thực thể
- OutgoingDocument
- Draft/Version
- Attachment
- Recipient/DispatchLog (lịch sử nơi nhận + trạng thái gửi)
- Link Incoming ↔ Outgoing (trả lời/căn cứ/liên quan)

### Chức năng chính/chức năng con (Sub-function)
1) Soạn thảo hoặc upload dự thảo; quản lý phiên bản:
   - version 1,2,3…; đánh dấu bản hiện hành
   - hạn chế chỉnh sửa bản đã phát hành theo policy
2) Trình duyệt nội bộ:
   - (tuỳ chọn) ký số
3) Phát hành:
   - cấu hình nơi nhận (cơ quan trong danh mục / nơi nhận ngoài danh mục)
   - ghi nhận dispatch log: sent_at, status (pending/sent/failed/confirmed), note
   - (tuỳ chọn) retry khi gửi thất bại
4) Liên kết outgoing với incoming:
   - loại liên kết: reply/reference/related
5) Thu hồi/đính chính văn bản đã phát hành:
   - bắt buộc lý do
   - có log đầy đủ

### Use-case
- Chuyên viên soạn dự thảo văn bản đi, trình trưởng phòng/lãnh đạo duyệt.
- Văn thư phát hành văn bản đi, ghi nhận danh sách nơi nhận và trạng thái gửi.
- Liên kết văn bản đi là “trả lời” cho một văn bản đến cụ thể.

### Rủi ro/thách thức
- Phát hành nhầm nơi nhận → rò rỉ văn bản mật.
- Dùng nhầm phiên bản dự thảo → sai nội dung ban hành.
- Thiếu dispatch log → khó chứng minh đã gửi/đã phát hành.

---

## 2.5. Module: Luân chuyển xử lý, bút phê & giao xử lý

### Thực thể
- Assignment
- Comment/Directive
- Timeline
- Trạng thái xử lý theo người/đơn vị

### Chức năng con (Sub-function)
1) Giao xử lý theo vai trò:
   - Chủ trì (primary)
   - Phối hợp (cooperate)
   - Nhận để biết (notify)
2) Điều chỉnh/thu hồi giao xử lý:
   - bắt buộc lý do, có log
3) Chuỗi bút phê nhiều cấp (lãnh đạo → trưởng phòng → chuyên viên…)
4) Theo dõi trạng thái xử lý theo từng người/đơn vị:
   - ai đang giữ, ai quá hạn, ai đã hoàn thành
5) Ghi nhận kết quả xử lý:
   - nội dung kết quả + file kết quả + (tuỳ chọn) draft/outgoing trả lời

### Policy đề xuất (để đảm bảo tính pháp lý)
- Bút phê dạng “directive” quan trọng:
  - hạn chế sửa/xoá
  - nếu cho sửa: bắt buộc `edit_reason`, ghi log và lưu phiên bản

### Use-case
- Lãnh đạo giao xử lý cho phòng/nhân sự, kèm chỉ đạo và thời hạn.
- Trưởng phòng điều phối chuyên viên; cập nhật tiến độ và kết quả.
- Thu hồi giao xử lý khi giao nhầm (bắt buộc ghi lý do, lưu vết).

### Rủi ro/thách thức
- Tranh chấp trách nhiệm nếu timeline/log không rõ.
- Luồng quá phức tạp → người dùng bỏ qua quy trình.
- Nếu bút phê có thể sửa/xoá không kiểm soát → mất tính pháp lý.

---

## 2.6. Module: Tra cứu – tìm kiếm – kết xuất

### Chức năng con (Sub-function)
1) Tra cứu theo:
   - Số/ký hiệu, trích yếu, cơ quan gửi/nhận, người ký
   - Ngày văn bản / ngày đến / ngày phát hành
2) Lọc theo:
   - Trạng thái xử lý, hạn xử lý (đúng hạn/quá hạn)
   - Độ mật/độ khẩn
   - Đơn vị/cá nhân xử lý
   - Lĩnh vực/loại văn bản
3) Export danh sách văn bản và báo cáo (Excel/PDF):
   - phải áp dụng RBAC/độ mật giống như khi xem/list
4) (Tuỳ chọn) Lưu bộ lọc mẫu theo người dùng.

### Use-case
- Trưởng phòng tra cứu “văn bản đến trong tuần, quá hạn, độ khẩn cao”.
- Văn thư xuất danh sách văn bản đến/đi theo thời gian để báo cáo.

### Rủi ro/thách thức
- Tìm kiếm chậm khi dữ liệu lớn → cần index, phân trang, tối ưu truy vấn.
- Rò rỉ dữ liệu nếu export/filter không kiểm soát theo quyền, đặc biệt văn bản mật.

---

## 2.7. Module: Thông báo – nhắc việc

### Chức năng con (Sub-function)
1) Nhắc hạn xử lý văn bản đến.
2) Thông báo:
   - có văn bản mới được giao xử lý
   - thay đổi hạn
   - thu hồi giao xử lý
   - thu hồi/phát hành văn bản đi
3) Đồng bộ thông báo:
   - web/mobile
   - (tuỳ chọn) email
4) Tuỳ chỉnh loại thông báo theo vai trò & mức độ ưu tiên *(production)*.

### Use-case
- Chuyên viên nhận thông báo “có văn bản mới được giao”, “còn 1 ngày đến hạn”.
- Trưởng phòng nhận cảnh báo “văn bản quá hạn” của đơn vị.

### Rủi ro/thách thức
- Spam thông báo → bỏ sót cảnh báo quan trọng.
- Không đồng bộ trạng thái đã đọc giữa nhiều thiết bị.

---

## 2.8. Module: Lịch sử, nhật ký, audit

### Chức năng con (Sub-function)
1) Audit log hành vi:
   - login/logout
   - view metadata (đặc biệt với văn bản mật)
   - preview/download/print attachments
   - export
   - publish/recall outgoing
   - create/update/change_status
   - assign/revoke/complete
2) Timeline nghiệp vụ:
   - giao ai, ai bút phê, ai hoàn thành, đổi hạn, đổi trạng thái
3) Giao diện diff thay đổi metadata (trước/sau) *(khuyến nghị)*.
4) Export báo cáo audit theo user/đơn vị/khoảng thời gian.
5) Định hướng log bất biến (append-only), hạn chế quyền can thiệp log.

### Use-case
- Khi có sự cố rò rỉ, truy xuất “ai đã preview/tải file văn bản X” và thời điểm.
- Đối chiếu thay đổi metadata văn bản qua các lần chỉnh sửa.

### Rủi ro/thách thức
- Log rất lớn → cần chiến lược index/partition/retention.
- Nếu log bị sửa/xoá → mất tính toàn vẹn và giá trị kiểm tra.
