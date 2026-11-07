# Tài liệu luồng chạy Chatbox1

Bộ này viết theo đúng đường đi của một thao tác: **người dùng bấm gì → FE gửi request nào → BE chạy hàm nào → Keycloak/MongoDB/9Router làm gì → kết quả về FE ra sao**. Mỗi file giải thích một việc để có thể đọc độc lập.

Đây là mô tả **code trong [repository Chatbox](https://github.com/Nguyen-Trung-Hieu1/Chatbox)**, không khẳng định pod production đang chạy đúng commit mới nhất. Những thứ mới thảo luận nhưng chưa viết code (chuyển user Mongo cũ sang Keycloak, gửi ảnh cho AI) được ghi rõ là chưa có.

## Cấu trúc bộ tài liệu

```text
docs/
├── README.md
├── 01-tong-quan/
│   ├── he-thong.md
│   └── duong-di-request.md
├── 02-dang-nhap/
│   ├── dang-nhap-keycloak.md
│   ├── token-va-f5.md
│   └── lien-ket-user-mongodb.md
├── 03-phan-quyen/
│   ├── doc-role-va-plan.md
│   └── admin-doi-plan.md
├── 04-hoi-thoai/
│   ├── quan-ly-hoi-thoai.md
│   ├── gui-tin-nhan.md
│   └── luu-tru-mongodb.md
├── 05-9router/
│   ├── tao-request.md
│   ├── xu-ly-stream.md
│   └── usage-va-credit.md
├── 06-api-key/
│   └── goi-tu-ide.md
└── 07-trien-khai/
    ├── cau-hinh-va-secret.md
    └── build-va-deploy.md
```

## Muốn xem luồng nào thì mở file nào?

| Thao tác/câu hỏi | File |
| --- | --- |
| Hệ thống gồm những gì, request thường đi qua đâu | [Hệ thống](01-tong-quan/he-thong.md), [đường đi request](01-tong-quan/duong-di-request.md) |
| Bấm Đăng nhập / Tạo tài khoản | [Đăng nhập Keycloak](02-dang-nhap/dang-nhap-keycloak.md) |
| Bấm F5 / token hết hạn / Đăng xuất | [Token và F5](02-dang-nhap/token-va-f5.md) |
| `sub` nối với Mongo user thế nào, tài khoản cũ ra sao | [Liên kết user](02-dang-nhap/lien-ket-user-mongodb.md) |
| Đọc plan Free/Paid, quyền admin, model được phép | [Đọc role và plan](03-phan-quyen/doc-role-va-plan.md) |
| Admin gạt Free/Paid | [Admin đổi plan](03-phan-quyen/admin-doi-plan.md) |
| Tạo, mở, đổi tên, xóa chat | [Quản lý hội thoại](04-hoi-thoai/quan-ly-hoi-thoai.md) |
| Bấm Gửi tin nhắn | [Gửi tin nhắn](04-hoi-thoai/gui-tin-nhan.md) |
| User, conversation, message lưu ra sao | [Lưu trữ MongoDB](04-hoi-thoai/luu-tru-mongodb.md) |
| BE gọi 9Router, nhận stream, tính credit | [Tạo request](05-9router/tao-request.md), [xử lý stream](05-9router/xu-ly-stream.md), [usage và credit](05-9router/usage-va-credit.md) |
| Tạo API key, dùng trong IDE | [Gọi từ IDE](06-api-key/goi-tu-ide.md) |
| Environment, Secret, build, deploy | [Cấu hình](07-trien-khai/cau-hinh-va-secret.md), [deploy](07-trien-khai/build-va-deploy.md) |

## Mấy tên dùng xuyên suốt

- **FE**: React trong trình duyệt. **BE**: API Go của Chatbox.
- **`sub`**: ID user của Keycloak trong token. **Mongo user ID**: `_id` trong collection `users`; hai ID khác nhau.
- **9Router**: AI gateway mà environment `AI_ENDPOINT` trỏ tới trong môi trường Chatbox.
- Trong code BE, URL là `/api/...`; trên domain production, FE gọi `/backend/api/...` vì Nginx proxy bỏ tiền tố `/backend`.
