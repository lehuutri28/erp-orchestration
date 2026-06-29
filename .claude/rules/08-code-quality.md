# Rule 08 — Code Quality

> Mức độ: 🟠 HIGH | Áp dụng: Mọi file code

---

## 1. Naming conventions

| Loại | Convention | Ví dụ |
|---|---|---|
| Variable | `camelCase` | `customerOrders`, `totalAmount` |
| Constant | `SCREAMING_SNAKE_CASE` | `MAX_RETRY_COUNT` |
| Function | `camelCase`, verb đầu | `calculateTax()`, `findOrderById()` |
| Class | `PascalCase`, noun | `OrderService` |
| Interface | `PascalCase`, KHÔNG prefix `I` | `Order` (không `IOrder`) |
| Type alias | `PascalCase` | `OrderStatus` |
| Enum | `PascalCase` cho name, `SCREAMING_SNAKE` cho value | `enum Status { ACTIVE }` |
| Generic | Single uppercase or descriptive | `<T>`, `<TUser>` |
| File (TS) | `kebab-case` | `order-service.ts` |
| File (React) | `PascalCase` cho component | `OrderList.tsx`, `use-orders.ts` |
| Folder | `kebab-case` | `order-management/` |
| URL path | `kebab-case` | `/api/v1/order-items` |
| DB table | `snake_case`, plural | `order_items` |
| DB column | `snake_case` | `created_at`, `tenant_id` |
| Env variable | `SCREAMING_SNAKE_CASE` | `DATABASE_URL` |
| Event name | `domain.action` lowercase | `order.created` |
| HTTP header (custom) | `X-Pascal-Case` | `X-Tenant-Id` |

**CẤM:** Viết tắt không phổ biến (`cusOrd`, `prodMgr`), tên 1 chữ (trừ loop `i`,`j`,`k`), Hungarian notation (`strName`), tên ngược nghĩa (`disableNotEnabled`), tên quá generic (`data`, `temp`, `obj`, `info`).

---

## 2. Function rules

```typescript
// ❌ SAI — function quá dài
async function processOrder(orderId: string) {
  // 200 dòng: validate, fetch, calculate, update stock, charge, send email...
}

// ✅ ĐÚNG — single responsibility
async function processOrder(orderId: string) {
  const order = await findOrderOrThrow(orderId);
  await validateOrder(order);
  const total = await calculateOrderTotal(order);
  await reserveStock(order.items);
  const payment = await chargePayment(order.customerId, total);
  await markOrderAsPaid(order.id, payment.id);
  await emitOrderPaidEvent(order);
  return order;
}
```

**Quy tắc:**
- ≤ 50 dòng (cảnh báo), ≤ 100 dòng (refuse)
- ≤ 4 tham số (dùng object nếu nhiều)
- Cyclomatic complexity ≤ 10
- Không nested > 3 cấp
- Return early (guard clauses)

```typescript
// ❌ Nested
function process(user: User | null) {
  if (user) {
    if (user.isActive) {
      if (user.hasPermission) { /* do work */ }
    }
  }
}

// ✅ Guard clauses
function process(user: User | null) {
  if (!user) throw new UnauthorizedException();
  if (!user.isActive) throw new ForbiddenException('Account inactive');
  if (!user.hasPermission) throw new ForbiddenException('No permission');
  
  // do work
}
```

---

## 3. Comments

```typescript
// ❌ SAI — nói lại điều code đã nói
// Increment counter
counter++;

// ✅ ĐÚNG — giải thích TẠI SAO
// Tăng version để optimistic lock detect concurrent update
counter++;

// HACK: GHN API trả về `delivery_time` đôi khi null cho đơn nội thành.
// Workaround: nếu null thì estimate +2h từ created_at.
// TODO(@trí, 2026-06-01): contact GHN support để fix root cause
const eta = response.delivery_time ?? new Date(order.createdAt.getTime() + 2 * 3600_000);
```

**Quy tắc:**
- TODO/FIXME/HACK PHẢI có owner + date + lý do
- Block comment chỉ cho file header hoặc class quan trọng
- JSDoc cho public API
- CẤM comment-out code (xoá đi, git có lịch sử)

---

## 4. Imports order (ESLint enforce)

```typescript
// 1. Node built-in
import { randomUUID } from 'crypto';
import { Readable } from 'stream';

// 2. External packages
import { Injectable, Inject, Logger } from '@nestjs/common';
import { eq, and, desc } from 'drizzle-orm';
import { z } from 'zod';

// 3. Workspace packages (alphabet)
import { type AuthUser } from '@wecha/contracts/auth';
import { ordersTable } from '@wecha/db/schema';
import { formatVnd } from '@wecha/shared/utils';

// 4. Aliased internal imports
import { DRIZZLE } from '@/database/database.module';
import { CurrentUser } from '@/shared/decorators';

// 5. Relative imports (gần nhất sau cùng)
import { OrderRepository } from '../repositories/orders.repository';
import { CreateOrderDto } from './dto/create-order.dto';

// 6. Type-only imports cuối
import type { Drizzle } from '@/database/database.module';
```

```json
{
  "import/order": ["error", {
    "groups": ["builtin", "external", "internal", "parent", "sibling", "index", "type"],
    "newlines-between": "always",
    "alphabetize": { "order": "asc" }
  }]
}
```

---

## 5. Type Safety

**CẤM:**
- `any` (dùng `unknown` rồi narrow)
- `@ts-ignore` (dùng `@ts-expect-error` + comment giải thích)
- `as` ép kiểu trừ khi cần (có comment)

**BẮT BUỘC:**
- Public function có return type rõ ràng
- Generic constraint hợp lý (`<T extends Entity>`)
- Discriminated union cho state machine
- `z.infer<typeof schema>` cho type từ Zod

```typescript
// ❌
function process(data: any) { return data.foo; }

// ✅
function process(data: unknown): string {
  if (typeof data === 'object' && data !== null && 'foo' in data) {
    return String(data.foo);
  }
  throw new Error('Invalid data');
}

// ✅ Discriminated union
type OrderState =
  | { status: 'draft' }
  | { status: 'paid', paidAt: Date, paymentId: string }
  | { status: 'cancelled', cancelledAt: Date, reason: string };

function handleOrder(order: OrderState) {
  switch (order.status) {
    case 'draft': /* ... */ break;
    case 'paid':
      console.log(order.paidAt);  // TS biết có paidAt
      break;
    case 'cancelled':
      console.log(order.reason);
      break;
  }
}
```

---

## 6. DRY — không lặp code

Lặp ≥ 3 lần → tách util.

```typescript
// ❌ Lặp
const totalA = items.reduce((sum, i) => sum + i.price * i.qty, 0);
const totalB = otherItems.reduce((sum, i) => sum + i.price * i.qty, 0);
const totalC = thirdItems.reduce((sum, i) => sum + i.price * i.qty, 0);

// ✅ Tách
function calculateTotal(items: Array<{ price: number, qty: number }>): number {
  return items.reduce((sum, i) => sum + i.price * i.qty, 0);
}

const totalA = calculateTotal(items);
const totalB = calculateTotal(otherItems);
const totalC = calculateTotal(thirdItems);
```

**Lưu ý:** DRY không có nghĩa là tách mọi thứ. Nếu 2 đoạn code "trông giống" nhưng có ý nghĩa khác — KHÔNG tách (premature abstraction).

---

## 7. ESLint config bắt buộc

```js
// .eslintrc.cjs
module.exports = {
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/strict-type-checked',
    'plugin:@typescript-eslint/stylistic-type-checked',
    'plugin:import/recommended',
    'plugin:import/typescript',
    'plugin:promise/recommended',
    'plugin:security/recommended',
  ],
  rules: {
    '@typescript-eslint/no-explicit-any': 'error',
    '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
    '@typescript-eslint/explicit-function-return-type': ['warn', { allowExpressions: true }],
    '@typescript-eslint/no-floating-promises': 'error',
    '@typescript-eslint/await-thenable': 'error',
    'no-console': ['error', { allow: ['warn', 'error'] }],
    'complexity': ['warn', 10],
    'max-lines-per-function': ['warn', { max: 50, skipBlankLines: true, skipComments: true }],
    'max-depth': ['warn', 3],
    'no-magic-numbers': ['warn', { ignore: [-1, 0, 1, 2, 100, 1000], ignoreArrayIndexes: true }],
  },
};
```

---

**END Rule 08**
