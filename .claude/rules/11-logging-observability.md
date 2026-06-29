# Rule 11 — Logging & Observability

> Mức độ: 🟠 HIGH | Áp dụng: Mọi log statement, mọi service

---

## 1. Structured logging (Pino)

```typescript
// shared/logger.ts
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  
  base: {
    env: process.env.NODE_ENV,
    service: 'erp-api',
    version: process.env.APP_VERSION,
  },
  
  timestamp: pino.stdTimeFunctions.isoTime,
  
  formatters: {
    level: (label) => ({ level: label }),
  },
  
  redact: {
    paths: [
      'password', '*.password', '*.password_hash',
      'token', 'refreshToken', 'accessToken', '*.token',
      'authorization', 'req.headers.authorization', 'req.headers.cookie',
      'apiKey', 'api_key', 'secret', '*.secret',
      'creditCard', 'cardNumber', 'cvv',
      'ssn', 'national_id', 'cccd',
      'email',
    ],
    censor: '[REDACTED]',
  },
  
  serializers: {
    req: (req) => ({
      method: req.method,
      url: req.url,
      requestId: req.id,
      tenantId: req.user?.tenantId,
      userId: req.user?.id,
    }),
    res: (res) => ({
      statusCode: res.statusCode,
    }),
    err: pino.stdSerializers.err,
  },
});
```

---

## 2. Log levels — khi nào dùng gì

| Level | Khi nào |
|---|---|
| `fatal` | Service không thể tiếp tục, sắp crash |
| `error` | Operation fail, cần investigate |
| `warn` | Bất thường nhưng vẫn xử lý được |
| `info` | Sự kiện quan trọng (start, deploy, login) |
| `debug` | Chi tiết debug, chỉ bật ở dev/staging |
| `trace` | Cực chi tiết, hiếm dùng |

```typescript
// ✅ ĐÚNG
logger.error({ orderId, err }, 'Failed to charge payment');
logger.warn({ ip, attempt }, 'Failed login attempt');
logger.info({ orderId, total }, 'Order created');
logger.debug({ query, params }, 'Executing query');

// ❌ SAI — không có context
logger.error('something went wrong');

// ❌ SAI — log object lớn
logger.info({ entireUserObject }, 'User updated');  // Có thể chứa PII
```

---

## 3. Bắt buộc log

| Sự kiện | Detail |
|---|---|
| Service start/stop | version, env, config (không secret) |
| HTTP request (mọi request) | method, url, status, duration, requestId |
| Slow query (> 100ms) | query (param redacted), duration |
| External API call | url, method, status, duration |
| Error (mọi exception) | err, stack, request context |
| Auth event | login/logout/2FA (qua audit log) |
| Tài chính event | invoice/payment/refund (qua audit log) |
| Job execution | jobId, type, status, duration |
| AI agent action | agentId, action, tokens, cost, confidence |

---

## 4. CẤM log

- Password (plain hay hash)
- Token (full JWT, refresh token)
- API key bên thứ ba
- CCCD, số thẻ tín dụng, CVV
- Cookie value
- Email (chỉ partial: a***@b.com)
- Request body có PII
- File content user upload

---

## 5. Distributed tracing (OpenTelemetry)

```typescript
// shared/tracing.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'erp-api',
    [SemanticResourceAttributes.SERVICE_VERSION]: process.env.APP_VERSION,
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV,
  }),
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT,
  }),
  instrumentations: [
    /* @opentelemetry/instrumentation-* */
  ],
});

sdk.start();
```

**Span naming:** `<verb>.<resource>` (e.g., `create.order`, `query.products`).

```typescript
import { trace } from '@opentelemetry/api';

const tracer = trace.getTracer('orders-service');

async function createOrder(dto) {
  return tracer.startActiveSpan('create.order', async (span) => {
    try {
      span.setAttribute('order.customer_id', dto.customerId);
      span.setAttribute('order.item_count', dto.items.length);
      
      const order = await this.repo.create(dto);
      span.setAttribute('order.id', order.id);
      span.setStatus({ code: SpanStatusCode.OK });
      return order;
    } catch (err) {
      span.recordException(err);
      span.setStatus({ code: SpanStatusCode.ERROR });
      throw err;
    } finally {
      span.end();
    }
  });
}
```

---

## 6. Metrics (Prometheus)

**Naming convention:** `wecha_<resource>_<metric>_<unit>`

```typescript
import { Counter, Histogram, Gauge } from 'prom-client';

export const ordersCreatedTotal = new Counter({
  name: 'wecha_orders_created_total',
  help: 'Total orders created',
  labelNames: ['tenant_id', 'channel', 'status'],
});

export const httpRequestDuration = new Histogram({
  name: 'wecha_http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.01, 0.05, 0.1, 0.3, 0.5, 1, 3, 5, 10],
});

export const dbConnectionsActive = new Gauge({
  name: 'wecha_db_connections_active',
  help: 'Active database connections',
});

export const aiAgentCostUsd = new Counter({
  name: 'wecha_ai_agent_cost_usd_total',
  help: 'AI agent cost in USD',
  labelNames: ['agent_id', 'model'],
});
```

**Endpoint /metrics:**
```typescript
@Get('/metrics')
async metrics(@Res() res: Response) {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
}
```

---

## 7. SLI / SLO / Error budget

**Service Level Indicators (SLI):**
- Availability: `(2xx + 3xx) / total_requests`
- Latency: `P95 latency < threshold`
- Error rate: `5xx / total_requests`

**Service Level Objectives (SLO):**

| Service | Availability | Latency P95 | Error rate |
|---|---|---|---|
| API (read) | 99.9% | < 300ms | < 0.1% |
| API (write) | 99.5% | < 1000ms | < 0.5% |
| Auth | 99.95% | < 500ms | < 0.05% |
| Web app | 99.9% | LCP < 2.5s | - |

**Error budget:** 99.9% availability/month = 43.2 phút downtime/tháng. Vượt → freeze deploy mới, focus stability.

---

## 8. Alerting rules

```yaml
# prometheus/alerts.yml
groups:
- name: api
  rules:
  - alert: HighErrorRate
    expr: rate(wecha_http_request_duration_seconds_count{status_code=~"5.."}[5m]) > 0.05
    for: 5m
    labels: { severity: critical }
    annotations:
      summary: "API error rate > 5%"
      runbook: "https://docs.wecha.vn/runbooks/high-error-rate"
  
  - alert: HighLatency
    expr: histogram_quantile(0.95, rate(wecha_http_request_duration_seconds_bucket[5m])) > 1
    for: 10m
    labels: { severity: warning }
    annotations:
      summary: "P95 latency > 1s"
  
  - alert: DatabaseConnectionsHigh
    expr: wecha_db_connections_active > 18
    for: 5m
    labels: { severity: warning }
  
  - alert: DiskSpaceLow
    expr: node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes < 0.1
    for: 10m
    labels: { severity: critical }
```

---

## 9. Log aggregation flow

```
NestJS pino → stdout
  ↓
Docker logging driver (json-file)
  ↓
Promtail (sidecar container)
  ↓
Loki (centralized log storage)
  ↓
Grafana (query + dashboard)
```

---

## 10. Audit log table schema

```sql
CREATE TABLE audit_logs (
  id BIGSERIAL PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  actor_id TEXT,
  actor_type TEXT NOT NULL,  -- 'user', 'system', 'ai_agent'
  actor_ip INET,
  actor_user_agent TEXT,
  action TEXT NOT NULL,       -- 'order.created', 'role.changed'
  resource_type TEXT NOT NULL, -- 'Order', 'User'
  resource_id TEXT NOT NULL,
  before_state JSONB,
  after_state JSONB,
  context JSONB,              -- Extra: reason, ticket_id, ...
  request_id TEXT,
  trace_id TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX audit_logs_tenant_created_idx ON audit_logs (tenant_id, created_at DESC);
CREATE INDEX audit_logs_actor_idx ON audit_logs (tenant_id, actor_id, created_at DESC);
CREATE INDEX audit_logs_resource_idx ON audit_logs (tenant_id, resource_type, resource_id);
CREATE INDEX audit_logs_action_idx ON audit_logs (tenant_id, action, created_at DESC);

-- Append-only: revoke UPDATE/DELETE
REVOKE UPDATE, DELETE ON audit_logs FROM PUBLIC;
```

---

**END Rule 11**
