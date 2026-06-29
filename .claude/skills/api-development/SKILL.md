# Skill: API Development

> ⚠️ Đồng bộ ARCHITECTURE_CONTRACT 29/5/2026: PK=uuid, tenant_id=uuid, TS camelCase, RBAC dấu `.`. Xem docs/ARCHITECTURE_CONTRACT.md.

> Khi nào load: Build endpoint hoàn chỉnh từ controller → service → repository → test

---

## Workflow chuẩn (8 bước)

### Bước 1: Đọc spec + threat model

- Đọc Jira/Notion ticket
- Đọc threat model nếu có (xem `skills/security-audit/`)
- Identify: tenant context? auth? rate limit? idempotent?

### Bước 2: Define Zod schema (shared FE+BE)

```typescript
// packages/shared/src/api/orders/schemas.ts
import { z } from 'zod';

export const createOrderSchema = z.object({
  customerId: z.string().min(1),
  items: z.array(z.object({
    productId: z.string(),
    quantity: z.number().int().positive(),
    note: z.string().max(200).optional(),
  })).min(1).max(100),
  customerNote: z.string().max(500).optional(),
  paymentMethod: z.enum(['cash', 'momo', 'vnpay', 'bank_transfer']),
}).strict();

export type CreateOrderInput = z.infer<typeof createOrderSchema>;

export const orderResponseSchema = z.object({
  id: z.string(),
  code: z.string(),
  status: z.enum(['draft', 'confirmed', 'paid', 'shipped', 'completed', 'cancelled', 'refunded']),
  // ...
});

export type OrderResponse = z.infer<typeof orderResponseSchema>;
```

### Bước 3: Define error classes

```typescript
// modules/orders/domain/errors/order-errors.ts
export class OrderNotFoundError extends DomainError {
  readonly code = 'ORDER_NOT_FOUND';
  readonly httpStatus = 404;
  constructor(orderId: string) { super(`Order ${orderId} not found`, { orderId }); }
}

export class InsufficientStockError extends DomainError {
  readonly code = 'INSUFFICIENT_STOCK';
  readonly httpStatus = 422;
  constructor(productId: string, requested: number, available: number) {
    super(`Sản phẩm hết hàng`, { productId, requested, available });
  }
}
```

### Bước 4: Repository (Drizzle queries)

> 📎 **Cross-ref:** Cursor pagination code đầy đủ (findMany +1 trick, hasMore, nextCursor) xem [rules/03-database-drizzle.md](../../rules/03-database-drizzle.md#cursor-pagination)

```typescript
// modules/orders/repositories/orders.repository.ts
@Injectable()
export class OrdersRepository {
  constructor(@Inject(DRIZZLE) private db: Drizzle) {}
  
  async create(data: NewOrder, tx?: Drizzle): Promise<Order> {
    const exec = tx ?? this.db;
    const [order] = await exec.insert(orders).values(data).returning();
    return order;
  }
  
  async findByIdForTenant(id: string, tenantId: string): Promise<Order | null> {
    const result = await this.db.query.orders.findFirst({
      where: and(
        eq(orders.id, id),
        eq(orders.tenant_id, tenantId),
        isNull(orders.deleted_at),
      ),
    });
    return result ?? null;
  }
  
  async listForTenant(params: ListOrdersParams): Promise<{ data: Order[]; nextCursor: string | null }> {
    const limit = Math.min(params.limit, 100);
    
    const items = await this.db.query.orders.findMany({
      where: and(
        eq(orders.tenant_id, params.tenantId),
        isNull(orders.deleted_at),
        params.status ? eq(orders.status, params.status) : undefined,
        params.cursor ? lt(orders.id, params.cursor) : undefined,
      ),
      orderBy: [desc(orders.created_at), desc(orders.id)],
      limit: limit + 1,
    });
    
    const hasMore = items.length > limit;
    const data = hasMore ? items.slice(0, -1) : items;
    
    return {
      data,
      nextCursor: hasMore ? data[data.length - 1].id : null,
    };
  }
}
```

### Bước 5: Service (business logic)

```typescript
// modules/orders/services/orders.service.ts
@Injectable()
export class OrdersService {
  constructor(
    private ordersRepo: OrdersRepository,
    private inventoryService: InventoryService,
    @Inject(DRIZZLE) private db: Drizzle,
    private eventBus: EventEmitter2,
  ) {}
  
  async create(input: CreateOrderInput, user: AuthUser): Promise<Order> {
    return this.db.transaction(async (tx) => {
      // 1. Validate stock
      for (const item of input.items) {
        const stock = await this.inventoryService.getStock(item.productId, user.tenantId, tx);
        if (stock < item.quantity) {
          throw new InsufficientStockError(item.productId, item.quantity, stock);
        }
      }
      
      // 2. Generate code — BẮT BUỘC dùng PG sequence (CONTRACT mục 8: CẤM Math.random/COUNT+1)
      // Ví dụ: const seq = await tx.execute(sql`SELECT nextval('seq_order_' || ${dateKey})`);
      // Format: HD-20260529-0001
      const code = await this.generateOrderCode(user.tenantId, tx);
      
      // 3. Create order
      const order = await this.ordersRepo.create({
        tenant_id: user.tenantId,
        customer_id: input.customerId,
        code,
        status: 'draft',
        subtotal: '0',
        tax_amount: '0',
        discount_amount: '0',
        total: '0',
        currency: 'VND',
        customer_snapshot: await this.snapshotCustomer(input.customerId, tx),
        customer_note: input.customerNote ?? null,
        created_by: user.id,
        // ...
      }, tx);
      
      // 4. Reserve stock + create items
      for (const item of input.items) {
        await this.inventoryService.reserve(item.productId, item.quantity, order.id, tx);
        await this.createOrderItem(order.id, item, tx);
      }
      
      // 5. Recalculate totals
      const updated = await this.recalculateTotals(order.id, tx);
      
      // 6. Outbox event
      await tx.insert(outboxEvents).values({
        aggregate_type: 'Order',
        aggregate_id: order.id,
        event_type: 'orders.order.created',
        payload: serializeOrder(updated),
      });
      
      return updated;
    });
  }
}
```

### Bước 6: Controller (HTTP layer)

```typescript
// modules/orders/controllers/orders.controller.ts
@Controller('api/v1/orders')
@UseGuards(JwtAuthGuard, TenantGuard, PermissionsGuard)
@ApiBearerAuth()
@ApiTags('orders')
export class OrdersController {
  constructor(private ordersService: OrdersService) {}
  
  @Get()
  @RequirePermissions('orders.read')
  @Throttle({ default: { limit: 30, ttl: 60_000 } })
  @ApiOperation({ summary: 'Danh sách đơn hàng' })
  async list(
    @Query() query: ListOrdersDto,
    @CurrentUser() user: AuthUser,
  ): Promise<ListResponse<OrderResponse>> {
    const { data, nextCursor } = await this.ordersService.list({
      tenantId: user.tenantId,
      ...query,
    });
    
    return {
      data: data.map(toOrderResponse),
      meta: {
        nextCursor,
        hasMore: !!nextCursor,
        limit: query.limit ?? 20,
      },
    };
  }
  
  @Get(':id')
  @RequirePermissions('orders.read')
  async findOne(@Param('id') id: string, @CurrentUser() user: AuthUser): Promise<DataResponse<OrderResponse>> {
    const order = await this.ordersService.findByIdForUser(id, user);
    if (!order) throw new OrderNotFoundError(id);
    return { data: toOrderResponse(order) };
  }
  
  @Post()
  @RequirePermissions('orders.create')
  @Throttle({ strict: { limit: 10, ttl: 60_000 } })
  @HttpCode(201)
  async create(
    @Headers('idempotency-key') idempotencyKey: string,
    @Body() body: unknown,
    @CurrentUser() user: AuthUser,
    @Res({ passthrough: true }) res: Response,
  ): Promise<DataResponse<OrderResponse>> {
    if (!idempotencyKey) throw new BadRequestException({ code: 'IDEMPOTENCY_KEY_REQUIRED' });
    
    const input = createOrderSchema.parse(body);
    
    const order = await this.idempotencyService.execute(
      `orders.create:${user.tenantId}:${idempotencyKey}`,
      () => this.ordersService.create(input, user),
    );
    
    res.setHeader('Location', `/api/v1/orders/${order.id}`);
    return { data: toOrderResponse(order) };
  }
  
  @Post(':id/cancel')
  @RequirePermissions('orders.update')
  async cancel(
    @Param('id') id: string,
    @Body() body: { reason: string },
    @CurrentUser() user: AuthUser,
  ): Promise<DataResponse<OrderResponse>> {
    if (!body.reason || body.reason.length < 10) {
      throw new BadRequestException('Reason >= 10 ký tự');
    }
    const order = await this.ordersService.cancel(id, body.reason, user);
    return { data: toOrderResponse(order) };
  }
}
```

### Bước 7: Test (theo `rules/06-testing.md`)

Bắt buộc viết:
- Unit test cho service (mock repo)
- Integration test cho controller (Supertest)
- Security test (cross-tenant, IDOR, unauthorized)

### Bước 8: Update OpenAPI + docs

```typescript
// Tự động qua @nestjs/swagger annotations
@ApiOperation({ summary: 'Tạo đơn hàng mới' })
@ApiHeader({ name: 'Idempotency-Key', required: true })
@ApiResponse({ status: 201, description: 'Đơn hàng được tạo', type: OrderResponseDto })
@ApiResponse({ status: 422, description: 'Validation error' })
@ApiResponse({ status: 409, description: 'Conflict (idempotency key reuse)' })
```

---

## Common patterns

### Pagination response
```typescript
interface ListResponse<T> {
  data: T[];
  meta: {
    nextCursor: string | null;
    prevCursor?: string | null;
    hasMore: boolean;
    limit: number;
    totalEstimate?: number;
  };
}
```

### Single resource response
```typescript
interface DataResponse<T> {
  data: T;
  meta?: { requestId?: string; idempotent?: boolean };
}
```

### Async job response (202)
```typescript
interface JobResponse {
  data: {
    jobId: string;
    status: 'queued' | 'running';
    statusUrl: string;
    estimatedDuration?: number;
  };
}
```

---

## Anti-patterns CẤM

```typescript
// ❌ Logic trong controller
@Post()
async create(@Body() dto) {
  // Validate
  if (!dto.items) throw new BadRequestException();
  // Calculate total
  let total = 0;
  for (const item of dto.items) total += item.price * item.quantity;
  // Insert
  return await db.insert(orders).values({...dto, total});
}

// ✅ Controller chỉ orchestrate
@Post()
async create(@Body() body: unknown, @CurrentUser() user) {
  const input = createOrderSchema.parse(body);
  return { data: await this.ordersService.create(input, user) };
}

// ❌ Trust client
async create(@Body() dto: { tenant_id, customer_id, total, ... }) {
  return db.insert(orders).values(dto);  // Client có thể truyền tenant khác!
}

// ✅ Override từ JWT
async create(@Body() dto: CreateOrderInput, @CurrentUser() user) {
  return db.insert(orders).values({
    ...dto,
    tenant_id: user.tenantId,  // From JWT
    created_by: user.id,
    total: await calculateTotalServerSide(dto.items),  // Recalculate
  });
}
```

---

**END Skill: API Development**
