# 02. Một request đi qua đâu?

[← Mục lục](README.md)

Lấy ví dụ user đã đăng nhập, FE muốn lấy danh sách hội thoại:

1. [`App.fetchConversations`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/App.js) gọi `api('/api/conversations')`.
2. [`services/api.js`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/services/api.js) ghép `API_URL`. Production build dùng `/backend`, nên browser gọi `GET https://chatbox.cyou/backend/api/conversations` với `Authorization: Bearer <user access token>`.
3. [`frontend/nginx.conf`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/nginx.conf) proxy `/backend/` tới `chatbox-backend:8080/` và bỏ tiền tố. Go nhận `GET /api/conversations`.
4. [`Server.Handler`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/server.go) tìm route. Middleware `authn` kiểm tra token, lấy `sub`, tìm Mongo user theo `keycloak_sub`, đồng bộ role nếu bật.
5. Handler `conversations` gọi [`Chat.List`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/chat.go) → [`ConversationRepository.ListOwned`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/persistence/repositories.go).
6. Mongo trả conversation có `owner_id` đúng Mongo user ID. BE trả JSON, FE cập nhật sidebar.

## Route được đăng ký ở đâu?

Hàm [`Server.Handler`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/server.go) tạo `http.NewServeMux` và đăng ký từng path. `method` chặn HTTP method sai bằng 405. `authn` bảo vệ route user; `adminOnly` kiểm tra quyền admin sau `authn`; `apiAuth` bảo vệ `/v1/*` bằng API key `cb_...`.

| Nhóm | Xác thực | Đọc chi tiết |
| --- | --- | --- |
| `/api/auth/*` | Luồng token/cookie, có kiểm tra Origin cho POST | [Đăng nhập](02-dang-nhap/dang-nhap-keycloak.md), [F5](02-dang-nhap/token-va-f5.md) |
| `/api/me`, `/api/conversations`, `/api/history`, `/api/chat`, `/api/entitlement`, `/api/usage`, `/api/keys` | `authn` | [Chat](04-hoi-thoai/gui-tin-nhan.md), [plan](03-phan-quyen/doc-role-va-plan.md), [API key](06-api-key/goi-tu-ide.md) |
| `/api/admin/*` | `authn` → `adminOnly` | [Admin đổi plan](03-phan-quyen/admin-doi-plan.md) |
| `/v1/*` | `apiAuth` | [IDE](06-api-key/goi-tu-ide.md) |

[`cors`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/server.go) chỉ cho request đổi trạng thái dưới `/api/` có `Origin` khớp `CORS_ORIGIN`; [`requestLogging`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/logging.go) ghi log. CORS không thay được kiểm tra user/role.

**Phân biệt URL:** Trong tài liệu BE, `/api/...` là path Go thấy. Browser production gọi `/backend/api/...`. URL Keycloak `/realms/...` và `/admin/realms/...` đi tới domain Keycloak, không đi qua Nginx của Chatbox.
