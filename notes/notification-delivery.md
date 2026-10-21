# Hợp đồng Mobile cảnh báo người thân — 3.21

Tài liệu này bổ sung hành vi nhận push cho các operation trong `openapi.json`; đây là thiết kế chờ duyệt, chưa chứng minh backend/Mobile đã triển khai.

## HTTP và quyền

| Method / path | Request | Thành công | Lỗi đặc thù |
|---|---|---|---|
| `PUT /me/device-tokens` | `fcm_token`, `platform=android`, tùy chọn `previous_fcm_token` | 204 | 409 `device_token_in_use` |
| `DELETE /me/device-tokens` | JSON body `fcm_token` | 204, cả khi không có token của user | Không 404/409 cho token |
| `GET /notifications` | `is_read?`, `limit=20` (1–100), `offset=0` (≥0) | 200 `NotificationList` | 403 admin; 503 lỗi PEP/audit |
| `GET /notifications/{id}` | `id` nguyên ≥1 | 200 `NotificationItem` | 403 mất quyền; 404 ID không thuộc hộp thư; 503 lỗi PEP/audit |
| `PATCH /notifications/{id}` | `is_read` boolean | 200 `NotificationItem` | Như GET chi tiết |

Mọi route cần app bearer, thiếu/sai trả 401; request không hợp lệ trả 422 theo schema chung. Chi tiết schema, ví dụ, mã lỗi và quy tắc idempotency nằm trong OpenAPI. Endpoint không có điều kiện xung đột thì không thêm mã 409 giả.

Hộp thư lọc quyền trước phân trang và tính số đếm, ghi `denied` cho patient bị loại. Chỉ lọc lỗi quyền sau khi ghi audit thành công; lỗi DB/audit làm toàn request thất bại. Chi tiết và PATCH kiểm quyền lại ở mỗi request. PATCH dùng PEP `ADHERENCE / READ` vì chỉ thay metadata hộp thư người nhận, không ghi liều/patient. Truy cập dữ liệu sức khỏe thành công chỉ trả sau khi commit audit.

## Payload push đang tích hợp

FCM adapter hiện hữu cung cấp `message.notification.title`, `message.notification.body` và `message.data.notification_id`. Android nhận phần data như sau (ví dụ giả):

```json
{"notification_id": "501"}
```

`notification_id` bắt buộc là chuỗi chữ số biểu diễn số nguyên dương, trong miền ID bigint có dấu (1–9223372036854775807). Không nhận float, số âm, 0, chuỗi rỗng, thiếu field hoặc số ngoài miền. Field lạ không được dùng để quyết định patient/quyền. Không bổ sung `patient_id`, thuốc, giờ hoặc chi tiết liều vào push. Payload lỗi không mở cảnh báo/patient suy đoán; UI thông báo không khả dụng và cho mở hộp thư qua API.

Tap hoặc mở từ foreground/background/terminated dùng ID để gọi `GET /notifications/{id}` bằng session hiện hành; chờ response hợp lệ rồi hiển thị patient từ `about_patient_user_id`. Không dùng push làm nguồn dữ liệu sức khỏe, không tự đánh dấu đọc bằng GET. Gặp 403/404 phải bỏ nội dung đã giữ cho cảnh báo đó; không chuyển sang patient khác. Chỉ hiển thị trạng thái đã đọc sau PATCH thành công, rồi tải lại list để lấy badge từ server.

Phần notification OS dùng nội dung đã được duyệt của luồng dispatch 3.20; không mang thuốc, giờ hoặc chi tiết liều. Khi ẩn xem trước, chỉ hiện nội dung chung; chế độ hiển thị phải tôn trọng cài đặt thiết bị, kể cả thông báo foreground do Mobile dựng. Task này không thay việc phát hiện/dispatch. iOS không đăng ký token ở API Android; hiện “Thông báo đẩy chưa được hỗ trợ”, vẫn dùng hộp thư HTTP.

## Dữ liệu cũ và tương thích

Giữ path, bộ lọc `limit/offset`, `items/total/unread_count`, `is_read` và các field item hiện có. `medication_name` và `scheduled_at` là field response bổ sung; dữ liệu lấy từ sự kiện gốc. Notification mới sau triển khai phải có thuốc và giờ khác null. Khi không khôi phục được sự kiện của notification cũ, trả null và UI hiện “Không có dữ liệu liều cũ”, không dùng `created_at` hoặc lịch đã chỉnh sửa để bịa giờ/thuốc. Plan phải chốt migration/backfill và kiểm dữ liệu mới.

Lọc theo PEP là thay đổi hành vi an ninh so với backend chưa kiểm PEP: list có thể loại item/giảm số đếm; detail/PATCH có thể 403 hoặc 503. Không tìm thấy client HTTP notification/token trong `frontend/mobile/lib` khi khảo sát phiên này. Không đổi client gọi API khác. Client tương lai phải xử lý các lỗi này theo OpenAPI.

## Logout, token rotation và giới hạn

PUT gửi lại cùng user/token không trùng; không chuyển token sang user khác. Refresh có thể thay token cũ của chính user bằng `previous_fcm_token` trong cùng transaction. Không gỡ token cũ nếu đăng ký mới thất bại.

Gỡ token bằng session cũ trước khi hoàn tất logout/đổi tài khoản. Khi offline phải retry; chưa được báo đã gỡ thành công. Nếu session hết hạn, client phải hủy token FCM cũ và tạo token mới trước đăng ký với tài khoản mới. Xóa token ở user mới không được xóa token user cũ.

DELETE không thu hồi thông báo đã được FCM nhận trước đó; server chỉ bảo đảm token không được chọn cho dispatch mới sau khi gỡ. Plan/test phải xử lý thao tác logout, token rotation, push đang bay và callback cũ để bảo đảm AC8; API GET chi tiết vẫn kiểm người nhận và quyền bằng session hiện hành. Không cam kết đã giải quyết hành vi thiết bị chỉ bằng contract HTTP.

## Việc cần kiểm khi triển khai

Đăng ký ba operation notification vào registry kiểm enforcement CI. Repo hiện có registry riêng cho family links, chưa có registry notification. Kiểm cross-patient, rút consent/liên kết/chia sẻ, audit denied transaction riêng và fail-closed, không trùng token, chạy đua đăng ký và refresh/logout. Không log token hoặc dữ liệu sức khỏe. Mobile không lưu inbox vào bộ nhớ bền; xóa dữ liệu phiên khi vào nền/mất mạng/logout/đổi user/bị từ chối quyền.
