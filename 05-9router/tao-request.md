# 11. Backend tạo request gửi 9Router ra sao?

[← Mục lục](../README.md) · [Bấm Gửi](../04-hoi-thoai/gui-tin-nhan.md) · [Stream](xu-ly-stream.md)

Browser không gọi 9Router. Nó chỉ gọi BE `/api/chat` hoặc `/v1/*`. Sau khi BE xác thực user, kiểm tra model và reserve credit, BE mới gọi AI gateway ở `AI_ENDPOINT`. Trong deployment hiện dùng 9Router; nếu đổi `AI_ENDPOINT` thì cùng client Go có thể gọi gateway tương thích khác.

## Request Chat Completions

[`Chat.SendStream`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/chat.go) chuẩn bị `contextMessages` bằng `buildContext`, gồm system prompt, summary cũ nếu có, các message gần đây và câu hỏi mới. Model ID lấy từ policy đã qua kiểm tra plan, ví dụ `ag/...` hoặc `cx/...`; không lấy một model bất kỳ do browser tự ý gửi.

[`ai.Client.newRequest`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/ai/client.go) serialize JSON dạng:

```json
{
  "stream": true,
  "model": "ag/gemini-3-flash",
  "messages": [
    { "role": "system", "content": "..." },
    { "role": "user", "content": "Câu hỏi của người dùng" }
  ],
  "max_completion_tokens": 1024,
  "stream_options": { "include_usage": true }
}
```

Đây là **ví dụ cấu trúc**, system prompt/context và giới hạn token thực tế phụ thuộc request. Client gửi `POST <AI_ENDPOINT>`, `Content-Type: application/json`; nếu `AI_API_KEY` có giá trị thì thêm `Authorization: Bearer <AI_API_KEY>`. Secret này ở BE, FE không biết.

## Context dài và request phụ

[`Chat.buildContext`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/chat.go) chỉ đưa những tin chưa summarized gần đây vào context theo `AI_TOKEN_BUDGET`. Khi tin cũ quá dài, nó có thể tạo **một request AI phụ** bằng [`ai.Client.Complete`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/ai/client.go) để tóm tắt đoạn cũ, lưu rolling summary ở Mongo rồi mới gửi request trả lời chính. Request tóm tắt cũng dùng credit và có usage riêng `kind=summary`. Vì vậy một lần bấm Gửi có thể tạo **hơn một request tới 9Router**.

## Request từ IDE

[`Chat.APIGenerate`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/api_chat.go) cũng đi qua `ai.Client.Stream` hoặc `Complete`. Nó dùng model policy/credit giống web, nhưng không lưu hội thoại Chatbox. `/v1/completions` bọc prompt text thành một user message trước khi gửi gateway. Xem [API key/IDE](../06-api-key/goi-tu-ide.md).

Hiện `domain.AIMessage.Content` là string. Client này chưa tạo content dạng `image_url`, nên 9Router chưa nhận ảnh từ Chatbox dù model phía sau có thể hỗ trợ ảnh.
