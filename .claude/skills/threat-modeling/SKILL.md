# Skill: Threat Modeling (STRIDE)

> Khi nào load: Tạo feature mới có auth / payment / PII / AI / multi-tenant / file upload / external integration
> Output: `docs/threat-models/<feature>-YYYY-MM-DD.md`

---

## 1. Khi nào BẮT BUỘC tạo threat model

| Trigger | Ví dụ |
|---------|-------|
| Feature mới có auth | Login provider mới, 2FA, SSO |
| Feature liên quan payment | Checkout, refund, payout |
| Xử lý PII | CCCD scan, lương nhân viên, địa chỉ khách |
| AI action có side effect | AI tạo order, AI phê duyệt PO |
| Multi-tenant boundary mới | Module LMS tách tenant |
| File upload | Ảnh sản phẩm, import Excel, PDF hợp đồng |
| External integration | Webhook sàn TMĐT, MISA, VNPay |
| Public API endpoint | API open cho đối tác |

**Không cần threat model cho:** UI style change, refactor nội bộ, thêm field không nhạy cảm.

---

## 2. STRIDE — 6 loại mối đe dọa

| Chữ | Loại | Câu hỏi cốt lõi |
|-----|------|----------------|
| **S** — Spoofing | Giả danh | Ai có thể giả vờ là người dùng/service khác? |
| **T** — Tampering | Sửa data | Dữ liệu có thể bị sửa trái phép ở đâu? |
| **R** — Repudiation | Chối bỏ | Ai có thể chối không thực hiện hành động? |
| **I** — Info Disclosure | Lộ thông tin | Dữ liệu nhạy cảm có thể lộ ở đâu? |
| **D** — Denial of Service | Từ chối dịch vụ | Có thể làm tê liệt service ở đâu? |
| **E** — Elevation of Privilege | Leo thang quyền | Có thể truy cập resource ngoài scope không? |

---

## 3. Template threat model (copy khi tạo file mới)

```markdown
# Threat Model: <Feature Name>
> Date: YYYY-MM-DD | Author: <worker/CEO> | Review: <Opus CEO>
> Feature: [mô tả ngắn 1-2 câu]
> Scope: [endpoint, module, data flow]

---

## Data Flow Diagram (mô tả text)

[Client] → [API Gateway] → [Service A] → [DB]
                        ↘ [Service B] → [External API]

Trust boundaries:
- Internet → API: TLS, JWT required
- API → DB: Private network, SSL
- API → External: mTLS hoặc API key

---

## S — Spoofing (Giả danh)

| # | Threat | Mitigation | Status | Risk |
|---|--------|-----------|--------|------|
| S1 | Attacker giả JWT của user khác | RS256 verify, jti blacklist | ✅ Done | High |
| S2 | Webhook giả từ Shopee | HMAC signature verify | ✅ Done | High |
| S3 | ... | ... | ⬜ TODO | ... |

---

## T — Tampering (Sửa data)

| # | Threat | Mitigation | Status | Risk |
|---|--------|-----------|--------|------|
| T1 | User sửa price trong request body | Server fetch price từ DB | ✅ Done | Critical |
| T2 | SQL injection qua input | Drizzle parameterized query | ✅ Done | Critical |
| T3 | ... | ... | ⬜ TODO | ... |

---

## R — Repudiation (Chối bỏ)

| # | Threat | Mitigation | Status | Risk |
|---|--------|-----------|--------|------|
| R1 | User chối tạo order | Audit log: user_id, IP, timestamp, payload | ✅ Done | Medium |
| R2 | Nhân viên chối duyệt PO | Immutable audit_logs + digital signature | ⬜ TODO | High |

---

## I — Information Disclosure (Lộ thông tin)

| # | Threat | Mitigation | Status | Risk |
|---|--------|-----------|--------|------|
| I1 | Stack trace trong error response | GlobalExceptionFilter ẩn detail | ✅ Done | Medium |
| I2 | Order ID guessable (sequential) | Dùng CUID/UUID không đoán được | ✅ Done | Low |
| I3 | PII trong log | Mask phone/CCCD trước khi log | ⬜ TODO | High |

---

## D — Denial of Service

| # | Threat | Mitigation | Status | Risk |
|---|--------|-----------|--------|------|
| D1 | Spam login | Rate limit 10 req/min/IP, throttler | ✅ Done | High |
| D2 | Upload file 10GB | fileSize limit 30MB | ✅ Done | Medium |
| D3 | Webhook flood từ sàn | Queue + rate limit per platform | ✅ Done | Medium |

---

## E — Elevation of Privilege

| # | Threat | Mitigation | Status | Risk |
|---|--------|-----------|--------|------|
| E1 | User thường gọi admin endpoint | RoleGuard + PermissionsGuard | ✅ Done | Critical |
| E2 | Tenant A truy cập data tenant B | RLS + tenant_id filter mọi query | ⚠️ Partial | Critical |
| E3 | AI agent tự approve PO >500k | Human-in-loop gate bắt buộc | ✅ Done | Critical |

---

## Residual Risk

| # | Risk còn lại | Lý do chấp nhận | Giảm thiểu bổ sung |
|---|-------------|-----------------|-------------------|
| R-1 | RLS chưa bật trên tất cả bảng | Wave 2 plan | App-layer filter x3 lớp đang active |

---

## Quyết định cần Letri phê duyệt

- [ ] [Ghi các quyết định security cần CEO/owner approve]

---

## Review checklist

- [ ] Mọi STRIDE category đã có ít nhất 1 entry
- [ ] Risk = Critical → Status = Done trước khi deploy
- [ ] Risk = High → Status = Done hoặc có Residual Risk documented
- [ ] Residual Risk được Opus CEO sign-off
```

---

## 4. Risk classification

| Mức | Tiêu chí | Xử lý |
|-----|----------|-------|
| **Critical** | Data loss, tiền, PII of nhiều user | PHẢI fix trước khi merge |
| **High** | Single user impact, auth bypass | Fix trong sprint, không deploy trước |
| **Medium** | Degraded experience, partial info leak | Fix trong 2 sprint |
| **Low** | Edge case, thông tin không nhạy cảm | Backlog, nice to have |

---

## 5. Workflow tạo threat model

```
1. Copy template → docs/threat-models/<feature>-YYYY-MM-DD.md
2. Điền Data Flow Diagram trước
3. STRIDE từng category — tối thiểu 2 threat mỗi category
4. Assign Status: Done / TODO / N/A
5. Mọi Critical+High phải Done trước khi code
6. Residual Risk list → Opus CEO review
7. Link file trong PR description
```

---

## 6. Ví dụ thực tế — File Upload feature

```markdown
## S — Spoofing
S1: Client fake Content-Type header để bypass MIME check
    → Mitigation: magic byte check (file-type lib)  ✅

## T — Tampering
T1: Attacker sửa storage_key trong presigned URL
    → Mitigation: R2 sign URL với expiry + scope  ✅

## I — Information Disclosure
I1: Presigned URL leak → file của tenant khác
    → Mitigation: storage key prefix tenant_id/  ✅
I2: Scan kết quả (infected) expose tên virus
    → Mitigation: log nội bộ, user chỉ thấy "rejected"  ✅

## D — DoS
D1: Upload 1000 file 30MB → storage exhausted
    → Mitigation: quota per tenant (config), file count limit  ⬜ TODO
```

---

## 7. Anti-pattern ❌

| Anti-pattern | Vấn đề |
|-------------|--------|
| Bỏ qua threat model vì "đơn giản" | Feature thanh toán "đơn giản" bị bypass |
| Threat model không review → xếp ngăn kéo | Risk Critical không được fix |
| Copy paste threat model cũ không update | Threat mới không có mitigation |
| Chỉ liệt kê threat không có mitigation | Checklist đẹp nhưng vô dụng |
| Không link threat model trong PR | Review không biết feature đã threat-model |

---

## Cross-ref

- `rules/01-security.md` — OWASP Top 10 + STRIDE context
- `skills/security-audit/SKILL.md` — Security review toàn diện
- `docs/threat-models/README.md` — Index tất cả threat model
- `rules/17-file-upload.md` — Threat model mẫu cho upload
- `rules/13-ai-agents.md` — AI threat model (scope, privilege)
