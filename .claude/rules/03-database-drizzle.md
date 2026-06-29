# Rule 03 — Database & Drizzle Pattern

> ⚠️ Đồng bộ ARCHITECTURE_CONTRACT 29/5/2026: PK=uuid, tenant_id=uuid, TS field camelCase, RBAC dấu `.`. Contract THẮNG nếu mâu thuẫn. Xem docs/ARCHITECTURE_CONTRACT.md.

> Mức độ: 🔴 CRITICAL | Áp dụng: Schema, migration, query

---

## 1. Schema definition pattern

**Mọi bảng nghiệp vụ PHẢI theo template:**

```typescript
// apps/api/src/database/schema/orders.ts
import {
  pgTable, uuid, text, timestamp, integer, decimal, jsonb,
  boolean, uniqueIndex, index, pgEnum,
} from 'drizzle-orm/pg-core';
import { sql, relations } from 'drizzle-orm';
import { tenants } from './tenants';
import { customers } from './customers';
import { users } from './users';

// ===== Enums =====
export const orderStatusEnum = pgEnum('order_status', [
  'draft', 'confirmed', 'paid', 'shipped',
  'completed', 'cancelled', 'refunded',
]);

// ===== Table =====
export const orders = pgTable('orders', {
  // ID — uuid (chuẩn duy nhất cho bảng mới)
  id: uuid('id').primaryKey().defaultRandom(),
  
  // Multi-tenant (BẮT BUỘC)
  tenantId: uuid('tenant_id').notNull()
    .references(() => tenants.id, { onDelete: 'restrict' }),
  
  // Business code (cho user thấy)
  // → Sinh mã qua PG sequence {PREFIX}-{YYYYMMDD}-{seq}: xem CONTRACT mục 8 (CẤM Math.random + COUNT+1)
  code: text('code').notNull(),  // VD: ORD-20260529-00001
  
  // Business fields
  customerId: uuid('customer_id').notNull()
    .references(() => customers.id, { onDelete: 'restrict' }),
  
  status: orderStatusEnum('status').notNull().default('draft'),
  
  // Tiền: LUÔN dùng decimal
  subtotal: decimal('subtotal', { precision: 15, scale: 2 }).notNull(),
  taxAmount: decimal('tax_amount', { precision: 15, scale: 2 }).notNull(),
  discountAmount: decimal('discount_amount', { precision: 15, scale: 2 }).notNull().default('0'),
  total: decimal('total', { precision: 15, scale: 2 }).notNull(),
  currency: text('currency').notNull().default('VND'),
  
  // Snapshot — KHÔNG join để hiển thị
  customerSnapshot: jsonb('customer_snapshot').$type<{
    name: string;
    phone: string;
    address: string;
  }>().notNull(),
  
  metadata: jsonb('metadata').$type<Record<string, unknown>>(),
  
  internalNote: text('internal_note'),
  customerNote: text('customer_note'),
  
  // Audit (BẮT BUỘC)
  createdAt: timestamp('created_at', { mode: 'date', withTimezone: true })
    .notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { mode: 'date', withTimezone: true })
    .notNull().defaultNow().$onUpdate(() => new Date()),
  createdBy: uuid('created_by').notNull()
    .references(() => users.id, { onDelete: 'restrict' }),
  updatedBy: uuid('updated_by').references(() => users.id),
  
  // Soft delete (BẮT BUỘC)
  // → Soft delete 3 cấp + thùng rác: xem ARCHITECTURE_CONTRACT mục 1 (delete_status, purge_after, duyệt xoá)
  deletedAt: timestamp('deleted_at', { mode: 'date', withTimezone: true }),
  deletedBy: uuid('deleted_by').references(() => users.id),
  deletedReason: text('deleted_reason'),
  
  // Optimistic locking
  version: integer('version').notNull().default(0),
}, (t) => ({
  // Unique trong tenant
  codeUnique: uniqueIndex('orders_tenant_code_unique').on(t.tenantId, t.code),
  
  // Indexes
  tenantIdx: index('orders_tenant_idx').on(t.tenantId),
  tenantStatusIdx: index('orders_tenant_status_idx').on(t.tenantId, t.status),
  tenantCreatedIdx: index('orders_tenant_created_idx').on(t.tenantId, t.createdAt),
  customerIdx: index('orders_customer_idx').on(t.tenantId, t.customerId),
  
  // Partial index cho active
  activeStatusIdx: index('orders_active_status_idx')
    .on(t.tenantId, t.status)
    .where(sql`${t.deletedAt} IS NULL`),
}));

// ===== Relations (cho query.with()) =====
export const ordersRelations = relations(orders, ({ one, many }) => ({
  tenant: one(tenants, { fields: [orders.tenantId], references: [tenants.id] }),
  customer: one(customers, { fields: [orders.customerId], references: [customers.id] }),
  creator: one(users, { fields: [orders.createdBy], references: [users.id] }),
}));

// ===== Inferred types (export) =====
export type Order = typeof orders.$inferSelect;
export type NewOrder = typeof orders.$inferInsert;
```

### Quy tắc bắt buộc:

| Rule | Mức |
|---|---|
| Mọi bảng nghiệp vụ có `tenant_id` notNull + FK + index | 🔴 |
| Mọi bảng có `created_at, updated_at, created_by, deleted_at` | 🔴 |
| Tiền: `decimal(15, 2)` — KHÔNG `numeric` không scale, KHÔNG `real`/`double` | 🔴 |
| Datetime: `timestamp(... withTimezone: true)` | 🔴 |
| ID: `uuid().primaryKey().defaultRandom()` 🔴 (cuid2 = legacy, KHÔNG dùng bảng mới) | 🔴 |
| Enum: `pgEnum`, KHÔNG string tự do | 🟠 |
| FK luôn có `onDelete` rõ ràng (`restrict`/`cascade`/`set null`) | 🔴 |
| Có `version` cho optimistic locking nếu concurrent update | 🟡 |
| Index: FK + cột query thường + `(tenant_id, *)` composite | 🟠 |
| Partial index cho `WHERE deleted_at IS NULL` | 🟢 |
| Snapshot field cho data có thể đổi (giá tại thời điểm đặt) | 🟠 |

---

## 2. Query patterns

### 2.1 Pagination — CURSOR-BASED (BẮT BUỘC)

```typescript
// ✅ ĐÚNG — cursor pagination
async listOrders(params: { cursor?: string; limit: number; status?: OrderStatus }) {
  const limit = Math.min(params.limit, 100);  // Cap
  
  const items = await db.query.orders.findMany({
    where: and(
      tenantFilter(orders),
      notDeleted(orders),
      params.status ? eq(orders.status, params.status) : undefined,
      params.cursor ? lt(orders.id, params.cursor) : undefined,
    ),
    orderBy: [desc(orders.createdAt), desc(orders.id)],
    limit: limit + 1,  // +1 để biết hasMore
  });
  
  const hasMore = items.length > limit;
  const data = hasMore ? items.slice(0, -1) : items;
  
  return {
    data,
    meta: {
      nextCursor: hasMore ? data[data.length - 1].id : null,
      hasMore,
      limit,
    },
  };
}

// ❌ SAI — offset (chậm với data lớn)
async listOrders(page: number, pageSize: number) {
  return db.select().from(orders).limit(pageSize).offset(page * pageSize);
}
```

### 2.2 N+1 — LUÔN dùng `with` hoặc `leftJoin`

```typescript
// ❌ N+1
const ordersList = await db.select().from(orders);
for (const order of ordersList) {
  const customer = await db.select().from(customers).where(eq(customers.id, order.customerId));
}

// ✅ Relational query
const result = await db.query.orders.findMany({
  with: {
    customer: { columns: { id: true, name: true, phone: true } },
    items: {
      with: {
        product: { columns: { id: true, name: true, sku: true } },
      },
    },
  },
});

// ✅ leftJoin manual control
const result = await db
  .select({
    order: orders,
    customer: { id: customers.id, name: customers.name },
  })
  .from(orders)
  .leftJoin(
    customers,
    and(eq(orders.customerId, customers.id), eq(customers.tenantId, orders.tenantId))
  );
```

### 2.3 Transaction + Optimistic Lock

```typescript
async createOrder(dto: CreateOrderDto, user: AuthUser) {
  return db.transaction(async (tx) => {
    // Tạo order
    const [order] = await tx.insert(orders).values({
      tenantId: user.tenantId,
      customerId: dto.customerId,
      code: await this.generateOrderCode(tx, user.tenantId),
      // ...
    }).returning();
    
    // Lock + check stock + giảm tồn (optimistic)
    for (const item of dto.items) {
      const [product] = await tx.execute(sql`
        SELECT id, stock, version FROM products
        WHERE id = ${item.productId}
        AND tenant_id = ${user.tenantId}
        FOR UPDATE
      `);
      
      if (product.stock < item.quantity) {
        throw new InsufficientStockError(item.productId, item.quantity, product.stock);
      }
      
      const updated = await tx.update(products)
        .set({
          stock: sql`${products.stock} - ${item.quantity}`,
          version: sql`${products.version} + 1`,
        })
        .where(and(
          eq(products.id, item.productId),
          eq(products.version, product.version),  // Optimistic check
        ))
        .returning();
      
      if (updated.length === 0) {
        throw new ConflictException('Stock changed concurrently — retry');
      }
      
      // Insert item với snapshot
      await tx.insert(orderItems).values({
        orderId: order.id,
        productId: item.productId,
        productSnapshot: { name: product.name, sku: product.sku },
        quantity: item.quantity,
        priceAtOrder: product.price,
        total: String(Number(product.price) * item.quantity),
      });
    }
    
    // Outbox event (xem rule 14)
    await tx.insert(outbox_events).values({
      eventType: 'order.created',
      payload: { orderId: order.id, tenantId: user.tenantId },
    });
    
    return order;
  }, {
    isolationLevel: 'serializable',  // Strongest
    deferrable: false,
  });
}
```

---

## 3. Migration rules

### 3.1 CẤM tuyệt đối 🔴

- Sửa migration đã merge vào main (kể cả format)
- Drop column còn data quan trọng (làm theo expand-contract)
- Đổi type column trực tiếp
- Migration không reversible (không có manual rollback)

### 3.2 Pattern an toàn

**Đổi tên column** — 3 deploy riêng:
```sql
-- Migration 0070_add_full_name_column.sql
ALTER TABLE users ADD COLUMN full_name TEXT;
UPDATE users SET full_name = first_name || ' ' || last_name;
-- Code deploy: ghi/đọc cả 3 cột

-- Migration 0071_make_full_name_required.sql (sau khi code deploy ổn)
ALTER TABLE users ALTER COLUMN full_name SET NOT NULL;
-- Code deploy: chỉ dùng full_name

-- Migration 0072_drop_old_name_columns.sql (sau 1 tuần stable)
ALTER TABLE users DROP COLUMN first_name;
ALTER TABLE users DROP COLUMN last_name;
```

**Đổi type column:**
```sql
-- Step 1: thêm column type mới
ALTER TABLE orders ADD COLUMN total_v2 DECIMAL(15, 2);
UPDATE orders SET total_v2 = total::DECIMAL(15, 2);

-- Step 2: deploy code dùng total_v2 song song

-- Step 3: drop cũ, rename mới
ALTER TABLE orders DROP COLUMN total;
ALTER TABLE orders RENAME COLUMN total_v2 TO total;
ALTER TABLE orders ALTER COLUMN total SET NOT NULL;
```

### 3.3 Naming convention

```
0070_add_audit_log_table.sql
0070_add_audit_log_table.down.sql
0071_enable_rls_orders.sql
0071_enable_rls_orders.down.sql
```

### 3.4 Mỗi migration BẮT BUỘC

1. Có file `.down.sql` để rollback
2. Có comment giải thích lý do (đầu file)
3. Test trên staging trước
4. Backup DB trước khi chạy production
5. Update `MIGRATION_REGISTRY.md`:

```markdown
## 0070_add_audit_log_table

- **Date**: 2026-04-26
- **Owner**: @trí
- **Ticket**: ERP-234
- **Reason**: Yêu cầu kế toán SOC 2 audit trail
- **Risk**: LOW — bảng mới, không động bảng cũ
- **Rollback time**: < 1 phút
- **Tested on staging**: 2026-04-25
```

---

## 4. Indexes — chiến lược

**Mọi bảng cần có:**
- PK (auto)
- Index trên mọi FK
- Index `tenant_id` (single)
- Index `(tenant_id, created_at DESC)` cho list endpoint mặc định
- Index `(tenant_id, status)` nếu hay filter status
- Partial index `WHERE deleted_at IS NULL` cho query active

**Composite index — thứ tự cột:**
```sql
-- Query: WHERE tenant_id = X AND status = 'paid' ORDER BY created_at DESC LIMIT 20
CREATE INDEX orders_tenant_status_created_idx
  ON orders (tenant_id, status, created_at DESC)
  WHERE deleted_at IS NULL;
```

**Quy tắc thứ tự:**
1. Equality column trước (`tenant_id`, `status`)
2. Range/sort column cuối (`created_at`)
3. Cardinality cao trước

**Full-text search tiếng Việt:**
```sql
CREATE INDEX customers_search_idx ON customers
  USING gin(to_tsvector('simple',
    COALESCE(name, '') || ' ' ||
    COALESCE(phone, '') || ' ' ||
    COALESCE(email, '')
  ));

-- Query
SELECT * FROM customers
WHERE to_tsvector('simple', name || ' ' || phone || ' ' || email)
  @@ plainto_tsquery('simple', 'nguyễn văn a 0901');
```

---

## 5. Data integrity

### CHECK constraints
```sql
ALTER TABLE orders ADD CONSTRAINT orders_total_positive CHECK (total >= 0);
ALTER TABLE order_items ADD CONSTRAINT order_items_qty_positive CHECK (quantity > 0);
ALTER TABLE order_items ADD CONSTRAINT order_items_subtotal_correct
  CHECK (subtotal = ROUND(price_at_order * quantity, 2));
```

### Triggers cho audit
```sql
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER orders_updated_at
  BEFORE UPDATE ON orders
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

---

## 6. Backup strategy (3-2-1)

**3 bản sao, 2 loại media, 1 offsite:**

```
Tier 1 — Hot (RPO=6h, RTO=1h)
  → pg_dump cron mỗi 6h → /opt/backup VPS
  → Giữ 7 ngày

Tier 2 — Warm (RPO=24h, RTO=4h)
  → Daily → Cloudflare R2 (mã hoá AES-256)
  → Giữ 30 ngày

Tier 3 — Cold (RPO=7d, RTO=24h)
  → Weekly → R2 Cold Storage
  → Giữ 12 tháng

Tier 4 — Archive (compliance TT 99/2025)
  → Monthly → R2 Archive
  → Giữ 10 năm
```

### Test restore HÀNG THÁNG 🔴

```bash
# 1st of every month, 02:00 AM
0 2 1 * * /opt/scripts/test-restore.sh

# test-restore.sh
#!/bin/bash
set -euo pipefail

BACKUP_FILE=$(ls -t /opt/backup/*.sql.gz | head -1)
TEST_DB="erp_restore_test_$(date +%Y%m%d)"

createdb "$TEST_DB"
gunzip -c "$BACKUP_FILE" | psql "$TEST_DB"

ROW_COUNT=$(psql "$TEST_DB" -t -c "SELECT COUNT(*) FROM orders;")
if [ "$ROW_COUNT" -lt 1 ]; then
  echo "RESTORE TEST FAILED" | mail -s "ALERT" admin@wecha.vn
  exit 1
fi

dropdb "$TEST_DB"
echo "Restore test passed: $ROW_COUNT orders"
```

---

## 7. Slow query log

```ini
# postgresql.conf
log_min_duration_statement = 100  # Log queries > 100ms
log_statement = 'mod'              # Log all DDL + writes
log_duration = on
```

EXPLAIN ANALYZE BẮT BUỘC cho query mới:
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT * FROM orders WHERE tenant_id = $1 AND status = $2 ORDER BY created_at DESC LIMIT 20;

-- Phải dùng index, không Seq Scan trên bảng > 1000 rows
-- Cost < 1000 cho query thường
```

---

## 8. Connection pooling

```typescript
import { Pool } from 'pg';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,                    // Max connections per Node instance
  min: 2,                     // Min idle
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 5_000,
  statement_timeout: 30_000,  // Cancel query > 30s
  query_timeout: 30_000,
});
```

---

**END Rule 03**
