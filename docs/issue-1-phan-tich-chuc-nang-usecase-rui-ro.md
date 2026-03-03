# Phân tích hệ thống – App Quản lý Văn bản (Đến/Đi) (Chuẩn hoá)

> Mục tiêu: Xây dựng **app quản lý văn bản** (văn bản đến/văn bản đi), có luân chuyển xử lý, bút phê/ý kiến, theo dõi hạn, tra cứu – kết xuất, thông báo và audit.

---

## 1. Giới thiệu
Hệ thống/app quản lý văn bản phục vụ nghiệp vụ văn thư – lưu trữ và luân chuyển xử lý văn bản trong cơ quan/đơn vị:
- Quản lý vòng đời văn bản: tiếp nhận/soạn thảo → trình duyệt/bút phê → giao xử lý → theo dõi tiến độ/hạn → hoàn thành → lưu trữ.
- Kiểm soát truy cập theo vai trò, đơn vị và **độ mật/độ khẩn**.
- Bảo đảm truy vết (audit) đầy đủ: **ai xem, ai tải, ai in, ai chuyển**, ai bút phê, ai giao xử lý.

---

## 2. Phân chia modules, chức năng chính, chức năng con (sub-function), use-case mẫu, rủi ro – thách thức

### 2.1. Module: Quản trị người dùng & phân quyền

#### Danh mục/Thực thể liên quan
- Người dùng (User)
- Nhóm quyền / Vai trò (Roles/Permission group)
- Phòng ban/đơn vị (Department/OrgUnit)
- Chức vụ (Position)
- Uỷ quyền xử lý khi vắng mặt (Delegation/Acting)
- Nhật ký truy cập & nhật ký thao tác (AuditLog)

#### Chức năng chính
- Đăng nhập/đăng xuất
- Quản lý người dùng (CRUD)
- Phân quyền theo vai trò nghiệp vụ:
  - Văn thư
  - Lãnh đạo
  - Trưởng phòng
  - Chuyên viên
  - Lưu trữ
  - Quản trị hệ thống (Admin)
- Quản lý phòng ban/chức vụ
- Xem nhật ký, lịch sử truy cập/xem/tải/in/chuyển văn bản

#### Chức năng con (Sub-function)
1. Xác thực tăng cường (2FA/OTP) cho tài khoản nhạy cảm (văn thư, lãnh đạo, admin)
2. Khoá tài khoản khi đăng nhập sai nhiều lần; cơ chế mở khoá theo quy trình
3. Cơ chế phân quyền theo độ mật:
   - Bình thường / Mật / Tối mật / Tuyệt mật
   - Tách quyền **xem metadata** và **xem/tải file**
4. Cơ chế uỷ quyền xử lý khi vắng mặt:
   - Có thời hạn (từ ngày… đến ngày…)
   - Có phạm vi (chỉ văn bản đến, chỉ văn bản độ mật X, chỉ đơn vị Y…)
   - Có log đầy đủ
5. Export danh sách user và lịch sử truy cập/audit
6. (Tuỳ chọn) Watermark khi xem/tải văn bản mật; hạn chế download theo cấp độ mật

#### Use-case chính
- Văn thư tạo tài khoản cán bộ mới, gán phòng ban/chức vụ, gán quyền xem văn bản theo đơn vị.
- Lãnh đạo có quyền bút phê/giao xử lý văn bản đến.
- Trưởng phòng phân công cán bộ xử lý văn bản được giao.
- Lưu trữ quản lý truy cập kho lưu văn bản và trích xuất văn bản theo thẩm quyền.

#### Rủi ro/thách thức
- Lộ văn bản mật do phân quyền sai; tải xuống trái phép.
- Thiếu audit hành vi xem/tải/in/chuyển văn bản gây khó truy trách nhiệm.
- Uỷ quyền sai phạm vi dẫn tới vượt quyền/lộ dữ liệu.

---

### 2.2. Module: Danh mục nghiệp vụ văn bản

#### Danh mục/Thực thể
- Loại văn bản (DocumentType)
- Độ khẩn (UrgencyLevel)
- Độ mật (ConfidentialityLevel)
- Cơ quan gửi/nhận (Organization)
- Lĩnh vực (Field/Sector)

#### Chức năng chính và chức năng con (Sub-function)
1. CRUD danh mục (cho phép active/inactive)
2. Ràng buộc không cho xoá danh mục đang được sử dụng (chỉ cho inactive)
3. Chuẩn hoá mã/định danh danh mục để phục vụ báo cáo và tra cứu
4. Import/Export danh mục (tuỳ chọn)

#### Use-case
- Quản trị viên cập nhật danh mục độ khẩn/độ mật.
- Văn thư cập nhật danh mục cơ quan gửi/nhận và loại văn bản.

#### Rủi ro/thách thức
- Danh mục không chuẩn gây sai thống kê, khó tra cứu, khó chuẩn hoá dữ liệu.

---

### 2.3. Module: Quản lý văn bản đến (Incoming)

#### Thực thể
- Văn bản đến (IncomingDocument)
- Tệp scan/đính kèm (Attachment)
- Giao xử lý (Assignment)
- Ý kiến/bút phê (Comment/Directive)
- Timeline xử lý

#### Chức năng chính/chức năng con (Sub-function)
1. Tiếp nhận văn bản đến: nhập metadata
   - Số/ký hiệu
   - Trích yếu
   - Cơ quan gửi
   - Ngày đến / ngày văn bản
   - Người ký
   - Loại văn bản
   - Độ khẩn / độ mật
   - Lĩnh vực
2. Đính kèm file scan, phân loại tài liệu (file chính, phụ lục…)
3. Trình lãnh đạo bút phê và giao xử lý
4. Theo dõi hạn xử lý, cảnh báo quá hạn
5. Cập nhật trạng thái xử lý theo vòng đời: mới tiếp nhận → đã trình → đang xử lý → hoàn thành → lưu trữ
6. Lưu trữ và kết thúc hồ sơ (đóng xử lý, chuyển trạng thái lưu)

#### Use-case
- Văn thư nhập văn bản đến, scan đính kèm, trình lãnh đạo.
- Lãnh đạo bút phê: giao phòng A chủ trì, phòng B phối hợp, deadline.
- Trưởng phòng phân công chuyên viên, theo dõi kết quả và thời hạn.

#### Rủi ro/thách thức
- Metadata sai làm giảm giá trị pháp lý; khó tra cứu về sau.
- File scan bị thất lạc/không đồng bộ phiên bản; dung lượng lớn gây chậm.
- Văn bản mật bị lộ qua tải xuống/chia sẻ ngoài hệ thống.

---

### 2.4. Module: Quản lý văn bản đi (Outgoing)

#### Thực thể
- Văn bản đi (OutgoingDocument)
- Dự thảo/phiên bản (Draft/Version)
- Tệp đính kèm (Attachment)
- Nơi nhận (Recipient/DispatchLog)
- Liên kết văn bản đi ↔ văn bản đến (trả lời/căn cứ)

#### Chức năng chính/chức năng con (Sub-function)
1. Soạn thảo hoặc upload dự thảo; quản lý phiên bản
2. Trình duyệt nội bộ; (tuỳ chọn) ký số
3. Phát hành: cấu hình nơi nhận, ghi nhận lịch sử gửi
4. Liên kết văn bản đi với văn bản đến (trả lời/căn cứ)
5. Thu hồi/đính chính văn bản đã phát hành (có log và lý do)

#### Use-case
- Chuyên viên soạn dự thảo văn bản đi, trình trưởng phòng/ lãnh đạo duyệt.
- Văn thư phát hành văn bản đi, ghi nhận danh sách nơi nhận và trạng thái gửi.
- Liên kết văn bản đi là “trả lời” cho một văn bản đến cụ thể.

#### Rủi ro/thách thức
- Phát hành nhầm nơi nhận; rò rỉ văn bản mật.
- Dùng nhầm phiên bản dự thảo gây sai nội dung ban hành.
- Thiếu log phát hành khiến khó chứng minh đã gửi/đã phát hành.

---

### 2.5. Module: Luân chuyển xử lý, bút phê & giao xử lý

#### Thực thể
- Assignment (giao xử lý)
- Comment/Directive (bút phê/ý kiến)
- Timeline xử lý
- Trạng thái xử lý theo người/đơn vị

#### Chức năng con (Sub-function)
1. Giao xử lý theo vai trò:
   - Chủ trì
   - Phối hợp
   - Nhận để biết
2. Điều chỉnh/thu hồi giao xử lý (bắt buộc lý do, có log đầy đủ)
3. Chuỗi bút phê nhiều cấp (lãnh đạo → trưởng phòng → chuyên viên…)
4. Theo dõi trạng thái xử lý theo từng người/đơn vị; ai đang giữ, ai quá hạn
5. Ghi nhận kết quả xử lý: nội dung xử lý, file kết quả, văn bản dự thảo trả lời…

#### Use-case
- Lãnh đạo giao xử lý cho phòng/nhân sự, kèm chỉ đạo và thời hạn.
- Trưởng phòng điều phối chuyên viên; cập nhật tiến độ và kết quả.
- Thu hồi giao xử lý khi giao nhầm (hệ thống bắt buộc ghi lý do, lưu vết).

#### Rủi ro/thách thức
- Tranh chấp trách nhiệm nếu timeline/log không rõ.
- Luồng quá phức tạp gây khó dùng; người dùng bỏ qua quy trình hệ thống.
- Nếu bút phê có thể sửa/xoá không kiểm soát → mất tính pháp lý.

---

### 2.6. Module: Tra cứu – tìm kiếm – kết xuất

#### Chức năng con (Sub-function)
1. Tra cứu theo:
   - Số/ký hiệu
   - Trích yếu
   - Cơ quan gửi/nhận
   - Người ký
   - Ngày văn bản / ngày đến / ngày phát hành
2. Lọc theo:
   - Trạng thái xử lý
   - Hạn xử lý (đúng hạn/quá hạn)
   - Độ mật/độ khẩn
   - Đơn vị/cá nhân xử lý
   - Lĩnh vực/loại văn bản
3. Export danh sách văn bản và báo cáo (Excel/PDF)
4. (Tuỳ chọn) Lưu bộ lọc mẫu theo người dùng

#### Use-case
- Trưởng phòng tra cứu “văn bản đến trong tuần, quá hạn, độ khẩn cao”.
- Văn thư xuất danh sách văn bản đến/đi theo thời gian để báo cáo.

#### Rủi ro/thách thức
- Tìm kiếm chậm khi dữ liệu lớn → cần index, phân trang, tối ưu truy vấn.
- Rò rỉ dữ liệu nếu export/filter không kiểm soát theo quyền, đặc biệt văn bản mật.

---

### 2.7. Module: Thông báo – nhắc việc

#### Chức năng con (Sub-function)
1. Nhắc hạn xử lý văn bản đến
2. Thông báo văn bản mới được giao xử lý / thay đổi hạn / thu hồi / chuyển xử lý
3. Đồng bộ thông báo: web/mobile/email (tuỳ điều kiện triển khai)
4. Tùy chỉnh loại thông báo theo vai trò và mức độ ưu tiên

#### Use-case
- Chuyên viên nhận thông báo “có văn bản mới được giao”, “còn 1 ngày đến hạn”.
- Trưởng phòng nhận cảnh báo “văn bản quá hạn” của đơn vị.

#### Rủi ro/thách thức
- Spam thông báo làm bỏ sót cảnh báo quan trọng.
- Không đồng bộ trạng thái đã đọc giữa nhiều thiết bị.

---

### 2.8. Module: Lịch sử, nhật ký, audit

#### Chức năng con (Sub-function)
1. Log xem/tải/in/chuyển văn bản (ai, lúc nào, hành động gì, thiết bị/IP nếu cần)
2. Timeline xử lý văn bản (giao ai, ai bút phê, ai hoàn thành, thay đổi hạn…)
3. Giao diện diff thay đổi metadata (trước/sau)
4. Export báo cáo audit theo người dùng/đơn vị/khoảng thời gian
5. Định hướng log bất biến (append-only), hạn chế quyền can thiệp log

#### Use-case
- Khi có sự cố rò rỉ, truy xuất “ai đã tải file văn bản X” và thời điểm.
- Đối chiếu thay đổi metadata văn bản qua các lần chỉnh sửa.

#### Rủi ro/thách thức
- Log rất lớn, khó lưu trữ/truy vấn → cần chiến lược index/partition/retention.
- Nếu log bị sửa/xoá → mất tính toàn vẹn và giá trị kiểm tra.

---
