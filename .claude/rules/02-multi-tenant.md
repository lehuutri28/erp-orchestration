# Rule 02 — Multi-Tenant Isolation

> ⚠️ Đồng bộ ARCHITECTURE_CONTRACT 29/5/2026: PK=uuid, tenant_id=uuid, TS field camelCase, RBAC dấu `.`. Contract THẮNG nếu mâu thuẫn. Xem docs/ARCHITECTURE_CONTRACT.md.

> Mức độ: 🔴 CRITICAL | Áp dụng: Mọi bảng nghiệp vụ, mọi query, mọi service

---

## Tại sao 4 lớp?

1 dòng quên `WHERE tenant_id = X` = leak data toàn bộ tenant khác = mất license PCI + phạt NĐ 13/2023 5-100M VND + mất uy tín thương hiệu vĩnh viễn.

Chiến lược **defense in depth** — mỗi tầng độc lập, một tầng fail không sập cả hệ thống.

---

## Lớp 1 — JWT có tenant_id + Guard

JWT payload BẮT BUỘC có `tenant_id`:
```typescript
interface JwtPayload {
  sub: string;           // user_id
  tenant_id: string;     // BẮT BUỘC
  email: string;
  roles: string[];
  store_ids: string[];   // Cho store_manager
  jti: string;           // JWT ID để revoke
  iat: number;
  exp: number;
}
```

TenantGuard:
```typescript
@Injectable()
export class TenantGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const req = context.switchToHttp().getRequest();
    if (!req.user?.tenantId) {
      throw new ForbiddenException('No tenant context');
    }
    return true;
  }
}

// app.module.ts
{ provide: APP_GUARD, useClass: TenantGuard }
```

---

## Lớp 2 — AsyncLocalStorage propagate context

```typescript
// shared/tenant-context.ts
import { AsyncLocalStorage } from 'async_hooks';

export interface TenantContext {
  tenantId: string;
  userId: string;
  requestId: string;
  storeIds?: string[];
}

export const tenantStorage = new AsyncLocalStorage<TenantContext>();

export function getCurrentTenantId(): string {
  const ctx = tenantStorage.getStore();
  if (!ctx) throw new Error('Tenant context missing — middleware not applied');
  return ctx.tenantId;
}

export function getCurrentUserId(): string {
  const ctx = tenantStorage.getStore();
  if (!ctx) throw new Error('Tenant context missing');
  return ctx.userId;
}

// Middleware
@Injectable()
export class TenantMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const ctx: TenantContext = {
      tenantId: req.user.tenantId,
      userId: req.user.id,
      storeIds: req.user.storeIds,
      requestId: req.headers['x-request-id'] as string ?? randomUUID(),
    };
    tenantStorage.run(ctx, () => next());
  }
}
```

---

## Lớp 3 — Drizzle helper auto-inject

```typescript
// shared/db-helpers.ts
import { eq, and, isNull, type SQL } from 'drizzle-orm';

export function tenantFilter<T extends { tenant_id: any }>(table: T): SQL {
  return eq(table.tenant_id, getCurrentTenantId());
}

export function notDeleted<T extends { deleted_at: any }>(table: T): SQL {
  return isNull(table.deleted_at);
}

// Usage — KHÔNG bao giờ quên
async listOrders(status?: string) {
  return db.query.orders.findMany({
    where: and(
      tenantFilter(orders),     // ← BẮT BUỘC
      notDeleted(orders),       // ← BẮT BUỘC
      status ? eq(orders.status, status) : undefined,
    ),
  });
}
```

**ESLint rule custom** để catch query thiếu tenant filter:
```js
// .eslintrc — custom rule
'no-restricted-syntax': [
  'error',
  {
    selector: 'CallExpression[callee.property.name="findMany"][arguments.length>0]:not(:has(Identifier[name="tenantFilter"]))',
    message: 'Query findMany phải có tenantFilter() hoặc giải thích lý do skip với // @allow-no-tenant-filter',
  },
],
```

---

## Lớp 4 — PostgreSQL Row-Level Security (RLS)

**Mọi bảng nghiệp vụ BẮT BUỘC bật RLS.** Đây là defense cuối — kể cả developer fail 3 lớp trên, Postgres vẫn từ chối query.

### Migration template enable RLS:
```sql
-- migrations/0070_enable_rls_orders.sql

-- 1. Enable RLS
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders FORCE ROW LEVEL SECURITY;

-- 2. Policy: chỉ thấy row của tenant hiện tại
CREATE POLICY tenant_isolation_orders ON orders
  USING (tenant_id = current_setting('app.current_tenant', true))
  WITH CHECK (tenant_id = current_setting('app.current_tenant', true));

-- 3. Service role bypass (cho admin tools, cron, migration)
CREATE POLICY service_role_bypass_orders ON orders
  TO service_role
  USING (true)
  WITH CHECK (true);
```

### Set Postgres session variable mỗi request:
```typescript
// Drizzle middleware
import { sql } from 'drizzle-orm';

export async function withTenantContext<T>(
  tenantId: string,
  fn: () => Promise<T>
): Promise<T> {
  return db.transaction(async (tx) => {
    await tx.execute(sql`SELECT set_config('app.current_tenant', ${tenantId}, true)`);
    return fn();
  });
}

// Usage trong service
async getOrders() {
  return withTenantContext(getCurrentTenantId(), () =>
    db.query.orders.findMany()  // RLS tự động filter
  );
}
```

### Connection pool setup:
```typescript
// database.module.ts — set search_path + reset trước khi return connection
const pool = new Pool({
  // ...
  afterConnect: async (client) => {
    await client.query("SET search_path TO public");
  },
});
```

---

## Bảng cần bật RLS — 20 bảng nghiệp vụ chính

🔴 **Wave 2 PRIORITY:**
1. orders
2. order_items
3. payments
4. invoices
5. customers
6. products
7. inventory
8. stock_movements
9. purchase_orders
10. suppliers
11. employees
12. salaries
13. attendance
14. campaigns
15. courses
16. enrollments
17. franchise_branches
18. royalty_reports
19. machine_assets
20. maintenance_logs

🟠 **Wave 3:**
- All audit logs
- All reporting tables
- All configuration tables

🟡 **Có thể skip RLS:**
- `tenants` (bảng master, scope theo super_admin)
- `users` (cần check theo tenant ở app layer)
- `system_configs` (global)
- Bảng `_lookup_*` (data tham chiếu chung)

---

## Kiểm tra nhanh — RLS đã bật chưa?

```sql
-- Liệt kê bảng có RLS enabled
SELECT schemaname, tablename, rowsecurity, forcerowsecurity
FROM pg_tables
WHERE schemaname = 'public' AND rowsecurity = true
ORDER BY tablename;

-- Liệt kê policy của 1 bảng
SELECT * FROM pg_policies WHERE tablename = 'orders';

-- Test RLS hoạt động
SET app.current_tenant = 'tenant-A';
SELECT COUNT(*) FROM orders;  -- Chỉ thấy của tenant-A

SET app.current_tenant = 'tenant-B';
SELECT COUNT(*) FROM orders;  -- Chỉ thấy của tenant-B

RESET app.current_tenant;
SELECT COUNT(*) FROM orders;  -- 0 rows (không set tenant)
```

---

## Test bắt buộc 🔴

> 📎 **Cross-ref:** Test cross-tenant (security perspective — 404 vs 403, IDOR) xem [rules/01-security.md](./01-security.md#a01--broken-access-control-)

```typescript
describe('Multi-tenant isolation', () => {
  it('Tenant A KHÔNG đọc được dữ liệu Tenant B (mọi bảng)', async () => {
    const orderB = await createOrderInTenant('tenant-B');
    
    const response = await request(app)
      .get(`/api/v1/orders/${orderB.id}`)
      .set('Authorization', `Bearer ${tenantAToken}`);
    
    expect(response.status).toBe(404);  // Không 403
  });
  
  it('Tenant A KHÔNG update được dữ liệu Tenant B', async () => {
    const orderB = await createOrderInTenant('tenant-B');
    
    const response = await request(app)
      .patch(`/api/v1/orders/${orderB.id}`)
      .set('Authorization', `Bearer ${tenantAToken}`)
      .send({ status: 'cancelled' });
    
    expect(response.status).toBe(404);
    
    // Verify trong DB không bị update
    const fresh = await db.query.orders.findFirst({
      where: eq(orders.id, orderB.id)
    });
    expect(fresh.status).not.toBe('cancelled');
  });
  
  it('Tenant A KHÔNG insert dữ liệu vào Tenant B', async () => {
    const response = await request(app)
      .post(`/api/v1/orders`)
      .set('Authorization', `Bearer ${tenantAToken}`)
      .send({ tenant_id: 'tenant-B', /* ... */ });  // Cố tình truyền tenant_id
    
    // Service phải tự override tenant_id từ JWT
    const created = await db.query.orders.findFirst({ /* ... */ });
    expect(created.tenant_id).toBe('tenant-A');  // KHÔNG phải tenant-B
  });
  
  it('RLS chặn raw SQL query không có tenant context', async () => {
    // KHÔNG set app.current_tenant
    const result = await db.execute(sql`SELECT COUNT(*) FROM orders`);
    expect(result.rows[0].count).toBe('0');  // RLS từ chối
  });
});
```

---

## Cross-tenant operations (admin tools)

Đôi khi cần query xuyên tenant (báo cáo super admin, migration). Cách an toàn:

```typescript
// 1. Decorator riêng đánh dấu
@CrossTenant()  // Custom decorator
@RequireRole('super_admin')
@RequireApproval()  // 2FA + lý do
@Get('admin/orders/all')
async getAllOrdersAcrossTenants() {
  return withServiceRole(() =>  // Bypass RLS
    db.query.orders.findMany()
  );
}

// 2. Audit log MẠNH
await this.audit.log({
  action: 'admin.cross_tenant_query',
  actor_id: user.id,
  reason: req.body.reason,  // Bắt buộc
  query_summary: 'list all orders',
  row_count: result.length,
  ip: req.ip,
});

// 3. Alert Telegram cho team security
await this.alerts.send('CROSS_TENANT_QUERY', { ... });
```

---

## Anti-patterns CẤM 🔴

```typescript
// ❌ Trust client tenant_id
@Post('orders')
async create(@Body() dto: { tenant_id: string, /* ... */ }) {
  return db.insert(orders).values(dto);  // dto.tenant_id có thể là tenant khác
}

// ✅ Override từ JWT
@Post('orders')
async create(@Body() dto: CreateOrderDto, @CurrentUser() user) {
  return db.insert(orders).values({ ...dto, tenant_id: user.tenantId });
}

// ❌ Quên filter trong join
const result = await db
  .select()
  .from(orders)
  .leftJoin(customers, eq(orders.customer_id, customers.id))
  .where(eq(orders.tenant_id, user.tenantId));  // CHỈ filter orders, customers leak!

// ✅ Filter cả 2 bảng
const result = await db
  .select()
  .from(orders)
  .leftJoin(
    customers,
    and(
      eq(orders.customer_id, customers.id),
      eq(customers.tenant_id, user.tenantId)
    )
  )
  .where(eq(orders.tenant_id, user.tenantId));

// ❌ Aggregate cross-tenant
const total = await db
  .select({ sum: sum(orders.total) })
  .from(orders);  // Tổng toàn hệ thống — leak business intelligence

// ✅ Filter tenant
const total = await db
  .select({ sum: sum(orders.total) })
  .from(orders)
  .where(tenantFilter(orders));
```

---

## Checklist trước commit 🔴

- [ ] Mọi `db.select`, `db.update`, `db.delete`, `db.insert` có tenant filter?
- [ ] Mọi join giữa 2 bảng nghiệp vụ → cả 2 bảng đều filter tenant?
- [ ] Mọi POST/PATCH override `tenant_id` từ JWT (không tin client body)?
- [ ] Bảng mới đã ALTER TABLE ENABLE ROW LEVEL SECURITY?
- [ ] Bảng mới đã CREATE POLICY?
- [ ] Test cross-tenant đã viết?
- [ ] Test thử SET app.current_tenant rỗng → query trả 0 rows?

---

**END Rule 02**
