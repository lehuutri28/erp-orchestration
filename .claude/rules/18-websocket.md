# Rule 18 — WebSocket (Socket.io Multi-tenant)

> Mức độ: 🟠 HIGH | Áp dụng: POS realtime, kitchen display, low-stock alert, chat nội bộ

---

## 1. Kiến trúc multi-tenant với Socket.io

```
[Client] ──connect(token)──▶ [Socket.io Server]
                               │
                               ├─ Verify JWT (giống HTTP guard)
                               ├─ Resolve tenant_id từ token
                               ├─ Join room: `tenant:{tenant_id}`
                               └─ Join room phụ theo role:
                                  ├─ `kitchen:{branch_id}`     (kitchen display)
                                  ├─ `pos:{terminal_id}`       (POS terminal cụ thể)
                                  └─ `manager:{branch_id}`     (manager alert)
```

**Nguyên tắc bất khả xâm phạm:**
- Event chỉ broadcast trong room `tenant:{id}` — KHÔNG global broadcast
- Mỗi room namespace theo tenant, KHÔNG share cross-tenant
- Server KHÔNG tin client tự khai `tenant_id` trong event payload

---

## 2. Auth — JWT trong handshake, verify mỗi connection

```typescript
// gateway/pos-realtime.gateway.ts
import { WebSocketGateway, WebSocketServer, OnGatewayConnection, OnGatewayDisconnect } from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';

@WebSocketGateway({
  namespace: '/pos',
  cors: { origin: process.env.ALLOWED_ORIGINS?.split(',') ?? [] },
  transports: ['websocket'],  // Không fallback HTTP long-poll trong production
})
export class PosRealtimeGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer() server: Server;

  constructor(
    private readonly jwtService: JwtService,
    private readonly rateLimitStore: Redis,
  ) {}

  async handleConnection(socket: Socket): Promise<void> {
    try {
      // 1. Lấy token từ handshake auth (KHÔNG từ query string khi production)
      const token = socket.handshake.auth?.token
        ?? socket.handshake.headers.authorization?.replace('Bearer ', '');

      if (!token) {
        socket.emit('error', { code: 'AUTH_REQUIRED' });
        socket.disconnect(true);
        return;
      }

      // 2. Verify JWT — giống HTTP guard
      const payload = await this.jwtService.verifyAsync(token);

      // 3. Rate limit per IP — ngăn connection flood
      const ip = socket.handshake.address;
      const connCount = await this.rateLimitStore.incr(`ws:conn:${ip}`);
      await this.rateLimitStore.expire(`ws:conn:${ip}`, 60);
      if (connCount > 20) {  // Tối đa 20 connection/IP/phút
        socket.emit('error', { code: 'RATE_LIMITED' });
        socket.disconnect(true);
        return;
      }

      // 4. Gán metadata vào socket data — để reuse không phải decode lại
      socket.data = {
        userId: payload.sub,
        tenantId: payload.tenantId,
        branchId: payload.branchId,
        roles: payload.roles,
        tokenExp: payload.exp,
      };

      // 5. Join room theo tenant — KHÔNG global
      await socket.join(`tenant:${payload.tenantId}`);

      // 6. Join room phụ theo role
      if (payload.roles.includes('kitchen_staff')) {
        await socket.join(`kitchen:${payload.branchId}`);
      }
      if (payload.roles.includes('pos_terminal')) {
        await socket.join(`pos:${payload.terminalId}`);
      }
      if (payload.roles.includes('store_manager')) {
        await socket.join(`manager:${payload.branchId}`);
      }

    } catch (err) {
      // JWT expired hoặc invalid
      socket.emit('error', { code: 'AUTH_INVALID' });
      socket.disconnect(true);
    }
  }

  async handleDisconnect(socket: Socket): Promise<void> {
    // Room được tự clean bởi socket.io khi disconnect
    // Log disconnect nếu cần audit
  }
}
```

---

## 3. Token expiry — disconnect khi JWT hết hạn

```typescript
// Trong handleConnection — set timer tự disconnect
const msUntilExpiry = (socket.data.tokenExp * 1000) - Date.now();
if (msUntilExpiry <= 0) {
  socket.disconnect(true);
  return;
}

// Tự disconnect khi token expire — client phải reconnect với token mới
const timer = setTimeout(() => {
  socket.emit('error', { code: 'TOKEN_EXPIRED' });
  socket.disconnect(true);
}, msUntilExpiry);

socket.on('disconnect', () => clearTimeout(timer));
```

---

## 4. Rate limit per socket — ngăn event spam

```typescript
// Middleware rate limit cho mỗi event
export function socketRateLimitMiddleware(
  redis: Redis,
  maxEvents: number,
  windowSecs: number,
) {
  return async (socket: Socket, next: () => void) => {
    socket.use(async ([event, ...args], next) => {
      const key = `ws:rate:${socket.id}:${event}`;
      const count = await redis.incr(key);
      if (count === 1) await redis.expire(key, windowSecs);

      if (count > maxEvents) {
        socket.emit('error', { code: 'EVENT_RATE_LIMITED', event });
        return;  // Drop event, không disconnect
      }
      next();
    });
    next();
  };
}

// Áp dụng: tối đa 60 event/phút mỗi socket
io.use(socketRateLimitMiddleware(redis, 60, 60));
```

---

## 5. Broadcast pattern — tenant-scoped

```typescript
// service emit event — KHÔNG dùng server.emit() global
@Injectable()
export class PosRealtimeService {
  constructor(
    @InjectSocketServer() private readonly server: Server,
  ) {}

  // ✅ Broadcast trong tenant room
  emitOrderStatusChanged(tenantId: string, payload: OrderStatusPayload): void {
    this.server
      .to(`tenant:${tenantId}`)
      .emit('order:status_changed', payload);
  }

  // ✅ Broadcast chỉ kitchen của branch
  emitNewKitchenTicket(branchId: string, tenantId: string, ticket: KitchenTicket): void {
    // Kitchen room đã được scope theo tenantId trong naming
    this.server
      .to(`kitchen:${branchId}`)
      .emit('kitchen:new_ticket', ticket);
    // Note: kitchen room chỉ join được khi token đúng tenant → isolation OK
  }

  // ✅ Low-stock alert → manager
  emitLowStockAlert(branchId: string, alert: LowStockAlert): void {
    this.server
      .to(`manager:${branchId}`)
      .emit('inventory:low_stock', alert);
  }

  // ❌ CẤM
  // this.server.emit('order:updated', payload)  // Global broadcast!
}
```

---

## 6. Message schema — structured events

```typescript
// Mọi event PHẢI có schema chuẩn
export interface WsEvent<T> {
  eventId: string;       // UUID — idempotency
  eventType: string;     // 'order:status_changed'
  tenantId: string;      // Redundant check client-side
  occurredAt: string;    // ISO 8601
  payload: T;
}

// Schema ví dụ — order status
export interface OrderStatusPayload {
  orderId: string;
  orderCode: string;
  oldStatus: OrderStatus;
  newStatus: OrderStatus;
  updatedBy: string;
  branchId: string;
}

// Zod validate khi nhận event từ client
const clientOrderUpdateSchema = z.object({
  orderId: z.string().uuid(),
  action: z.enum(['confirm', 'cancel', 'mark_ready']),
});
```

---

## 7. POS realtime use cases

| Use case | Room | Event | Payload tối thiểu |
|----------|------|-------|-------------------|
| Order mới từ kiosk | `kitchen:{branchId}` | `kitchen:new_ticket` | orderId, items[], priority |
| Order status thay đổi | `tenant:{tenantId}` | `order:status_changed` | orderId, newStatus |
| Thanh toán thành công | `pos:{terminalId}` | `payment:confirmed` | orderId, amount, method |
| Tồn kho cảnh báo thấp | `manager:{branchId}` | `inventory:low_stock` | productId, currentQty, threshold |
| Màn hình bếp refresh | `kitchen:{branchId}` | `kitchen:queue_updated` | queueSnapshot |

---

## 8. Anti-pattern ❌

| Anti-pattern | Vấn đề |
|-------------|--------|
| `socket.handshake.query.token` trong production | Token lộ trong server log URL |
| `server.emit(event, data)` global | Cross-tenant data leak |
| Không verify JWT trong `handleConnection` | Bất kỳ ai connect được |
| Tin `socket.data.tenantId` từ client | Client có thể fake tenant |
| Không đặt rate limit | Event flood → CPU spike |
| Room name không prefix tenant | `kitchen:1` của tenant A có thể join từ tenant B |
| Không disconnect khi JWT expire | Zombie connection giữ room |
| Không giới hạn max connection | DoS bằng mở 10k connection |

---

## 9. Checklist trước khi deploy WebSocket feature

- [ ] JWT verify trong `handleConnection` — không skip
- [ ] Room join chỉ sau khi verify token thành công
- [ ] Room name prefix với tenant/branch context phù hợp
- [ ] Rate limit per socket và per IP connection
- [ ] Token expiry timer — auto disconnect
- [ ] Broadcast dùng `.to(room)` — không `.emit()` global
- [ ] Message schema validate bằng Zod khi nhận từ client
- [ ] Test: client với token tenant A KHÔNG nhận event của tenant B
- [ ] Test: JWT expired → disconnect trong ≤5s
- [ ] Test: spam 100 event/s → rate limit kick in

---

## Cross-ref

- `rules/02-multi-tenant.md` — Isolation pattern, tenant context
- `rules/04-api-design.md` — Auth pattern chung (JWT)
- `rules/01-security.md` — OWASP A07 (Auth failures)
- `rules/10-error-handling.md` — Error code format
- `rules/14-event-driven.md` — Event schema + idempotency
