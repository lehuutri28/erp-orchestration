# Skill: E-commerce Sync (Shopee / Lazada / Tiki)

> ⚠️ Đồng bộ ARCHITECTURE_CONTRACT 29/5/2026: PK=uuid, tenant_id=uuid, TS camelCase, RBAC dấu `.`. Xem docs/ARCHITECTURE_CONTRACT.md.

> Khi nào load: Tích hợp sàn TMĐT, webhook xử lý, sync đơn hàng, đồng bộ tồn kho
> Module: `apps/api/src/modules/ecommerce/` — gồm `platform-shopee`, `platform-lazada`, `platform-tiki`

---

## 1. Kiến trúc tổng quan

```
[Sàn TMĐT] ──webhook──▶ [ERP Webhook Endpoint]
                          │
                          ├─ Verify signature (HMAC)
                          ├─ Idempotency check (event_id)
                          ├─ Map to internal schema
                          └─ Enqueue pg-boss job
                               │
                               ├─ order.pull      → Tạo order nội bộ
                               ├─ inventory.push  → Cập nhật tồn kho lên sàn
                               └─ status.push     → Cập nhật trạng thái đơn
```

**ERP là source-of-truth:**
- Tồn kho: ERP → Sàn (push khi có thay đổi)
- Đơn hàng: Sàn → ERP (pull qua webhook hoặc polling)
- Conflict: ERP thắng trong mọi trường hợp

---

## 2. Webhook receive + signature verify

### Shopee

```typescript
// ecommerce/platform-shopee/controllers/shopee-webhook.controller.ts
@Post('webhook/shopee')
@HttpCode(200)
async handleShopeeWebhook(
  @Headers('Authorization') auth: string,
  @RawBody() rawBody: Buffer,
  @Body() body: ShopeeWebhookPayload,
): Promise<{ msg: string }> {
  // 1. Verify signature trước khi làm bất cứ điều gì
  await this.shopeeWebhookService.verifySignature(rawBody, auth);

  // 2. Idempotency — check đã xử lý chưa
  const alreadyProcessed = await this.idempotencyService.check(
    `shopee:webhook:${body.code}:${body.timestamp}`,
  );
  if (alreadyProcessed) return { msg: 'ok' };

  // 3. Enqueue job — KHÔNG xử lý sync trong webhook handler
  await this.pgBoss.send(
    `shopee:${this.mapWebhookType(body.code)}`,
    body,
    { key: `shopee:${body.code}:${body.timestamp}` },
  );

  return { msg: 'ok' };  // Phải trả 200 ngay, không để sàn timeout retry liên tục
}

// Verify Shopee HMAC-SHA256
async verifySignature(rawBody: Buffer, authHeader: string): Promise<void> {
  const expected = crypto
    .createHmac('sha256', process.env.SHOPEE_PUSH_SECRET)
    .update(rawBody)
    .digest('hex');

  // Constant-time compare để chống timing attack
  const received = authHeader?.replace('sha256=', '') ?? '';
  if (!crypto.timingSafeEqual(
    Buffer.from(expected, 'hex'),
    Buffer.from(received, 'hex').subarray(0, expected.length / 2),
  )) {
    throw new UnauthorizedException('Shopee webhook signature invalid');
  }
}
```

### Lazada

```typescript
// Lazada dùng query param signature
async verifyLazadaSignature(query: Record<string, string>): Promise<void> {
  const { sign, ...params } = query;
  const sortedParams = Object.entries(params)
    .sort(([a], [b]) => a.localeCompare(b))
    .map(([k, v]) => `${k}${v}`)
    .join('');

  const expected = crypto
    .createHmac('sha256', process.env.LAZADA_APP_SECRET)
    .update(sortedParams)
    .digest('hex')
    .toUpperCase();

  if (expected !== sign?.toUpperCase()) {
    throw new UnauthorizedException('Lazada signature invalid');
  }
}
```

---

## 3. Webhook event type mapping

| Platform | Code | Loại sự kiện | Job queue |
|---------|------|-------------|-----------|
| Shopee | 3 | Order created | `shopee:order.created` |
| Shopee | 4 | Order item cancelled | `shopee:order.cancelled` |
| Shopee | 5 | Order shipped | `shopee:order.shipped` |
| Shopee | 6 | Fulfillment done | `shopee:order.completed` |
| Lazada | order_created | Order created | `lazada:order.created` |
| Lazada | order_cancelled | Order cancelled | `lazada:order.cancelled` |
| Tiki | ORDER_CREATED | Order created | `tiki:order.created` |
| Tiki | ORDER_CANCELLED | Order cancelled | `tiki:order.cancelled` |

---

## 4. Order pull — tạo ERP order từ sàn

```typescript
@Injectable()
export class ShopeeOrderSyncWorker implements OnModuleInit {
  async onModuleInit() {
    await this.boss.work<ShopeeOrderPayload>(
      'shopee:order.created',
      { teamSize: 3 },
      this.handleOrderCreated.bind(this),
    );
  }

  async handleOrderCreated(job: PgBoss.Job<ShopeeOrderPayload>): Promise<void> {
    const shopeeOrder = job.data;

    // 1. Check đơn chưa tồn tại trong ERP (idempotency ở DB layer)
    const existing = await this.ordersRepo.findByExternalId(
      'shopee',
      shopeeOrder.ordersn,
    );
    if (existing) return;  // Đã sync rồi

    // 2. Fetch đơn chi tiết từ Shopee API (webhook chỉ có ordersn)
    const detail = await this.shopeeApiClient.getOrderDetail(shopeeOrder.ordersn);

    // 3. Map sang ERP schema
    const erpOrder = this.shopeeMapper.toErpOrder(detail, tenantId);

    // 4. Tạo order trong ERP với transaction
    await this.db.transaction(async (tx) => {
      const [order] = await tx.insert(orders).values(erpOrder).returning();

      // Ghi external_id để track nguồn gốc
      await tx.insert(orderExternalRefs).values({
        orderId: order.id,      // camelCase TS field
        platform: 'shopee',
        externalId: shopeeOrder.ordersn,
        externalData: detail,
      });
    });

    // 5. Trigger inventory reservation
    await this.boss.send('inventory:reserve', {
      orderId: erpOrder.id,
      source: 'shopee',
    });
  }
}
```

---

## 5. Inventory sync 2-way — ERP ↔ Sàn

### ERP → Sàn (push khi stock thay đổi)

```typescript
// Trigger khi tồn kho thay đổi trong ERP
@OnEvent('inventory.stock_changed')
async onStockChanged(event: StockChangedEvent): Promise<void> {
  const { productId, tenantId, newQuantity } = event;

  // Tìm tất cả sàn mà sản phẩm đang bán
  const listings = await this.listingsRepo.findByProduct(productId);

  for (const listing of listings) {
    await this.boss.send(
      `${listing.platform}:inventory.update`,
      {
        externalProductId: listing.external_id,
        shopId: listing.shop_id,
        quantity: newQuantity,
        tenantId,
      },
      { key: `${listing.platform}:inventory:${listing.id}:${Date.now()}` },
    );
  }
}

// Worker push lên Shopee
async pushInventoryToShopee(job: PgBoss.Job<ShopeeInventoryUpdatePayload>) {
  const { externalProductId, shopId, quantity } = job.data;

  await this.shopeeApi.updateStock({
    shop_id: shopId,
    item_id: externalProductId,
    stock: Math.max(0, quantity),  // Không push số âm
  });
}
```

### Sàn → ERP (reconcile định kỳ)

```typescript
// Cron 4h — so sánh tồn kho ERP vs sàn
@Cron('0 */4 * * *')
async reconcileInventory() {
  const tenants = await this.tenantsRepo.findAllWithEcommerceEnabled();

  for (const tenant of tenants) {
    await this.boss.send(
      'inventory:reconcile-ecommerce',
      { tenantId: tenant.id },
      { key: `reconcile:${tenant.id}:${new Date().toISOString().slice(0,13)}` },
    );
  }
}
```

---

## 6. Conflict resolution — ERP thắng

```typescript
// Khi nhận tồn kho từ sàn khác ERP
async resolveInventoryConflict(
  erpQty: number,
  platformQty: number,
  productId: string,
  platform: string,
): Promise<void> {
  if (erpQty !== platformQty) {
    // ERP là source-of-truth → push ERP qty lên sàn
    await this.boss.send(`${platform}:inventory.update`, {
      quantity: erpQty,
      // ...
    });

    // Log để audit
    await this.auditLog.log({
      action: 'inventory.conflict_resolved',
      metadata: {
        productId, platform,
        erpQty, platformQty,
        resolution: 'erp_wins',
      },
    });
  }
}
```

---

## 7. Rate limit per platform

| Platform | API rate limit | Strategy |
|---------|---------------|---------|
| Shopee | 1000 calls/min per shop | pg-boss throttle: `teamSize: 5` |
| Lazada | 100 calls/min | `teamSize: 2`, delay 600ms |
| Tiki | 60 calls/min | `teamSize: 1`, delay 1000ms |

```typescript
// Lazada — rate limit với throttle
await this.boss.send('lazada:order.fetch', payload, {
  startAfter: new Date(Date.now() + 600),  // Delay 600ms giữa các job
});
```

---

## 8. Anti-pattern ❌

| Anti-pattern | Vấn đề |
|-------------|--------|
| Xử lý sync trong webhook handler | Sàn timeout 5s → retry lũy thừa |
| Không verify signature | Bất kỳ ai gọi webhook được |
| Không idempotency check | Duplicate order khi sàn retry webhook |
| Sàn là source-of-truth cho tồn kho | Conflict và oversell |
| Push số âm lên sàn | Shopee/Lazada block shop |
| Không log external API call | Không debug được khi sàn thay đổi format |
| Gọi Shopee API trực tiếp trong webhook | Tăng latency, risk timeout |

---

## 9. Checklist tích hợp sàn mới

- [ ] Đọc API docs chính thức + ghi chú rate limit
- [ ] Tạo module `platform-<name>` theo cấu trúc Shopee
- [ ] Implement signature verify (constant-time compare)
- [ ] Webhook handler chỉ verify + enqueue, không xử lý
- [ ] Idempotency key trên mọi pg-boss job
- [ ] Rate limit config phù hợp quota sàn
- [ ] `order_external_refs` table để track external ID
- [ ] Alert khi API sàn trả error 4xx/5xx liên tiếp
- [ ] Test: webhook replay → không tạo duplicate order
- [ ] Test: sàn rate limit → không crash ERP

---

## Cross-ref

- `skills/api-development/SKILL.md` — Endpoint pattern + auth
- `skills/n8n-workflow/SKILL.md` — n8n cho simple sync workflow
- `rules/19-queue-pgboss.md` — Job retry + dead-letter
- `rules/14-event-driven.md` — Outbox pattern + idempotency
- `rules/01-security.md` — Signature verify, webhook security
- `rules/02-multi-tenant.md` — Tenant isolation cho listing/order
