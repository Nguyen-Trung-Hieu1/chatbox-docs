# 01. Hệ thống gồm những gì?

[← Mục lục](../README.md)

Trang Chatbox nhìn như một ứng dụng, nhưng thực ra có mấy phần chạy riêng:

```text
Trình duyệt
  ├─ React: hiển thị chat, gọi BE
  └─ Keycloak: trang đăng nhập

Nginx frontend → Go backend
                  ├─ MongoDB: user, chat, ví, API key
                  ├─ Keycloak: đổi token, đọc/gán role
                  └─ 9Router: gửi câu hỏi tới model AI

Keycloak → PostgreSQL riêng của Keycloak
```

| Thành phần | Việc của nó |
| --- | --- |
| FE React | Giữ state giao diện, access token trong RAM, gửi request và hiển thị stream. |
| Nginx frontend | Phục vụ React build, chuyển `/backend/...` đến service Go. |
| BE Go | Xác thực, áp dụng quy tắc plan/chat/credit, gọi MongoDB, Keycloak và AI gateway. |
| Keycloak | Đăng nhập OIDC, cấp token, lưu các realm role Free/Paid/Admin. |
| PostgreSQL Keycloak | DB riêng của Keycloak; BE Chatbox không query trực tiếp. |
| MongoDB Chatbox | User nội bộ, conversations, messages, wallets, API keys. |
| 9Router | Nhận request AI từ BE và chuyển đến model. |

Ở BE, [`cmd/api/main.go`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/cmd/api/main.go) là nơi khởi động: đọc config, mở Mongo, tạo repository, use case, AI client, Keycloak client, HTTP server. Một chức năng thường đi theo chiều `HTTP handler → use case → repository/client`. Các interface nằm trong [`domain/ports.go`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/domain/ports.go).

- [`delivery/http`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http): route, middleware, handler, JSON, cookie, stream.
- [`usecase`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase): logic auth, chat, admin, API key.
- [`infrastructure/persistence`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/persistence): query MongoDB.
- [`infrastructure/keycloak`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/keycloak): Keycloak Admin API.
- [`infrastructure/ai`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/ai): AI gateway.

Ở FE, [`App.js`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/App.js) điều phối user, hội thoại, tin nhắn và model; [`services/api.js`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/services/api.js) gọi BE; [`keycloak.js`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/keycloak.js) xử lý OIDC; [`components`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/components) chứa các phần nhìn thấy.

Đọc tiếp [đường đi request](duong-di-request.md) để thấy các phần này nối với nhau ra sao.
