# 06. Backend đọc role và quyết định plan thế nào?

[← Mục lục](../README.md) · [Đăng nhập](../02-dang-nhap/dang-nhap-keycloak.md)

Trong realm `chatbox` có ba role ứng dụng: `chatbox-free`, `chatbox-paid`, `chatbox-admin`. Free có thể là default role. User có cả Free và Paid thì code chọn **Paid**. Role nằm ở Keycloak; credit/balance và usage nằm ở MongoDB.

## Khi nào BE đọc role?

Sau `/api/auth/exchange`, `/api/auth/refresh` và trên request có Bearer user access token, BE gọi `syncKeycloakRole` nếu cấu hình `KEYCLOAK_ROLES_ENABLED=true`. API key của IDE cũng đồng bộ role của owner trước khi dùng model. Nếu cờ tắt, code plan Mongo cũ vẫn được dùng. Giá trị cờ thực tế nằm trong environment/Secret của pod, không suy ra được từ `.env.example`.

## Từng request BE → Keycloak

1. [`Admin.SyncUser`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/admin.go) nhận Mongo user đã được tìm bằng `sub`; lấy `u.KeycloakSub`.
2. [`keycloak.Admin.Roles`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/keycloak/admin.go) cần service-account token, nên gọi `token()`.
3. `token()` gửi form `grant_type=client_credentials`, `client_id=chatbox-backend-admin`, `client_secret=<secret>` tới `POST https://auth.chatbox.cyou/realms/chatbox/protocol/openid-connect/token`. Đây là token **của backend**, không phải token đăng nhập của user.
4. `Roles()` gửi `GET https://auth.chatbox.cyou/admin/realms/chatbox/users/{sub}/role-mappings/realm/composite` với `Authorization: Bearer <service token>`.
5. Keycloak trả mảng realm role hiệu lực, gồm role thừa kế. BE lấy tên từng role vào map: có `chatbox-admin` thì `u.IsAdmin=true`; có `chatbox-paid` thì plan `paid`, còn lại `free`.
6. [`AdminRepository.SyncRolePlan`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/persistence/admin.go) ghi plan tương ứng vào `wallets` của Mongo user ID, đặt `plan_source=keycloak` và điều chỉnh balance/reset khi plan hoặc nguồn plan thay đổi. Vì ví nằm trong Mongo nên FE sau đó vẫn nhận `plan` từ endpoint entitlement đọc ví.

## FE biết model nào được dùng như thế nào?

[`App.fetchEntitlement`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/App.js) gọi `GET /api/entitlement`. Route chạy `authn` trước, nên role đã được đồng bộ. Handler gọi [`Chat.Entitlement`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/chat.go): bảo đảm ví tồn tại, đọc plan, balance và mốc reset, chọn `freeModels` hoặc `paidModels` rồi trả JSON. FE hiển thị danh sách đó ở bộ chọn model.

Khi user bấm Gửi, [`Chat.policy`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/chat.go) **kiểm tra lại model ở BE** bằng entitlement. Sửa danh sách model trên FE không cho phép gọi model Paid nếu BE xác định tài khoản Free. Nếu thiếu credit, request dừng ở bước reserve trước khi gọi 9Router. Xem [usage và credit](../05-9router/usage-va-credit.md).

## Quyền admin được kiểm tra ở đâu?

FE có thể ẩn trang admin nếu `currentUser.is_admin=false`, nhưng đây chỉ là giao diện. Các route `/api/admin/*` chạy [`authn`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/server.go) để lấy user/role mới nhất, rồi [`adminOnly`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/admin.go) kiểm tra `IsAdmin`. Nếu Keycloak không trả được role lúc cờ đang bật, request lỗi; BE không tự lấy `is_admin` Mongo cũ để cho qua.

## Hai điều không nên nhầm

- BE kiểm tra **chữ ký user access token** bằng Keycloak JWKS. Việc BE **đọc role hiện hành** là một request Admin API khác, dùng service token.
- `wallets.plan` là bản đồng bộ phục vụ entitlement/billing. Khi role mode bật, muốn đổi quyền plan thì phải đổi role Keycloak; chỉ sửa `wallets.plan` có thể bị đồng bộ đè lại ở request kế tiếp.
