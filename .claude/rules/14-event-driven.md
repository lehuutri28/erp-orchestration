# Rule 14 — Event-Driven Architecture

> Mức độ: 🟠 HIGH | Áp dụng: Cross-module communication, async processing

---

## 1. Outbox pattern (BẮT BUỘC cho cross-service event)

### Tại sao Outbox?
- Đảm bảo event được publish sau khi DB transaction commit thành công
- Không mất event nếu broker xuống tạm thời
- Replay được khi cần debug

### Implementation:

```typescript
// Schema
CREATE TABLE outbox_events (
  id BIGSERIAL PRIMARY KEY,
  aggregate_type TEXT NOT NULL,     -- 'Order', 'Customer'
  aggregate_id TEXT NOT NULL,
  event_type TEXT NOT NULL,         -- 'order.created'
  payload JSONB NOT NULL,
  metadata JSONB,                   -- requestId, userId, tenantId
  occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  published BOOLEAN NOT NULL DEFAULT false,
  published_at TIMESTAMPTZ,
  attempts INTEGER NOT NULL DEFAULT 0,
  last_error TEXT
);

CREATE INDEX outbox_events_unpublished_idx ON outbox_events (published, occurred_at)
  WHERE NOT published;

// Trong transaction tạo order
await db.transaction(async (tx) => {
  const [order] = await tx.insert(orders).values({...}).returning();
  
  await tx.insert(outboxEvents).values({
    aggregate_type: 'Order',
    aggregate_id: order.id,
    event_type: 'order.created',
    payload: serializeOrder(order),
    metadata: {
      tenant_id: getCurrentTenantId(),
      user_id: getCurrentUserId(),
      request_id: req.id,
    },
  });
});

// Background worker — poll outbox → publish
@Injectable()
export class OutboxPublisher {
  @Cron('*/5 * * * * *')  // Every 5s
  async publishPendingEvents() {
    const events = await db.query.outboxEvents.findMany({
      where: eq(outboxEvents.published, false),
      limit: 100,
      orderBy: asc(outboxEvents.occurred_at),
    });
    
    for (const event of events) {
      try {
        await this.eventBus.publish(event.event_type, event.payload, event.metadata);
        
        await db.update(outboxEvents)
          .set({ published: true, published_at: new Date() })
          .where(eq(outboxEvents.id, event.id));
      } catch (error) {
        await db.update(outboxEvents)
          .set({
            attempts: sql`${outboxEvents.attempts} + 1`,
            last_error: String(error),
          })
          .where(eq(outboxEvents.id, event.id));
        
        if (event.attempts >= 5) {
          await this.alertOps('outbox_publish_failed', { eventId: event.id });
        }
      }
    }
  }
}
```

---

## 2. Event store (append-only)

```sql
CREATE TABLE event_log (
  id BIGSERIAL PRIMARY KEY,
  event_id TEXT NOT NULL UNIQUE,           -- ULID hoặc cuid2
  tenant_id TEXT NOT NULL,
  aggregate_type TEXT NOT NULL,
  aggregate_id TEXT NOT NULL,
  event_type TEXT NOT NULL,
  event_version INT NOT NULL DEFAULT 1,
  payload JSONB NOT NULL,
  metadata JSONB,
  occurred_at TIMESTAMPTZ NOT NULL,
  recorded_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX event_log_tenant_aggregate_idx 
  ON event_log (tenant_id, aggregate_type, aggregate_id, occurred_at);

CREATE INDEX event_log_type_time_idx 
  ON event_log (event_type, occurred_at);

-- Append-only: revoke UPDATE/DELETE
REVOKE UPDATE, DELETE ON event_log FROM PUBLIC;
```

---

## 3. Event naming convention

```
<bounded_context>.<aggregate>.<action>

✅ orders.order.created
✅ orders.order.cancelled
✅ inventory.stock.depleted
✅ payments.payment.failed
✅ hr.employee.terminated
✅ lms.enrollment.completed
✅ franchise.branch.activated

❌ created                       # Quá generic
❌ orderUpdated                  # camelCase + thiếu domain
❌ order_was_created             # past tense, dùng past participle
```

**Versioning:**
- v1 → `orders.order.created`
- v2 (breaking) → `orders.order.created.v2`

---

## 4. Event payload schema

```typescript
// packages/shared/src/events/orders.ts
import { z } from 'zod';

export const orderCreatedEventSchema = z.object({
  // Envelope
  eventId: z.string(),
  eventType: z.literal('orders.order.created'),
  eventVersion: z.literal(1),
  occurredAt: z.string().datetime(),
  
  // Context
  tenantId: z.string(),
  userId: z.string().nullable(),
  requestId: z.string().nullable(),
  
  // Payload
  data: z.object({
    orderId: z.string(),
    customerId: z.string(),
    total: z.string(),  // Decimal as string
    currency: z.string(),
    itemCount: z.number(),
    channel: z.enum(['pos', 'web', 'mobile', 'shopee', 'lazada']),
  }),
});

export type OrderCreatedEvent = z.infer<typeof orderCreatedEventSchema>;
```

---

## 5. Event handler pattern

```typescript
// inventory/automation/handlers.ts
@Injectable()
export class InventoryEventHandlers {
  constructor(
    private readonly inventoryService: InventoryService,
    private readonly logger: Logger,
  ) {}
  
  @OnEvent('orders.order.created')
  async handleOrderCreated(event: OrderCreatedEvent) {
    try {
      // Idempotent — check đã xử lý chưa
      const processed = await this.cache.get(`evt_processed:${event.eventId}`);
      if (processed) return;
      
      // Validate event
      const validated = orderCreatedEventSchema.parse(event);
      
      // Set tenant context
      await withTenantContext(validated.tenantId, async () => {
        // Business logic
        await this.inventoryService.reserveStockForOrder(validated.data.orderId);
      });
      
      // Mark processed
      await this.cache.set(`evt_processed:${event.eventId}`, '1', 'EX', 86400);
      
    } catch (error) {
      this.logger.error('Failed to handle order.created', { event, error });
      throw error;  // Will be retried by event bus
    }
  }
}
```

---

## 6. Cross-bounded-context communication

> 📎 **Cross-ref:** Port/adapter pattern tổng quát (bounded contexts, DI token, forbidden direct import) xem [rules/00-supreme-principles.md](./00-supreme-principles.md#cross-bounded-context)

```typescript
// ❌ CẤM — manufacturing import trực tiếp file của hr
import { HrEmployeesService } from '../../hr/hr-employees/services/hr-employees.service';

// ✅ ĐÚNG — qua public interface (port/adapter)
// packages/contracts/src/hr/employee-query.port.ts
export interface EmployeeQueryPort {
  findById(id: string): Promise<Employee | null>;
  findByDepartment(departmentId: string): Promise<Employee[]>;
}

// hr/hr-employees/hr-employees.module.ts
@Module({
  providers: [
    {
      provide: 'EMPLOYEE_QUERY_PORT',
      useClass: HrEmployeesService,  // Implementation
    },
  ],
  exports: ['EMPLOYEE_QUERY_PORT'],
})
export class HrEmployeesModule {}

// manufacturing/services/...
constructor(
  @Inject('EMPLOYEE_QUERY_PORT') private employeeQuery: EmployeeQueryPort,
) {}

// ✅ ĐÚNG — qua event
@OnEvent('hr.employee.terminated')
async handleEmployeeTerminated(event: EmployeeTerminatedEvent) {
  // Manufacturing tự xử lý
}
```

---

## 7. Saga pattern (multi-step transaction)

```typescript
// Order placement saga
async function placeOrderSaga(dto: CreateOrderDto): Promise<Order> {
  const compensations: Array<() => Promise<void>> = [];
  
  try {
    // Step 1: Create order (draft)
    const order = await this.ordersService.createDraft(dto);
    compensations.push(() => this.ordersService.deleteDraft(order.id));
    
    // Step 2: Reserve stock
    await this.inventoryService.reserveStock(order.items);
    compensations.push(() => this.inventoryService.releaseStock(order.items));
    
    // Step 3: Charge payment
    const payment = await this.paymentsService.charge(order.total, dto.paymentMethod);
    compensations.push(() => this.paymentsService.refund(payment.id));
    
    // Step 4: Confirm order
    const confirmed = await this.ordersService.confirm(order.id, payment.id);
    
    return confirmed;
    
  } catch (error) {
    // Run compensations in reverse
    this.logger.error('Saga failed, running compensations', { error });
    
    for (const compensate of compensations.reverse()) {
      try {
        await compensate();
      } catch (compensateError) {
        this.logger.error('Compensation failed', { compensateError });
        // Alert ops — manual intervention needed
        await this.alertOps('saga_compensation_failed', { error, compensateError });
      }
    }
    
    throw error;
  }
}
```

---

## 8. n8n integration

n8n workflow trigger qua webhook → ERP API → emit event:

```
n8n: Cron 6AM
  ↓
Webhook → POST /api/v1/inventory/auto-reorder-check
  ↓
ERP API: query products có stock < min
  ↓
For each product:
  - Create draft PO
  - Emit event: 'inventory.po.draft_created'
  ↓
n8n subscribe via webhook → notify Telegram team procurement
```

**Quy tắc:**
- n8n KHÔNG chứa business logic phức tạp
- Logic ở ERP API, n8n chỉ orchestrate
- Mọi n8n call vào ERP có HMAC signature
- Mọi execution lưu vào `workflow_executions` table

---

**END Rule 14**
