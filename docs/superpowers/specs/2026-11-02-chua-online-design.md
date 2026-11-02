# Chùa Online: Spec thiết kế

**Ngày:** 2026-11-02
**Trạng thái:** Đã chốt, dùng làm căn cứ cho plan triển khai.

## 1. Mục tiêu

Website chùa ảo cho phép người dùng lạy Phật, lần tràng hạt, nghe kinh, thắp hương và chắp tay chào người khác, hoàn toàn ẩn danh. Đây là đồ án tốt nghiệp của sinh viên, ưu tiên hoàn thành đúng hạn và chi phí gần bằng 0.

**Nguyên tắc:** Không chat, không nhắn tin, không hiển thị danh tính. Tương tác chỉ qua hành động mang tính nghi thức.

## 2. Đối tượng và vai trò

- **Khách (ẩn danh):** dùng toàn bộ tính năng chính, không cần tài khoản.
- **Quản trị viên:** một tài khoản duy nhất, quản lý nội dung kinh và xem thống kê.

## 3. Chức năng MVP

| # | Chức năng | Mô tả |
|---|---|---|
| F1 | Vào chùa ảo | Vào không cần đăng nhập, chọn không gian (chính điện, sân chùa). |
| F2 | Lạy Phật | Bấm để lạy, đếm 108 lạy, có hiệu ứng và âm thanh mõ (bật/tắt được). |
| F3 | Tràng hạt | 108 hạt, bấm để lần, hạt sáng theo tiến độ. |
| F4 | Nghe kinh | Phát audio từ danh sách kinh, bật/tắt, chọn bài. |
| F5 | Thắp hương | Hiệu ứng khói hương, tự tắt sau khoảng 8 giây. |
| F6 | Chắp tay chào | Gửi một cái chắp tay ẩn danh, hiển thị thông báo ngắn cho mọi người đang online. |
| F7 | Hiển thị online | Số người đang kết nối, cập nhật realtime. |
| F8 | Thống kê cá nhân | Số lạy hôm nay và chuỗi ngày liên tiếp có lạy. |
| F9 | Giao diện responsive | Dùng tốt trên điện thoại. |
| F10 | Chống spam | Giới hạn tần suất lạy, chắp tay và request. |

## 4. Ngoài phạm vi (làm sau nếu còn thời gian)

- Đăng nhập Google để đồng bộ tiến trình.
- Thống kê toàn hệ thống nâng cao.
- Nhiều không gian và nhạc nền.
- Bảng xếp hạng số lạy.

## 5. Quản trị (admin)

- Đăng nhập bằng mật khẩu băm (bcrypt) lưu trong biến môi trường.
- Thêm, sửa, xóa nội dung kinh.
- Xem thống kê lượt truy cập và số lạy.

## 6. Yêu cầu phi chức năng

- Trang tải nhanh trên mạng di động, mục tiêu dưới 3 giây.
- Không thu thập dữ liệu cá nhân. Không lưu nội dung chắp tay.
- Nội dung tôn trọng tôn giáo. Không có bảng xếp hạng công khai trong giai đoạn đầu.
- Chỉ dùng audio kinh thuộc phạm vi công cộng, ghi nguồn và bản quyền trong dữ liệu.

## 7. Quyết định kỹ thuật (đã chốt)

- **Backend:** Node.js 20 LTS, Express, Socket.IO.
- **Database:** MySQL 8, truy cập qua `mysql2` và Knex (migration).
- **Triển khai:** iNET Cloud VPS (Ubuntu LTS), Nginx reverse proxy, PM2, Certbot.
- **Docker:** chỉ dùng trên máy dev để chạy MySQL. Không dùng trên VPS.
- **Tên miền:** mua riêng.
- **Frontend:** HTML, CSS, JavaScript thuần, phục vụ từ Express.

## 8. Dữ liệu

- `sutras`: id, title, audio_url, source, license, is_active, created_at.
- `visits`: session_id, day, khóa chính (session_id, day). Đếm lượt duy nhất mỗi ngày.
- `bows`: session_id, day, count, updated_at, khóa chính (session_id, day).
- `daily_stats`: day (khóa chính), visits, bows_total.

Ngày (`day`) được tính trong code theo múi giờ Asia/Ho_Chi_Minh, không dùng hàm ngày của MySQL.

## 9. Ràng buộc chung

- Mỗi lần gửi lạy: số lạy từ 1 đến 200.
- Chắp tay: tối đa một lần mỗi 3 giây cho mỗi kết nối socket.
- `POST /api/bows`: tối đa 30 request mỗi phút cho mỗi IP.
- Cổng MySQL không mở ra internet.
- Mỗi trình duyệt có một session ẩn danh (UUID v4) lưu trong localStorage.

## 10. Tiêu chí nghiệm thu

- Có link production chạy ổn định, HTTPS, mở được trên điện thoại.
- F1 đến F8 hoạt động, F9 và F10 kiểm tra được bằng thử nghiệm.
- Admin quản lý được kinh và xem được thống kê.
- Có backup hằng ngày và đã thử khôi phục thành công.
- Có test tự động cho API và các service chính.
