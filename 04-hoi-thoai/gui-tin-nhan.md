# 09. User bấm Gửi tin nhắn

[← Mục lục](../README.md) · [Quản lý hội thoại](quan-ly-hoi-thoai.md) · [9Router](../05-9router/tao-request.md)

Đây là luồng trọn vẹn từ nút **Gửi** đến lúc câu trả lời hiện trên màn hình. Phần request BE ↔ 9Router được tách kỹ hơn ở thư mục `05-9router`.

```text
ChatView → App.sendMessage → services/api.streamChat
        → POST /api/chat → authn → Server.send → Chat.SendStream
        → Mongo + 9Router → NDJSON delta/done → FE cập nhật câu trả lời
```

1. [`Composer`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/components/ChatView.js) gọi `onSend`. [`App.sendMessage`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/App.js) lấy text và model đang chọn, xóa ô nhập, thêm user message và assistant placeholder **tạm trong React state**, tạo `AbortController` để có thể bấm Dừng.
2. Nếu chưa có `currentId`, FE gọi `POST /api/conversations`, lấy ID mới. Việc bấm “Đoạn chat mới” trước đó chưa tạo Mongo document.
3. [`streamChat`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/services/api.js) gửi `POST /api/chat` với `{ "conversation_id": "...", "content": "...", "model": "..." }`, Bearer access token và abort signal. Nginx production bỏ `/backend` trước khi Go nhận request.
4. BE [`authn`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/server.go) xác thực user, lấy Mongo ID, đồng bộ role nếu bật. [`Server.send`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/server.go) kiểm tra JSON và text không rỗng, tạo response `application/x-ndjson`.
5. [`Chat.SendStream`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/chat.go) xác nhận conversation thuộc user, giới hạn text tối đa 20.000 rune, gọi `policy` để kiểm tra model thuộc plan. Nếu model sai hoặc credit không đủ, dừng trước khi gọi 9Router.
6. `buildContext` lấy tin chưa tóm tắt và summary cũ để tạo context. Sau đó BE reserve credit, insert user message vào `messages`, cập nhật `conversations.updated_at`; nếu title vẫn mặc định thì đặt title từ 30 rune đầu câu hỏi.
7. BE gọi [`ai.Client.Stream`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/ai/client.go). 9Router stream các phần câu trả lời về BE; `Server.send` đổi mỗi phần thành một dòng `{"type":"delta","content":"..."}` và flush cho FE.
8. `streamChat` đọc từng dòng, nối `delta` vào assistant placeholder. Khi BE nhận đủ usage/hoàn tất, nó quyết toán credit, lưu assistant message, gửi `{"type":"done","message":{...}}`. FE thay placeholder bằng message thật, tải lại sidebar và entitlement.

**Dừng hoặc lỗi:** `AbortController` hủy request ở FE. BE có thể đã lưu user message trước khi AI ngắt; nếu có phần trả lời, BE có thể lưu phần đó và ghi usage `interrupted`. Nếu không có nội dung/usage, phần credit tạm giữ được hoàn. Đừng coi một lượt gửi là một Mongo transaction duy nhất.

Hiện `/api/chat` chỉ nhận **text**. Chưa có upload ảnh hay `content` đa phương thức trong luồng này.
