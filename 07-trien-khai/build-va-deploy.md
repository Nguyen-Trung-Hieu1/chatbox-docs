# 16. Push Git rồi bản chạy thay đổi thế nào?

[← Mục lục](../README.md) · [Cấu hình](cau-hinh-va-secret.md)

Push code lên Git **chưa đồng nghĩa** pod đã chạy image mới. Trong dự án này có nhiều bước nối nhau:

1. [`.github/workflows/build-images.yml`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/.github/workflows/build-images.yml) chỉ chạy tự động khi push `main` có thay đổi dưới `backend/**`, `frontend/**` hoặc chính workflow. Chỉ sửa `docs/**` không kích hoạt build image. Có thể chạy `workflow_dispatch` thủ công.
2. Hai job build Docker image BE/FE, push lên GHCR với tag `main` và `sha-<Git commit SHA>`.
3. Job `update-deployment` sửa tag image trong [`deploy/aws/app.yaml`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/deploy/aws/app.yaml) và [`deploy/local-app.yaml`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/deploy/local-app.yaml), rồi commit manifest về Git.
4. [`argocd/chatbox1-aws-application.yaml`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/argocd/chatbox1-aws-application.yaml) theo dõi `main`, path `deploy/aws`, bật auto sync/self heal. Khi Argo CD hoạt động, nó áp manifest mới vào namespace `chatbox1` của K3s.
5. K3s kéo image mới và rollout pod BE/FE. Service frontend/backend giữ endpoint ổn định; request mới vào pod đã sẵn sàng. Các nguồn dữ liệu MongoDB/Keycloak nằm ngoài image nên không bị thay khi thay image ứng dụng.

## Request production thật đi qua đâu?

Browser mở `https://chatbox.cyou`, FE Nginx phục vụ HTML/JS từ image. FE gọi `/backend/api/...`; [`nginx.conf`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/nginx.conf) proxy tới service `chatbox-backend:8080` và bỏ `/backend`. BE gọi Mongo, Keycloak hoặc 9Router theo việc cần làm. Keycloak có domain riêng `auth.chatbox.cyou` và namespace riêng `keycloak`.

## Kiểm tra đã chạy đúng bản mới

Xem commit GitHub Actions đã build thành công, tag `sha-...` trong `deploy/aws/app.yaml`, trạng thái sync Argo CD, image thực trong Deployment/pod và rollout. Đừng chỉ nhìn `git push` thành công hoặc source code local. Nếu FE mới nhưng BE cũ (hoặc ngược lại), các route/cookie mới có thể không khớp nhau.

**Đặc biệt với F5:** sau khi sửa proxy cookie path và FE image đã rollout, user cần đăng nhập lại một lần để nhận cookie mới có `Path=/backend/api/auth`; cookie cũ với path `/api/auth` không đi tới `/backend/api/auth/refresh`.

## Các phần triển khai khác

- [`deploy/aws/data.yaml`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/deploy/aws/data.yaml) chứa MongoDB và 9Router cùng storage/service liên quan.
- [`infra/new-account`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/infra/new-account) chứa Terraform/SSM cho AWS, Keycloak, PostgreSQL, proxy Keycloak; không phải một phần của image FE/BE.
- [`argocd/monitoring-stack.yaml`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/argocd/monitoring-stack.yaml) và [`monitoring`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/monitoring) dành cho giám sát/log.
- [`deploy/local-app.yaml`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/deploy/local-app.yaml) và [`deploy/local-data.yaml`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/deploy/local-data.yaml) mô tả môi trường K3s local; nó có thể dùng image/tag khác production AWS.
