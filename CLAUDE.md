# CLAUDE.md — erp.wecha.vn

> **Hệ thống ERP cấp doanh nghiệp** cho WECHA (chuỗi trà sữa nhượng quyền) + CÔNG TY TNHH VUA AN TOÀN (sản xuất nguyên liệu, máy móc pha chế, đào tạo F&B).
>
> File này là **MỤC LỤC + RÀNG BUỘC TỐI THƯỢNG**. Chi tiết kỹ thuật nằm trong `.claude/rules/` và `.claude/skills/`.
>
> **Phiên bản:** 3.0 | **Cập nhật:** 2026-04-26 | **Owner:** Lê Hữu Trí | **Status:** PRODUCTION-BINDING

---

## ⚖️ NGUYÊN TẮC NỀN TẢNG

Mọi đoạn code Claude viết PHẢI tuân thủ. Nếu có mâu thuẫn giữa rule cụ thể và 5 nguyên tắc này — nguyên tắc thắng.

1. **🔴 BẢO MẬT TRƯỚC, TÍNH NĂNG SAU** — Không đánh đổi bảo mật để code nhanh hơn.
2. **🔴 KHÔNG ĐOÁN, KHÔNG BỊA** — Không biết → hỏi user, KHÔNG bịa tên file/function/version.
3. **🔴 MULTI-TENANT TỪ SCHEMA** — Mọi bảng nghiệp vụ có `tenant_id` + RLS bật.
4. **🔴 AUDIT MỌI THAY ĐỔI NHẠY CẢM** — Tài chính, phân quyền, PII, AI action — tất cả ghi audit log immutable.
5. **🔴 ĐỌC FILE TRƯỚC KHI VIẾT** — Trước khi viết bất kỳ code nào: đọc CLAUDE.md → đọc rule liên quan → grep code tương tự → check package.json.

> Chi tiết 15 nguyên tắc đầy đủ: [`.claude/rules/00-supreme-principles.md`](.claude/rules/00-supreme-principles.md)

---

## 🏢 CONTEXT DOANH NGHIỆP

**Pháp nhân:** CÔNG TY TNHH VUA AN TOÀN (MST 0313334177, Bà Điểm, TP.HCM)

**Lĩnh vực hoạt động (đa ngành):**
1. **Sản xuất nguyên liệu F&B** — bột pha trà sữa, syrup, topping (xem `skills/manufacturing-fnb/`)
2. **Bán + bảo trì máy móc pha chế** — máy pha cà phê, máy lắc trà sữa, có CMMS theo dõi (xem `skills/manufacturing-fnb/`)
3. **Đào tạo F&B + LMS mở quán** — khoá public có thanh toán + chứng chỉ điện tử (xem `skills/lms-courses/`)
4. **Nhượng quyền thương hiệu WECHA** — chuỗi trà sữa, ERP quản lý đa chi nhánh (xem `skills/franchise-wecha/`)
5. **POS + bán lẻ** — POS cho quán, đơn online qua Shopee/Lazada/Tiki (xem `skills/pos-fnb/`)

**Bounded contexts kỹ thuật:** 86 modules NestJS được tổ chức thành 11 contexts (`_core, _platform, commerce, inventory, manufacturing, hr, ecommerce, marketing, ai, lms, ops`). Chi tiết: [`.claude/rules/00-supreme-principles.md#bounded-contexts`](.claude/rules/00-supreme-principles.md).

---

## 💻 TECH STACK (PINNED)

> Stack đã chốt. Mọi đề xuất thay đổi PHẢI có ADR + Letri duyệt.

| Layer | Tech | Trạng thái |
|---|---|---|
| Frontend | Next.js 14.2 + TypeScript 5.7 strict + Tailwind 3.4 + React 18 | ✅ Production |
| Backend | NestJS 10.4 + TypeScript 5.7 strict + Node 20 LTS | ✅ Production |
| ORM | **Drizzle 0.38** (chốt 26/4/2026, KHÔNG migrate Prisma) | ✅ Production |
| Database | PostgreSQL 16 + pgvector 0.7 + Row-Level Security | ⚠️ RLS chưa bật |
| Cache/Queue | Redis 7 + pg-boss 10.1 | ✅ Production |
| Auth | JWT (passport) + 2FA TOTP (Wave 2) | ⚠️ 2FA chưa có |
| Storage | Local filesystem → Cloudflare R2 (Wave 3) | ⚠️ Migrating |
| Reverse proxy | Caddy 2 (auto Let's Encrypt) | ✅ Production |
| CDN/WAF | Cloudflare | ✅ Production |
| Container | Docker + Compose | ✅ Production |
| Build | pnpm 9.15 + Turborepo 2.4 | ✅ Production |
| Testing | Jest 30 + Supertest + Playwright (E2E mới) | ⚠️ Coverage thấp |
| CI/CD | GitHub Actions (Wave 3) | ❌ Chưa có |
| Monitoring | Sentry + Grafana + Loki (Wave 3) | ❌ Chưa có |
| Workflow engine | n8n self-hosted (9 workflows) | ✅ Production |
| AI LLM | Anthropic Claude API (Opus/Sonnet/Haiku) | 📅 Wave 4 |
| AI Vision | Google Gemini 2.5 Flash | ✅ Production |

**Stack chi tiết + version chính xác + plan migrate:** [`.claude/rules/00-supreme-principles.md#stack`](.claude/rules/00-supreme-principles.md)

---

## 🔚 CUỐI MỖI SESSION — BẮT BUỘC BÀN GIAO

> ⭐ **Từ 30/5/2026: Trước khi dừng, MỌI session PHẢI cập nhật mục "BÀN GIAO CHO SESSION SAU" trong `docs/PROGRESS.md`** (hoặc TASK-LOG của C nếu là CEO con).
>
> Ghi ĐỦ 5 dòng, CỤ THỂ tới **file:dòng** + **bước tiếp theo chính xác làm gì**:
> - **Vừa làm:** [việc]
> - **Dừng tại:** [file:dòng cụ thể]
> - **Bước tiếp theo CHÍNH XÁC:** [làm gì trước]
> - **Cạm bẫy cần nhớ:** [vd: nhớ apply migration trước khi test]
> - **Hệ khác cần biết:** [bàn giao chéo — vd: orders giờ có thêm trạng thái X]
>
> 🔴 **CẤM bàn giao mơ hồ** kiểu "đã xong phần khách hàng, tiếp theo làm đơn hàng" — vô dụng. Phải tới file:dòng + bước chính xác.
> 🔴 **Không bàn giao = session sau làm sai.** Việc bắt buộc, không bỏ qua, KHÔNG đợi Letri nhắc.
> 📁 **Việc lớn dở dang** → thêm file chi tiết `docs/handoff/<việc>.md` (tạm; xong thì archive).

---

## 📜 HIẾN PHÁP + TIẾN ĐỘ (ĐỌC TRƯỚC — ưu tiên CAO NHẤT)

> ⭐ **Từ 29/5/2026: `docs/ARCHITECTURE_CONTRACT.md` là HIẾN PHÁP — thắng mọi rule cũ nếu mâu thuẫn (Contract mục 0).**

| File | Vai trò | Khi nào đọc |
|---|---|---|
| [`docs/ARCHITECTURE_CONTRACT.md`](docs/ARCHITECTURE_CONTRACT.md) | 🔴 HIẾN PHÁP — 10 luật nền tảng (soft delete+thùng rác, PK uuid, tenant, branch, audit, response {data,meta}, pagination cursor, mã chứng từ, RBAC resource.action, naming) + bảng HUB đóng băng + checklist | **MỌI task code** |
| [`docs/PROGRESS.md`](docs/PROGRESS.md) | Xương sống tiến độ + release + QA nghiệm thu. Worker ghi nhật ký mỗi session | Trước/sau mỗi việc |
| [`docs/ARCHITECTURE_AUDIT.md`](docs/ARCHITECTURE_AUDIT.md) | Hiện trạng 400 bảng, 9 domain | Khi cần ngữ cảnh hệ thống |

> ⚠️ **20 rule cũ + skills đang ĐỒNG BỘ DẦN với Contract** (29/5 phát hiện 26 chỗ lệch: PK cuid2→uuid, tenant_id text→uuid, RBAC `:`→`.`, TS field snake→camelCase, soft delete 3 cấp). Khi rule cũ ghi khác Contract → **theo Contract**. Chi tiết: `docs/RULES_SYNC_PLAN.md`.

### `.claude/rules/` — CHI TIẾT KỸ THUẬT (Contract trỏ tới cho chi tiết từng mảng)

| # | File | Khi nào áp dụng | Mức độ |
|---|---|---|---|
| 00 | [`supreme-principles.md`](.claude/rules/00-supreme-principles.md) | Mọi task | 🔴 |
| 01 | [`security.md`](.claude/rules/01-security.md) | Auth, payment, PII, upload, mọi endpoint | 🔴 |
| 02 | [`multi-tenant.md`](.claude/rules/02-multi-tenant.md) | Mọi bảng nghiệp vụ, mọi query | 🔴 |
| 03 | [`database-drizzle.md`](.claude/rules/03-database-drizzle.md) | Schema, migration, query | 🔴 |
| 04 | [`api-design.md`](.claude/rules/04-api-design.md) | Tạo/đổi endpoint | 🔴 |
| 05 | [`frontend.md`](.claude/rules/05-frontend.md) | Tạo page/component | 🟠 |
| 06 | [`testing.md`](.claude/rules/06-testing.md) | Mọi feature mới | 🟠 |
| 07 | [`git-workflow.md`](.claude/rules/07-git-workflow.md) | Branch, commit, PR | 🟠 |
| 08 | [`code-quality.md`](.claude/rules/08-code-quality.md) | Mọi file code | 🟠 |
| 09 | [`performance.md`](.claude/rules/09-performance.md) | Endpoint mới, query mới | 🟡 |
| 10 | [`error-handling.md`](.claude/rules/10-error-handling.md) | Mọi nơi có error | 🔴 |
| 11 | [`logging-observability.md`](.claude/rules/11-logging-observability.md) | Mọi log statement | 🟠 |
| 12 | [`devops-cicd.md`](.claude/rules/12-devops-cicd.md) | Deploy, infra, CI/CD | 🟠 |
| 13 | [`ai-agents.md`](.claude/rules/13-ai-agents.md) | Mọi AI agent action | 🔴 |
| 14 | [`event-driven.md`](.claude/rules/14-event-driven.md) | Cross-module communication | 🟠 |
| 15 | [`compliance-vn.md`](.claude/rules/15-compliance-vn.md) | NĐ 13/2023, **TT 99/2025**, NĐ 70/2025 hoá đơn | 🔴 |
| 16 | [`claude-behavior.md`](.claude/rules/16-claude-behavior.md) | Cách Claude Code phải hành xử | 🔴 |

### `.claude/skills/` — HƯỚNG DẪN CHUYÊN MÔN (load on-demand)

| Skill | Khi nào load | Lĩnh vực |
|---|---|---|
| [`architecture/`](.claude/skills/architecture/SKILL.md) | Tạo module mới, refactor lớn | Kiến trúc |
| [`security-audit/`](.claude/skills/security-audit/SKILL.md) | Review bảo mật, threat model | Bảo mật |
| [`database-migration/`](.claude/skills/database-migration/SKILL.md) | Tạo migration phức tạp | Database |
| [`api-development/`](.claude/skills/api-development/SKILL.md) | Build endpoint hoàn chỉnh | API |
| [`frontend-component/`](.claude/skills/frontend-component/SKILL.md) | Build component có form/state | Frontend |
| [`testing-strategy/`](.claude/skills/testing-strategy/SKILL.md) | Viết test unit/integration/E2E | Testing |
| [`code-review/`](.claude/skills/code-review/SKILL.md) | Review PR | Quy trình |
| [`n8n-workflow/`](.claude/skills/n8n-workflow/SKILL.md) | Tạo/sửa workflow n8n | Automation |
| [`ai-agent-design/`](.claude/skills/ai-agent-design/SKILL.md) | Định nghĩa AI agent mới | AI |
| **[`accounting-tt99/`](.claude/skills/accounting-tt99/SKILL.md)** | **Hạch toán theo TT 99/2025/TT-BTC** | **Kế toán** |
| **[`manufacturing-fnb/`](.claude/skills/manufacturing-fnb/SKILL.md)** | **SX nguyên liệu, BOM, máy móc + CMMS** | **Sản xuất F&B** |
| **[`lms-courses/`](.claude/skills/lms-courses/SKILL.md)** | **Khoá học, thanh toán, chứng chỉ** | **LMS** |
| **[`pos-fnb/`](.claude/skills/pos-fnb/SKILL.md)** | **POS quán trà sữa, recipe, đơn ship** | **POS F&B** |
| **[`franchise-wecha/`](.claude/skills/franchise-wecha/SKILL.md)** | **Nhượng quyền, royalty, multi-brand** | **Franchise** |

---

## 🛡️ CLAUDE CODE BEHAVIOR (BẮT BUỘC)

> Đầy đủ ở [`.claude/rules/16-claude-behavior.md`](.claude/rules/16-claude-behavior.md). Tóm tắt:

### Trước khi viết code:
1. Đọc CLAUDE.md (file này)
2. Đọc rule liên quan (`.claude/rules/`)
3. Đọc skill liên quan (`.claude/skills/`)
4. Grep code tương tự trong repo
5. Đọc README + package.json của module

### Khi user yêu cầu vi phạm rule:
- **🔴 CRITICAL** → TỪ CHỐI + giải thích lý do + đề xuất giải pháp đúng
- **🟠 HIGH** → TỪ CHỐI + đề xuất giải pháp
- **🟡 MEDIUM** → CẢNH BÁO + xin xác nhận trước khi tiếp tục
- **🟢 LOW** → GỢI Ý cải thiện

### Khi không chắc:
- HỎI user, KHÔNG bịa
- Grep + đọc file thực tế trước khi trả lời
- Trích dẫn file:line khi nói về code có thật
- Nói "không tìm thấy" nếu không có, KHÔNG suy đoán

### Output format:
- Báo cáo có % phải có công thức tính (vd: 16/38 = 42%)
- Mỗi item gap analysis có cột Verify (📖🔍🖥❓)
- Phân biệt rõ "đã có", "có chưa đủ", "chưa có", "không xác định"

---

## 📋 QUICK REFERENCE — Khi nhận task...

| Task | Đọc trước |
|---|---|
| Tạo endpoint mới | `rules/01,02,04,06,10` + `skills/api-development/` |
| Tạo schema/migration | `rules/02,03,15` + `skills/database-migration/` |
| Tạo component frontend | `rules/05,08` + `skills/frontend-component/` |
| Sửa lỗi bảo mật | `rules/01,02` + `skills/security-audit/` |
| Tích hợp payment VNPay/MoMo | `rules/01,15` + `skills/api-development/` |
| Hạch toán/báo cáo tài chính | `rules/15` + `skills/accounting-tt99/` |
| Recipe/BOM trà sữa | `skills/pos-fnb/` + `skills/manufacturing-fnb/` |
| Khoá học mới + thanh toán | `skills/lms-courses/` + `rules/15` |
| Onboard franchise mới | `skills/franchise-wecha/` + `rules/02` |
| Tạo workflow n8n | `rules/13,14` + `skills/n8n-workflow/` |
| Build AI agent | `rules/13` + `skills/ai-agent-design/` |
| Tạo PR | `rules/07` + `skills/code-review/` |
| Orchestrate CEO con / multi-agent | `orchestrator-ceo/SKILL.md` + `orchestrator-ceo/MANDATE-*.md` |
| **File upload** (image, PDF, attachment) | `rules/01,17` + `skills/api-development/` |
| **WebSocket realtime** (POS kitchen, order status) | `rules/02,18` + `skills/api-development/` |
| **Background job / queue** (pg-boss) | `rules/14,19` + `skills/n8n-workflow/` |
| **Sync e-commerce** (Shopee/Lazada/Tiki) | `rules/01,14` + `skills/ecommerce-sync/` |
| **Threat model feature mới** (auth/payment/AI) | `skills/threat-modeling/` + `rules/01` |
| **Deploy production VPS** | `scripts/safe-deploy.sh` (5-step + verify env + auto-rollback) |
| Trước task tương tự task cũ | `grep` `docs/lessons-learned/<domain>/` |
| Sau khi mắc lỗi / bug / rollback | Tạo NGAY `docs/lessons-learned/<domain>/YYYY-MM-DD-NN-<slug>.md` |
| Memory rule project share team | `docs/memory-rules/` (KHÔNG `~/.claude/`) |
| Archive file cũ | IA Section F + `docs/memory-rules/no-delete-archive-only.md` |

---

## 📞 LIÊN HỆ

| Vai trò | Tên | Email |
|---|---|---|
| Owner / Tech lead | Lê Hữu Trí | engineering@wecha.vn |
| DPO (Data Protection) | Lê Hữu Trí (tạm) | dpo@wecha.vn |
| Kế toán trưởng | Nhi (vợ Trí) | kt.vuaantoan@gmail.com |
| Hotline doanh nghiệp | — | 0777 722 777 |

---

## 📅 LỊCH SỬ THAY ĐỔI

| Version | Date | Author | Changes |
|---|---|---|---|
| 3.2 | 2026-04-26 | Trí + Opus | Add 3 rules (17 file-upload, 18 websocket, 19 queue-pgboss) + 2 skills (ecommerce-sync, threat-modeling). 14 cross-link Phase 1. Safe-deploy script + 2 hooks pre-deploy. ADR 0005 inheritance. RLS rollout plan 20 bảng. Cách B (.env.production track git). |
| 3.0 | 2026-04-26 | Trí + Claude | Tách module: CLAUDE.md mục lục + 17 rules + 14 skills. Thêm TT 99/2025, manufacturing F&B, máy móc CMMS, LMS public, franchise. |
| 2.0 | 2026-04-26 | Trí + Claude | Single CLAUDE.md ~5200 dòng (deprecated, replaced by 3.0) |
| 1.0 | 2026-04-26 | Trí + Claude | Initial CLAUDE.md + 11 skills (deprecated) |

---

> **"Bảo mật là chi phí, không phải tính năng. Quality là attitude, không phải process."**
>
> — *Mọi dòng code chúng ta viết hôm nay, 5 năm sau vẫn còn người đọc. Hãy viết để người đó cảm ơn, không phải nguyền rủa.*
