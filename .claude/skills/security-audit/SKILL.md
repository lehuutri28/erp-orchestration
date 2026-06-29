# Skill: Security Audit

> ⚠️ Đồng bộ ARCHITECTURE_CONTRACT 29/5/2026: PK=uuid, tenant_id=uuid, TS camelCase, RBAC dấu `.`. Xem docs/ARCHITECTURE_CONTRACT.md.

> Khi nào load: Review bảo mật, threat model, pen test, security incident

---

## Threat modeling (STRIDE)

Khi build feature có động chạm tiền/PII/public endpoint, BẮT BUỘC tạo file `docs/threat-models/<feature>.md`:

```markdown
# Threat Model: <Feature Name>

## Scope
[Mô tả feature, data luồng đi đâu]

## Spoofing (giả danh)
| Threat | Mitigation | Status |
|---|---|---|
| User giả danh admin | JWT verify + 2FA cho admin | ✅ |

## Tampering (sửa data)
| Threat | Mitigation | Status |
|---|---|---|
| User sửa price trong request | Server fetch price từ DB, không tin client | ✅ |

## Repudiation (chối)
| Threat | Mitigation | Status |
|---|---|---|
| User chối tạo order | Audit log có user_id, IP, timestamp | ✅ |

## Information Disclosure (leak)
| Threat | Mitigation | Status |
|---|---|---|
| Order detail leak qua URL | ID dùng uuid v4 (unguessable, chuẩn Contract), check ownership | ✅ |

## Denial of Service
| Threat | Mitigation | Status |
|---|---|---|
| Spam tạo order rỗng | Rate limit 30 order/giờ/user, captcha sau 10 | ✅ |

## Elevation of Privilege
| Threat | Mitigation | Status |
|---|---|---|
| User thường tạo order với discount admin | CASL check field-level | ⚠️ Wave 4 |
```

---

## Security checklist khi review feature

### Authentication
- [ ] JWT có TTL hợp lý (access 15m, refresh 7d)
- [ ] Refresh token rotation
- [ ] Account lockout sau N lần fail
- [ ] 2FA cho admin role
- [ ] Password policy NIST 800-63B (12+ ký tự, breach check)

### Authorization
- [ ] Mọi endpoint có guard
- [ ] Permission check ở field level (CASL)
- [ ] Tenant isolation 4 lớp (Rule 02)
- [ ] Test cross-tenant + IDOR

### Input validation
- [ ] Zod schema cho mọi user input
- [ ] `.strict()` để reject extra fields
- [ ] File upload check MIME + size + magic bytes
- [ ] URL từ user → SSRF check

### Output encoding
- [ ] React auto-escape (không `dangerouslySetInnerHTML` không sanitize)
- [ ] Response không có internal field (password_hash, ...)
- [ ] Error message không leak stack trace

### Crypto
- [ ] Password hash Argon2id
- [ ] Secret từ Doppler/Vault, không `.env` plain
- [ ] TLS 1.3 only
- [ ] HMAC verify cho webhook
- [ ] Timing-safe compare

### Logging
- [ ] PII redact list đầy đủ
- [ ] Audit log cho action nhạy cảm
- [ ] Không log password/token/CCCD

### Rate limiting
- [ ] Login: 5/phút
- [ ] Register/forgot password: 3/giờ
- [ ] Default API: 100/phút
- [ ] Export: 10/giờ

### Compliance
- [ ] NĐ 13/2023: consent + export + delete
- [ ] TT 99/2025: audit log retention 10 năm cho tài chính

---

## Penetration testing tools

```bash
# OWASP ZAP — automated scan
docker run -v $(pwd):/zap/wrk/:rw -t zaproxy/zap-stable zap-full-scan.py \
  -t https://api-staging.wecha.vn \
  -r zap-report.html

# Nuclei — template-based scanner
nuclei -u https://api-staging.wecha.vn -tags cve,exposure

# SQLMap — SQL injection test
sqlmap -u "https://api-staging.wecha.vn/api/v1/orders?id=1" \
  --headers="Authorization: Bearer $TOKEN" \
  --batch --random-agent

# nikto — web server scanner
nikto -h https://api-staging.wecha.vn

# Trivy — container scan
trivy image ghcr.io/wecha/erp-api:latest
```

---

## Incident response runbook

### Phát hiện security incident → 5 bước

**1. Detect & Triage (0-15 phút)**
- Confirm incident (false positive vs real)
- Severity: Low / Medium / High / Critical
- Escalate: Trí + DPO + on-call

**2. Contain (15-60 phút)**
- Isolate affected system
- Revoke compromised credentials
- Block IP attacker (Cloudflare WAF)
- Snapshot evidence (logs, DB state)

**3. Eradicate (1-4h)**
- Patch vulnerability
- Remove malicious code
- Reset all secrets nếu nghi ngờ compromise

**4. Recover (4-24h)**
- Restore from clean backup nếu cần
- Re-deploy với patch
- Verify integrity (checksum, audit log)
- Monitor 48h sau

**5. Lessons Learned (1 tuần)**
- Postmortem document
- Action items
- Update runbook
- Notify users nếu PII leaked (NĐ 13 — 72 giờ)

---

## Common vulnerabilities to check

### Stored XSS
```typescript
// Test: gửi `<script>alert(1)</script>` vào mọi text field
// Verify: render ra trang khác có execute không
```

### Reflected XSS
```typescript
// Test: gửi `?q=<script>alert(1)</script>`
// Verify: response có encode không
```

### CSRF
```typescript
// Test: tạo HTML form ở domain khác POST tới ERP API
// Verify: SameSite cookie + CSRF token
```

### SSRF
```typescript
// Test: gửi URL `http://localhost:6379` (Redis), `http://169.254.169.254` (cloud metadata)
// Verify: blocked
```

### Mass Assignment
```typescript
// Test: PATCH /users/me với body { roles: ['admin'] }
// Verify: roles bị reject
```

### IDOR
```typescript
// Test: User A request `/orders/<userB_order_id>`
// Verify: 404 (not 403, not 200)
```

### Path traversal
```typescript
// Test: GET /files/../../etc/passwd
// Verify: blocked
```

### Broken object level authorization
```typescript
// Test: User A của tenant-1 request `/orders/<tenant-2 order id>`
// Verify: 404
```

---

## Bug bounty / responsible disclosure

`/.well-known/security.txt`:
```
Contact: security@wecha.vn
Expires: 2027-01-01T00:00:00Z
Encryption: https://wecha.vn/pgp-key.txt
Acknowledgments: https://wecha.vn/security/hall-of-fame
Preferred-Languages: vi, en
Canonical: https://wecha.vn/.well-known/security.txt
Policy: https://wecha.vn/security/disclosure-policy
```

**Bounty (tham khảo):**
- Critical (RCE, SQL injection, auth bypass): 5-50M VND
- High (IDOR, sensitive data leak): 2-10M VND
- Medium (XSS, CSRF): 500k-2M VND
- Low (info disclosure): 100k-500k VND

---

**END Skill: Security Audit**
