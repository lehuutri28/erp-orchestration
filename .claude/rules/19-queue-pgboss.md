# Rule 19 — Queue với pg-boss

> Mức độ: 🟠 HIGH | Áp dụng: Background job, async processing, scheduled task
> Stack chính thức: **pg-boss 10.1.x** (KHÔNG phải BullMQ — queue backend là PostgreSQL)

---

## 1. Tại sao pg-boss (không phải Redis queue)

| Tiêu chí | pg-boss | BullMQ (Redis) |
|---------|---------|----------------|
| Backend | PostgreSQL (đã có) | Redis (infra thêm) |
| Durability | ACID — không mất job | Best-effort nếu Redis crash |
| Idempotency | Built-in với `key` param | Tự implement |
| Dead-letter | Built-in | Cần config thêm |
| Visibility | Query SQL trực tiếp | Cần Redis CLI |
| Phù hợp cho | Job số lượng vừa, cần ACID | Job throughput cực cao (>10k/s) |

**Kết luận:** ERP có tần suất job vừa phải, cần audit trail, pg-boss là lựa chọn đúng.

---

## 2. Setup pg-boss trong NestJS

```typescript
// modules/queue/pg-boss.module.ts
import PgBoss from 'pg-boss';

@Global()
@Module({})
export class PgBossModule implements OnModuleInit, OnModuleDestroy {
  private boss: PgBoss;

  async onModuleInit() {
    this.boss = new PgBoss({
      connectionString: process.env.DATABASE_URL,
      schema: 'pgboss',              // Schema riêng, không đụng schema app
      monitorStateIntervalSeconds: 30,
      deleteAfterDays: 30,           // Tự cleanup job cũ 30 ngày
      archiveCompletedAfterSeconds: 3600,  // Archive sau 1h complete
    });

    await this.boss.start();
    await this.boss.work('*', { teamSize: 5, teamConcurrency: 2 }, this.logUnhandledJob);
  }

  async onModuleDestroy() {
    await this.boss.stop({ graceful: true, timeout: 30000 });
  }

  getInstance(): PgBoss { return this.boss; }
}
```

---

## 3. Job retry strategy — exponential backoff

```typescript
// Tạo job với retry policy chuẩn
await boss.send(
  'email:send-invoice',                  // Queue name
  { orderId, recipientEmail },           // Payload
  {
    // Idempotency key — BẮTBUỘC để tránh duplicate
    key: `invoice:${orderId}`,

    // Retry: tối đa 3 lần
    retryLimit: 3,

    // Exponential backoff: 30s → 90s → 270s
    retryDelay: 30,                      // Giây — lần đầu
    retryBackoff: true,                  // pg-boss tự tính exponential

    // Job expire nếu không xử lý được trong 10 phút
    expireInMinutes: 10,

    // Priority — 0 là mặc định, số dương = ưu tiên cao hơn
    priority: 0,
  },
);
```

**Backoff thực tế với `retryBackoff: true`:**
- Attempt 1 fail → chờ 30s
- Attempt 2 fail → chờ 30 × 2 = 60s
- Attempt 3 fail → chờ 30 × 4 = 120s → Dead-letter

---

## 4. Idempotency key — bắt buộc

```typescript
// Rule: MỌI job send PHẢI có `key` nếu có nguy cơ duplicate

// ✅ Idempotency key = business identifier duy nhất
await boss.send('invoice:create', { orderId }, {
  key: `invoice:create:${orderId}`,     // Không send 2 invoice cho cùng order
});

await boss.send('sync:shopee-order', { shopeeOrderId }, {
  key: `shopee:order:${shopeeOrderId}`, // Idempotent khi webhook retry
});

// ❌ Không có key — dễ duplicate khi retry
await boss.send('email:send', { to, subject, body });
```

---

## 5. Worker implementation chuẩn

```typescript
@Injectable()
export class InvoiceJobWorker implements OnModuleInit {
  constructor(
    @InjectPgBoss() private readonly boss: PgBoss,
    private readonly invoiceService: InvoiceService,
    private readonly auditLogService: AuditLogService,
    private readonly alertService: AlertService,
  ) {}

  async onModuleInit() {
    // Register worker — teamSize = concurrent job per worker instance
    await this.boss.work<{ orderId: string }>(
      'invoice:create',
      { teamSize: 3, teamConcurrency: 1 },
      this.handleCreateInvoice.bind(this),
    );
  }

  async handleCreateInvoice(job: PgBoss.Job<{ orderId: string }>): Promise<void> {
    const { orderId } = job.data;
    const startedAt = Date.now();

    try {
      await this.invoiceService.createForOrder(orderId);

      // Log thành công vào audit
      await this.auditLogService.log({
        action: 'invoice.created',
        entity: 'order',
        entityId: orderId,
        jobId: job.id,
        durationMs: Date.now() - startedAt,
      });

    } catch (err) {
      // Log thất bại — KHÔNG nuốt lỗi
      await this.auditLogService.log({
        action: 'invoice.create_failed',
        entity: 'order',
        entityId: orderId,
        jobId: job.id,
        error: err.message,
        attempt: job.retryCount,
      });

      // Alert Telegram nếu đây là lần cuối cùng (sắp vào dead-letter)
      if (job.retryCount >= (job.retryLimit ?? 3) - 1) {
        await this.alertService.telegram(
          `CRITICAL: job invoice:create FAIL lần cuối\nOrder: ${orderId}\nError: ${err.message}`,
        );
      }

      // Re-throw để pg-boss biết job fail → retry hoặc dead-letter
      throw err;
    }
  }
}
```

---

## 6. Dead-letter queue — handling

```typescript
// Subscribe dead-letter queue — job fail hết retry
await boss.work('__state__failed', async (job) => {
  const failedJob = job.data as PgBoss.Job;

  // Alert Telegram với context đầy đủ
  await alertService.telegram(
    `DEAD LETTER: ${failedJob.name}\n` +
    `ID: ${failedJob.id}\n` +
    `Payload: ${JSON.stringify(failedJob.data)}\n` +
    `Last error: ${failedJob.output?.message ?? 'unknown'}\n` +
    `Attempts: ${failedJob.retryCount}/${failedJob.retryLimit}`,
  );

  // Ghi vào audit log cho compliance
  await auditLogService.log({
    action: 'job.dead_lettered',
    metadata: {
      jobId: failedJob.id,
      jobName: failedJob.name,
      payload: failedJob.data,
      lastError: failedJob.output,
    },
  });

  // Manual investigation / replay sau
});

// Replay job từ dead-letter (khi fix xong)
// pg-boss không có built-in replay — query thủ công:
// UPDATE pgboss.job SET state = 'created', retrycount = 0 WHERE id = '<id>'
```

---

## 7. Scheduled job (cron)

```typescript
// Cron hàng ngày — sync inventory Shopee lúc 2AM
await boss.schedule(
  'sync:shopee-inventory',
  '0 2 * * *',            // Cron expression
  {},                     // Payload (empty nếu không cần)
  {
    tz: 'Asia/Ho_Chi_Minh',
    key: 'daily-shopee-sync',  // Đảm bảo chỉ 1 scheduled job tồn tại
  },
);
```

---

## 8. Anti-pattern ❌

| Anti-pattern | Vấn đề |
|-------------|--------|
| Không có `key` (idempotency) | Duplicate job khi retry webhook |
| `try { ... } catch {}` nuốt lỗi | pg-boss không biết fail → không retry |
| `retryLimit: 0` | Job fail một lần → đi thẳng dead-letter |
| Không alert dead-letter | Job "biến mất" không ai biết |
| Logic phức tạp trong 1 job | Khó debug khi fail ở giữa — tách job nhỏ |
| Job payload có binary/buffer lớn | PostgreSQL JSONB không phù hợp file |
| Không log duration | Không biết bottleneck |
| `teamSize` quá cao | Connection pool exhausted |

---

## 9. Queue name convention

```
<context>:<action>           → invoice:create, email:send-receipt
sync:<platform>-<entity>     → sync:shopee-order, sync:lazada-inventory
report:<type>                → report:daily-revenue, report:kpi-monthly
notify:<channel>             → notify:zalo, notify:telegram
```

---

## 10. Checklist trước khi deploy job mới

- [ ] `key` idempotency đặt đúng (business identifier)
- [ ] `retryLimit: 3` + `retryBackoff: true`
- [ ] Worker re-throw error (không nuốt)
- [ ] Alert Telegram khi hết retry
- [ ] Audit log cho cả success và failure
- [ ] Dead-letter subscriber active
- [ ] Queue name theo convention `<context>:<action>`
- [ ] Test: job fail lần 1, 2 → retry; lần 3 → dead-letter + alert

---

## Cross-ref

- `rules/10-error-handling.md` — Error log pattern, không nuốt lỗi
- `rules/14-event-driven.md` — Outbox pattern + idempotency key
- `rules/11-logging-observability.md` — Audit log format
- `skills/n8n-workflow/SKILL.md` — Phân biệt pg-boss job vs n8n workflow
