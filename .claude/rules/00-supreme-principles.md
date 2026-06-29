# Rule 00 — Nguyên tắc tối thượng & Stack chính thức

> Mức độ: 🔴 CRITICAL | Áp dụng: Mọi task | Owner: Lê Hữu Trí

---

## 1. 15 Nguyên tắc bất khả xâm phạm

### N1 — Bảo mật trước, tính năng sau 🔴
Không đánh đổi bảo mật để code nhanh. Thà refuse build feature còn hơn build với SQL injection / XSS / CSRF.

### N2 — Không đoán, không bịa 🔴
Không biết → "tôi không biết" + HỎI user. Cấm bịa: tên thư viện, version, API method, file path, class name, % completion. Trước khi import → check `package.json`. Trước khi nói "X đã có" → grep + đọc file.

### N3 — Code cho người đọc 🟠
Code đọc nhiều hơn viết 10 lần. Tên rõ nghĩa, function ngắn, comment giải thích "TẠI SAO".

### N4 — Đa nền tảng từ ngày đầu 🟠
Mọi API phải dùng được cho ≥ 3 client: web Next.js, mobile React Native, n8n/external. Không phụ thuộc cookie session, không phụ thuộc User-Agent.

### N5 — Multi-tenant từ schema 🔴
Mọi bảng nghiệp vụ có `tenant_id` notNull + FK + index + RLS bật. App filter là defense thứ 2, RLS là defense cuối.

### N6 — Audit mọi thay đổi nhạy cảm 🔴
Tài chính, phân quyền, PII, AI action → ghi `audit_logs` immutable (no UPDATE, no DELETE). Retention 10 năm cho tài chính (TT 99/2025), 7 năm cho phân quyền, 2 năm cho AI.

### N7 — Không có "magic" 🟠
Cấm decorator làm 5 việc khác nhau. Cấm middleware sửa body không log. Cấm auto-retry không có limit/backoff/log.

### N8 — Tự động hoá việc lặp lại 🟢
Task > 2 lần/ngày → workflow n8n hoặc AI agent. Mục tiêu: AI vận hành 80% công việc lặp lại.

### N9 — Mọi automation lưu trace vào PostgreSQL ERP 🔴
Không tin n8n internal log. Mọi workflow execution, AI decision, event → lưu `workflow_executions`, `agent_executions`, `event_log` trong Postgres ERP.

### N10 — AI agent có scope giới hạn 🔴
AI KHÔNG BAO GIỜ là superuser. Mặc định READ-only. Action tài chính > 500k VND → human approval. Confidence < 0.85 → escalate `human_review_tasks`.

### N11 — Idempotent by default 🟠
Mọi POST/PUT/PATCH hỗ trợ `Idempotency-Key` header. Mọi job nền safe khi chạy lại.

### N12 — Defense in depth 🟠
Mọi rule bảo mật áp dụng ≥ 2 lớp. VD multi-tenant: Guard (lớp 1) + Service filter (lớp 2) + Drizzle middleware (lớp 3) + RLS (lớp 4).

### N13 — Fail fast, fail loud 🟠
Có lỗi → crash sớm + log đầy đủ + alert. CẤM nuốt lỗi (`try {} catch {}`). CẤM silent degradation.

### N14 — Reversible by default 🟡
Mọi action quan trọng (tạo đơn, charge tiền, sa thải) phải rollback hoặc compensate được. Soft delete > hard delete. Saga pattern cho multi-step.

### N15 — Trước khi xây mới, đọc file đã có 🔴
Bắt buộc trước khi viết code:
1. Đọc CLAUDE.md
2. Đọc rule liên quan
3. Đọc skill liên quan
4. Grep code tương tự (`grep -r "<keyword>" apps/`)
5. Đọc README module sẽ động chạm
6. Check `package.json` xem dependencies thực sự có gì

---

## 2. Bounded Contexts (86 modules → 11 contexts)

```
apps/api/src/modules/
├── _core/              auth, users, roles, tenants, audit-log, health
├── _platform/          notifications, media, search, monitoring, databases
├── commerce/           products, orders, pos, customers, payments, shipments...
├── inventory/          warehouses, stock-audits, purchase-orders, suppliers...
├── manufacturing/      manufacturing, mrp, sops, rnd-formulas, haccp, cmms...
├── hr/                 hr-employees, hr-attendance, hr-kpi, hr-payroll...
├── ecommerce/          platform-shopee, platform-lazada, platform-tiki
├── marketing/          campaigns, care-reminders, ads-tracking
├── ai/                 chatbot, knowledge, n8n-webhooks, automation-events
├── lms/                lms, courses, drinks, assets
└── ops/                excel-import, excel-export, wiki, branches
```

**Quy tắc bounded context** 🟠 HIGH:
1. Module **cùng** context import nhau qua DI tự do
2. Module **khác** context CHỈ giao tiếp qua **events** hoặc **public service interface**
3. CẤM `import` xuyên context trực tiếp:
   ```typescript
   // ❌ CẤM
   import { HrEmployeesService } from '../../hr/hr-employees/services/hr-employees.service';
   
   // ✅ ĐÚNG
   import { EmployeeQueryPort } from '@wecha/contracts/hr';
   @Inject('EMPLOYEE_QUERY_PORT') private employeeQuery: EmployeeQueryPort;
   ```

---

## 3. Stack chính thức (PINNED — đổi cần ADR)

### Frontend
| Hạng mục | Tech | Version | Status |
|---|---|---|---|
| Framework | Next.js | 14.2.x | ✅ |
| Language | TypeScript strict | 5.7.x | ✅ |
| UI | React | 18.3.x | ✅ |
| Styling | Tailwind CSS | 3.4.x | ✅ |
| Component pattern | CVA → shadcn/ui | - | ⚠️ Migrating |
| Icon | lucide-react | 0.460.x | ✅ |
| Chart | Recharts | 2.15.x | ✅ |
| State server | TanStack Query v5 | latest | ❌ Wave 4 |
| Form | react-hook-form + Zod | latest | ❌ Wave 4 |
| Date | date-fns | 3.x | ✅ |
| i18n | next-intl | latest | ❌ Wave 5 |

### Backend
| Hạng mục | Tech | Version | Status |
|---|---|---|---|
| Framework | NestJS | 10.4.x | ✅ |
| Runtime | Node.js LTS | 20-alpine | ✅ |
| API style | REST + OpenAPI 3.1 | - | ✅ |
| Validation | class-validator → Zod + nestjs-zod | - | ⚠️ Migrating |
| ORM | **Drizzle ORM** | **0.38.x** | ✅ **Chốt 26/4/2026** |
| DB driver | node-postgres (pg) | 8.13.x | ✅ |
| Auth | Passport JWT | 0.7.x | ✅ |
| Password hash | bcrypt → **argon2id** | - | ⚠️ Migrating Wave 2 |
| 2FA | otplib (TOTP) | latest | ❌ Wave 2 |
| RBAC | Custom Guard → CASL 6 | - | ⚠️ Migrating Wave 4 |
| Queue | pg-boss | 10.1.x | ✅ |
| Cache | Redis | 7-alpine | ✅ |
| Logger | NestJS default → Pino | - | ⚠️ Wave 3 |
| Rate limit | @nestjs/throttler | latest | ❌ Wave 1.2 |
| Security headers | helmet | latest | ❌ Wave 1.2 |
| Event bus | @nestjs/event-emitter → Outbox + BullMQ | - | ⚠️ Wave 4 |
| API docs | @nestjs/swagger | 7.4.x | ✅ |
| Testing | Jest 30 + Supertest | - | ✅ |
| E2E | Playwright | latest | ❌ Wave 3 |

### Database & Storage
| Hạng mục | Tech | Status |
|---|---|---|
| Primary DB | PostgreSQL 16 | ✅ |
| Multi-tenant | Row-Level Security | ❌ Wave 2 |
| Vector | pgvector 0.7 (768-dim Gemini) | ✅ |
| FTS | PostgreSQL tsvector + GIN | ✅ |
| Cache | Redis 7 | ✅ |
| Queue backend | PostgreSQL (qua pg-boss) | ✅ |
| Object storage | Local FS → Cloudflare R2 | ⚠️ Wave 3 |
| Backup | Manual → cron 6h + R2 weekly + monthly cold | ⚠️ Wave 1.2 |

### DevOps & Observability
| Hạng mục | Tech | Status |
|---|---|---|
| Container | Docker + Compose | ✅ |
| Reverse proxy | Caddy 2 (auto Let's Encrypt) | ✅ |
| Build | pnpm 9.15 + Turborepo 2.4 | ✅ |
| CI/CD | GitHub Actions | ❌ Wave 3 |
| Monitoring errors | Sentry | ❌ Wave 3 |
| Monitoring logs | Grafana + Loki | ❌ Wave 3 |
| Monitoring metrics | Prometheus + Grafana | ❌ Wave 3 |
| APM | OpenTelemetry → Tempo | ❌ Wave 4 |
| Health check | NestJS HealthModule (`/api/health`) | ✅ |

### Automation & AI
| Hạng mục | Tech | Status |
|---|---|---|
| Workflow engine | n8n self-hosted (queue mode) | ✅ 9 workflows |
| Workflow tracking | `workflow_executions` table | ❌ Wave 4 |
| Event store | `event_log` (append-only) | ❌ Wave 4 |
| AI Orchestrator | Custom NestJS module | ❌ Wave 4 |
| AI Workers | Claude Opus/Sonnet/Haiku | 📅 Wave 4 |
| Local LLM | Ollama (Qwen/Llama) | 📅 Wave 5 |
| Vision AI | Gemini 2.5 Flash | ✅ |
| Embedding | Gemini text-embedding-004 (768-dim) | ✅ |
| Vector DB | pgvector | ✅ |
| Tool calling | Anthropic SDK function calling | 📅 Wave 4 |
| Human-in-the-loop | `human_review_tasks` + Zalo OA + Telegram | ❌ Wave 4 |
| Cost tracking | Per-workflow + per-agent (token + USD) | ❌ Wave 4 |

### Integrations
| Loại | Provider | Status |
|---|---|---|
| Payment | VNPay + MoMo | ✅ |
| Shipping | GHN, GHTK, J&T, Viettel Post | ✅ |
| E-commerce | Shopee, Lazada, Tiki | ✅ |
| E-invoice | MISA meInvoice (NĐ 70/2025 + TT 32/2025) | ✅ |
| Accounting | MISA AMIS (TT 99/2025/TT-BTC) | ✅ |
| Customer comm | Zalo OA | ✅ |
| AI LLM | Anthropic Claude API | 📅 Wave 4 |
| AI Vision | Google Gemini API | ✅ |
| Object Storage | Cloudflare R2 (S3-compat) | 📅 Wave 3 |

---

## 4. Layered Architecture (Clean lite)

```
┌────────────────────────────────────────────────────┐
│  PRESENTATION    Controllers, Pages, n8n webhooks  │
├────────────────────────────────────────────────────┤
│  APPLICATION     Services, Server Actions          │
├────────────────────────────────────────────────────┤
│  DOMAIN          Entities, VO, Events, Errors      │
├────────────────────────────────────────────────────┤
│  INFRASTRUCTURE  Repositories (Drizzle), External  │
└────────────────────────────────────────────────────┘
```

**Dependency rule** 🔴 CRITICAL — Outer biết inner, KHÔNG ngược:
- Domain KHÔNG import NestJS, Drizzle, Express
- Service KHÔNG import HTTP types (Request, Response)
- Repository CHỈ data access, KHÔNG business logic

```typescript
// ❌ Domain entity import Drizzle
class Order {
  async save() { await db.insert(orders).values(this); }
}

// ✅ Domain pure, Repository handle persistence
class Order { /* pure data + behavior */ }
class OrderRepository {
  async save(order: Order) { await db.insert(orders).values(order.toRow()); }
}
```

---

## 5. Cấu trúc 1 NestJS module chuẩn

```
modules/orders/
├── orders.module.ts
├── README.md                    # BẮT BUỘC
├── controllers/
│   ├── orders.controller.ts
│   └── orders.controller.spec.ts
├── services/
│   ├── orders.service.ts
│   └── orders.service.spec.ts
├── repositories/
│   └── orders.repository.ts
├── domain/
│   ├── entities/order.entity.ts
│   ├── value-objects/money.vo.ts
│   ├── events/order-created.event.ts
│   └── errors/insufficient-stock.error.ts
├── dto/
│   ├── create-order.dto.ts
│   └── order-response.dto.ts
├── automation/                  # BẮT BUỘC
│   ├── events.ts                # Events module emit
│   ├── handlers.ts              # Events listen
│   └── ai-tools.ts              # Tools cho AI agent gọi
├── guards/
└── decorators/
```

**README.md template** (BẮT BUỘC mọi module):
```markdown
# <Module Name>
## Mục đích
## Owner (@github-handle)
## Use cases (1, 2, 3...)
## Dependencies (modules khác)
## Events emit
## Events subscribe
## AI agent tools available
## AI agent CẤM
```

---

## 6. Compliance chuẩn quốc tế áp dụng

| Chuẩn | Phiên bản | Áp dụng cho |
|---|---|---|
| OWASP Top 10 | 2021 | Bảo mật web (rule 01) |
| OWASP API Top 10 | 2023 | API security (rule 01) |
| OWASP ASVS | 4.0.3 L2 | Verify checklist |
| CIS Benchmark | PostgreSQL 16, Docker, Node | Hardening |
| NIST SP 800-53 | Rev 5 | Access control, audit |
| NIST SP 800-63B | Rev 3 | Auth & MFA |
| PCI DSS | v4.0 | Payment (qua VNPay/MoMo) |
| ISO/IEC 27001 | 2022 | ISMS |
| SOC 2 Type II | Latest | Audit trail |
| GDPR | 2016/679 | Privacy (khách quốc tế) |
| **NĐ 13/2023/NĐ-CP** | **VN** | **Bảo vệ dữ liệu cá nhân** |
| **TT 99/2025/TT-BTC** | **VN, từ 01/01/2026** | **Chế độ kế toán doanh nghiệp** |
| **NĐ 70/2025/NĐ-CP** | **VN** | **Hoá đơn điện tử** |
| **TT 32/2025/TT-BTC** | **VN** | **Hướng dẫn HĐĐT từ máy tính tiền** |
| Conventional Commits | 1.0.0 | Git commit format |
| Semantic Versioning | 2.0.0 | Package + API version |
| RFC 7807 | - | HTTP error format |
| RFC 7519 + 8725 | - | JWT + JWT BCP |
| RFC 6238 | - | TOTP 2FA |
| OpenAPI | 3.1 | API docs |
| WCAG | 2.1 AA | Accessibility |

Chi tiết tham chiếu trong từng rule cụ thể.

---

## 7. Câu hỏi trước khi viết code

Claude PHẢI tự trả lời được những câu sau trước khi gõ phím:

- [ ] Đã đọc CLAUDE.md chưa?
- [ ] Đã đọc rule(s) liên quan chưa?
- [ ] Đã đọc skill(s) liên quan chưa?
- [ ] Đã grep code tương tự trong repo chưa?
- [ ] Đã check `package.json` xem dependencies thực sự có gì chưa?
- [ ] Đã đọc README của module sẽ động chạm chưa?
- [ ] Tôi có chắc chắn về tên class/file/import không, hay đang đoán?
- [ ] Code này có vi phạm rule 🔴 nào không?
- [ ] Có cần multi-tenant filter không?
- [ ] Có cần audit log không?
- [ ] Có cần test không?
- [ ] Có break backward compat của API không?

Nếu trả lời "không chắc" cho bất kỳ câu nào — **HỎI USER, KHÔNG ĐOÁN**.

---

**END Rule 00**
