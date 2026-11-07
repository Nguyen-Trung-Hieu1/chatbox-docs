# 08. Tạo, mở, đổi tên và xóa cuộc trò chuyện

[← Mục lục](../README.md) · [Bấm Gửi](gui-tin-nhan.md) · [MongoDB](luu-tru-mongodb.md)

Các thao tác này đều bắt đầu ở [`App.js`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/App.js), đi qua [`Server.Handler`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/server.go), rồi xuống [`Chat`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/chat.go) và Mongo repository. Trước khi handler chạy, `authn` đã lấy Mongo user ID từ access token.

## Tạo cuộc trò chuyện

Khi user bấm **Đoạn chat mới**, FE chỉ xóa lựa chọn hiện tại (`currentId`), tin nhắn đang hiển thị và lỗi trong state. **Chưa có document Mongo mới ngay lúc bấm.** Khi user gửi tin đầu tiên, `App.sendMessage` thấy `currentId` rỗng thì gọi `POST /api/conversations` với `{}`. BE `conversations` → `Chat.CreateConversation` tạo `title="Cuộc trò chuyện mới"`, `owner_id=<Mongo user ID>`, `created_at`, `updated_at`; repository insert vào collection `conversations` và trả ID. FE dùng ID này để gọi `/api/chat`.

## Danh sách ở sidebar

Sau khi login/gửi tin, `App.fetchConversations` gọi `GET /api/conversations`. `Chat.List` → [`ConversationRepository.ListOwned`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/persistence/repositories.go) query `owner_id` của user hiện tại, sort `updated_at` giảm dần. FE đặt mảng JSON vào sidebar. Không query tất cả conversation của mọi người rồi lọc ở FE.

## Mở một cuộc trò chuyện

User bấm một mục trong sidebar, `App.selectConversation` đặt ID đang xem, gọi `GET /api/history?id=<conversation ID>`. [`Chat.History`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/chat.go) gọi `owned`: repository tìm conversation bằng **cả** `_id` và `owner_id`. Nếu không thuộc user thì trả lỗi. Nếu đúng, [`MessageRepository.List`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/persistence/repositories.go) lấy messages theo `conversation_id`, sắp xếp `created_at` rồi `_id` tăng dần. FE hiện lịch sử.

## Đổi tên

`App.renameConversation` gửi `PATCH /api/conversations?id=<ID>` với `{ "title": "..." }`. `Chat.Rename` xác nhận owner, trim tiêu đề, giới hạn 1–100 rune, cập nhật `title` và `updated_at` trong Mongo. BE trả conversation đã đổi; FE thay đúng item trong sidebar.

## Xóa

`App.deleteConversation` gửi `DELETE /api/conversations?id=<ID>`. `Chat.Delete` xác nhận owner, gọi `MessageRepository.DeleteByConversation` xóa các message của cuộc trò chuyện, rồi `ConversationRepository.Delete` xóa document conversation. BE trả 204; FE bỏ item khỏi sidebar và xóa state nếu đang mở nó. Hai lệnh xóa chạy tuần tự, **không nằm trong một Mongo transaction**.

Nếu ID sai định dạng, không tồn tại hoặc thuộc người khác, `owned` chặn trước khi đọc/sửa/xóa messages. Các endpoint này không nhận một `owner_id` do FE tự khai báo; owner lấy từ user đã xác thực.
