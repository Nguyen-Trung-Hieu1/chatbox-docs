# 13. Usage và credit được tính lúc nào?

[← Mục lục](../README.md) · [Tạo request](tao-request.md)

Credit là **đơn vị nội bộ của Chatbox**, không phải số tiền nhà cung cấp AI thu. Rate của từng model ở [`config.go`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/config/config.go) hoặc được override bằng `AI_FREE_MODELS_JSON`/`AI_PAID_MODELS_JSON`; rate tính theo 1.000 input/output token.

## Trước khi gọi 9Router: tạm giữ credit

1. [`Chat.policy`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/chat.go) lấy entitlement, kiểm tra model được phép với plan hiện hành. `Entitlement` gọi [`BillingRepository.Ensure`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/persistence/billing.go) để tạo ví nếu thiếu, rồi đọc plan/balance.
2. BE ước lượng input token từ text trong context, tính mức tạm giữ bằng rate model, input ước lượng nhân hệ số dự phòng 4 và output tối đa.
3. [`BillingRepository.Reserve`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/persistence/billing.go) chạy Mongo `UpdateOne` có điều kiện `balance >= reserve`, rồi `$inc` balance âm. Nếu không đủ, trả lỗi quota; chưa gọi 9Router. Điều kiện nằm trong cùng thao tác update để hai request đồng thời không chi quá số dư.

## Sau khi 9Router trả lời: quyết toán

1. AI client lấy `prompt_tokens/completion_tokens` hoặc `input_tokens/output_tokens` do gateway báo; nếu gateway không báo thì [`billableUsage`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/chat.go) ước lượng từ text và đánh `reported=false`.
2. [`charge`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/chat.go) tính credit thật, chặn không vượt mức đã reserve, gọi [`BillingRepository.Settle`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/persistence/billing.go). Repository hoàn `reserve - spent` vào balance và `$push` usage record.
3. Usage record có model, loại (`chat`, `summary`, `api`), input/output tokens, credits, `reported`, status và thời gian. `wallets.usage` giữ 1.000 record gần nhất. `GET /api/usage` trả các record này cho user; trang admin tổng hợp theo ngày UTC+7.
4. Nếu request thất bại khi chưa có nội dung và gateway chưa báo usage, `spent=0`, khoản tạm giữ được hoàn. Nếu stream đã tạo nội dung rồi ngắt, status là `interrupted` và vẫn có thể tính usage cho phần đã xử lý.

## Ví và reset

Ví dùng `_id` bằng Mongo user ID. Free mặc định `FREE_INITIAL_CREDITS` (thường 1.000), Paid 2.000. Mốc reset cách 2 giờ; [`BillingRepository.Plan`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/persistence/billing.go) áp dụng reset ở **request đầu tiên sau mốc**, không có cron cứ đúng giờ quét mọi ví. Khi bật role Keycloak, plan trong ví được đồng bộ từ role; balance/usage vẫn ở Mongo.

`kind=summary` giải thích vì sao một lần user bấm Gửi có thể thấy usage lớn hơn chỉ câu trả lời chính: `buildContext` đôi khi gọi thêm AI để tóm tắt hội thoại cũ.
