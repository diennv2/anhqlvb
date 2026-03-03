# Chi tiết cơ sở dữ liệu cho ứng dụng quản lý ca làm việc

## Các bảng trong cơ sở dữ liệu

1. **Users**: Lưu thông tin người dùng.
   - `id`: Khóa chính, định danh người dùng.
   - `username`: Tên đăng nhập của người dùng.
   - `password`: Mật khẩu của người dùng được mã hóa.
   - `role`: Vai trò của người dùng (admin/user).

2. **Shifts**: Lưu thông tin về ca làm việc.
   - `id`: Khóa chính, định danh ca làm việc.
   - `user_id`: Khóa ngoại, liên kết đến bảng Users.
   - `start_time`: Thời gian bắt đầu ca làm việc.
   - `end_time`: Thời gian kết thúc ca làm việc.
   - `date`: Ngày thực hiện ca làm việc.

3. **Departments**: Lưu thông tin về các phòng ban.
   - `id`: Khóa chính, định danh phòng ban.
   - `name`: Tên phòng ban.
   - `description`: Mô tả về phòng ban.

## Tương tác giữa các bảng

- Bảng Users liên kết với bảng Shifts thông qua `user_id`.
- Có thể mở rộng cơ sở dữ liệu để thêm nhiều bảng khác như bảng Lịch sử ca làm việc, Bảng phân công công việc, v.v.

## Vấn đề cần giải quyết

Dự kiến sẽ cần vẽ một sơ đồ ER cho cơ sở dữ liệu này để có cái nhìn tổng quát hơn về mối quan hệ giữa các bảng.