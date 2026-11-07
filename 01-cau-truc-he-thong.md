# 01. Cấu trúc hệ thống Chatbox trên K3s

[← Mục lục](README.md)

File này trả lời câu hỏi: **trên máy chủ có những gì, trong K3s có những gì, và các phần đó nối với nhau thế nào?** Mình mô tả theo manifest trong [repository Chatbox](https://github.com/Nguyen-Trung-Hieu1/Chatbox). Trạng thái pod đang chạy thực tế có thể khác nếu manifest chưa được áp dụng hoặc image chưa được cập nhật.

## Nhìn từ ngoài vào

```text
Người dùng (trình duyệt)
  │
  ├── https://chatbox.cyou ── Caddy trên máy EC2 ── NodePort 30080
  │                                                │
  │                                      K3s / namespace chatbox1
  │                                                │
  │                                      Service chatbox-frontend:80
  │                                                │
  │                                      Pod frontend (React + Nginx)
  │                                                │ /backend/*
  │                                      Service chatbox-backend:8080
  │                                                │
  │                                      Pod backend (Go API)
  │                                       ├── MongoDB
  │                                       ├── 9Router → model AI bên ngoài
  │                                       └── Keycloak Admin/Token API
  │
  └── https://auth.chatbox.cyou ── Caddy ── NodePort 30081
                                               │
                                     K3s / namespace keycloak
                                               │
                                     Pod Keycloak → PostgreSQL riêng
```

**K3s** là cụm Kubernetes chạy trên máy chủ. Nó chạy các container thành pod, cấp địa chỉ nội bộ qua Service và giữ dữ liệu qua volume. **Caddy** là reverse proxy HTTPS chạy trên máy EC2, ở ngoài K3s; nó chuyển hai domain tới hai NodePort khác nhau. Cổng `30080` và `30081` là cổng của K3s trên máy chủ, không phải cổng mà người dùng phải gõ vào trình duyệt.

## Namespace `chatbox1`: ứng dụng chính

Trong [manifest ứng dụng](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/deploy/aws/app.yaml) có hai Deployment:

| Thành phần | Chạy gì | Service và cổng | Dùng để làm gì |
| --- | --- | --- | --- |
| `chatbox-frontend` | React đã build, được Nginx phục vụ trong container | `chatbox-frontend:80`, NodePort `30080` | Trả giao diện và chuyển request `/backend/*` đến backend. |
| `chatbox-backend` | API viết bằng Go | `chatbox-backend:8080` | Kiểm tra đăng nhập/quyền, xử lý chat, plan, credit, hội thoại và API key. |

Hai Deployment này đang khai báo `replicas: 1`: mỗi loại có một pod khi hệ thống ở trạng thái mong muốn. Browser **không gọi thẳng** Service backend. Browser gọi `https://chatbox.cyou/backend/api/...`; [Nginx của frontend](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/nginx.conf) bỏ phần `/backend` rồi chuyển đến `chatbox-backend:8080`, nên Go nhận `/api/...`.

Trong [manifest dữ liệu](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/deploy/aws/data.yaml) có thêm:

| Thành phần | Service và cổng | Dữ liệu giữ ở đâu | Dùng để làm gì |
| --- | --- | --- | --- |
| `mongodb` | `mongodb:27017` | PVC `mongodb-data` 5 GiB, gắn vào `/data/db` | Lưu user nội bộ, hội thoại, tin nhắn, ví/credit, API key. |
| `router` (9Router) | `router:20128` | `hostPath` `/var/lib/chatbox1-k3s/chatbox1-router-copy` trên máy chủ | Nhận request AI từ backend rồi gọi model/upstream tương ứng. |

`mongodb` và `router` cũng khai báo mỗi loại một pod. MongoDB có Service nội bộ; 9Router cũng có Service nội bộ. Backend dùng URL/credential từ cấu hình để kết nối tới chúng; frontend không được truy cập trực tiếp MongoDB hay 9Router.

Trong namespace này còn có những cấu hình không chứa trong manifest công khai: Secret `chatbox-backend-config` cấp biến môi trường cho backend, Secret `mongodb-auth` cấp tài khoản MongoDB và ConfigMap `mongo-init` chứa script khởi tạo MongoDB. Manifest chỉ **tham chiếu tên** các tài nguyên đó, không chứa giá trị secret. PVC MongoDB giữ dữ liệu qua lần pod khởi động lại; riêng 9Router dùng thư mục trên chính máy chủ, nên việc chuyển sang máy chủ khác cần mang dữ liệu theo.

## Namespace `keycloak`: đăng nhập và role

Keycloak chạy ở namespace riêng, không nằm chung pod với Chatbox. Cấu hình triển khai nằm ở [Keycloak server](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/infra/new-account/keycloak-server.tf) và [Keycloak PostgreSQL](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/infra/new-account/keycloak-postgres.tf):

| Thành phần | Loại | Service và cổng | Dùng để làm gì |
| --- | --- | --- | --- |
| `keycloak` | Deployment, một pod | `keycloak:8080`, NodePort `30081` | Trang đăng nhập, OIDC token, realm role và Admin API. |
| `keycloak-postgres` | StatefulSet, một pod | `keycloak-postgres:5432` | Database riêng của Keycloak. |

PostgreSQL của Keycloak dùng PVC `keycloak-postgres-data`. Password database và tài khoản bootstrap admin được đọc từ Kubernetes Secret trong namespace `keycloak`. Chatbox backend **không đọc trực tiếp** PostgreSQL này: backend giao tiếp với Keycloak qua HTTP API. Người dùng đăng nhập trên `auth.chatbox.cyou`; [Caddy proxy](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/infra/new-account/keycloak-proxy.tf) chuyển domain này tới NodePort `30081`.

MongoDB và PostgreSQL là **hai database khác nhau**. MongoDB giữ dữ liệu ứng dụng Chatbox; PostgreSQL giữ dữ liệu do Keycloak quản lý, như tài khoản và role. Khi cần nối một user ở hai bên, backend dùng `sub` của Keycloak để tìm `keycloak_sub` trong MongoDB.

## Ai quản lý các pod và cấu hình?

[Argo CD Application](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/argocd/chatbox1-aws-application.yaml) ở namespace `argocd` theo dõi thư mục `deploy/aws` trong Git, rồi đồng bộ manifest ứng dụng và dữ liệu vào namespace `chatbox1`. Vì vậy, việc push code hay tài liệu lên Git **không tự đồng nghĩa** pod đã chạy bản mới; image/manifest cần được build và đồng bộ. Phần triển khai Keycloak ở trên dùng Terraform tạo AWS SSM document/association để áp cấu hình lên K3s, không nằm trong `deploy/aws` mà Argo CD theo dõi.

Trong K3s, **Deployment/StatefulSet** mô tả pod cần chạy, **Service** cho các pod một tên và cổng ổn định, **Secret/ConfigMap** cấp cấu hình, còn **PVC/hostPath** giữ dữ liệu. Khi pod backend được tạo lại, nó kết nối lại bằng tên Service; địa chỉ IP của pod có thể đổi.

Muốn theo dõi một thao tác cụ thể từ browser tới backend, đọc tiếp [đường đi request](luong-request-chung.md). Các luồng đăng nhập, gửi chat và đổi plan có file riêng trong mục lục.
