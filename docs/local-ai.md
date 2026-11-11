# AI local và xử lý tin tức

## Định hướng

Người dùng muốn dựng AI local để tóm tắt bản tin. Có thể sử dụng model có sẵn, chưa cần huấn luyện từ đầu. Chưa chọn model hoặc runtime; cần đo trên phần cứng thực tế trước khi quyết định.

Python phù hợp để xây dựng pipeline xử lý nội dung và tích hợp model. Worker có thể gọi model trực tiếp hoặc qua endpoint nội bộ của runtime. FastAPI chỉ cần khi muốn cung cấp một API HTTP riêng cho chức năng AI.

## Hai phương án vận hành

| Phương án | Ưu điểm | Hạn chế |
| --- | --- | --- |
| Model trên máy cá nhân, ứng dụng trên AWS | Tận dụng phần cứng hiện có, phù hợp thử nghiệm | Máy tắt/mất mạng sẽ dừng xử lý; tin nóng có thể bị chậm |
| Model tự host trên server | Chủ động vận hành liên tục | Phải tính tài nguyên, chi phí và quản lý dịch vụ model |

Nếu dùng máy cá nhân, worker có thể chủ động đọc job từ hàng đợi AWS rồi lưu kết quả qua giao diện backend được xác thực. Không cần mở cổng model ra Internet. Quyền truy cập và credential phải được cấu hình phù hợp.

## Pipeline đề xuất

1. Thu thập bài từ các nguồn đã chọn.
2. Trích xuất nội dung, chuẩn hóa URL, thời gian và nguồn.
3. Lọc trùng, gom các bài nói về cùng sự kiện.
4. Phân loại và gọi model để tóm tắt dựa trên nội dung nguồn.
5. Kiểm tra cấu trúc đầu ra và lưu kết quả kèm liên kết nguồn.
6. Tái sử dụng bản tóm tắt theo sự kiện/ngôn ngữ khi ghép email.

Prompt cần yêu cầu giữ dữ kiện quan trọng, không thêm thông tin ngoài nguồn và thể hiện sự không chắc chắn. Nội dung bài báo là dữ liệu đầu vào, không phải chỉ dẫn để model thực thi. Với các nguồn mâu thuẫn, cần giữ dấu vết nguồn thay vì tự suy đoán kết luận.

## Lịch xử lý và lỗi

- Tóm tắt dần khi có tin, chuẩn bị trước giờ gửi bản tin.
- Lưu kết quả để không xử lý lại cùng nội dung khi không cần thiết.
- Khi model lỗi hoặc không hoạt động, giữ job để retry có giới hạn.
- Có thể dùng các nội dung đã xử lý xong cho bản tin; chính sách khi thiếu tin cần được chốt.
- Tách lịch gửi email khỏi tốc độ xử lý model.

## Thông tin cần có để chọn model

- GPU, dung lượng VRAM và RAM; khả năng chạy CPU nếu không có GPU.
- Ngôn ngữ nguồn và ngôn ngữ bản tóm tắt.
- Số bài/ngày, độ dài bài và thời gian máy có thể hoạt động.
- Độ trễ chấp nhận cho tin nóng.
- Giấy phép model phù hợp với mục đích sử dụng.

Đánh giá thử trên một tập bài đại diện: độ chính xác dữ kiện, chất lượng tiếng Việt, khả năng gom/tóm tắt nhiều nguồn, thời gian xử lý và mức dùng bộ nhớ. Chưa có benchmark hoặc xác nhận cấu hình phần cứng trong giai đoạn trao đổi này.
