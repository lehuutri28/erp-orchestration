# Rule 09 — Performance Budget

> Mức độ: 🟡 MEDIUM | Áp dụng: Endpoint mới, query mới, page mới

---

## 1. Latency targets (P50/P95/P99)

| Endpoint type | P50 | P95 | P99 |
|---|---|---|---|
| Health check | < 10ms | < 50ms | < 100ms |
| Auth (login) | < 200ms | < 500ms | < 1000ms |
| Read single resource | < 50ms | < 200ms | < 500ms |
| Read list (paginated) | < 100ms | < 300ms | < 800ms |
| Search (full-text) | < 200ms | < 500ms | < 1000ms |
| Create resource | < 200ms | < 500ms | < 1000ms |
| Complex transaction | < 500ms | < 1500ms | < 3000ms |
| Report sync | < 1s | < 3s | < 5s |
| File upload (10MB) | < 2s | < 5s | < 10s |
| AI agent call | < 3s | < 10s | < 20s |

**Action when exceeded:**
- P95 vượt 2x → log warning, investigate
- P99 vượt 3x → alert Sentry
- Liên tục P95 vượt 1 giờ → page on-call

---

## 2. Throughput targets

| Layer | Target |
|---|---|
| API requests | 1000 req/s sustained |
| Database reads | 5000 query/s |
| Database writes | 500 query/s |
| Background jobs | 100 job/s |
| WebSocket connections | 5000 concurrent |

---

## 3. Resource limits

```yaml
# docker-compose.prod.yml
services:
  api:
    deploy:
      resources:
        limits: { cpus: '2', memory: 2G }
        reservations: { cpus: '0.5', memory: 512M }
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4000/api/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 30s
```

---

## 4. Caching strategy

### Client-side (browser)
```http
# Static assets
Cache-Control: public, max-age=31536000, immutable

# API list responses (đã có ETag)
Cache-Control: private, max-age=60, must-revalidate

# API mutation
Cache-Control: no-store
```

### Server-side (Redis)

| Data type | TTL | Invalidation |
|---|---|---|
| User session | 7 days | Logout, password change |
| User permissions | 5 min | Role change |
| Product catalog | 10 min | Product update |
| Tenant config | 1 hour | Config change |
| AI response (same prompt) | 1 hour | TTL |
| Rate limit counter | 1-60 min | Sliding window |

**Cache key naming:**
```
{tenant_id}:{resource}:{id}           # User session
{tenant_id}:product:{product_id}      # Product cache
rate_limit:{ip}:{endpoint}            # Rate limit
idem:{tenant_id}:{idempotency_key}    # Idempotency
agent:cache:{prompt_hash}             # AI response cache
```

---

## 5. Frontend performance

**Bundle size target:**
- First Load JS (route): < 200 KB gzip
- Per-route JS: < 100 KB gzip
- Total page weight: < 1 MB

**Core Web Vitals (target P75):**
- LCP (Largest Contentful Paint): < 2.5s
- INP (Interaction to Next Paint): < 200ms
- CLS (Cumulative Layout Shift): < 0.1
- FCP (First Contentful Paint): < 1.8s

**Tools:**
```bash
# Analyze bundle
ANALYZE=true pnpm build

# Lighthouse CI
pnpm dlx @lhci/cli@0.13.x autorun
```

---

## 6. Database performance

> 📎 **Cross-ref:** Chi tiết database performance (EXPLAIN ANALYZE, connection pool config, slow query log setup) xem [rules/03-database-drizzle.md](./03-database-drizzle.md#performance) — file này chỉ giữ summary.

**Slow query log:**
```ini
# postgresql.conf
log_min_duration_statement = 100  # Log queries > 100ms
log_statement = 'mod'              # Log all DDL + writes
log_duration = on
```

**EXPLAIN ANALYZE bắt buộc cho query mới:**
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT * FROM orders WHERE tenant_id = $1 AND status = $2 ORDER BY created_at DESC LIMIT 20;

-- Phải dùng index, không Seq Scan trên bảng > 1000 rows
-- Cost < 1000 cho query thường
```

**Connection pooling:**
```typescript
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20, min: 2,
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 5_000,
  statement_timeout: 30_000,
  query_timeout: 30_000,
});
```

---

## 7. Anti-patterns CẤM

```typescript
// ❌ N+1 query
for (const order of orders) {
  const customer = await db.select().from(customers).where(eq(customers.id, order.customer_id));
}

// ❌ Fetch toàn bộ rồi filter
const all = await db.select().from(orders);
const active = all.filter(o => o.status === 'active');

// ❌ Loop sync trong transaction
await db.transaction(async (tx) => {
  for (const item of items) {
    await tx.insert(orderItems).values(item);  // Sequential
  }
});

// ✅ Bulk insert
await db.transaction(async (tx) => {
  await tx.insert(orderItems).values(items);  // 1 query
});

// ❌ Aggregate không index
const total = await db.execute(sql`SELECT SUM(total) FROM orders WHERE tenant_id = ${tenantId}`);
// Nếu không có index trên tenant_id → seq scan toàn bảng

// ❌ JSON column không index
await db.select().from(orders).where(sql`metadata->>'priority' = 'high'`);
// Cần: CREATE INDEX orders_metadata_priority_idx ON orders ((metadata->>'priority'));
```

---

**END Rule 09**
