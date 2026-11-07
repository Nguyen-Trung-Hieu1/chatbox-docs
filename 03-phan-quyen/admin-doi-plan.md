# 07. Admin gạt Free/Paid thì luồng chạy ra sao?

[← Mục lục](../README.md) · [Đọc role và plan](doc-role-va-plan.md)

Ví dụ admin chọn một user đang Free rồi gạt sang Paid. `PATCH /api/admin/plan` là **API của Chatbox tự định nghĩa**. Các URL `/admin/realms/.../role-mappings/...` mới là Keycloak Admin API có sẵn.

1. Khi mở trang admin, [`AdminDashboard.js`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/components/AdminDashboard.js) gọi `GET /api/admin/users`; chọn user thì gọi `GET /api/admin/usage?user_id=<Mongo ID>`. BE chỉ trả dữ liệu sau `authn` và `adminOnly`.
2. Admin gạt nút. `changePlan` gửi `PATCH /api/admin/plan` với JSON `{ "user_id": "<Mongo user ID>", "plan": "paid" }`. `user_id` ở đây **chưa phải Keycloak sub**.
3. [`adminSetPlan`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/admin.go) giải mã JSON và gọi [`Admin.SetPlan`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/admin.go).
4. `Admin.SetPlan` xác nhận plan chỉ là `free`/`paid`, tra `users.keycloak_sub` từ Mongo theo Mongo user ID. Nếu chưa liên kết Keycloak thì không có sub để sửa role, request trả lỗi.
5. BE gọi `Roles(sub)` để nhớ plan trước khi đổi. Hàm này xin service token bằng `client_credentials` rồi `GET /admin/realms/chatbox/users/{sub}/role-mappings/realm/composite`.
6. [`keycloak.Admin.SetPlan`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/keycloak/admin.go) lấy realm role object qua `GET /admin/realms/chatbox/roles/chatbox-paid`. Đổi sang Paid: gửi `POST /admin/realms/chatbox/users/{sub}/role-mappings/realm` với danh sách role Paid. Đổi về Free: gán Free bằng `POST`, rồi gỡ Paid bằng `DELETE` cùng path. Free default có thể vẫn hiện là role thừa kế, nên BE dùng sự có mặt của Paid để quyết định plan.
7. BE `GET` lại effective roles để xác nhận Paid đã được thêm hoặc gỡ. Tiếp theo [`AdminRepository.SetRolePlan`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/persistence/admin.go) ghi `wallets.plan`, `plan_source=keycloak`, balance và `credit_reset_at` vào MongoDB.
8. Nếu ghi Mongo thất bại sau khi đổi role, use case thử đổi role Keycloak về plan trước đó. Hai hệ thống không có transaction chung; nếu rollback cũng lỗi, BE trả cả hai lỗi để operator xử lý.
9. Thành công: BE trả account JSON; [`AdminDashboard`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/components/AdminDashboard.js) thay dòng user trong React state. Ở request tiếp theo của chính user đó, BE lại đọc role Keycloak và chọn model theo plan mới.

**Nút trên FE không trực tiếp gọi Keycloak.** FE chỉ có user access token và gọi BE; BE kiểm tra `chatbox-admin`, rồi mới dùng service account bí mật để gọi Keycloak. Client secret nằm ở backend/Secret, không nằm trong JavaScript của browser.

Trang admin còn hiển thị thống kê usage. [`Admin.Usage`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/admin.go) lấy usage trong Mongo wallet, tổng hợp theo ngày UTC+7, tách token Web/API và model được dùng nhiều nhất. Ví chỉ giữ tối đa 1.000 usage record gần nhất.
