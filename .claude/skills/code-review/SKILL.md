# Skill: Code Review

> Khi nào load: Review PR, viết review checklist

---

## 90+ items checklist

### Correctness (10)
- [ ] Logic đúng yêu cầu (đọc lại issue/PR description)
- [ ] Edge cases: null, empty, 0, negative, max value
- [ ] Error handling đầy đủ
- [ ] Async đúng (no `forEach async`, có `await`)
- [ ] Race condition đã xử lý (lock, transaction)
- [ ] Off-by-one: pagination, array index, date range
- [ ] Timezone đúng (UTC store, convert ở presentation)
- [ ] Decimal cho tiền, không float
- [ ] Idempotent
- [ ] Rollback được nếu fail

### Security (15)

> 📎 **Cross-ref:** Security self-check checklist 12 items (commit-time perspective) xem [rules/01-security.md](../../rules/01-security.md#self-check-trước-commit-)

- [ ] Input validation (Zod) cho mọi data từ user
- [ ] Không hardcode secret
- [ ] Auth guard cho mọi endpoint (trừ public whitelist)
- [ ] Authorization (CASL) cho mutation
- [ ] Tenant isolation: filter `tenant_id`
- [ ] SQL: chỉ Drizzle/parameterized
- [ ] Không trả password/token trong response
- [ ] Audit log cho action nhạy cảm
- [ ] File upload: check MIME + size + magic bytes
- [ ] Không log PII/secret
- [ ] CORS không `*`
- [ ] Rate limit cho endpoint nhạy cảm
- [ ] CSRF protection (nếu form-based)
- [ ] XSS prevention (no dangerouslySetInnerHTML không sanitize)
- [ ] SSRF prevention nếu có user-provided URL

### Maintainability (8)
- [ ] Tên biến/hàm rõ nghĩa
- [ ] Function ngắn (≤ 50 dòng), single responsibility
- [ ] Class ngắn (≤ 300 dòng)
- [ ] Không nested if/for > 3 cấp
- [ ] Không magic number/string
- [ ] Comment "TẠI SAO" cho logic phức tạp
- [ ] Không duplicate code (DRY) — tách util nếu lặp ≥ 3 lần
- [ ] Không dead code

### Type Safety (6)
- [ ] Không `any`
- [ ] Không `@ts-ignore` (dùng `@ts-expect-error` + comment)
- [ ] Không dùng `as` ép kiểu trừ khi cần (có comment)
- [ ] Public function có return type rõ ràng
- [ ] Generic constraint hợp lý
- [ ] Discriminated union cho state machine

### Performance (8)
- [ ] Không N+1 query
- [ ] List endpoint có pagination
- [ ] Index cho cột query thường
- [ ] Cache (Redis) cho data ít đổi
- [ ] Lazy load component nặng (FE)
- [ ] Image dùng `next/image`
- [ ] Không re-render thừa
- [ ] Bundle size không tăng đột biến

### Testing (8)
- [ ] Unit test cho business logic mới
- [ ] Integration test cho endpoint mới
- [ ] Test happy path
- [ ] Test error cases
- [ ] Test edge cases
- [ ] Test security: cross-tenant, IDOR
- [ ] Coverage không tụt
- [ ] Test name rõ ràng

### Architecture (7)
- [ ] Controller → Service → Repository (không skip layer)
- [ ] Service không import Drizzle trực tiếp
- [ ] DTO dùng Zod schema từ `packages/shared`
- [ ] Module mới có README.md
- [ ] Cross-module qua DI hoặc EventBus
- [ ] State machine cho workflow phức tạp
- [ ] Không circular dependency

### Database (8)
- [ ] Bảng có `tenant_id` + audit fields
- [ ] Soft delete (`deleted_at`)
- [ ] Index cho FK + cột query
- [ ] Decimal cho tiền
- [ ] Enum cho status
- [ ] Migration có `.down.sql`
- [ ] RLS policy đã update nếu cần
- [ ] EXPLAIN ANALYZE cho query mới

### API Contract (10)
- [ ] URL theo REST convention
- [ ] Có version prefix `/api/v1/`
- [ ] Response format chuẩn
- [ ] HTTP status code đúng
- [ ] Pagination cursor-based
- [ ] Datetime ISO 8601 với timezone
- [ ] Field naming camelCase
- [ ] Swagger annotation đầy đủ
- [ ] Idempotency key cho POST mutate
- [ ] Không break backward compat (hoặc bump version)

### UX (10, nếu có FE)
- [ ] Loading state cho mọi async
- [ ] Error state với message rõ ràng (i18n)
- [ ] Empty state có hint
- [ ] Form validation client-side
- [ ] Optimistic update khi phù hợp
- [ ] Toast cho action thành công
- [ ] Confirm dialog cho action destructive
- [ ] Mobile responsive (test 360px)
- [ ] Keyboard accessible
- [ ] i18n cho mọi text

### Compliance (5)

> 📎 **Cross-ref:** Compliance checklist đầy đủ (PII encrypt, TT99, NĐ70, audit retention 10 năm) xem [rules/15-compliance-vn.md](../../rules/15-compliance-vn.md#checklist)
- [ ] PII có encrypt + audit log
- [ ] Tài chính có audit log retention 10 năm (TT 99)
- [ ] User consent ghi rõ
- [ ] Hoá đơn điện tử theo NĐ 70/2025
- [ ] Không lưu PAN (payment outsource VNPay/MoMo)

---

## Review process

### 1. Self-review (developer trước khi request)
- Chạy lint, type-check, test local
- Đọc lại code 1 lần như reviewer
- Check checklist trên
- Self-comment chỗ phức tạp ("@reviewers: lý do tôi làm thế này...")

### 2. Reviewer (manh dạn 30-60 phút)
- Đọc PR description trước
- Đọc test trước (hiểu yêu cầu)
- Review từng file theo thứ tự: schema → repo → service → controller → test
- Comment cụ thể (file:line + suggestion)
- Approve hoặc Request changes

### 3. Author response
- Trả lời mọi comment
- Resolve hoặc tranh luận
- Force push nếu rebase, regular push nếu fix

### 4. Final approval
- ≥ 1 approver (≥ 2 cho security/payment/AI)
- All conversations resolved
- CI green
- Squash merge

---

## Comment templates

### Suggestion
```
💡 **Suggestion:** Có thể tách logic này thành function `calculateOrderTotal()` để dễ test.

```ts
function calculateOrderTotal(items: OrderItem[]): Money {
  // ...
}
```
```

### Issue (must fix)
```
🔴 **Bug:** Function này không filter `tenant_id`. Vi phạm Rule 02 (Multi-tenant).

```ts
// Hiện tại
return db.query.orders.findMany();

// Phải sửa
return db.query.orders.findMany({ where: tenantFilter(orders) });
```
```

### Question
```
❓ **Question:** Tại sao cần lock pessimistic ở đây? Optimistic không đủ?
```

### Praise (đừng quên!)
```
👍 **Nice:** Pattern saga ở đây rất rõ ràng, dễ hiểu rollback.
```

---

## Review không bao giờ làm

- ❌ Comment style "tôi sẽ viết khác" (subjective)
- ❌ Block PR vì preference cá nhân
- ❌ Yêu cầu refactor trong PR mới (tạo follow-up issue)
- ❌ Approve mà chưa đọc test
- ❌ Approve qua loa cho qua chuyện
- ❌ Comment xúc phạm tác giả

---

**END Skill: Code Review**
