# 04. Access token, F5 và Đăng xuất

[← Mục lục](../README.md) · [Đăng nhập](dang-nhap-keycloak.md)

Sau khi đăng nhập, FE giữ **user access token trong RAM**. Browser giữ **refresh token trong cookie HttpOnly**. BE không lưu hai token này vào MongoDB. Khi đóng tab hoặc F5, RAM mất; cookie vẫn còn nếu phiên Keycloak và cookie chưa hết hạn.

## Một request bình thường

1. [`api()`/`authenticatedFetch()`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/services/api.js) lấy access token RAM và gắn `Authorization: Bearer <token>` vào request tới BE, ví dụ `GET /api/me`.
2. [`authn`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/server.go) gọi [`verifyAccessToken`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/keycloak.go): đọc JWT, kiểm tra chữ ký qua JWKS `/certs`, issuer, thời hạn, `azp`, type và `sub`.
3. BE lấy Mongo user theo `sub`, rồi mới chạy handler. BE **không POST access token tới Keycloak token endpoint mỗi request**. Nếu bật role Keycloak, BE có một request Admin API riêng để đọc role; xem [đọc role](../03-phan-quyen/doc-role-va-plan.md).

## Bấm F5

1. React load lại; biến access token RAM trở thành rỗng. [`App.js`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/App.js) hiện “Đang kiểm tra phiên đăng nhập...” và gọi [`refreshSession()`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/services/api.js).
2. FE gửi `POST /api/auth/refresh` với `credentials: include`. Browser tự chọn cookie `chatbox_refresh` theo domain/path rồi gửi. JavaScript không đọc được giá trị cookie HttpOnly.
3. [`refreshKeycloakToken`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/keycloak_session.go) lấy cookie, gọi Keycloak `POST /realms/chatbox/protocol/openid-connect/token` với `grant_type=refresh_token`, `client_id=chatbox-web`, `refresh_token=<cookie>`.
4. Keycloak cấp access token mới; nếu có refresh token xoay vòng, BE đặt lại cookie. BE kiểm tra access token mới, tìm Mongo user, đồng bộ role nếu bật, rồi trả `{user, access_token, expires_in}`.
5. FE nạp access token vào RAM và `currentUser` vào React state. Người dùng vẫn thấy trang chat, không phải nhập lại mật khẩu.

Cookie được đặt ở [`keycloak_session.go`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/keycloak_session.go) với `Path=/api/auth`, `HttpOnly`, `MaxAge` tối đa 30 ngày; HTTPS production bật `Secure` và `SameSite=None`. [`nginx.conf`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/nginx.conf) đổi `Path` sang `/backend/api/auth` trên response production, vì browser thực sự gọi `/backend/api/auth/refresh`. **MaxAge 30 ngày không bảo đảm Keycloak cho phiên tồn tại 30 ngày**; idle/max session của Keycloak có thể kết thúc sớm hơn.

## Access token hết hạn khi đang dùng

Nếu một API trả 401 trong lúc FE vẫn có access token RAM, [`authenticatedFetch`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/services/api.js) gọi `refreshSession()` rồi thử gửi lại request **một lần**. Nhiều request refresh đồng thời trong cùng một tab dùng chung `refreshInFlight`. Nếu refresh thất bại, FE xóa access token; request vẫn không thành công và màn hình có thể quay về đăng nhập.

## Bấm Đăng xuất

[`logoutKeycloak`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/keycloak.js) gọi `POST /api/auth/logout`. BE gửi refresh token tới Keycloak `/protocol/openid-connect/revoke`, xóa cookie refresh và cookie legacy, trả `logout_url`. FE xóa access token RAM rồi chuyển browser đến Keycloak logout URL. Đây là hai việc: bỏ phiên ở Chatbox/browser và đi qua logout của Keycloak.

## Nếu F5 lại bị out

Nhìn request `POST /backend/api/auth/refresh` trong Network: có Cookie không, status là gì, response có `Set-Cookie` mới không. Kiểm tra cookie `chatbox_refresh` có `Path=/backend/api/auth` trên production. Cookie cũ với `Path=/api/auth` sẽ không được gửi cho URL `/backend/...`; cần đăng nhập lại sau khi sửa proxy path. Nếu request có cookie nhưng Keycloak trả lỗi, kiểm tra hạn phiên/refresh token thay vì đoán lỗi FE.
