# Business Requirements — AI Daily News Collective

Cập nhật: 13/11/2026. Phạm vi: nghiệp vụ bản tin cá nhân hóa và cảnh báo tin khẩn cấp qua email.

## 1. Mục đích và cách đọc

Tài liệu tổng hợp logic nghiệp vụ từ tài liệu trong repository, dùng làm cơ sở trao đổi sản phẩm, thiết kế luồng và nghiệm thu. Đây là mô tả yêu cầu, không phải xác nhận app đã triển khai đầy đủ.

Nguồn đối chiếu:

- [Database v1](docs/databases.md): cập nhật ngày 13/11/2026; nguồn chính cho các yêu cầu mới về nhiều khung giờ, topic, lịch sử và gửi email.
- [Kiến trúc](docs/architecture.md): cập nhật theo quyết định ngày 12/11/2026; mô tả trách nhiệm các thành phần và pipeline.
- [Sản phẩm](docs/product.md): định hướng ban đầu về bản tin hằng ngày, cá nhân hóa và tin nóng.
- [AI local](docs/local-ai.md): nguyên tắc xử lý AI và lỗi model.
- Mã nguồn trong `apps/`: web Angular còn giao diện mẫu, API NestJS trả `Hello World!`, Search Engine mới có Spring Boot entry point, file AI `main.py` hiện chưa có nội dung. Chưa có triển khai luồng nghiệp vụ end-to-end trong các file này.

Quy ước trạng thái yêu cầu:

| Nhãn | Ý nghĩa |
| --- | --- |
| **Đã thống nhất** | Được tài liệu nguồn ghi nhận là yêu cầu hoặc quyết định đã xác định. |
| **Thiết kế v1** | Quy tắc được mô tả trong thiết kế hiện tại để triển khai nhất quán; chưa đồng nghĩa đã nghiệm thu hoặc vận hành. |
| **Đề xuất / cần chốt** | Giả định hoặc lựa chọn còn mở; không tự chuyển thành cam kết sản phẩm. |

Khi tài liệu cũ khác tài liệu mới, bản tổng hợp ưu tiên cập nhật nghiệp vụ trong `docs/databases.md` và chỉ rõ thay đổi ở phần 14. Các mục dưới đây mô tả yêu cầu dự kiến; trạng thái triển khai thực tế vẫn như phần đối chiếu mã nguồn ở trên.

## 2. Mục tiêu và phạm vi

**Đã thống nhất:** hệ thống thu thập tin từ nhiều nguồn, gom các bài về cùng sự kiện, dùng AI chọn lọc/tóm tắt và gửi bản tin theo sở thích, giờ địa phương của người dùng.

- Người dùng đăng ký email, chọn topic và cấu hình giờ nhận.
- Một người dùng có thể nhận nhiều bản tin trong ngày; mỗi khung giờ có topic, giới hạn số tin và mức chi tiết riêng.
- Danh mục topic được quản lý tập trung, hỗ trợ xem xét topic người dùng đề xuất.
- Có cảnh báo tin khẩn cấp liên quan đến topic người dùng quan tâm.
- MVP sử dụng email làm kênh gửi.
- Cấu hình và bản tin đã tạo là hai đối tượng riêng; đổi cấu hình không làm thay đổi lịch sử.

Mục tiêu 5–10 phút đọc của Daily Digest trong kiến trúc là định hướng trải nghiệm ban đầu. Với cấu hình nhiều khung giờ, thời lượng cụ thể và phạm vi mức chi tiết cần được chốt; không áp đặt mọi bản tin luôn dài 5–10 phút.

Ngoài phạm vi MVP đã nêu: Telegram/push, sản phẩm Deep Dive riêng và các mức cảnh báo ngoài Critical. Mức `detailed` của một cấu hình bản tin không tự đồng nghĩa với tính năng Deep Dive. Chưa có yêu cầu được xác nhận về thu phí, gói thuê bao, quảng cáo hoặc thanh toán.

## 3. Tác nhân và quyền hạn

| Tác nhân | Trách nhiệm nghiệp vụ |
| --- | --- |
| Người dùng | Đăng ký/xác minh email, quản lý cấu hình nhận tin, chọn topic, đề xuất topic, cấu hình cảnh báo, xem lịch sử, tạm dừng hoặc hủy đăng ký. |
| Quản trị viên | Quản lý danh mục topic, nguồn tin, xem xét yêu cầu topic, quản lý template và theo dõi vận hành. Cơ chế duyệt topic bắt buộc hay tự động còn mở. |
| Hệ thống thu thập | Lấy nội dung được phép, chuẩn hóa, nhận diện bài trùng, theo dõi tình trạng nguồn. |
| Hệ thống AI | Gom sự kiện/diễn biến, phân loại, tóm tắt và cung cấp tín hiệu đánh giá khẩn cấp kèm nguồn. |
| Hệ thống điều phối/phát hành | Áp dụng cấu hình và chính sách, lập lịch, chọn tin, chống trùng, tạo email, gửi và lưu kết quả. |
| Nhà cung cấp email | Nhận yêu cầu gửi, cung cấp kết quả hoặc sự kiện giao email để đối soát. |

**Thiết kế v1:** mọi thao tác truy cập cấu hình và lịch sử phải kiểm tra quyền sở hữu. Người dùng không được đọc/sửa dữ liệu của người khác bằng cách thay ID. AI và bộ thu thập không tự quyết định phát hành email.

## 4. Các khái niệm nghiệp vụ

| Khái niệm | Ý nghĩa |
| --- | --- |
| Topic | Chủ đề dùng chung, có tên, mô tả phạm vi và định nghĩa tìm kiếm. |
| Nguồn tin | RSS, API hoặc trang web được phép thu thập. |
| Bài viết (`article`) | Một bài của một nguồn; có URL gốc, metadata và có thể có nội dung đã trích xuất. |
| Sự kiện (`news_event`) | Sự việc thực tế có thể được nhiều bài viết đề cập. |
| Diễn biến (`event_update`) | Một thay đổi thông tin có ý nghĩa của sự kiện; đơn vị quan trọng để chống gửi lại. |
| Bản tóm tắt | Nội dung AI theo diễn biến, ngôn ngữ, mức chi tiết và phiên bản. |
| Cấu hình bản tin | Quy tắc nhận tin theo khung giờ của một người dùng. |
| Bản tin (`newsletter`) | Nội dung thực tế tạo cho một cấu hình tại một lần chạy lịch. `Digest` trong tài liệu cũ tương ứng khái niệm này. |
| Cảnh báo (`breaking_alert`) | Nội dung khẩn cấp cho một người dùng và một diễn biến đủ điều kiện. |
| Delivery | Một email logic cần gửi; tách biệt với quá trình tạo nội dung và các lần thử gửi. |
| Snapshot | Bản chụp cấu hình, nội dung, nguồn hoặc chính sách tại thời điểm xử lý để giữ đúng lịch sử. |

## 5. Đăng ký, xác minh và quyền nhận email

### 5.1. Luồng đăng ký và quản lý

1. Người dùng cung cấp email và đồng ý nhận tin.
2. Hệ thống ghi nhận đăng ký và yêu cầu xác minh email.
3. Sau xác minh, người dùng đủ điều kiện kích hoạt nhận tin khi các điều kiện khác cũng hợp lệ.
4. Người dùng chọn ngôn ngữ, múi giờ và tạo cấu hình bản tin.
5. Người dùng có thể sửa tùy chọn, tạm dừng hoặc hủy đăng ký.

**Thiết kế v1:** chỉ gửi bản tin/cảnh báo cho email đã xác minh và user đủ điều kiện nhận. Email xác minh hoặc truy cập quản lý là email phục vụ tài khoản, có mục đích riêng.

| Trạng thái user | Ý nghĩa đối với bản tin/cảnh báo |
| --- | --- |
| `pending_verification` | Chưa đủ điều kiện nhận. |
| `active` | Có thể nhận khi cấu hình và nội dung hợp lệ. |
| `paused` | Tạm dừng gửi. |
| `unsubscribed` | Đã hủy đăng ký; chặn các email bản tin/cảnh báo chưa gửi. |
| `suppressed` | Bị chặn gửi do vấn đề địa chỉ hoặc phản hồi email, ví dụ bounce/complaint. |

Xác minh email, đồng ý nhận tin, tạm dừng và suppression là các điều kiện khác nhau. Xác minh không tự xóa một suppression hoặc thay thế đồng ý nhận tin.

### 5.2. Quy tắc bảo vệ quyền nhận tin

- Kiểm tra lại quyền nhận ngay trước lần gửi thực tế, kể cả email đã nằm trong hàng chờ.
- Đổi địa chỉ email phải xác minh lại; không tiếp tục gửi đến địa chỉ snapshot cũ nếu quyền nhận tại đó đã bị thu hồi.
- **Đề xuất v1:** chuẩn hóa email bằng trim và chuyển chữ thường để đối chiếu duy nhất; không tự bỏ dấu chấm hoặc `+tag`.
- **Thiết kế v1:** nếu dùng token, chỉ lưu hash; token xác minh/quản lý dùng một lần, có hết hạn và có thể bị thu hồi.
- **Cần chốt:** luồng truy cập trang quản lý và endpoint hủy đăng ký. Chưa mặc định yêu cầu mật khẩu hoặc đăng nhập mạng xã hội.

## 6. Topic và đề xuất topic

**Đã thống nhất:** người dùng chọn từ danh mục chung và có thể đề xuất topic cần bổ sung.

**Thiết kế v1:**

- Topic có trạng thái `draft`, `active`, `paused`, `archived`; việc dùng topic không hoạt động trong cấu hình/lịch chờ cần được xử lý nhất quán theo chính sách sản phẩm.
- Alias hỗ trợ các tên như “AI”, “Artificial Intelligence”, “Trí tuệ nhân tạo” trỏ về cùng topic. Alias có thể đa nghĩa, không mặc định duy nhất toàn hệ thống.
- Yêu cầu topic bắt đầu ở `pending`, có thể cần làm rõ (`needs_clarification`), được tạo thành topic mới (`approved`), ánh xạ topic có sẵn (`mapped`) hoặc bị từ chối (`rejected`).
- Yêu cầu được duyệt/ánh xạ phải chỉ ra topic đích và lưu kết quả xem xét.
- Gửi yêu cầu không tự làm topic hoạt động hoặc tự thêm vào mọi cấu hình bản tin.
- Nếu hỗ trợ tự thêm sau duyệt, phải biết cấu hình đích và kiểm tra lại quyền sở hữu trước khi thêm.
- Topic dùng chung giúp tái sử dụng việc thu thập và xử lý AI giữa nhiều người dùng.

**Cần chốt:** tự kích hoạt sau đối chiếu hay admin duyệt; cấu hình nhận topic sau duyệt; cách xử lý khi topic đang theo dõi bị tạm dừng. “Upload topic” hiện chỉ được hiểu là nhập tên/mô tả, chưa bao gồm nhập file hàng loạt.

## 7. Cấu hình bản tin và lịch nhận

### 7.1. Cấu hình theo khung giờ

**Đã thống nhất:** mỗi khung giờ có thể có topic, mức chi tiết và giới hạn số tin riêng.

**Đề xuất v1:** một cấu hình có một giờ gửi mỗi ngày. Muốn cùng cách đọc ở nhiều giờ thì tạo/sao chép nhiều cấu hình. Số lần nhận định kỳ mỗi ngày bằng số cấu hình đang bật; cảnh báo khẩn cấp không tính vào số lần này.

Mỗi cấu hình gồm tên, giờ nhận địa phương, mức chi tiết, giới hạn tổng số tin, thời lượng đọc mục tiêu nếu có, danh sách topic và hạn mức từng topic. Trạng thái cấu hình: `active`, `paused`, `archived`.

- **Đề xuất v1:** ngôn ngữ và múi giờ đặt ở user; mức chi tiết gồm `brief`, `standard`, `detailed` đặt ở cấu hình.
- **Thiết kế v1:** giới hạn số tin là số nguyên dương và là trần cứng; thời lượng đọc là mục tiêu ước lượng.
- **Đề xuất v1:** hạn mức topic tính theo mỗi bản tin. Đơn vị mỗi bản tin hay cả ngày vẫn cần xác nhận.
- Tổng hạn mức topic có thể vượt giới hạn bản tin; nội dung thực tế vẫn không vượt giới hạn tổng và không cam kết đủ quota của mọi topic.
- Một sự kiện liên quan nhiều topic chỉ chiếm một mục, có một topic chính để tính hạn mức.

Ví dụ minh họa, không phải giá trị mặc định: “Buổi sáng” lúc 08:00, đọc lướt, tối đa 5 tin; AI tối đa 3 và kinh tế tối đa 3. Bản tin vẫn chỉ có tối đa 5 tin. “Buổi tối” lúc 20:00 có thể đọc chi tiết với tối đa 10 tin và tập topic khác.

### 7.2. Lập lịch

**Thiết kế v1:**

- Giờ nhận được diễn giải theo IANA timezone của user; thiết kế dữ liệu dùng mặc định `Asia/Ho_Chi_Minh`.
- 08:00 là định hướng giờ nhận ban đầu, không phải giờ bắt buộc cho mọi user/cấu hình.
- Thu thập và xử lý AI chạy trước; giờ gửi là thời điểm bắt đầu phát hành, không bảo đảm email vào mọi hộp thư đúng cùng một giây.
- Đổi múi giờ, giờ nhận, trạng thái hoặc quyền nhận email phải tính lại lần chạy kế tiếp; lịch sử đã tạo giữ nguyên snapshot và thời điểm.
- Một cấu hình chỉ có một lần chạy tại một thời điểm lịch xác định, kể cả scheduler chạy lại hoặc cấu hình tăng revision.
- **Đề xuất v1:** giờ không tồn tại do DST chuyển tới thời điểm hợp lệ kế tiếp; giờ lặp chỉ gửi một lần.
- **Đề xuất / cần chốt:** bỏ qua lịch đã quá hạn thay vì gửi dồn sau gián đoạn; chưa có ngưỡng trễ. Cách xử lý nhiều cấu hình cùng giờ cũng chưa chốt.

## 8. Thu thập và xử lý nội dung

### 8.1. Thu thập nguồn

1. Lấy tin từ nguồn phù hợp topic theo lịch hoặc yêu cầu điều phối.
2. Lưu metadata, URL gốc và URL chuẩn hóa.
3. Trích xuất phần văn bản chính được phép từ RSS/API/web, loại bỏ menu và quảng cáo.
4. Kiểm tra trùng, ngôn ngữ, nội dung thiếu và giới hạn truy cập.
5. Chuyển dữ liệu đủ điều kiện sang gom sự kiện và xử lý AI.

**Thiết kế v1:**

- Tôn trọng quyền truy cập, giới hạn tốc độ và điều kiện sử dụng nguồn; không vượt paywall hoặc giới hạn truy cập.
- URL chuẩn hóa giúp nhận diện cùng bài; hash nội dung hỗ trợ nhận diện sao chép, không chứng minh dữ kiện đúng.
- Nhiều URL hoặc nhiều trang cùng nhà xuất bản không tự được tính là nhiều xác nhận độc lập. Nguồn chưa rõ nhóm nhà xuất bản không tự được xem là độc lập đã xác minh.
- Theo dõi lỗi/tiến độ thu thập riêng với trạng thái xử lý AI.
- Không lấy đủ nội dung thì giữ metadata và nguyên nhân; không tạo tóm tắt chi tiết như đã đọc đầy đủ bài.
- Chỉ lưu tạm văn bản sạch theo thiết kế v1, không lưu raw HTML mặc định. Khi hết hạn và xử lý thành công, có thể xóa văn bản nhưng giữ metadata, hash, kết quả và trích dẫn theo chính sách lưu trữ.

### 8.2. Sự kiện, diễn biến và AI

- Gom bài nói về cùng sự việc thành một sự kiện. Một bài có thể đề cập nhiều sự kiện.
- Không tạo diễn biến mới chỉ vì có thêm bài cùng thông tin. Diễn biến thể hiện thay đổi có ý nghĩa.
- Mỗi diễn biến lưu dữ kiện và nguồn hỗ trợ, phản bác hoặc cung cấp bối cảnh.
- AI phân loại topic, tạo tiêu đề/tóm tắt/bối cảnh/ý nghĩa và tín hiệu đánh giá khẩn cấp.
- Tóm tắt bám dữ kiện nguồn, thể hiện sự không chắc chắn; không suy đoán kết luận khi các nguồn mâu thuẫn.
- Nội dung web là dữ liệu, không phải chỉ dẫn cho AI thực thi.
- Kết quả AI có cấu trúc và tham chiếu nguồn; phải kiểm tra trước khi sử dụng.
- Tái sử dụng tóm tắt theo diễn biến, ngôn ngữ và mức chi tiết giữa nhiều user; không bắt buộc gọi model riêng cho từng người.
- Khi sửa bản tóm tắt đã dùng, tạo phiên bản mới; khi sửa việc gom sự kiện, giữ dấu vết ánh xạ và lịch sử.
- Đánh giá xác minh gồm `unverified`, `corroborated`, `verified`, `disputed`; tiêu chí đạt từng mức và ngưỡng Critical chưa chốt. Điểm AI không phải bằng chứng xác minh.

## 9. Tạo và chọn nội dung bản tin

### 9.1. Luồng tạo bản tin

1. Nhận lần chạy đến hạn của cấu hình đủ điều kiện.
2. Tạo bản tin gắn cấu hình, người nhận, thời điểm lịch và snapshot cấu hình.
3. Chốt mốc dữ liệu đầu vào; chọn diễn biến và tóm tắt phù hợp.
4. Kiểm tra giới hạn tổng/topic, chống lặp và giữ chỗ các diễn biến được chọn.
5. Lưu thứ tự mục tin, nội dung thực tế và nguồn trích dẫn trong snapshot.
6. Tạo tiêu đề/tổng quan; render template email đang hoạt động và tạo delivery.
7. Đưa email đủ điều kiện vào luồng gửi; trạng thái tạo bản tin và trạng thái gửi email được theo dõi riêng.

Các phần nội dung được kiến trúc định hướng gồm tổng quan, tin nổi bật, lý do đáng quan tâm và diễn biến cần theo dõi. Chưa có yêu cầu mọi bản tin bắt buộc đủ cả bốn phần hoặc số mục cố định.

### 9.2. Quy tắc chọn tin

**Thiết kế v1:**

- Tin phải liên quan topic cấu hình, đủ điều kiện nội dung và có nguồn.
- Xếp hạng dựa trên độ mới, liên quan, ảnh hưởng và đa dạng topic; công thức và cách phân bổ quota chưa chốt.
- Không vượt trần tổng số tin hoặc hạn mức topic áp dụng.
- Một sự kiện chỉ xuất hiện một lần trong bản tin, kể cả có nhiều diễn biến phù hợp; thông thường chọn diễn biến mới nhất phù hợp.
- Không đủ tin mới thì gửi ít hơn; không bổ sung tin không phù hợp chỉ để đủ số lượng.
- **Đề xuất v1:** không có tin phù hợp thì `skipped`, ghi lý do và không gửi email rỗng.
- Bản tin `ready` phải có tiêu đề, thời điểm tạo xong và ít nhất một mục.
- Snapshot lưu nội dung, ngôn ngữ, mức chi tiết, topic và nguồn thực tế. Đổi cấu hình, nguồn hoặc tóm tắt sau đó không làm đổi bản đã phát hành.

Trạng thái tạo bản tin: `pending` → `building` → `ready`; có thể kết thúc bằng `skipped`, `failed` hoặc `cancelled` theo kết quả. `ready` chỉ nghĩa nội dung sẵn sàng, chưa chứng minh email đã gửi.

## 10. Cảnh báo tin khẩn cấp

**Đã thống nhất:** hỗ trợ cảnh báo cho sự kiện khẩn cấp liên quan topic người dùng. Kiến trúc hiện giới hạn MVP ở mức Critical.

### 10.1. Luồng đánh giá

1. Phát hiện sự kiện hoặc diễn biến mới đáng kể.
2. AI cung cấp đánh giá ảnh hưởng, khẩn cấp, độ mới và tín hiệu xác minh kèm bằng chứng.
3. Hệ thống áp dụng chính sách Critical và yêu cầu xác minh tối thiểu.
4. Xác định người nhận phù hợp; kiểm tra quyền nhận, tùy chọn cảnh báo, hạn mức, khoảng cách gửi và giờ yên lặng.
5. Kiểm tra chống lặp; lưu nội dung/chính sách/topic khớp tại thời điểm xét.
6. Khi đủ điều kiện, tạo email và dùng chung quy trình phát hành, đối soát như bản tin.

### 10.2. Quy tắc cảnh báo

- **Thiết kế v1:** một user chỉ có một cảnh báo cho một diễn biến, dù khớp nhiều topic hoặc topic nằm ở nhiều khung giờ.
- Diễn biến tiếp theo của sự kiện cũ chỉ được cảnh báo khi thay đổi đáng kể và đạt lại điều kiện; thêm URL không đủ để gửi lại.
- **Đề xuất v1:** chỉ đủ điều kiện nếu topic còn trong ít nhất một cấu hình bản tin đang hoạt động, cảnh báo toàn cục và cảnh báo topic đều bật.
- Cấu hình cảnh báo độc lập với giờ gửi bản tin; có hạn mức mỗi ngày, khoảng cách tối thiểu giữa lần gửi và giờ yên lặng theo timezone user.
- Giờ yên lặng có thể đi qua nửa đêm. Không mặc định Critical được bỏ qua giờ yên lặng hoặc không bị giới hạn.
- Nếu bị giữ lại, phải kiểm tra tính thời sự trước khi gửi; không gửi dồn cảnh báo cũ.
- Trạng thái nội dung cảnh báo gồm `pending`, `ready`, `cancelled`, `expired`; kết quả gửi nằm ở delivery.

**Cần chốt:** ngưỡng Critical, tiêu chí xác minh, độ trễ mục tiêu, mặc định bật/tắt, hạn mức, khoảng cách gửi, thời hạn cảnh báo, quyền bỏ qua giờ yên lặng và cách xử lý tin bị chặn. Gửi email ngoài lịch là đề xuất v1 chưa được xác nhận cuối cùng trong tài liệu dữ liệu mới nhất.

## 11. Chống lặp giữa bản tin và cảnh báo

**Đề xuất v1:** cùng user không nhận lại cùng diễn biến qua nhiều khung giờ hoặc qua cả bản tin và cảnh báo. Diễn biến mới đáng kể của cùng sự kiện vẫn có thể được chọn.

**Thiết kế v1 để hỗ trợ đề xuất này:**

- Dùng sổ giữ chỗ theo user và diễn biến chung cho hai luồng; xử lý đồng thời phải giữ chỗ nguyên tử trước khi phát hành.
- `reserved`: một delivery đang giữ quyền gửi; luồng khác không chọn lại diễn biến đó.
- `submitted`: provider đã chấp nhận email; không giải phóng chỉ vì lease hết hạn.
- `released`: đã xác định chưa gửi và cho phép luồng khác giữ chỗ lại.
- Không dùng trạng thái “đã đọc” làm căn cứ chống lặp.
- **Đề xuất v1:** ưu tiên cảnh báo khi bản tin chưa gửi và còn có thể loại mục đó an toàn; cập nhật giữ chỗ và email chờ đồng bộ. Không sửa snapshot đã phát hành.
- Nếu kết quả gửi chưa xác định, đối soát trước khi giải phóng hoặc gửi lại.

**Cần chốt:** có nhắc lại tin đã cảnh báo trong bản tổng kết không. Nếu cho phép, phải bổ sung chính sách/mục đích gửi để không vô hiệu hóa cơ chế chống lặp hiện tại.

## 12. Template, gửi email và đối soát

### 12.1. Tạo email

**Thiết kế v1 mới nhất:** template được quản lý theo loại và ngôn ngữ, gồm `newsletter`, `breaking_alert`, `verify_email`, `manage_preferences`; mỗi cặp loại/ngôn ngữ có một template. Template có trạng thái hoạt động và được chỉnh sửa trực tiếp, không quản lý version template trong v1.

- Chọn template đang hoạt động phù hợp mục đích và ngôn ngữ.
- Kiểm tra dữ liệu, escape nội dung động, không đưa raw HTML không tin cậy từ web/AI vào email.
- Lưu người nhận, tiêu đề, HTML và bản text đã render cho từng delivery.
- Sửa template không làm đổi lịch sử hoặc email retry đã tạo. Phiên bản tóm tắt AI vẫn được quản lý riêng.

### 12.2. Delivery và lần thử gửi

- Một bản tin/cảnh báo hoặc email truy cập tài khoản có một delivery logic; retry dùng lại delivery đó và nội dung đã render.
- Mỗi lần hệ thống gọi provider là một attempt, có kết quả và lỗi riêng. Attempt không phải lượt mở email hoặc lần provider tự thử giao thư.
- `submitted`: provider chấp nhận yêu cầu gửi.
- `delivered`: provider báo giao thành công; không chứng minh nằm ở inbox hoặc đã được đọc.
- Trạng thái khác gồm `queued`, `sending`, `deferred`, `failed`, `bounced`, `complained`, `cancelled`, `unknown`.
- Lỗi tạm thời được retry có giới hạn/backoff. Timeout sau khi gọi provider có thể là `unknown`; không retry mù.
- Dùng idempotency provider nếu có hoặc đối soát kết quả trước khi thử lại. Khóa chống trùng nội bộ không tự bảo đảm hệ thống bên ngoài gửi đúng một lần.

### 12.3. Phản hồi provider

- Lưu webhook và chống xử lý lặp; hỗ trợ sự kiện đến sai thứ tự hoặc chưa tìm thấy delivery tương ứng.
- Không để sự kiện cũ làm mất kết quả bounce/complaint mới hơn.
- Cập nhật suppression từ phản hồi phù hợp để chặn các lần gửi sau; chính sách phân loại bounce cần cụ thể hóa theo provider.
- Theo dõi provider message ID, lịch sử attempt, lỗi và thời điểm đối soát.

## 13. Lỗi, lịch sử và vận hành

| Tình huống | Hành vi yêu cầu/định hướng |
| --- | --- |
| Nguồn lỗi hoặc chặn truy cập | Lưu lỗi thu thập, giữ tiến độ; không giả lập nội dung hoặc vượt quyền truy cập. |
| AI lỗi hoặc model không hoạt động | Lưu lỗi/job và retry có giới hạn; lịch gửi tách khỏi tốc độ model. Chính sách gửi bản thiếu tin cần chốt. |
| Thiếu dữ liệu để tóm tắt | Giữ metadata/lý do; không khẳng định đã đọc toàn văn. |
| Không có tin phù hợp | Đề xuất `skipped`, không gửi email rỗng. |
| Worker dừng giữa chừng | Có cơ chế phục hồi công việc; không tạo trùng lần chạy hoặc gửi lại tùy tiện. |
| Người dùng hủy đăng ký khi email đang chờ | Kiểm tra lại quyền nhận và chặn email bản tin/cảnh báo chưa gửi. |
| Provider timeout, chưa rõ đã nhận email | Đối soát; giữ trạng thái chưa xác định thay vì mặc định thất bại rồi gửi lại. |
| Sửa cấu hình/template/tóm tắt | Áp dụng thay đổi theo luồng tương ứng, giữ nguyên lịch sử đã phát hành; retry dùng email đã render. |

**Thiết kế v1:** lưu riêng trạng thái thu thập, kết quả AI, tạo bản tin, cảnh báo, delivery và attempt để truy vết đúng công đoạn lỗi. Có thể truy ngược nội dung đã gửi tới diễn biến, bản tóm tắt và nguồn được sử dụng.

Theo dõi tối thiểu: lỗi nguồn, lỗi AI, job/lịch quá hạn, độ trễ phát hiện–đánh giá–gửi, email lỗi, bounce/complaint và cảnh báo bị giới hạn. Chỉ tiêu định lượng về độ trễ, tỷ lệ thành công, quy mô và ngân sách chưa được chốt.

Thời hạn giữ bài thu thập, bản tin, email render, token, webhook và job phải có chính sách riêng. Không mặc định giữ vô hạn. Xóa dữ liệu cá nhân phải xử lý cả snapshot và payload có thông tin cá nhân. Quyền đọc token, email render và dữ liệu provider phải được hạn chế; không ghi secret/token vào log.

## 14. Các cập nhật thay thế mô tả cũ

| Mô tả trước | Cách hiểu trong bản tổng hợp này |
| --- | --- |
| Một bản tin khoảng 08:00 mỗi ngày | Có nhiều cấu hình theo khung giờ; 08:00 là ví dụ/định hướng ban đầu. |
| Chưa rõ dùng giờ Việt Nam hay từng user | Yêu cầu mới dùng giờ địa phương và timezone của user. |
| Độ dài cố định 5–10 phút, tùy chỉnh sau MVP | Yêu cầu mới có mức chi tiết và giới hạn riêng từng khung giờ; trần sản phẩm cụ thể còn mở. |
| Email template quản lý version | Database v1 mới nhất dùng template chỉnh trực tiếp và lưu email đã render để giữ lịch sử/retry. |
| Bắt buộc đăng nhập tài khoản theo cách truyền thống | Database v1 chưa mặc định mật khẩu/social login; có thiết kế token quản lý email, luồng cuối cùng chưa chốt. |
| Tin khẩn cấp gửi ngay ngoài lịch | Đây là định hướng kiến trúc; database mới nhất vẫn đánh dấu cách gửi ngoài lịch và các chính sách hạn chế là đề xuất cần chốt. |

## 15. Tiêu chí nghiệm thu nghiệp vụ dự kiến

Các tiêu chí theo **thiết kế/đề xuất v1** dưới đây dùng để rà soát sau khi xác nhận các chính sách còn mở; không phải báo cáo kiểm thử đã chạy.

| Mã | Tình huống | Kết quả cần kiểm chứng |
| --- | --- | --- |
| AC-01 | Email chưa xác minh hoặc user không đủ quyền nhận | Không gửi bản tin/cảnh báo. |
| AC-02 | User tạo cấu hình sáng và tối khác nhau | Mỗi lần tạo nội dung tuân theo topic, giờ và mức chi tiết của cấu hình tương ứng. |
| AC-03 | Tổng quota topic là 6 nhưng trần bản tin là 5 | Bản tin không vượt 5; không topic nào vượt hạn mức được áp dụng. |
| AC-04 | Một sự kiện khớp nhiều topic hoặc có nhiều diễn biến ứng viên | Sự kiện chỉ xuất hiện một mục trong bản tin. |
| AC-05 | Scheduler xử lý lại cùng cấu hình và thời điểm lịch | Không tạo thêm một lần chạy cho cùng lịch. |
| AC-06 | Đổi cấu hình sau khi đã phát hành | Nội dung và nguồn của bản tin lịch sử giữ nguyên. |
| AC-07 | Cùng diễn biến được chọn ở hai khung giờ hoặc đồng thời với cảnh báo | Reservation/chính sách chống lặp ngăn phát hành trùng theo đề xuất v1. |
| AC-08 | Sự kiện cũ có diễn biến mới đáng kể | Có thể được chọn hoặc cảnh báo nếu đạt lại điều kiện; thêm bài giống nhau không đủ. |
| AC-09 | Cảnh báo khớp nhiều topic cùng user | Chỉ một cảnh báo cho diễn biến đó. |
| AC-10 | Chưa đạt ngưỡng xác minh hoặc Critical | Không gửi cảnh báo chỉ vì điểm AI cao. |
| AC-11 | Hủy đăng ký khi email chưa gửi | Worker chặn cả bản tin và cảnh báo đang chờ. |
| AC-12 | Lỗi tạm thời rồi retry, template đã bị sửa | Giữ một delivery, nhiều attempt; nội dung retry giống snapshot email đã tạo. |
| AC-13 | Provider timeout sau khi gửi yêu cầu | Không tự động gửi lại trước khi áp dụng idempotency/đối soát. |
| AC-14 | Webhook trùng hoặc đến sai thứ tự | Không xử lý lặp hoặc làm mất trạng thái suppression phù hợp. |
| AC-15 | Không lấy được toàn văn | Không tạo tóm tắt chi tiết như đã có toàn văn; giữ nguyên nhân và nguồn. |
| AC-16 | Không có tin phù hợp | Theo đề xuất v1, ghi `skipped` có lý do và không gửi email rỗng. |
| AC-17 | User truy cập cấu hình/lịch sử của người khác | Từ chối truy cập. |
| AC-18 | Giờ gửi gặp DST hoặc hệ thống phục hồi sau gián đoạn | Tuân theo chính sách DST/lịch quá hạn sau khi được chốt; không tạo lần gửi trùng. |

## 16. Các quyết định cần xác nhận

1. Luồng đăng ký/quản lý email, thời hạn token, cách hủy đăng ký và điều kiện kích hoạt lại sau tạm dừng/suppression.
2. Topic đề xuất được tự kích hoạt hay duyệt; tự thêm vào cấu hình nào; xử lý topic bị dừng hoặc lưu trữ.
3. Số cấu hình tối đa, số topic tối đa, giới hạn số tin và phạm vi mức chi tiết MVP.
4. Hạn mức topic tính theo từng bản tin hay cả ngày; cách phân bổ khi tổng quota vượt trần.
5. Chính sách không có tin, thiếu tin, AI chưa xử lý xong, nguồn lỗi hoặc thiếu template/ngôn ngữ phù hợp.
6. Cửa sổ độ mới, mốc chốt dữ liệu và công thức xếp hạng/phân bổ topic.
7. Xác nhận chính sách chống gửi lại diễn biến giữa các khung giờ; có tổng kết lại tin đã cảnh báo không.
8. Ngưỡng Critical, xác minh tối thiểu, nguồn được chấp nhận, độ trễ mục tiêu và cách sửa/đính chính tin đã gửi.
9. Có gửi cảnh báo ngoài lịch; mặc định bật/tắt; hạn mức, khoảng cách gửi, giờ yên lặng, quyền bỏ qua và xử lý cảnh báo hết thời sự.
10. Xác nhận quy tắc DST, lịch quá hạn, đổi cấu hình khi bản tin đang chờ và nhiều cấu hình cùng giờ.
11. Ngôn ngữ hỗ trợ, phạm vi nguồn tin, thời hạn lưu và quy trình xóa dữ liệu cá nhân.
12. Email provider, đối soát kết quả không xác định, giới hạn retry và chính sách bounce/complaint.
13. Quy mô phục vụ, ngân sách và chỉ tiêu chất lượng/độ trễ để nghiệm thu vận hành.

Các quyết định này cần được cập nhật đồng bộ vào tài liệu sản phẩm, kiến trúc và database sau khi xác nhận. Tài liệu này không tự gán giá trị mặc định cho các giới hạn sản phẩm còn mở.
