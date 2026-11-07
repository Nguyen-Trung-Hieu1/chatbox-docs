# 10. MongoDB lưu những gì?

[← Mục lục](../README.md) · [Quản lý hội thoại](quan-ly-hoi-thoai.md)

[`NewMongo`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/persistence/mongo.go) mở sáu collection chính. Dưới đây là trường BE đang dùng; đây là quan hệ logic bằng ID trong code, không phải foreign key như SQL.

| Collection | ID/liên kết | Dữ liệu chính |
| --- | --- | --- |
| `users` | `_id` ObjectID, `keycloak_sub` string tùy user | Username nội bộ, profile, hash password legacy, cờ admin legacy. |
| `conversations` | `_id`, `owner_id → users._id` | Title, rolling summary, created/updated time. |
| `messages` | `_id`, `conversation_id → conversations._id` | `role` (`user`/`assistant`), `content` text, cờ `summarized`, thời gian. |
| `wallets` | `_id = users._id` | Plan đồng bộ, balance, mốc reset, usage gần nhất. |
| `api_keys` | `user_id → users._id` | Hash SHA-256 của key, prefix, tên, revoke/last used. |
| `sessions` | `user_id → users._id` | Phiên legacy; luồng Keycloak mới không ghi access/refresh token tại đây. |

Struct map collection nằm trong [`persistence/repositories.go`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/persistence/repositories.go), [`billing.go`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/persistence/billing.go), [`api_keys.go`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/persistence/api_keys.go). Kiểu domain/API JSON nằm ở [`domain/entities.go`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/domain/entities.go). Một số trường domain có `json:"-"` nên không trả ra FE, ví dụ hash mật khẩu và `keycloak_sub`.

## Ví dụ quan hệ

```text
users._id = A
  ├─ conversations.owner_id = A
  │    └─ messages.conversation_id = conversations._id
  ├─ wallets._id = A
  └─ api_keys.user_id = A
```

BE lấy `A` từ `users.keycloak_sub=<Keycloak sub>`. Nếu tạo một Mongo user mới B cho cùng một người, A và B có hội thoại/ví/API key riêng; không tự gộp. Xem [liên kết user](../02-dang-nhap/lien-ket-user-mongodb.md).

## Ghi và đọc conversation/message

- [`ConversationRepository.Create`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/persistence/repositories.go) insert conversation; `ListOwned` lọc `owner_id` và sort `updated_at` giảm; `OwnedByID` lọc **cả** `_id` và owner trước mọi thao tác.
- [`MessageRepository.Create`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/persistence/repositories.go) insert một message với `conversation_id`; `List` sort `created_at`, `_id` tăng; `Unsummarized` bỏ những message đã đánh dấu `summarized=true` khỏi **context AI**, không bỏ khỏi lịch sử FE.
- Khi context dài, [`Chat.buildContext`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/chat.go) tóm tắt đoạn cũ, cập nhật `conversations.summary`, rồi đánh dấu một nhóm message là summarized. Lịch sử gốc vẫn còn trong `messages`.
- Xóa conversation gọi xóa messages rồi xóa conversation. Không có transaction chung, nên nếu lỗi giữa hai bước cần xử lý trạng thái còn lại.

## Index và giới hạn

[`ensureIndexes`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/persistence/mongo.go) tạo unique index `users.username`, partial unique index `users.keycloak_sub`, index owner/update cho conversation, conversation/time cho message và index cho API key. Wallet chỉ giữ tối đa **1.000 usage record gần nhất** bằng Mongo `$slice: -1000`; thống kê admin cũng chỉ dựa trên số này.

Hiện `messages.content` là **chuỗi text**, không có trường ảnh, file ID hay nơi lưu binary ảnh.
