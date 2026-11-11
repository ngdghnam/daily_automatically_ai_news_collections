# Sản phẩm và phạm vi MVP

## Mục tiêu đã xác định

Xây dựng dịch vụ tổng hợp tin tức từ nhiều nguồn trên thế giới và tạo bản tin theo sở thích của từng người dùng.

- Gửi bản tóm tắt hằng ngày vào khoảng 8h sáng.
- Chọn nội dung theo loại tin tức người dùng quan tâm.
- Email là kênh gửi dự kiến ban đầu.
- Cập nhật kịp thời khi có tin nóng.
- Dùng AI để tóm tắt tin; người dùng đang định hướng chạy model local.
- Frontend dùng Next.js, backend dùng NestJS; hạ tầng hướng tới AWS.

## Luồng bản tin hằng ngày đề xuất

1. Thu thập tin định kỳ từ các nguồn RSS/API đã chọn.
2. Làm sạch nội dung, phát hiện bài trùng và gom bài nói về cùng sự kiện.
3. Phân loại, tóm tắt và lưu kết quả trước giờ gửi.
4. Chọn và sắp xếp tin theo sở thích của từng người.
5. Ghép email, gửi theo lịch và lưu trạng thái gửi.

Không đợi đến 8h mới bắt đầu thu thập và gọi model. Mục tiêu là bắt đầu gửi vào giờ đã chọn; không thể bảo đảm mọi email đến hộp thư đúng cùng thời điểm.

## Cá nhân hóa đề xuất

- Chủ đề quan tâm.
- Ngôn ngữ bản tin.
- Múi giờ và giờ nhận.
- Bật/tắt thông báo tin nóng, giới hạn số thông báo và giờ yên lặng.

Nên tóm tắt một lần cho mỗi sự kiện/ngôn ngữ rồi tái sử dụng. Giai đoạn đầu, cá nhân hóa tập trung vào chọn và xếp hạng tin, tránh gọi model riêng cho từng người.

## Tin nóng

Luồng tin nóng dùng chung dữ liệu đã thu thập nhưng có bước đánh giá riêng:

- Độ mới và mức độ ảnh hưởng.
- Số nguồn độc lập xác nhận.
- Mức độ liên quan đến sở thích người nhận.
- Sự kiện đã thông báo hay có diễn biến mới đáng kể.
- Giới hạn tần suất và giờ yên lặng.

Chưa chốt ngưỡng xác định tin nóng, độ trễ mục tiêu hoặc chính sách xác minh. AI hỗ trợ đánh giá; cần giữ nguồn và biểu thị thông tin chưa xác nhận.

## MVP đề xuất

Ưu tiên hoàn thiện bản tin hằng ngày trước khi triển khai thông báo tin nóng:

1. Đăng ký email và chọn một số chủ đề.
2. Thu thập từ một tập nguồn RSS/API giới hạn.
3. Tóm tắt có liên kết nguồn.
4. Gửi bản tin khoảng 8h, lưu lịch sử và hỗ trợ hủy đăng ký.
5. Theo dõi lỗi thu thập, xử lý AI và gửi email.

## Cần chốt trước khi triển khai

- 8h theo giờ Việt Nam hay theo múi giờ từng người; mặc định đề xuất là `Asia/Ho_Chi_Minh`.
- Chủ đề, ngôn ngữ, nguồn tin và phạm vi sử dụng nội dung.
- Số người dùng, số bài mỗi ngày và ngân sách vận hành.
- Định nghĩa tin nóng, độ trễ chấp nhận và giới hạn email.
- Cấu hình máy chạy model và mức độ sẵn sàng cần có.
