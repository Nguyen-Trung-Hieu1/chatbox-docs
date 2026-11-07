# 14. Tạo API key và gọi Chatbox từ IDE

[← Mục lục](../README.md) · [Credit](../05-9router/usage-va-credit.md)

API key `cb_...` là key **của Chatbox**, không phải Keycloak access token và cũng không phải `AI_API_KEY` của 9Router. Nó cho IDE gọi `/v1/*` dưới quyền một Mongo user; BE vẫn kiểm tra plan/credit của owner.

## User bấm Tạo key trên FE

1. User đã đăng nhập mở **API keys**. [`APIKeysPanel`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/components/APIKeysPanel.js) gọi `GET /api/keys`; BE đi qua `authn` và trả danh sách key của Mongo user hiện tại (tên, prefix, thời gian, trạng thái), không trả raw key.
2. User nhập tên và bấm **Tạo key**. FE gửi `POST /api/keys` với `{ "name": "VS Code" }`.
3. [`manageKeys`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/openai.go) lấy owner từ request context và gọi [`APIKeys.Create`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/api_keys.go). Use case kiểm tra tên 1–80 rune, tạo 32 byte ngẫu nhiên, mã hóa thành raw key bắt đầu `cb_`, lấy SHA-256 hash.
4. [`APIKeyRepository.Create`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/persistence/api_keys.go) chỉ lưu hash, prefix, tên, `user_id`, thời gian vào Mongo. BE trả raw key **một lần** cùng metadata, đặt `Cache-Control: no-store`. FE cho user sao chép. Không thể đọc lại raw key từ Mongo.

Khi user bấm **Thu hồi**, FE xác nhận rồi gửi `DELETE /api/keys?id=<key ID>`. Backend chỉ revoke key thuộc owner; key đó không xác thực request mới được nữa.

## IDE gửi một request

Ví dụ IDE gọi `POST /backend/v1/chat/completions` trên domain production với `Authorization: Bearer cb_...` và JSON gồm `model`, `messages`, có thể có `stream`.

1. Nginx bỏ `/backend`; [`apiAuth`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/openai.go) nhận raw key, gọi [`APIKeys.Authenticate`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/api_keys.go). BE hash key, tìm bản ghi chưa revoke, lấy `user_id`, tìm Mongo user và cập nhật thời gian dùng.
2. Nếu bật Keycloak roles, BE đọc role của `user.KeycloakSub` rồi đồng bộ plan. API key **không né** quyền Free/Paid.
3. [`openAIChat`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/openai.go) giải mã và kiểm tra request. Nó chỉ chấp nhận **text** ở `messages`, role `system`, `developer`, `user`, `assistant`; từ chối tool calls, content mảng/ảnh và nhiều choices.
4. [`Chat.APIGenerate`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/api_chat.go) kiểm tra model trong plan, giới hạn text/max output, reserve credit, gọi [`ai.Client.Stream` hoặc `Complete`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/ai/client.go), quyết toán usage với `kind=api`.
5. Nếu `stream=true`, BE trả SSE kiểu Chat Completions (`data: ...`, cuối cùng `[DONE]`). Nếu không, trả JSON `chat.completion`. IDE dùng response đó để hiển thị. Luồng này **không tạo** `conversations` hay `messages` trong Chatbox; chỉ ghi usage vào ví.

`GET /v1/models` trả model được phép theo plan của owner. `POST /v1/completions` nhận text prompt cho Continue Edit, đóng prompt thành một user message trước khi gọi gateway; dùng cùng API key, plan và ví. Base URL production được panel hiển thị ở dạng `https://chatbox.cyou/backend/v1`.

Nếu model không thuộc gói thì trả 403 trước khi gọi AI; nếu credit không đủ thì trả 402; key sai/đã revoke thì 401. Các lỗi `/v1/*` dùng dạng `{"error":{"message":"...","type":"...","code":"..."}}`.
