# Rule 06 — Testing

> Mức độ: 🟠 HIGH | Áp dụng: Mọi feature mới

---

## 1. Test Pyramid

```
       ╱╲
      ╱E2E╲       ← 5%   Playwright/Detox — slow, fragile
     ╱─────╲
    ╱  INT  ╲     ← 15%  Supertest        — medium speed
   ╱─────────╲
  ╱   UNIT    ╲   ← 80%  Jest/Vitest      — fast, deterministic
 ╱─────────────╲
```

---

## 2. Coverage targets BẮT BUỘC

| Layer | Tool | Coverage min | Notes |
|---|---|---|---|
| Domain entities/VO | Jest | 95% | Pure logic |
| Services | Jest | 80% | Mock repository |
| Repositories | Jest + integration DB | 60% | Test critical query |
| Controllers | Supertest | 100% endpoint | Happy + error path |
| Utils (`packages/shared`) | Vitest | 95% | Pure functions |
| UI components có logic | Vitest + RTL | 70% | User interaction |
| E2E critical flows | Playwright | 100% happy path | Login, checkout, payment |

CI fail nếu coverage tụt:
```yaml
- run: pnpm test --coverage --coverageThreshold='{ "global": { "lines": 70, "functions": 70, "branches": 65 } }'
```

---

## 3. Test naming + AAA pattern

> 📎 **Cross-ref:** Code đầy đủ AAA pattern + naming convention chi tiết xem [skills/testing-strategy/SKILL.md](../skills/testing-strategy/SKILL.md#aaa-pattern)

```typescript
describe('OrdersService.createOrder', () => {
  it('phải tạo đơn thành công khi đủ tồn kho', async () => {
    // Arrange
    const dto = createOrderFixture({ items: [{ productId: 'p1', quantity: 5 }] });
    const user = mockUser({ tenantId: 'tenant-1' });
    mockRepo.getStock.mockResolvedValue(10);
    mockRepo.create.mockResolvedValue({ id: 'order-1' });
    
    // Act
    const result = await service.createOrder(dto, user);
    
    // Assert
    expect(result.id).toBe('order-1');
    expect(mockRepo.create).toHaveBeenCalledOnce();
    expect(mockRepo.create).toHaveBeenCalledWith(
      expect.objectContaining({ tenant_id: 'tenant-1' })
    );
  });
  
  it('phải throw InsufficientStockError khi tồn kho không đủ', async () => {
    const dto = createOrderFixture({ items: [{ productId: 'p1', quantity: 100 }] });
    mockRepo.getStock.mockResolvedValue(50);
    
    await expect(service.createOrder(dto, mockUser()))
      .rejects.toThrow(InsufficientStockError);
    
    expect(mockRepo.create).not.toHaveBeenCalled();
  });
  
  it('phải tự động set tenant_id từ user (không tin client)', async () => {
    const dto = createOrderFixture({ tenantId: 'fake-tenant' as any });
    const user = mockUser({ tenantId: 'real-tenant' });
    
    await service.createOrder(dto, user);
    
    expect(mockRepo.create).toHaveBeenCalledWith(
      expect.objectContaining({ tenant_id: 'real-tenant' })
    );
  });
});
```

**Quy tắc naming:**
- `describe`: tên class hoặc method (`OrdersService.createOrder`)
- `it`: "phải <action> khi <condition>" (tiếng Việt)
- 1 `it` = 1 behavior

---

## 4. Fixtures + Factories

```typescript
// test/fixtures/order.ts
import { faker } from '@faker-js/faker/locale/vi';

export function createOrderFixture(overrides?: Partial<Order>): Order {
  return {
    id: `order_${faker.string.nanoid()}`,
    tenant_id: 'tenant-test',
    code: `ORD-2026-${faker.number.int({ min: 1, max: 99999 }).toString().padStart(5, '0')}`,
    customer_id: `cust_${faker.string.nanoid()}`,
    status: 'draft',
    subtotal: '100000',
    tax_amount: '10000',
    discount_amount: '0',
    total: '110000',
    currency: 'VND',
    customer_snapshot: {
      name: faker.person.fullName(),
      phone: faker.phone.number(),
      address: faker.location.streetAddress(),
    },
    metadata: null, internal_note: null, customer_note: null,
    created_at: new Date(), updated_at: new Date(),
    created_by: 'user-test', updated_by: null,
    deleted_at: null, deleted_by: null, deleted_reason: null,
    version: 0,
    ...overrides,
  };
}
```

---

## 5. Integration test (Supertest) — security tests BẮT BUỘC

```typescript
describe('POST /api/v1/orders', () => {
  it('phải tạo order thành công với input hợp lệ', async () => {
    const response = await request(app.getHttpServer())
      .post('/api/v1/orders')
      .set('Authorization', `Bearer ${authToken}`)
      .set('Idempotency-Key', `test-${Date.now()}`)
      .send(validPayload);
    
    expect(response.status).toBe(201);
    expect(response.body.data).toMatchObject({
      id: expect.any(String),
      code: expect.stringMatching(/^ORD-\d{4}-\d{5}$/),
      status: 'draft',
    });
  });
  
  it('phải return 400 nếu thiếu Idempotency-Key', async () => {
    const response = await request(app.getHttpServer())
      .post('/api/v1/orders')
      .set('Authorization', `Bearer ${authToken}`)
      .send(validPayload);
    
    expect(response.status).toBe(400);
    expect(response.body.error.code).toBe('IDEMPOTENCY_KEY_REQUIRED');
  });
  
  it('phải return same response cho cùng Idempotency-Key', async () => {
    const idempKey = `test-${Date.now()}`;
    
    const r1 = await request(app.getHttpServer())
      .post('/api/v1/orders')
      .set('Authorization', `Bearer ${authToken}`)
      .set('Idempotency-Key', idempKey)
      .send(validPayload);
    
    const r2 = await request(app.getHttpServer())
      .post('/api/v1/orders')
      .set('Authorization', `Bearer ${authToken}`)
      .set('Idempotency-Key', idempKey)
      .send(validPayload);
    
    expect(r1.body.data.id).toBe(r2.body.data.id);
    expect(r2.body.meta.idempotent).toBe(true);
  });
  
  // 🔴 SECURITY TESTS BẮT BUỘC
  it('phải KHÔNG thấy đơn của tenant khác', async () => {
    const tenantBOrder = await createOrderInTenant(db, 'tenant-b');
    const response = await request(app.getHttpServer())
      .get(`/api/v1/orders/${tenantBOrder.id}`)
      .set('Authorization', `Bearer ${authToken}`);
    
    expect(response.status).toBe(404);  // 404, không 403
  });
  
  it('phải reject SQL injection trong query param', async () => {
    const response = await request(app.getHttpServer())
      .get(`/api/v1/orders?customer_id=' OR '1'='1`)
      .set('Authorization', `Bearer ${authToken}`);
    
    expect(response.status).toBe(422);
  });
  
  it('phải rate limit khi gọi quá nhiều', async () => {
    const requests = Array.from({ length: 110 }, () =>
      request(app.getHttpServer())
        .get('/api/v1/orders')
        .set('Authorization', `Bearer ${authToken}`)
    );
    
    const responses = await Promise.all(requests);
    expect(responses.filter(r => r.status === 429).length).toBeGreaterThan(0);
  });
});
```

---

## 6. E2E (Playwright)

```typescript
// e2e/orders.spec.ts
import { test, expect } from '@playwright/test';
import { loginAs } from './helpers/auth';

test.describe('Order management flow', () => {
  test.beforeEach(async ({ page }) => {
    await loginAs(page, 'manager@wecha.vn', 'StrongPass!2026');
  });
  
  test('phải tạo và xem đơn hàng mới', async ({ page }) => {
    await page.goto('/orders');
    await page.getByRole('button', { name: 'Tạo đơn' }).click();
    
    await page.getByLabel('Khách hàng').click();
    await page.getByPlaceholder('Tìm khách hàng').fill('Nguyễn Văn A');
    await page.getByRole('option', { name: /Nguyễn Văn A.*0901/ }).click();
    
    await page.getByLabel('Sản phẩm').click();
    await page.getByRole('option', { name: 'Trà sữa size M' }).click();
    await page.getByLabel('Số lượng').fill('2');
    
    await page.getByRole('button', { name: 'Tạo đơn' }).click();
    
    await expect(page.getByText('Tạo đơn thành công')).toBeVisible();
    await expect(page).toHaveURL(/\/orders\/[a-z0-9]+$/);
  });
});
```

---

## 7. Test database strategy

```typescript
// test/setup.ts
import { migrate } from 'drizzle-orm/node-postgres/migrator';

export async function globalSetup() {
  execSync(`createdb erp_test_${process.pid}`);
  
  const db = drizzle(new Pool({
    connectionString: `postgresql://localhost/erp_test_${process.pid}`,
  }));
  
  await migrate(db, { migrationsFolder: './apps/api/src/database/migrations' });
  await seedTestData(db);
}

export async function globalTeardown() {
  execSync(`dropdb erp_test_${process.pid}`);
}

// Per-test isolation qua transaction rollback
beforeEach(async () => {
  await db.transaction(async (tx) => {
    await runTest(tx);
    throw new Error('Rollback');  // Auto rollback
  }).catch(() => {});
});
```

---

## 8. Anti-patterns CẤM

```typescript
// ❌ Test implementation detail
expect(service['_calculateTax']).toHaveBeenCalled();

// ❌ Shared state
let order;
beforeAll(() => { order = createOrder(); });
test('test 1', () => { order.status = 'paid'; });
test('test 2', () => { /* order.status đã bị đổi */ });

// ❌ Conditional assertions
if (response.body.data) {
  expect(response.body.data.id).toBeDefined();
}

// ❌ expect.any cho mọi field
expect(result).toEqual({
  id: expect.any(String),
  total: expect.any(Number),
});

// ❌ Snapshot test cho data thay đổi
expect(orders).toMatchSnapshot();  // Date đổi mỗi lần

// ❌ sleep
await new Promise(r => setTimeout(r, 1000));  // Flaky
```

---

**END Rule 06**
