# 03. Người dùng bấm Đăng nhập thì chuyện gì xảy ra?

[← Mục lục](../README.md) · [Token, F5 và logout](token-va-f5.md)

Ở đây là lần đăng nhập bằng Keycloak, bắt đầu từ nút **Đăng nhập** trên FE. Nút **Tạo tài khoản** đi gần giống vậy, chỉ chuyển tới endpoint `registrations` thay vì `auth`.

```text
FE → trang Keycloak → FE callback có code → BE → Keycloak token endpoint
                                      BE kiểm tra token → Mongo user → role
                                      BE trả access token + cookie refresh → FE
```

## Từng bước

1. [`Auth.js`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/Auth.js) gọi [`startKeycloakLogin(false)`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/keycloak.js). Hàm tạo `state`, `nonce`, PKCE `verifier` và `challenge`; lưu `state/nonce/verifier` tạm trong `sessionStorage`. Đây là dữ liệu chờ callback, **không phải access token**.
2. FE chuyển browser tới `GET https://auth.chatbox.cyou/realms/chatbox/protocol/openid-connect/auth` với `client_id=chatbox-web`, `redirect_uri=<origin>/auth/callback`, `response_type=code`, `scope=openid profile email`, `state`, `nonce`, `code_challenge` và `code_challenge_method=S256`. Username/password được nhập trên trang Keycloak, không gửi qua FE Chatbox.
3. Keycloak xác thực xong, redirect browser về `/auth/callback?code=...&state=...`. `code` dùng một lần và sống ngắn; nó chưa cho phép gọi Chatbox API.
4. [`App.js`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/App.js) thấy path callback, gọi [`finishKeycloakLogin`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/keycloak.js). Hàm đối chiếu `state`, giới hạn thời gian 5 phút, lấy `code`, xóa dữ liệu chờ và POST tới BE `/api/auth/exchange` với `code`, `code_verifier`, `nonce`, `redirect_uri`.
5. [`Server.Handler`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/server.go) đưa request tới [`exchangeKeycloakCode`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/keycloak_session.go). Handler kiểm tra dữ liệu và redirect URI đúng origin đã cấu hình.
6. [`requestTokens`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/keycloak_session.go) gửi form `grant_type=authorization_code`, `client_id=chatbox-web`, `code`, `code_verifier`, `redirect_uri` tới `POST /realms/chatbox/protocol/openid-connect/token` của Keycloak. **Không dùng client secret của `chatbox-backend-admin` ở bước này.**
7. Keycloak trả `access_token`, `id_token`, `refresh_token`. BE dùng [`verifyIDToken`/`verifyAccessToken`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/keycloak.go) kiểm tra chữ ký RS256 theo public key từ `/protocol/openid-connect/certs`, issuer, thời hạn, nonce, audience/`azp` và `sub` của hai token phải giống nhau.
8. BE gọi [`ResolveKeycloak`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/auth.go) → [`UpsertKeycloak`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/persistence/repositories.go) để lấy Mongo user theo `keycloak_sub`. Chưa có thì tạo Mongo user nội bộ mới. Chi tiết ID ở [file liên kết user](lien-ket-user-mongodb.md).
9. Nếu bật `KEYCLOAK_ROLES_ENABLED`, BE gọi [`SyncUser`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/admin.go) để đọc role hiệu lực từ Keycloak và đồng bộ plan vào ví Mongo. Luồng này được tách ở [đọc role và plan](../03-phan-quyen/doc-role-va-plan.md).
10. BE trả JSON `{user, access_token, expires_in}`; đồng thời đặt refresh token vào cookie `chatbox_refresh` HttpOnly. FE đặt access token vào biến RAM trong [`services/api.js`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/services/api.js), đặt `currentUser` vào React state và mở trang chat.

## Ba token ở bước 7 khác nhau thế nào?

| Token | Ai dùng ở luồng này | Dùng để làm gì |
| --- | --- | --- |
| ID token | BE kiểm tra lúc callback | Xác nhận user vừa đăng nhập và `nonce`. |
| User access token | FE giữ trong RAM, gửi Bearer tới BE | Chứng minh request Chatbox tiếp theo thuộc user nào. |
| Refresh token | Browser giữ trong cookie HttpOnly do BE đặt | Xin access token mới khi F5 hoặc access token hết hạn. |

Service-account access token của client admin là **token thứ tư, khác hẳn**, BE xin riêng để gọi Keycloak Admin API. Xem [đọc role](../03-phan-quyen/doc-role-va-plan.md).

## Nếu lỗi ở giữa?

- `state` không khớp, callback quá 5 phút hoặc thiếu code: FE dừng trước khi gọi BE.
- Token endpoint từ chối code/verifier, thiếu refresh token, JWT sai chữ ký/claims: BE trả lỗi; FE không coi là đăng nhập thành công.
- Mongo hoặc Keycloak Admin API lỗi ở các bước sau: BE không trả phiên thành công. Không lấy username/email trùng để tự nối với Mongo user cũ.
