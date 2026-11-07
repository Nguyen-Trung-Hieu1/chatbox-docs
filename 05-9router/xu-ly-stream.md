# 12. Stream từ 9Router về FE đi thế nào?

[← Mục lục](../README.md) · [Tạo request](tao-request.md) · [Usage](usage-va-credit.md)

Ở đây có **hai kiểu stream nối nhau**: 9Router → BE là SSE kiểu Chat Completions; BE → FE web là NDJSON. Đừng nhầm rằng browser đang đọc trực tiếp SSE của 9Router.

1. [`ai.Client.Stream`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/ai/client.go) gửi request `stream=true`, lấy HTTP response của gateway. Nếu status không phải 2xx, client đọc tối đa một phần body lỗi và trả lỗi cho use case.
2. Client đọc từng dòng gateway. Nó chỉ xử lý dòng bắt đầu `data:`; bỏ `[DONE]`. Với JSON chunk, nó lấy `choices[0].delta.content` để nối câu trả lời, và lấy usage nếu gateway gửi.
3. Mỗi `delta` được đưa vào callback `emit` do [`Server.send`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/server.go) tạo. Callback encode một dòng JSON `{"type":"delta","content":"..."}` rồi `Flush()` tới browser. HTTP response BE đặt `Content-Type: application/x-ndjson`.
4. [`streamChat`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/services/api.js) đọc `response.body`, tách theo newline, parse JSON. `delta` được nối vào assistant placeholder trong [`App.sendMessage`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/App.js), nên chữ xuất hiện dần.
5. Khi gateway kết thúc, use case quyết toán usage, lưu assistant message ở Mongo. Handler gửi dòng cuối `{"type":"done","message":{...}}`. FE dùng message thật thay placeholder. Nếu handler gửi `{"type":"error","error":"..."}`, FE hiện lỗi.

**Dừng tạo câu trả lời:** nút Dừng gọi `AbortController.abort()` ở [`App.js`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/App.js). Request BE có thể bị hủy sau khi đã lưu user message. Nếu đã có một phần nội dung, BE cố lưu phần trả lời và usage với trạng thái `interrupted`; nếu không có nội dung/usage thì giải phóng phần credit tạm giữ.

**API IDE khác web:** `/v1/chat/completions` do [`openAIChat`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/openai.go) xử lý, trả SSE kiểu OpenAI cho IDE, không đổi sang NDJSON. Cả hai dùng cùng AI client để gọi 9Router.
