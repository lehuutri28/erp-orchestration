# Skill: Testing Strategy

> Khi nào load: Viết test cho feature mới, fix flaky test, setup test infra

---

## Test layer & tool

| Layer | Tool | File pattern | Run |
|---|---|---|---|
| Unit (BE) | Jest 30 | `*.spec.ts` cùng folder | `pnpm test:unit` |
| Integration (BE) | Jest + Supertest | `*.controller.spec.ts` | `pnpm test:integration` |
| Unit (FE) | Vitest + RTL | `*.test.tsx` | `pnpm test:fe` |
| E2E | Playwright | `e2e/*.spec.ts` | `pnpm test:e2e` |
| Load | k6 | `load-tests/*.js` | `pnpm test:load` |
| Visual | Chromatic | (auto on PR) | - |

---

## Test setup template

### Jest config
```js
// jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  rootDir: 'src',
  testRegex: '.*\\.spec\\.ts$',
  transform: { '^.+\\.(t|j)s$': 'ts-jest' },
  collectCoverageFrom: ['**/*.(t|j)s', '!**/*.module.ts', '!**/main.ts'],
  coverageDirectory: '../coverage',
  coverageThreshold: {
    global: { branches: 65, functions: 70, lines: 70, statements: 70 },
  },
  setupFilesAfterEach: ['<rootDir>/test/setup.ts'],
  testTimeout: 10_000,
};
```

### Test database isolation
```typescript
// test/setup.ts
import { execSync } from 'child_process';
import { Pool } from 'pg';
import { drizzle } from 'drizzle-orm/node-postgres';
import { migrate } from 'drizzle-orm/node-postgres/migrator';

const TEST_DB = `erp_test_${process.pid}`;

export async function globalSetup() {
  execSync(`createdb ${TEST_DB}`);
  
  const pool = new Pool({
    connectionString: `postgresql://localhost/${TEST_DB}`,
  });
  const db = drizzle(pool);
  
  await migrate(db, { migrationsFolder: './apps/api/src/database/migrations' });
  await seedTestData(db);
  
  process.env.DATABASE_URL = `postgresql://localhost/${TEST_DB}`;
}

export async function globalTeardown() {
  execSync(`dropdb ${TEST_DB}`);
}

// Per-test cleanup
beforeEach(async () => {
  // Truncate non-system tables
  await db.execute(sql`TRUNCATE orders, order_items, customers RESTART IDENTITY CASCADE`);
});
```

---

## Unit test patterns

> 📎 **Cross-ref:** AAA pattern quy tắc đặt tên test (naming convention ngắn gọn + ví dụ) xem [rules/06-testing.md](../../rules/06-testing.md#3-test-naming--aaa-pattern)

> 📎 **Cross-ref:** Security tests bắt buộc (cross-tenant 404, SQL injection 422, rate limit 429) xem [rules/06-testing.md](../../rules/06-testing.md#-security-tests-bắt-buộc)

### Service test (mock repository)

```typescript
// orders.service.spec.ts
import { Test } from '@nestjs/testing';
import { OrdersService } from './orders.service';
import { OrdersRepository } from '../repositories/orders.repository';
import { InventoryService } from '@/modules/inventory/services/inventory.service';

describe('OrdersService', () => {
  let service: OrdersService;
  let mockRepo: jest.Mocked<OrdersRepository>;
  let mockInventory: jest.Mocked<InventoryService>;
  
  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        OrdersService,
        {
          provide: OrdersRepository,
          useValue: {
            create: jest.fn(),
            findByIdForTenant: jest.fn(),
            update: jest.fn(),
          },
        },
        {
          provide: InventoryService,
          useValue: {
            getStock: jest.fn(),
            reserve: jest.fn(),
          },
        },
        // Mock event bus, db, etc.
      ],
    }).compile();
    
    service = module.get(OrdersService);
    mockRepo = module.get(OrdersRepository);
    mockInventory = module.get(InventoryService);
  });
  
  describe('create', () => {
    it('phải tạo đơn thành công khi đủ tồn kho', async () => {
      // Arrange
      const dto = createOrderInputFixture({
        items: [{ productId: 'p1', quantity: 5 }],
      });
      const user = mockUser();
      
      mockInventory.getStock.mockResolvedValue(10);
      mockRepo.create.mockResolvedValue(orderFixture({ id: 'order-1' }));
      
      // Act
      const result = await service.create(dto, user);
      
      // Assert
      expect(result.id).toBe('order-1');
      expect(mockInventory.getStock).toHaveBeenCalledWith('p1', user.tenantId, expect.anything());
      expect(mockInventory.reserve).toHaveBeenCalledWith('p1', 5, 'order-1', expect.anything());
      expect(mockRepo.create).toHaveBeenCalledWith(
        expect.objectContaining({ tenant_id: user.tenantId }),
        expect.anything(),
      );
    });
    
    it('phải throw InsufficientStockError khi tồn kho không đủ', async () => {
      const dto = createOrderInputFixture({
        items: [{ productId: 'p1', quantity: 100 }],
      });
      mockInventory.getStock.mockResolvedValue(50);
      
      await expect(service.create(dto, mockUser())).rejects.toThrow(InsufficientStockError);
      expect(mockRepo.create).not.toHaveBeenCalled();
      expect(mockInventory.reserve).not.toHaveBeenCalled();
    });
    
    it('phải tự override tenant_id từ user (KHÔNG tin client)', async () => {
      const dto = createOrderInputFixture({ tenantId: 'fake-tenant' as any });
      const user = mockUser({ tenantId: 'real-tenant' });
      
      mockInventory.getStock.mockResolvedValue(100);
      mockRepo.create.mockResolvedValue(orderFixture());
      
      await service.create(dto, user);
      
      expect(mockRepo.create).toHaveBeenCalledWith(
        expect.objectContaining({ tenant_id: 'real-tenant' }),
        expect.anything(),
      );
    });
  });
});
```

### Repository test (real DB)

```typescript
// orders.repository.spec.ts
describe('OrdersRepository (integration)', () => {
  let repo: OrdersRepository;
  let db: Drizzle;
  
  beforeAll(async () => {
    db = drizzle(new Pool({ connectionString: process.env.DATABASE_URL }));
    repo = new OrdersRepository(db);
  });
  
  beforeEach(async () => {
    await db.execute(sql`TRUNCATE orders RESTART IDENTITY CASCADE`);
  });
  
  describe('listForTenant', () => {
    it('phải return cursor pagination đúng', async () => {
      // Seed 25 orders
      for (let i = 0; i < 25; i++) {
        await db.insert(orders).values(orderFixture({ tenant_id: 'tenant-1' }));
      }
      
      const page1 = await repo.listForTenant({ tenantId: 'tenant-1', limit: 10 });
      expect(page1.data).toHaveLength(10);
      expect(page1.nextCursor).toBeTruthy();
      
      const page2 = await repo.listForTenant({
        tenantId: 'tenant-1',
        limit: 10,
        cursor: page1.nextCursor,
      });
      expect(page2.data).toHaveLength(10);
      
      const page3 = await repo.listForTenant({
        tenantId: 'tenant-1',
        limit: 10,
        cursor: page2.nextCursor,
      });
      expect(page3.data).toHaveLength(5);
      expect(page3.nextCursor).toBeNull();
    });
    
    it('phải KHÔNG trả order của tenant khác', async () => {
      await db.insert(orders).values(orderFixture({ tenant_id: 'tenant-1' }));
      await db.insert(orders).values(orderFixture({ tenant_id: 'tenant-2' }));
      
      const result = await repo.listForTenant({ tenantId: 'tenant-1', limit: 10 });
      
      expect(result.data).toHaveLength(1);
      expect(result.data[0].tenant_id).toBe('tenant-1');
    });
  });
});
```

---

## E2E test patterns (Playwright)

```typescript
// e2e/checkout-flow.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Checkout flow', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/login');
    await page.getByLabel('Email').fill('manager@wecha.vn');
    await page.getByLabel('Mật khẩu').fill('TestPass!2026');
    await page.getByRole('button', { name: 'Đăng nhập' }).click();
    await page.waitForURL('/dashboard');
  });
  
  test('phải hoàn tất đơn hàng từ POS đến thanh toán', async ({ page }) => {
    // 1. Mở POS
    await page.goto('/pos');
    
    // 2. Add product
    await page.getByRole('button', { name: 'Trà sữa size M' }).click();
    await expect(page.getByText('1 món')).toBeVisible();
    
    // 3. Tăng số lượng
    await page.getByRole('button', { name: '+' }).click();
    await expect(page.getByText('2 món')).toBeVisible();
    
    // 4. Add khách
    await page.getByRole('button', { name: 'Chọn khách' }).click();
    await page.getByPlaceholder('Tìm theo SĐT').fill('0901234567');
    await page.getByRole('option').first().click();
    
    // 5. Thanh toán
    await page.getByRole('button', { name: 'Thanh toán' }).click();
    await page.getByLabel('Tiền mặt').check();
    await page.getByRole('button', { name: 'Xác nhận' }).click();
    
    // 6. Verify thành công
    await expect(page.getByText('Thanh toán thành công')).toBeVisible();
    await expect(page.getByText(/Hoá đơn ORD-/)).toBeVisible();
    
    // 7. Verify trong DB qua API
    const ordersResponse = await page.request.get('/api/v1/orders?limit=1');
    const orders = await ordersResponse.json();
    expect(orders.data[0].status).toBe('paid');
  });
});
```

```typescript
// playwright.config.ts
export default {
  testDir: './e2e',
  timeout: 30_000,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 2 : 4,
  reporter: [
    ['html'],
    ['junit', { outputFile: 'test-results/junit.xml' }],
  ],
  use: {
    baseURL: process.env.E2E_BASE_URL ?? 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    { name: 'chromium', use: { browserName: 'chromium' } },
    { name: 'mobile', use: { ...devices['iPhone 13'] } },
  ],
};
```

---

## Load test với k6

```javascript
// load-tests/orders-create.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 50 },   // Ramp up
    { duration: '5m', target: 50 },   // Stay
    { duration: '2m', target: 200 },  // Spike
    { duration: '5m', target: 200 },
    { duration: '2m', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    http_req_failed: ['rate<0.01'],   // < 1% error
  },
};

const BASE_URL = __ENV.BASE_URL ?? 'https://api-staging.wecha.vn';
const TOKEN = __ENV.AUTH_TOKEN;

export default function () {
  const res = http.post(
    `${BASE_URL}/api/v1/orders`,
    JSON.stringify({
      customerId: 'cust-test',
      items: [{ productId: 'prod-test', quantity: 1 }],
      paymentMethod: 'cash',
    }),
    {
      headers: {
        'Authorization': `Bearer ${TOKEN}`,
        'Content-Type': 'application/json',
        'Idempotency-Key': `load-test-${__VU}-${__ITER}`,
      },
    },
  );
  
  check(res, {
    'status 201': (r) => r.status === 201,
    'has order id': (r) => r.json('data.id') !== undefined,
    'duration < 500ms': (r) => r.timings.duration < 500,
  });
  
  sleep(1);
}
```

---

## Common pitfalls

### Flaky test fix

```typescript
// ❌ Race condition
test('order created', async () => {
  await createOrder();
  const orders = await listOrders();
  expect(orders).toHaveLength(1);  // Có thể fail vì cron job xoá
});

// ✅ Wait + isolation
test('order created', async () => {
  await db.transaction(async (tx) => {
    await createOrder(tx);
    const orders = await listOrders(tx);
    expect(orders).toHaveLength(1);
    throw new Error('rollback');
  }).catch(() => {});
});

// ❌ Time-dependent
test('expires after 1 hour', async () => {
  const token = createToken();
  setTimeout(() => {
    expect(token.isExpired()).toBe(true);  // Wait 1 giờ?!
  }, 3600 * 1000);
});

// ✅ Mock time
test('expires after 1 hour', () => {
  jest.useFakeTimers();
  const token = createToken();
  jest.advanceTimersByTime(3600 * 1000);
  expect(token.isExpired()).toBe(true);
});
```

---

**END Skill: Testing Strategy**
