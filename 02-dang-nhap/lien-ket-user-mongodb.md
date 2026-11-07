# 05. `sub` và Mongo user ID nối với nhau thế nào?

[← Mục lục](../README.md) · [Đăng nhập](dang-nhap-keycloak.md)

Keycloak có ID user riêng gọi là `sub`. MongoDB Chatbox có `_id` riêng trong collection `users`. Chúng **không phải cùng một ID**.

```text
Keycloak user: sub = "abc-123"
Mongo users:   _id = ObjectID("...")
               keycloak_sub = "abc-123"
Mongo conversations.owner_id = users._id
Mongo wallets._id             = users._id
Mongo api_keys.user_id        = users._id
```

Sau khi BE kiểm tra token, nó lấy `sub` rồi gọi [`Auth.ResolveKeycloak`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/auth.go) → [`UserRepository.UpsertKeycloak`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/persistence/repositories.go). Repository query `users` bằng `keycloak_sub`. Nếu đã có, nó trả **đúng Mongo `_id` cũ** và cập nhật vài trường profile. Nếu chưa có, nó tạo một Mongo user mới với username nội bộ `kc-<sub>`. Username/email chỉ là profile; code **không tự ghép tài khoản theo tên giống nhau**.

Vì sao phải cẩn thận? Nếu tạo user mới ở Keycloak có email trùng với Mongo user cũ rồi đăng nhập ngay, `UpsertKeycloak` vẫn tạo **Mongo user khác**. Cuộc trò chuyện và ví cũ đang trỏ tới `_id` cũ, nên người dùng sẽ thấy dữ liệu rỗng. Để chuyển user cũ thật sự, phải xác minh đúng người rồi ghi `keycloak_sub` vào **bản ghi Mongo cũ**, giữ `_id` của nó.

Collection `users` có partial unique index cho `keycloak_sub` trong [`mongo.go`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/persistence/mongo.go); một Keycloak `sub` không thể gắn đồng thời cho nhiều Mongo user.

## Tình trạng hiện tại của tài khoản cũ

Code vẫn còn hàm đăng nhập bcrypt legacy trong [`auth.go`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/auth.go) và route `/api/auth/legacy-login`, nhưng FE đã bỏ nút đăng nhập tài khoản cũ. Khi `KEYCLOAK_ROLES_ENABLED=true`, route legacy bị từ chối. **Chưa có** User Storage SPI, API cầu nối kiểm tra mật khẩu Mongo cũ từ Keycloak hoặc migration tự nối `keycloak_sub` cho user cũ. Việc này là dự định sau, không phải một luồng hiện đang chạy.
