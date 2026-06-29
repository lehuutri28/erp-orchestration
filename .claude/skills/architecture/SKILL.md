# Skill: Architecture

> Khi nào load: Tạo module mới, refactor lớn, viết ADR, design hệ thống

---

## Bounded contexts hiện tại (11 contexts, 86 modules)

Xem [`rules/00-supreme-principles.md#bounded-contexts`](../../rules/00-supreme-principles.md).

```
_core | _platform | commerce | inventory | manufacturing | hr 
ecommerce | marketing | ai | lms | ops
```

---

## Khi nào tạo module mới?

### Tiêu chí TẠO module mới
- Có domain rõ ràng, khác biệt với module hiện tại
- Có ít nhất 5 use cases độc lập
- Có owner team riêng (hoặc dự kiến)
- Sẽ phát triển dài hạn (không phải one-off feature)

### Tiêu chí MỞ RỘNG module hiện tại
- Sub-feature của domain đã có
- < 5 use cases
- Liên quan chặt với module gốc

### Ví dụ
- ✅ Tạo mới: "Loyalty program" — domain mới, 10+ use cases
- ❌ Tạo mới: "Order export Excel" — chỉ là feature của module orders
- ✅ Tạo mới: "Franchise management" — domain mới hoàn toàn
- ❌ Tạo mới: "Customer phone number validator" — chỉ là util

---

## Quy trình tạo module mới

### Bước 1: ADR (Architecture Decision Record)
File: `docs/adr/ADR-XXX-create-<module-name>.md`

```markdown
# ADR-042: Tạo module Loyalty Program

## Status
Proposed | Accepted | Rejected | Deprecated

## Context
[Tại sao cần module này? Vấn đề gì đang cần giải quyết?]

## Decision
Tạo bounded context mới `marketing/loyalty` với các module:
- loyalty-programs
- loyalty-tiers
- loyalty-rewards
- loyalty-redemptions

## Consequences
### Positive
- [...]
### Negative
- [...]

## Alternatives considered
1. [Alternative 1] — rejected because [...]
2. [Alternative 2] — rejected because [...]
```

### Bước 2: Tạo cấu trúc folder

```bash
mkdir -p apps/api/src/modules/marketing/loyalty/{controllers,services,repositories,domain/{entities,value-objects,events,errors},dto,automation,guards,decorators}

# Skeleton files
touch apps/api/src/modules/marketing/loyalty/{loyalty.module.ts,README.md}
touch apps/api/src/modules/marketing/loyalty/controllers/loyalty.controller.ts
touch apps/api/src/modules/marketing/loyalty/services/loyalty.service.ts
# ...
```

### Bước 3: Viết README.md (BẮT BUỘC)

Theo template trong `rules/00-supreme-principles.md#cấu-trúc-1-nestjs-module-chuẩn`.

### Bước 4: Viết schema (Drizzle)

Theo `rules/03-database-drizzle.md`.

### Bước 5: Viết test plan

Theo `rules/06-testing.md`.

### Bước 6: PR + review

Theo `rules/07-git-workflow.md`.

---

## Refactor lớn — quy tắc

### Khi nào refactor?
- Code đã xong, đã có test, đã hoạt động ở production ≥ 1 tháng
- Có pain point cụ thể (slow, hard to maintain, hard to test)
- Có owner cam kết maintain sau refactor

### Khi nào KHÔNG refactor?
- "Code trông không đẹp" (subjective)
- Chưa hiểu rõ business logic
- Không có test coverage > 80%
- Đang sắp release lớn

### Quy tắc
1. KHÔNG refactor + thêm feature trong cùng 1 PR
2. KHÔNG refactor xuyên 5+ module trong 1 PR
3. PHẢI có rollback plan
4. PHẢI có benchmark trước/sau (nếu refactor performance)

---

## Hexagonal architecture (Ports & Adapters)

```
┌─────────────────────────────────────┐
│        APPLICATION CORE             │
│  ┌─────────────────────────────┐    │
│  │  Domain (entities, VO)      │    │
│  └─────────────────────────────┘    │
│  ┌─────────────────────────────┐    │
│  │  Use cases (services)       │    │
│  └────────┬─────────────┬──────┘    │
│           │             │            │
│  ┌────────▼─────┐ ┌─────▼──────┐    │
│  │ Inbound Port │ │ Outbound   │    │
│  │ (interface)  │ │ Port       │    │
│  └────────▲─────┘ └─────▲──────┘    │
└───────────┼─────────────┼────────────┘
            │             │
   ┌────────┴────┐  ┌─────┴──────────┐
   │ Inbound     │  │ Outbound       │
   │ Adapter     │  │ Adapter        │
   │ (REST, gRPC,│  │ (Drizzle, S3,  │
   │  GraphQL)   │  │  email, queue) │
   └─────────────┘  └────────────────┘
```

**Ưu điểm:**
- Test core không cần infra
- Swap adapter (e.g., PostgreSQL → MySQL) không sửa core
- Thêm channel mới (REST → gRPC) không sửa core

---

## Cross-context communication

### Pattern 1: Public service interface (port)

```typescript
// packages/contracts/src/hr/employee-query.port.ts
export interface EmployeeQueryPort {
  findById(id: string): Promise<Employee | null>;
  findByDepartment(departmentId: string): Promise<Employee[]>;
}

// hr/hr-employees/hr-employees.module.ts (provider)
@Module({
  providers: [
    HrEmployeesService,
    { provide: 'EMPLOYEE_QUERY_PORT', useExisting: HrEmployeesService },
  ],
  exports: ['EMPLOYEE_QUERY_PORT'],
})

// manufacturing/services/... (consumer)
constructor(@Inject('EMPLOYEE_QUERY_PORT') private employeeQuery: EmployeeQueryPort) {}
```

### Pattern 2: Domain event

```typescript
// hr emit event
@OnEvent('employee.terminated')
async handleTermination(event: EmployeeTerminatedEvent) {
  // Manufacturing tự xử lý: revoke machine access, ...
}
```

### Pattern 3: Saga (multi-step)

Xem [`rules/14-event-driven.md#7-saga-pattern`](../../rules/14-event-driven.md).

---

## Anti-patterns CẤM 🔴

```typescript
// ❌ Import xuyên context
import { HrEmployeesService } from '../../hr/...';

// ❌ God service (1 service làm 10 việc)
class OrderService {
  async createOrder() {}
  async sendEmail() {}        // Should be in NotificationService
  async generatePdf() {}      // Should be in PdfService
  async syncShopee() {}       // Should be in ShopeeAdapter
  async updateInventory() {}  // Should be in InventoryService
  // ...
}

// ❌ Anemic domain (entity chỉ là data, no behavior)
class Order {
  id: string;
  total: number;
  status: string;
  // No methods!
}
// Service làm hết: validateOrder(), calculateTotal(), ...
// → Logic scatter, hard to test

// ✅ Rich domain
class Order {
  constructor(
    public readonly id: string,
    private items: OrderItem[],
    private status: OrderStatus,
  ) {}
  
  cancel(reason: string): void {
    if (this.status === 'shipped') {
      throw new OrderAlreadyShippedError();
    }
    this.status = 'cancelled';
    // Emit domain event
  }
  
  calculateTotal(): Money {
    return this.items.reduce((sum, item) => sum.add(item.subtotal()), Money.zero('VND'));
  }
}
```

---

**END Skill: Architecture**
