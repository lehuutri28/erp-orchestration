# Rule 12 — DevOps & CI/CD

> Mức độ: 🟠 HIGH | Áp dụng: Deploy, infrastructure, CI/CD config

---

## 1. Environments

| Env | URL | Purpose | Branch | Auto-deploy |
|---|---|---|---|---|
| `local` | localhost | Dev local | feature/* | - |
| `dev` | dev.wecha.vn | Tích hợp test | develop | ✅ on push |
| `staging` | staging.wecha.vn | UAT, dryrun | release/* | ✅ on push |
| `production` | erp.wecha.vn | Live | main + tag | ✅ on tag (manual approval) |

**Mỗi env phải có:**
- DB riêng (KHÔNG share)
- Secret riêng (Doppler/Infisical project khác nhau)
- Domain + SSL cert riêng
- Monitoring riêng (Sentry project)

---

## 2. CI Pipeline (GitHub Actions)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  lint-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env: { POSTGRES_PASSWORD: test }
        options: >-
          --health-cmd pg_isready --health-interval 10s
        ports: ['5432:5432']
      redis:
        image: redis:7-alpine
        ports: ['6379:6379']
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      
      - uses: pnpm/action-setup@v3
        with: { version: 9.15 }
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'
      
      - name: Install
        run: pnpm install --frozen-lockfile
      
      - name: Lint
        run: pnpm lint
      
      - name: Type check
        run: pnpm typecheck
      
      - name: Unit tests
        run: pnpm test --coverage
      
      - name: Integration tests
        run: pnpm test:integration
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/test
          REDIS_URL: redis://localhost:6379
      
      - name: Upload coverage
        uses: codecov/codecov-action@v4
      
      - name: Bundle size check
        run: pnpm size-limit

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Dependency audit
        run: pnpm audit --audit-level=high
      
      - name: Snyk scan
        uses: snyk/actions/node@master
        with: { args: --severity-threshold=high }
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      
      - name: Gitleaks
        uses: gitleaks/gitleaks-action@v2
      
      - name: Trivy container scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/wecha/erp-api:${{ github.sha }}
          severity: CRITICAL,HIGH
          exit-code: 1
      
      - name: License check
        run: pnpm dlx license-checker --failOn 'GPL-3.0;AGPL-3.0'

  build:
    needs: [lint-test, security]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build & push API
        uses: docker/build-push-action@v5
        with:
          context: .
          file: apps/api/Dockerfile
          push: true
          tags: |
            ghcr.io/wecha/erp-api:${{ github.sha }}
            ghcr.io/wecha/erp-api:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
          provenance: true
          sbom: true
```

---

## 3. CD Pipeline

```yaml
# .github/workflows/deploy-production.yml
name: Deploy Production

on:
  push:
    tags: ['v*']
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # Manual approval
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup SSH
        uses: webfactory/ssh-agent@v0.9.0
        with: { ssh-private-key: ${{ secrets.DEPLOY_SSH_KEY }} }
      
      - name: Database backup before deploy
        run: |
          ssh deploy@erp.wecha.vn "
            pg_dump -Fc -d erp > /opt/backup/pre-deploy-$(date +%Y%m%d-%H%M%S).dump
          "
      
      - name: Run migrations (with safety check)
        run: |
          ssh deploy@erp.wecha.vn "
            cd /opt/erp
            docker compose run --rm api pnpm db:migrate:status
            docker compose run --rm api pnpm db:migrate:check  # Detect breaking
            docker compose run --rm api pnpm db:migrate
          "
      
      - name: Blue-green deploy
        run: |
          ssh deploy@erp.wecha.vn "
            cd /opt/erp
            docker compose pull
            docker compose up -d --no-deps --scale api=4 api
            sleep 30
            for i in {1..10}; do
              if curl -f http://localhost:4000/api/health/ready; then
                echo 'Health OK'
                break
              fi
              sleep 5
            done
            docker compose up -d --no-deps --scale api=2 api
          "
      
      - name: Smoke test
        run: |
          curl -f https://erp.wecha.vn/api/health
          curl -f https://erp.wecha.vn/api/version | grep ${{ github.sha }}
      
      - name: Notify Telegram
        if: always()
        run: |
          curl -X POST "https://api.telegram.org/bot${{ secrets.TG_BOT }}/sendMessage" \
            -d chat_id=${{ secrets.TG_CHAT }} \
            -d text="🚀 Deploy ${{ github.ref_name }} ${{ job.status == 'success' && '✅' || '❌' }}"
      
      - name: Rollback on failure
        if: failure()
        run: |
          ssh deploy@erp.wecha.vn "
            cd /opt/erp
            docker compose down
            docker tag ghcr.io/wecha/erp-api:previous ghcr.io/wecha/erp-api:latest
            docker compose up -d
          "
```

---

## 4. Dockerfile (multi-stage, secure, small)

```dockerfile
# apps/api/Dockerfile
FROM node:20-alpine@sha256:<digest> AS base
RUN corepack enable && corepack prepare pnpm@9.15.0 --activate
WORKDIR /app

FROM base AS deps
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY apps/api/package.json apps/api/
COPY packages/*/package.json packages/*/
RUN pnpm install --frozen-lockfile --prod=false

FROM deps AS build
COPY . .
RUN pnpm --filter @wecha/api build

FROM base AS runtime
ENV NODE_ENV=production
RUN addgroup -g 1001 nodejs && adduser -u 1001 -G nodejs -s /bin/sh -D nodejs
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build --chown=nodejs:nodejs /app/apps/api/dist ./dist
COPY --from=build --chown=nodejs:nodejs /app/apps/api/package.json ./

USER nodejs
EXPOSE 4000

HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost:4000/api/health || exit 1

CMD ["node", "dist/main.js"]
```

**Image size target:** < 200MB

---

## 5. Secrets management 🔴

**🔴 CẤM TUYỆT ĐỐI:**
- Commit `.env` lên git
- Hardcode secret trong code
- Truyền secret qua command-line args (visible trong `ps`)
- Log secret (dù chỉ debug)

**Tools — chọn 1:**
- **Doppler** — SaaS, dễ dùng nhất
- **Infisical** — Open-source, self-host được
- **HashiCorp Vault** — Enterprise

**Pattern:**
```typescript
// shared/config/config.module.ts
import { z } from 'zod';

const configSchema = z.object({
  NODE_ENV: z.enum(['development', 'staging', 'production']),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  JWT_REFRESH_SECRET: z.string().min(32),
  VNPAY_SECRET: z.string().min(16),
  MOMO_SECRET: z.string().min(16),
  ANTHROPIC_API_KEY: z.string().startsWith('sk-ant-'),
  // ...
});

export const config = configSchema.parse(process.env);
// → Crash early nếu thiếu/sai env
```

---

## 6. Database migration safety

```typescript
// scripts/migrate-check.ts — chạy trước migrate
const breakingChanges = [
  'DROP COLUMN', 'DROP TABLE', 'ALTER COLUMN ... TYPE',
  'RENAME COLUMN', 'NOT NULL' /* without default */,
];

const sqlContent = readFileSync(migrationFile, 'utf8');
for (const pattern of breakingChanges) {
  if (sqlContent.toUpperCase().includes(pattern)) {
    throw new Error(`Breaking change detected: ${pattern}. Require manual approval.`);
  }
}
```

**Pre-deploy checklist:**

> 📎 **Cross-ref:** Pre-deploy migration safety checklist chi tiết (expand-contract, down.sql, staging test) xem [rules/03-database-drizzle.md](./03-database-drizzle.md#34-migration-safety)

1. Backup DB
2. Test migration on staging với production-like data
3. Có rollback script (`.down.sql`)
4. Documented downtime estimate
5. Alert team trước khi chạy

---

## 7. Monitoring stack (Wave 3)

```
Application
    ↓
Prometheus (metrics)  →  Grafana (dashboards) →  Alertmanager → Telegram/PagerDuty
    ↑
Loki (logs)           →  Grafana
    ↑
Tempo (traces)        →  Grafana
    ↑
Sentry (errors)       →  Webhook → Telegram
```

**Dashboards bắt buộc:**
1. **API Overview** — Req/s, latency P50/P95/P99, error rate per endpoint
2. **Database** — Connection pool, slow queries, replication lag
3. **Business** — Orders/min, revenue/hour, active users
4. **Infrastructure** — CPU, memory, disk, network
5. **AI Agents** — Token usage, cost, success rate
6. **Multi-tenant** — Top tenants by usage, anomalies

---

## 8. Disaster Recovery

**RTO (Recovery Time Objective):** 1 giờ  
**RPO (Recovery Point Objective):** 6 giờ

**Backup tier:**
- Tier 1 — Hot (RPO=6h, RTO=1h): pg_dump 6h → /opt/backup VPS, giữ 7 ngày
- Tier 2 — Warm (RPO=24h, RTO=4h): Daily → Cloudflare R2 (AES-256), giữ 30 ngày
- Tier 3 — Cold (RPO=7d, RTO=24h): Weekly → R2 Cold, giữ 12 tháng
- Tier 4 — Archive (TT 99/2025): Monthly → R2 Archive, giữ 10 năm

**Test restore HÀNG THÁNG** — quy trình ở Rule 03.

---

**END Rule 12**
