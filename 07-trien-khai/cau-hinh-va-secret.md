# 15. Cấu hình và Secret nằm ở đâu?

[← Mục lục](../README.md) · [Build/deploy](build-va-deploy.md)

[`backend/internal/config/config.go`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/config/config.go) đọc environment khi BE khởi động. [`backend/.env.example`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/.env.example) là mẫu tên biến; secret thật không nên commit vào Git. [`cmd/api/main.go`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/cmd/api/main.go) lấy config đó để khởi tạo Mongo, Keycloak client, AI client và HTTP server.

| Biến | Dùng vào việc gì | Nên ở đâu |
| --- | --- | --- |
| `MONGO_URI`, `MONGO_DATABASE` | Kết nối MongoDB Chatbox | Secret/env BE |
| `KEYCLOAK_ISSUER` | Realm Keycloak để đổi/kiểm tra token | Env BE |
| `KEYCLOAK_CLIENT_ID` | Client đăng nhập user, mặc định `chatbox-web` | Env BE/FE tương ứng |
| `KEYCLOAK_ADMIN_CLIENT_ID` | Client service account của BE | Env BE |
| `KEYCLOAK_ADMIN_CLIENT_SECRET` | BE xin service-account token gọi Admin API | **Secret BE**, không FE |
| `KEYCLOAK_ROLES_ENABLED` | Bật Keycloak role làm nguồn plan/admin | Env BE |
| `AI_ENDPOINT`, `AI_API_KEY` | URL và key để BE gọi 9Router | Env/Secret BE |
| `AI_FREE_MODELS_JSON`, `AI_PAID_MODELS_JSON` | Override model/rate nội bộ | Env BE nếu cần |
| `FREE_INITIAL_CREDITS`, `AI_TOKEN_BUDGET` | Ví Free và context budget | Env BE |
| `CORS_ORIGIN`, `COOKIE_SECURE`, `REFRESH_COOKIE_NAME` | Origin hợp lệ, cờ cookie HTTPS, tên cookie | Env BE |

Ở AWS, [`deploy/aws/app.yaml`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/deploy/aws/app.yaml) dùng `envFrom.secretRef.name=chatbox-backend-config` để đưa Secret vào backend pod, cùng vài env không bí mật như `CORS_ORIGIN`, `COOKIE_SECURE=true`. Tên Secret ở đây **không có nghĩa giá trị Secret nằm trong repo**. FE là JavaScript chạy trong browser: mọi biến `REACT_APP_*` sẽ nằm trong build và có thể xem được, nên không đặt client secret hay `AI_API_KEY` ở đó.

Keycloak chạy tách khỏi Chatbox backend, có PostgreSQL riêng; cấu hình triển khai nằm ở [`infra/new-account`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/infra/new-account). Backend đọc/sửa user role qua API Keycloak bằng service account, không kết nối thẳng PostgreSQL này.

Local development và production khác URL: image frontend production đặt `REACT_APP_API_URL=/backend` trong [`frontend/Dockerfile`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/Dockerfile); Nginx proxy tới BE. Local FE không đặt biến này thì mặc định gọi `http://localhost:8080`. Cookie refresh BE đặt path `/api/auth`, production Nginx viết lại thành `/backend/api/auth` trong [`nginx.conf`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/nginx.conf).

Tài liệu này không in giá trị secret, access token, refresh token hay credential của 9Router. Khi debug, chỉ kiểm tra biến **có tồn tại** và request trả status nào; không chép secret ra log/tài liệu.
