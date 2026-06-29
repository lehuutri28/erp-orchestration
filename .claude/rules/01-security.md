# Rule 01 — Bảo mật (OWASP / NIST / PCI / ISO)

> ⚠️ Đồng bộ ARCHITECTURE_CONTRACT 29/5/2026: PK=uuid, tenant_id=uuid, TS field camelCase, RBAC dấu `.`. Contract THẮNG nếu mâu thuẫn. Xem docs/ARCHITECTURE_CONTRACT.md.

> Mức độ: 🔴 CRITICAL | Áp dụng: Auth, payment, PII, upload, mọi endpoint

---

## OWASP Top 10 (2021) — MAPPING & MITIGATION

### A01 — Broken Access Control 🔴

**3 lớp defense bắt buộc:**

```typescript
// Lớp 1 — Guard ở controller
@Get('orders/:id')
@UseGuards(JwtAuthGuard, PermissionsGuard)
@RequirePermissions('orders.read')
async getOrder(@Param('id') id: string, @CurrentUser() user: AuthUser) {
  
  // Lớp 2 — Filter trong service
  const order = await this.ordersService.findByIdForUser(id, user);
  if (!order) throw new NotFoundException();  // 404, KHÔNG 403 (không leak existence)
  return order;
}

// Lớp 3 — Filter trong query
async findByIdForUser(id: string, user: AuthUser) {
  return db.query.orders.findFirst({
    where: and(
      eq(orders.id, id),
      eq(orders.tenant_id, user.tenantId),  // Tenant
      user.roles.includes('store_manager')
        ? inArray(orders.store_id, user.storeIds)  // Store-scoped
        : sql`true`
    ),
  });
}

// Lớp 4 — RLS Postgres (defense cuối, xem rule 02)
```

**Test bắt buộc** 🔴:
```typescript
it('user A KHÔNG truy cập order của user B (cùng tenant)', async () => {
  const res = await request(app).get(`/orders/${userBOrderId}`)
    .set('Authorization', `Bearer ${userAToken}`);
  expect(res.status).toBe(404);  // NOT 403
});

it('tenant A KHÔNG truy cập đơn của tenant B', async () => {
  const res = await request(app).get(`/orders/${tenantBOrderId}`)
    .set('Authorization', `Bearer ${tenantAToken}`);
  expect(res.status).toBe(404);
});
```

### A02 — Cryptographic Failures 🔴

**Bảng dữ liệu cần bảo vệ:**

| Loại | Cách bảo vệ |
|---|---|
| Password | Argon2id (m=19456, t=2, p=1) — NIST 800-63B |
| Session token | crypto.randomBytes(32), httpOnly cookie hoặc Bearer |
| API key bên thứ ba | AES-256-GCM, lưu `secrets` table mã hoá |
| CCCD / số TK ngân hàng | pgcrypto encrypt + SHA-256 hash để search |
| Lương nhân viên | Column-level encryption, chỉ HR admin decrypt |
| DB password | Doppler/Infisical/Vault, KHÔNG `.env` plain |
| TLS | TLS 1.3 only (Caddy default), HSTS preload |
| Backup file | AES-256 encrypt trước khi upload R2 |

**Code mẫu — Argon2:**
```typescript
import argon2 from 'argon2';

export async function hashPassword(password: string): Promise<string> {
  return argon2.hash(password, {
    type: argon2.argon2id,
    memoryCost: 19456, timeCost: 2, parallelism: 1,
  });
}

export async function verifyPassword(hash: string, password: string): Promise<boolean> {
  try { return await argon2.verify(hash, password); }
  catch { return false; }
}
```

**Code mẫu — Encrypt CCCD:**
```typescript
// Schema
national_id_encrypted: customType<{data: string}>({dataType: () => 'bytea'})('national_id_encrypted'),
national_id_hash: text('national_id_hash'),  // SHA-256 để search

// Insert
await db.execute(sql`
  INSERT INTO customers (id, name, national_id_encrypted, national_id_hash) VALUES (
    ${id}, ${name},
    pgp_sym_encrypt(${nationalId}, ${process.env.PGP_KEY}),
    encode(digest(${nationalId}, 'sha256'), 'hex')
  )
`);

// Search
await db.execute(sql`
  SELECT id, name, pgp_sym_decrypt(national_id_encrypted, ${process.env.PGP_KEY}) AS national_id
  FROM customers WHERE national_id_hash = encode(digest(${searchValue}, 'sha256'), 'hex')
`);
```

**🔴 CẤM TUYỆT ĐỐI:** MD5/SHA-1 cho password, AES-ECB, tự viết crypto, lưu private key trong git.

### A03 — Injection 🔴

```typescript
// SQL Injection
// ❌ String concat
await db.execute(`SELECT * FROM users WHERE email = '${email}'`);

// ✅ Drizzle query builder (auto parameterize)
await db.select().from(users).where(eq(users.email, email));

// ✅ sql tag với placeholder
await db.execute(sql`SELECT * FROM users WHERE email = ${email}`);

// NoSQL Injection (Redis)
// ❌ User input thành key
await redis.get(`session:${userInput}`);

// ✅ Validate trước
if (!/^[a-zA-Z0-9_-]+$/.test(userInput)) throw new BadRequestException();

// Command Injection
// ❌ exec(`pdftotext ${userFilename}`);
// ✅
import { spawn } from 'child_process';
spawn('pdftotext', [userFilename], { shell: false });

// XSS
// ❌ <div dangerouslySetInnerHTML={{ __html: userContent }} />
// ✅ <div>{userContent}</div>  // React auto-escape

// ✅ Cho rich text — sanitize
import DOMPurify from 'isomorphic-dompurify';
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userContent) }} />
```

### A04 — Insecure Design 🟠

**Threat modeling BẮT BUỘC** trước khi build feature có động chạm:
- Tiền → race condition, double charge, replay
- PII → leak qua API, log
- Public endpoint → abuse, scraping, DoS

**Output:** `docs/threat-models/<feature>.md` theo format STRIDE (Spoofing/Tampering/Repudiation/Information Disclosure/DoS/Elevation of Privilege).

### A05 — Security Misconfiguration 🟠

**main.ts BẮT BUỘC:**
```typescript
import helmet from 'helmet';

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", 'data:', 'blob:', 'https:'],
      connectSrc: ["'self'", 'wss:', 'https:'],
      objectSrc: ["'none'"],
      frameAncestors: ["'none'"],
    },
  },
  hsts: { maxAge: 31536000, includeSubDomains: true, preload: true },
  frameguard: { action: 'deny' },
}));

// CORS — KHÔNG '*'
const allowedOrigins = (process.env.CORS_ORIGINS ?? '').split(',').filter(Boolean);
app.enableCors({
  origin: (origin, cb) => {
    if (!origin) return cb(null, true);
    if (allowedOrigins.includes(origin)) return cb(null, true);
    cb(new Error(`CORS: ${origin} not allowed`));
  },
  credentials: true,
  methods: ['GET','POST','PUT','PATCH','DELETE','OPTIONS'],
  allowedHeaders: ['Content-Type','Authorization','X-Tenant-Id','X-Request-Id','X-Idempotency-Key'],
  exposedHeaders: ['X-RateLimit-Remaining','X-RateLimit-Reset'],
  maxAge: 86400,
});

app.use(json({ limit: '1mb' }));
app.disable('x-powered-by');
app.set('trust proxy', 1);
```

**🔴 CẤM:**
- `swagger` enable production (chỉ staging/dev)
- Default credentials (postgres/postgres, redis no password)
- Stack trace expose ra client
- Debug log có data nhạy cảm

### A06 — Vulnerable Components 🟠

```bash
# Tuần
pnpm audit --audit-level=high
pnpm dlx snyk test

# Khi có CVE
# CVSS ≥ 9.0 → hotfix 24h
# CVSS ≥ 7.0 → update 7 ngày
```

CI bắt buộc:
```yaml
- run: pnpm audit --audit-level=high
- run: pnpm dlx license-checker --failOn 'GPL-3.0'
```

### A07 — Auth Failures 🔴

**Password policy (NIST 800-63B AAL2):**
- Min 12 ký tự
- Check breach qua haveibeenpwned k-anonymity
- Không có common password (top 10000)
- Không có user info (email, name)

**Account lockout:** 10 lần fail → lock, reset password để unlock.

**JWT TTL:** Access 15 phút, Refresh 7 ngày.

**Refresh token rotation** (BẮT BUỘC):
```typescript
@Post('refresh')
async refresh(@Body() { refreshToken }: RefreshDto) {
  const payload = await this.verifyRefreshToken(refreshToken);
  
  // Check chưa bị revoke
  const isBlacklisted = await this.cache.get(`refresh_blacklist:${payload.jti}`);
  if (isBlacklisted) {
    // Token reuse — security incident, revoke ALL
    await this.revokeAllTokensForUser(payload.sub);
    throw new UnauthorizedException('Token reuse detected');
  }
  
  // Blacklist token cũ
  await this.cache.set(`refresh_blacklist:${payload.jti}`, '1', 'EX', 7 * 24 * 3600);
  
  return this.issueTokens(payload.sub);
}
```

**2FA TOTP** (Wave 2):
```typescript
import { authenticator } from 'otplib';
import QRCode from 'qrcode';

// Setup
const secret = authenticator.generateSecret();
const otpauth = authenticator.keyuri(user.email, 'WECHA ERP', secret);
const qrDataUrl = await QRCode.toDataURL(otpauth);

// Verify
authenticator.options = { window: 1 };  // ±30s clock skew
return authenticator.verify({ token, secret });

// Backup codes (10 cái, hash Argon2, dùng 1 lần)
const backupCodes = Array.from({length: 10}, () => randomBytes(5).toString('hex'));
await db.insert(backup_codes).values(
  backupCodes.map(code => ({user_id, code_hash: argon2.hash(code), used: false}))
);
```

**Login response code (KHÔNG leak info):**
```typescript
// ❌ "Wrong password" hoặc "User not found" → leak existence
// ✅ "Invalid credentials" cho cả 2 case
```

### A08 — Software/Data Integrity 🟡

- Lockfile committed (`pnpm-lock.yaml`)
- CI: `pnpm install --frozen-lockfile`
- Docker: pin digest (`node:20-alpine@sha256:...`)
- CDN script: SRI (Subresource Integrity)
- Webhook: verify HMAC signature + timing-safe compare

```typescript
// Verify VNPay webhook
function verifyVnpayIpn(query, secret): boolean {
  const receivedHash = query.vnp_SecureHash;
  delete query.vnp_SecureHash;
  delete query.vnp_SecureHashType;
  
  const sortedKeys = Object.keys(query).sort();
  const signData = sortedKeys.map(k => `${k}=${query[k]}`).join('&');
  const expectedHash = crypto.createHmac('sha512', secret).update(signData).digest('hex');
  
  return crypto.timingSafeEqual(Buffer.from(receivedHash), Buffer.from(expectedHash));
}
```

### A09 — Logging & Monitoring 🔴

**BẮT BUỘC log:**

| Sự kiện | Detail | Retention |
|---|---|---|
| Login success | user_id, ip, ua, geo | 1 năm |
| Login fail | email, ip, ua, reason | 1 năm |
| Logout | user_id, jti, ip | 90 ngày |
| Password change | user_id, ip, by_admin | **10 năm** (TT 99) |
| 2FA enable/disable | user_id, ip, method | 7 năm |
| Role change | actor_id, target_id, before, after | 7 năm |
| Tài chính (invoice/payment/refund) | snapshot before/after | **10 năm** (TT 99) |
| Export PII | actor_id, query, row_count, ip | 7 năm |
| AI agent action | agent_id, action, input, output, confidence | 2 năm |
| Failed permission check | user_id, action, resource | 90 ngày |
| Webhook signature fail | source, ip, payload_hash | 1 năm |

**CẤM log:** Password (plain/hash), token full, CCCD, số thẻ, API key, cookie value.

> 📎 **Cross-ref:** Audit log table schema (`audit_logs`, CREATE INDEX, REVOKE UPDATE/DELETE) xem [rules/11-logging-observability.md](./11-logging-observability.md#audit-log)

**Pino redact list (BẮT BUỘC):**

> 📎 **Cross-ref:** Pino redact list canonical (đầy đủ field) xem [rules/11-logging-observability.md](./11-logging-observability.md#pino-redact-list)
```typescript
const logger = pino({
  redact: {
    paths: [
      'password', '*.password', '*.password_hash',
      'token', 'refreshToken', 'accessToken', '*.token',
      'authorization', 'req.headers.authorization', 'req.headers.cookie',
      'apiKey', 'api_key', 'secret', '*.secret',
      'creditCard', 'cardNumber', 'cvv',
      'ssn', 'national_id', 'cccd',
      'email',  // PII — chỉ log hash hoặc partial
    ],
    censor: '[REDACTED]',
  },
});
```

### A10 — SSRF 🟠

```typescript
const BLOCKED_HOSTS = ['localhost','127.0.0.1','0.0.0.0','::1','169.254.169.254'];

async function safeFetch(userUrl: string): Promise<Response> {
  const url = new URL(userUrl);
  
  if (!['http:','https:'].includes(url.protocol)) throw new BadRequestException('HTTP(S) only');
  if (BLOCKED_HOSTS.includes(url.hostname.toLowerCase())) throw new BadRequestException('Blocked');
  
  const addrs = await dns.resolve4(url.hostname);
  for (const addr of addrs) if (isPrivateIP(addr)) throw new BadRequestException('Private network');
  
  const controller = new AbortController();
  setTimeout(() => controller.abort(), 10_000);
  
  return fetch(userUrl, {
    signal: controller.signal,
    redirect: 'manual',
    headers: { 'User-Agent': 'WECHA-ERP/1.0' },
  });
}
```

---

## OWASP API Security Top 10 (2023)

### API1 — BOLA → đã cover A01

### API2 — Broken Auth → đã cover A07

### API3 — Property-Level Authorization 🔴

```typescript
// ❌ User update được cả role!
@Patch('users/:id')
async updateUser(@Param('id') id, @Body() dto: any) {
  return this.usersService.update(id, dto);  // dto.role = 'admin' bypass
}

// ✅ Whitelist field theo role với .strict()
const userSelfUpdateSchema = z.object({
  fullName: z.string().min(1).max(100).optional(),
  avatar: z.string().url().optional(),
  phone: z.string().regex(/^0[0-9]{9}$/).optional(),
  // KHÔNG có role, permissions, tenant_id
}).strict();

const userAdminUpdateSchema = userSelfUpdateSchema.extend({
  roles: z.array(z.string()).optional(),
  is_active: z.boolean().optional(),
}).strict();

@Patch('users/:id')
async updateUser(@Param('id') id, @Body() body, @CurrentUser() user) {
  const isSelf = user.id === id;
  const isAdmin = user.roles.includes('admin');
  if (!isSelf && !isAdmin) throw new ForbiddenException();
  
  const schema = isAdmin ? userAdminUpdateSchema : userSelfUpdateSchema;
  const dto = schema.parse(body);  // throw 422 nếu field thừa
  return this.usersService.update(id, dto);
}
```

### API4 — Resource Consumption 🟠

**Rate limit nhiều chiều:**
```typescript
ThrottlerModule.forRootAsync({
  useFactory: () => ({
    throttlers: [
      { name: 'default', ttl: 60_000, limit: 100 },     // 100 req/phút/IP
      { name: 'strict',  ttl: 60_000, limit: 5 },        // 5 req/phút (login)
      { name: 'export',  ttl: 3600_000, limit: 10 },     // 10 export/giờ
    ],
    storage: new ThrottlerStorageRedisService(redis),
  }),
})

@Throttle({ strict: { limit: 5, ttl: 60_000 } })
@Post('auth/login')

@Throttle({ export: { limit: 10, ttl: 3600_000 } })
@Get('reports/export')
```

**File upload limits:**
- Image: 10 MB
- Video: 50 MB
- Document: 5 MB
- Max files/request: 10
- Max total request: 100 MB

**Pagination forced:**
```typescript
const limit = Math.min(query.limit ?? 20, 100);  // Cap 100
```

### API5 — Function-Level Authorization 🔴

CASL ABAC (Wave 4):
```typescript
import { AbilityBuilder, createMongoAbility } from '@casl/ability';

export function defineAbilitiesFor(user: AuthUser) {
  const { can, cannot, build } = new AbilityBuilder(createMongoAbility);
  
  can('read', 'User', { id: user.id });
  can('update', 'User', { id: user.id });
  cannot('update', 'User', ['roles', 'permissions', 'tenant_id']);
  
  if (user.roles.includes('store_manager')) {
    can('read', 'Order', { store_id: { $in: user.storeIds } });
    can('update', 'Order', { store_id: { $in: user.storeIds } });
    cannot('delete', 'Order');
  }
  
  if (user.roles.includes('finance_admin')) {
    can('manage', 'Invoice');
    can('create', 'Refund');
  }
  
  if (user.roles.includes('super_admin')) {
    can('manage', 'all');
    cannot('delete', 'AuditLog');  // KHÔNG ai xoá audit
  }
  
  return build();
}
```

### API6 — Sensitive Business Flows 🟠

- Captcha cho register, password reset (Cloudflare Turnstile)
- Email verify trước login lần đầu
- SMS OTP cho action giá trị cao
- Velocity check: 1 user không tạo > 5 đơn 100M VND/giờ

### API7 — SSRF → đã cover A10

### API8 — Misconfiguration → đã cover A05

### API9 — Inventory Management 🟡

URL versioning:
```
/api/v1/...   ← Stable, support 24 tháng sau khi v2 ra
/api/v2/...   ← Breaking changes
```

Deprecation header:
```http
X-API-Deprecated: true
X-API-Sunset: 2027-01-01
X-API-Migration-Guide: https://docs.wecha.vn/api/migration-v2
```

OpenAPI export mỗi build → CI artifact cho mobile dev.

### API10 — Unsafe API Consumption 🟠

```typescript
import { z } from 'zod';
import CircuitBreaker from 'opossum';

const ghnResponseSchema = z.object({
  code: z.number(),
  message: z.string(),
  data: z.object({
    order_code: z.string(),
    expected_delivery_time: z.string().datetime(),
  }).optional(),
});

const breaker = new CircuitBreaker(
  async (orderData) => {
    const res = await fetch('https://api.ghn.vn/v2/order/create', {
      method: 'POST',
      headers: { 'Token': process.env.GHN_TOKEN, 'Content-Type': 'application/json' },
      body: JSON.stringify(orderData),
      signal: AbortSignal.timeout(10_000),
    });
    if (!res.ok) throw new Error(`GHN ${res.status}`);
    return ghnResponseSchema.parse(await res.json());
  },
  { timeout: 10_000, errorThresholdPercentage: 50, resetTimeout: 5 * 60_000 }
);
```

---

## File upload checklist 🔴

```typescript
import sharp from 'sharp';
import { fileTypeFromBuffer } from 'file-type';

async function safeUpload(file: Express.Multer.File): Promise<string> {
  // 1. Size check
  if (file.size > 10 * 1024 * 1024) throw new BadRequestException('File too large');
  
  // 2. MIME check (client-provided, không tin)
  const ALLOWED_MIME = ['image/jpeg', 'image/png', 'image/webp'];
  if (!ALLOWED_MIME.includes(file.mimetype)) throw new BadRequestException('Invalid MIME');
  
  // 3. Magic bytes check (server-side verify)
  const detected = await fileTypeFromBuffer(file.buffer);
  if (!detected || !ALLOWED_MIME.includes(detected.mime)) {
    throw new BadRequestException('File content does not match MIME type');
  }
  
  // 4. Re-encode để strip metadata + virus payload
  const safeBuffer = await sharp(file.buffer).rotate().toBuffer();
  
  // 5. Random filename (KHÔNG tin client filename)
  const safeName = `${createId()}.${detected.ext}`;
  
  // 6. Upload với Content-Type chuẩn
  const url = await this.storage.upload(safeBuffer, safeName, detected.mime);
  
  // 7. Audit log
  await this.audit.log({
    action: 'file.uploaded',
    file_name_original: file.originalname,
    file_name_stored: safeName,
    size: file.size,
    mime: detected.mime,
  });
  
  return url;
}
```

---

## Self-check trước commit 🔴

- [ ] Mọi user input đã validate (Zod)?
- [ ] Mọi DB query có `tenant_id` filter?
- [ ] Không hardcode secret/API key?
- [ ] Không log password/token/PII?
- [ ] Auth guard cho mọi endpoint (trừ public whitelist)?
- [ ] Authorization (CASL/permissions) cho mutation?
- [ ] Audit log cho action nhạy cảm?
- [ ] Rate limit cho endpoint nhạy cảm?
- [ ] File upload check MIME + size + magic bytes?
- [ ] CORS không có `*`?
- [ ] Không trả password/internal field trong response?
- [ ] Test: cross-tenant + IDOR + permission?

---

**END Rule 01**
