# Rule 04 — API Design

> Mức độ: 🔴 CRITICAL | Áp dụng: Tạo/đổi endpoint

---

## 1. URL conventions (REST)

```
✅ ĐÚNG
GET    /api/v1/orders                    # List (cursor pagination)
GET    /api/v1/orders/:id                # Detail
POST   /api/v1/orders                    # Create
PATCH  /api/v1/orders/:id                # Partial update
PUT    /api/v1/orders/:id                # Full replace (hiếm)
DELETE /api/v1/orders/:id                # Soft delete

# Action không phải CRUD → POST + verb
POST   /api/v1/orders/:id/cancel
POST   /api/v1/orders/:id/refund
POST   /api/v1/orders/:id/duplicate
POST   /api/v1/invoices/:id/send-email
POST   /api/v1/users/:id/reset-password

# Nested (max 2 cấp)
GET    /api/v1/orders/:id/items
POST   /api/v1/customers/:id/notes

# Search/filter
GET    /api/v1/orders?status=paid&from=2026-01-01&cursor=xxx&limit=20
GET    /api/v1/orders?q=nguyễn+văn+a
GET    /api/v1/orders?fields=id,code,total

# Bulk
POST   /api/v1/orders/bulk-status        # { ids: [...], status: '...' }
POST   /api/v1/orders/bulk-export        # async, returns job ID

# Health & meta
GET    /api/v1/health                    # Liveness
GET    /api/v1/health/ready              # Readiness
GET    /api/v1/version                   # API version + build SHA

❌ SAI
GET    /api/getOrders                    # Verb trong URL
POST   /api/orders/delete/:id            # Nên DELETE
GET    /api/orders/list                  # Thừa "list"
GET    /api/v1/customers/:id/orders/:oid/items/:iid  # Nested quá sâu
GET    /api/orders                       # Thiếu version
POST   /api/v1/createOrder               # camelCase + verb
```

---

## 2. Response format chuẩn

### Success — single
```json
{
  "data": {
    "id": "clx123...",
    "code": "ORD-2026-00001",
    "status": "paid",
    "total": "1500000.00",
    "currency": "VND",
    "createdAt": "2026-04-26T10:00:00+07:00"
  },
  "meta": {
    "version": "1.0",
    "requestId": "req_01HQ..."
  }
}
```

### Success — list
```json
{
  "data": [/* items */],
  "meta": {
    "nextCursor": "clx456...",
    "prevCursor": "clx100...",
    "hasMore": true,
    "limit": 20,
    "totalEstimate": 1234
  }
}
```

### Success — async job
```json
{
  "data": {
    "jobId": "job_01HQ...",
    "status": "queued",
    "statusUrl": "/api/v1/jobs/job_01HQ..."
  }
}
```

### Error — RFC 7807 Problem Details
```json
{
  "error": {
    "code": "INSUFFICIENT_STOCK",
    "type": "https://docs.wecha.vn/errors/INSUFFICIENT_STOCK",
    "title": "Sản phẩm hết hàng",
    "status": 422,
    "detail": "Sản phẩm Trà sữa size M chỉ còn 5, không đủ 10",
    "instance": "/api/v1/orders",
    "fields": {
      "items.0.quantity": "Vượt quá tồn kho"
    },
    "context": {
      "productId": "clx789",
      "requested": 10,
      "available": 5
    },
    "traceId": "01HQ...",
    "requestId": "req_01HQ..."
  }
}
```

---

## 3. HTTP Status Codes

| Code | Khi nào | Body |
|---|---|---|
| 200 | GET/PATCH/PUT thành công | `{ data }` |
| 201 | POST tạo mới | `{ data }` + `Location` header |
| 202 | Async job started | `{ data: { jobId, statusUrl } }` |
| 204 | DELETE thành công | (empty) |
| 304 | ETag match | (empty) |
| 400 | Request malformed | `{ error }` |
| 401 | Chưa auth | `{ error }` |
| 402 | Plan giới hạn | `{ error }` |
| 403 | Đã auth nhưng không có quyền | `{ error }` |
| 404 | Resource không tồn tại HOẶC user không có quyền xem | `{ error }` |
| 405 | Method Not Allowed | `{ error }` |
| 409 | Conflict (unique, optimistic lock fail) | `{ error }` |
| 410 | Gone (đã xoá vĩnh viễn) | `{ error }` |
| 412 | Precondition Failed (If-Match mismatch) | `{ error }` |
| 413 | Payload Too Large | `{ error }` |
| 415 | Unsupported Media Type | `{ error }` |
| 422 | Validation fail (Zod parse error) | `{ error.fields }` |
| 423 | Resource locked | `{ error }` |
| 425 | Too Early (replay) | `{ error }` |
| 428 | Precondition Required | `{ error }` |
| 429 | Rate limit | `{ error }` + `Retry-After` |
| 500 | Server bug | `{ error: { traceId } }` |
| 502 | Upstream lỗi (VNPay down) | `{ error }` |
| 503 | Service Unavailable / maintenance | `{ error }` + `Retry-After` |
| 504 | Gateway Timeout | `{ error }` |

---

## 4. Versioning

**URL versioning (chính):**
```
/api/v1/...   ← Stable, support 24 tháng sau khi v2 ra
/api/v2/...   ← Breaking changes
```

**Khi nào bump version:**
- Đổi shape response (xoá field, đổi type, đổi semantic)
- Đổi auth mechanism
- Đổi error code semantic

**Khi nào KHÔNG bump:**
- Thêm field optional
- Thêm endpoint mới
- Thêm enum value mới (nếu client handle "unknown")
- Sửa bug

**Deprecation header:**
```http
X-API-Deprecated: true
X-API-Sunset: 2027-01-01
X-API-Migration-Guide: https://docs.wecha.vn/api/migration-v1-to-v2
Warning: 299 - "API v1 will be sunset on 2027-01-01"
```

---

## 5. Idempotency 🔴

```typescript
@Post('orders')
async createOrder(
  @Headers('idempotency-key') idempotencyKey: string,
  @Body() dto: CreateOrderDto,
  @CurrentUser() user: AuthUser,
) {
  if (!idempotencyKey) {
    throw new BadRequestException('Idempotency-Key header required');
  }
  
  if (!/^[a-zA-Z0-9_-]{16,128}$/.test(idempotencyKey)) {
    throw new BadRequestException('Invalid Idempotency-Key format');
  }
  
  const cacheKey = `idem:${user.tenantId}:${idempotencyKey}`;
  
  const cached = await redis.get(cacheKey);
  if (cached) {
    const { body } = JSON.parse(cached);
    return { ...body, meta: { ...body.meta, idempotent: true } };
  }
  
  // Acquire lock
  const lockKey = `idem-lock:${cacheKey}`;
  const acquired = await redis.set(lockKey, '1', 'NX', 'EX', 30);
  if (!acquired) {
    throw new ConflictException('Request with same key in progress');
  }
  
  try {
    const order = await this.ordersService.create(dto, user);
    const response = { data: order, meta: { requestId: req.id } };
    
    await redis.set(cacheKey, JSON.stringify({ status: 201, body: response }), 'EX', 86400);
    return response;
  } finally {
    await redis.del(lockKey);
  }
}
```

---

## 6. Mobile-ready checklist 🔴

Mỗi endpoint:
- [ ] Hoạt động qua REST thuần (không phụ thuộc cookie)
- [ ] Auth qua `Authorization: Bearer <token>` header
- [ ] Response < 100KB cho list (cap pagination)
- [ ] Pagination cursor-based
- [ ] Field naming **camelCase**
- [ ] Datetime ISO 8601 với timezone (`2026-04-26T10:00:00+07:00`)
- [ ] Hỗ trợ `If-None-Match` (ETag)
- [ ] `Cache-Control` header rõ ràng
- [ ] Error code stable (mobile dùng để show i18n)
- [ ] Hỗ trợ `Accept-Language` cho server-side i18n

### Mobile-friendly patterns

```typescript
// 1. Sparse fieldset
GET /api/v1/orders?fields=id,code,total,status

// 2. Compact list
GET /api/v1/orders/compact

// 3. Bulk
POST /api/v1/orders/bulk-status
{ "orderIds": ["..."], "status": "shipped" }

// 4. Sync endpoint (offline-first)
GET /api/v1/sync?since=2026-04-26T10:00:00Z&types=orders,products
{
  "data": {
    "orders": { "created": [...], "updated": [...], "deleted": ["id1"] },
    "products": { ... }
  },
  "meta": { "syncedAt": "2026-04-26T11:00:00Z" }
}

// 5. ETag
GET /api/v1/products/:id
Headers (response): ETag: "v123"

GET /api/v1/products/:id
Headers (request): If-None-Match: "v123"
→ 304 Not Modified (no body)
```

---

## 7. Webhook design (outgoing)

```
POST <webhook-url>
Headers:
  X-Wecha-Event: order.created
  X-Wecha-Signature: t=1714123200,v1=hmac_sha256_hex
  X-Wecha-Delivery: 01HQ...
  X-Wecha-Tenant: tenant_id

Body:
{
  "id": "evt_01HQ...",
  "type": "order.created",
  "createdAt": "2026-04-26T10:00:00+07:00",
  "tenantId": "...",
  "data": { /* event payload */ },
  "previousAttributes": null
}
```

**Signing:**
```typescript
function signWebhook(payload: string, secret: string): string {
  const timestamp = Math.floor(Date.now() / 1000);
  const signedPayload = `${timestamp}.${payload}`;
  const signature = crypto.createHmac('sha256', secret).update(signedPayload).digest('hex');
  return `t=${timestamp},v1=${signature}`;
}

function verifyWebhook(payload: string, header: string, secret: string): boolean {
  const parts = Object.fromEntries(header.split(',').map(p => p.split('=')));
  const timestamp = parseInt(parts.t);
  
  // Reject nếu > 5 phút (replay)
  if (Math.abs(Date.now() / 1000 - timestamp) > 300) return false;
  
  const expected = crypto.createHmac('sha256', secret)
    .update(`${timestamp}.${payload}`).digest('hex');
  
  return crypto.timingSafeEqual(Buffer.from(parts.v1), Buffer.from(expected));
}
```

**Retry:** 3 lần exponential backoff (1m, 5m, 30m). Sau 3 fail → mark inactive, alert admin. Lưu webhook delivery log.

---

## 8. OpenAPI auto-generation

```typescript
// main.ts
const config = new DocumentBuilder()
  .setTitle('WECHA ERP API')
  .setDescription('API cho hệ thống ERP và chuỗi WECHA')
  .setVersion('1.0')
  .addBearerAuth()
  .addServer('https://api.wecha.vn', 'Production')
  .addServer('https://api-staging.wecha.vn', 'Staging')
  .addTag('orders', 'Quản lý đơn hàng')
  .addTag('inventory', 'Quản lý kho')
  .build();

const document = SwaggerModule.createDocument(app, config);

if (process.env.NODE_ENV !== 'production') {
  SwaggerModule.setup('api/docs', app, document, {
    swaggerOptions: { persistAuthorization: true },
  });
}

// Export OpenAPI JSON cho mobile dev
fs.writeFileSync('./openapi.json', JSON.stringify(document, null, 2));
```

---

**END Rule 04**
