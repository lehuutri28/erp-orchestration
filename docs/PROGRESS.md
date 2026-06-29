# PROGRESS — Tiến độ hệ ERP WECHA / Vua An Toàn

> **Xương sống tiến độ + bàn giao session.** File này là nguồn tiến độ chính thức cho điều phối (kể cả mobile clone repo này về xem).
>
> **Phiên bản:** 1.0 (dựng lại) | **Cập nhật:** 2026-06-29 | **Owner:** Lê Hữu Trí | **Người ghi:** CEO ERP (đọc & báo cáo)
>
> 🔒 **0 secret** — IP VPS để dạng `<VPS_IP>`, KHÔNG chứa pass DB / token / key. Mobile cần IP thật để deploy → lấy từ phiên local, KHÔNG để trên GitHub.
>
> 📌 **Nguồn dữ liệu:** Audit thực tế 5 repo ngày 2026-06-29 (verify bằng `ls/grep/git/wc`) + `erp-wecha-pos/docs/TASK-LOG.md` + `docs/wiki/SYSTEM-SNAPSHOT-20260416.md` + `api-google-appscript-vps/orchestrator-ceo/ceos/handovers/`. Mọi con số có công thức + verify icon (📖 đọc / 🔍 grep / 🖥 chạy lệnh).

---

## 1. TỔNG QUAN 5 REPO

| Repo | Bản chất | Tiến độ | Trạng thái |
|---|---|---|---|
| **erp-wecha-pos** | Code CHÍNH (ERP/POS) | Rộng ~78%, chất lượng ~25–35% | ⚠️ Chạy prod nhưng hổng phi-chức-năng |
| **webphache** | Bundle tích hợp phache.com.vn | Phase A ~83% code, deploy 0% | ⚠️ Soạn sẵn, chưa deploy |
| **api-google-appscript-vps** | Điều phối CEO con + docs + infra script | Governance đủ; KHÔNG chứa code app | ✅ Đúng thiết kế |
| **erp-orchestration** | Hiến pháp / governance | 100% docs, 0 code (cố ý) | ✅ Đúng thiết kế |
| **c10-erp-wecha** | Repo rỗng | 0% (0 commit / 0 file) | ❌ Chưa bắt đầu |

---

## 2. erp-wecha-pos — REPO CHÍNH

### ✅ Đã có (code thật, đang chạy production: erp.wecha.vn / pos.vuaantoan.com — VPS `<VPS_IP>`)

| Thành phần | Verify |
|---|---|
| 67 module NestJS = 32,714 LOC (orders 782, payments 866, manufacturing 773…) | 🔍 `ls apps/api/src/modules` = 67 |
| 176 trang Next.js = 51,262 LOC (admin/hr/kho/pos/crm/einvoice…) | 🔍 `find apps/web -name page.tsx` = 176 |
| 52 schema + 29 migration SQL (0000→0029), PK=uuid, tenant_id=uuid | 🔍 `ls schema` / `ls migrations` |
| 9 n8n workflow | 🔍 `ls n8n/workflows` = 9 JSON |
| Tích hợp Shopee/Lazada/Tiki + VNPay/MoMo + GHN/GHTK + HĐĐT | 📖 modules platform-*/payment-gateway/shipping-adapters/einvoice |
| Stack khớp CLAUDE.md: NestJS 10.4 + Drizzle 0.38 + pg-boss 10.1 + Next 14.2 | 📖 package.json |

**📐 % chiều rộng:** 67/86 module = **78%** | 22 phân hệ nghiệp vụ tự báo hoàn thành (snapshot 16/04).

### 🔴 Hổng nặng phi-chức-năng (vi phạm rule CRITICAL)

| Hạng mục | Trạng thái | Công thức / Verify |
|---|---|---|
| RLS Postgres (Rule 02) | ❌ chưa có | 🔍 grep `ROW LEVEL SECURITY` = **0/20 bảng** |
| Soft-delete + thùng rác (Contract mục 1) | ❌ chưa có | 🔍 grep `deleted_at` schema = **0** |
| RBAC `@RequirePermissions` (Rule 20) | ⚠️ chưa đủ | **8/66 controller = 12%**; 56 còn lại chỉ JwtAuthGuard |
| Test (Rule 06 target 80%) | ❌ chưa có | **2 spec / ~145 file = ~1%** |
| Security `main.ts` (Rule 01 A05) | ❌ chưa có | thiếu helmet, rate-limit, CORS hardcode, Swagger không guard |
| Password hash | ⚠️ chưa đủ | bcrypt (target argon2id), chưa 2FA TOTP |
| CI/CD | ❌ chưa có | `.github` không tồn tại |
| Outbox event (Rule 14) | ⚠️ chưa đủ | EventEmitter in-memory, mất nếu crash |

> 📐 **Tuân thủ rule production-grade ước ~25–35%.** Làm được nhiều tính năng, nhưng chưa "cứng" về bảo mật / đa-tenant / test theo hiến pháp.

---

## 3. webphache — Tích hợp phache.com.vn ↔ ERP

- ✅ **Phase A** (đẩy lead web→CRM): `CrmClient.php` 419 dòng (circuit breaker + retry + JWT rotate) + **test PASS 77/0 assertion** 🖥. Code chất lượng cao, PHP 5.6-compat.
- ⚠️ **Phase B**: chỉ có CSS + auto-fill + courses-list. **Chatbot AI (mục tiêu gốc dự án) = 0 file** 🔍 grep=0.
- ❌ **Chưa deploy**: A.4 wrap form chưa apply (mốc 12/5), `.env` còn placeholder, JS/CSS chưa nhúng vào site.

> 📐 Code soạn sẵn ~70–75%, đã-tích-hợp-vào-site = **0%**.

---

## 4. 2 repo GOVERNANCE + 1 repo rỗng

- **api-google-appscript-vps**: tên gây hiểu nhầm — KHÔNG có code bridge AppScript (🔍 `find *.gs`=0). Thực chất là repo điều phối CEO con (C1–C11) + TASK_LOG (8,700 dòng) + monitoring scripts. Code ERP thật ở repo riêng `claude/pos-system` (bị .gitignore).
- **erp-orchestration** (repo này): hiến pháp đầy đủ — **21 rule + 16 skill + ARCHITECTURE_CONTRACT** = 42 file `.md`, 0 code (cố ý). Governance ~100%.
- **c10-erp-wecha**: ❌ **0%** — git ls-files=0, 0 commit, working tree chỉ có `.git/`.

---

## 5. KẾ HOẠCH CODE TIẾP THEO

> Ưu tiên theo mức độ rule (🔴 trước). Nguồn: gap đã verify + `nextSteps` trong TASK_LOG/handoff thật.

### 🔴 ĐỢT 1 — "Cứng hoá" hệ chính erp-wecha-pos (nợ kỹ thuật CRITICAL)
1. **Bật RLS 20 bảng** (Rule 02). Bắt đầu: `orders, order_items, payments, invoices, customers`. Mỗi bảng: `ENABLE + FORCE RLS` + policy `tenant_isolation` + `service_role_bypass`.
2. **Soft-delete 3 cấp + thùng rác** (Contract mục 1): thêm `deletedAt/deletedBy/deletedReason/version` vào 52 schema + helper `notDeleted()`.
3. **Phủ `@RequirePermissions` cho 56 controller còn lại** (Rule 20). ⚠️ *Cạm bẫy: seed quyền + gán policy TRƯỚC khi thêm guard, kẻo khoá NV 403.*
4. **Security `main.ts`** (Rule 01 A05): helmet + `@nestjs/throttler` rate-limit + CORS từ env + guard Swagger theo `NODE_ENV`.

### 🟠 ĐỢT 2 — Hoàn thiện dở dang (từ handoff thật)
5. Fix **BUG-002** (inventory không emit event) + **BUG-003** (payments tính lại công nợ) — `erp-wecha-pos/docs/wiki/BUGS-ACTIVE.md`.
6. Gỡ blocker **C4 MISA migration 0060–0067** (đang BLOCKED do branch merge dở + .gitignore chặn *.sql) — `api-google-appscript-vps/orchestrator-ceo/TASK_LOG.md`.
7. Apply migration **0043 / 0048 / 0052** lên prod + thay mock data dashboard/MRP/cost bằng API thật. Resolve xung đột số migration 0043 (C1 vs C10).
8. **webphache**: apply A.4 wrap (form→ERP) + nhúng JS/CSS + code **chatbot AI** (ChatWidget/chatbot.php/ClaudeClient/KnowledgeBase + bảng chat_session/chat_message).

### 🟡 ĐỢT 3 — Nền tảng
9. CI/CD (lint+typecheck+test+security-scan) → tăng test coverage (hiện ~1%) → migrate argon2id + 2FA TOTP → Outbox pattern cho event.

---

## 6. ⚠️ ĐIỂM CẦN XÁC NHẬN (chưa kết luận)

1. **❓ Lệch số liệu repo:** Handover 26/4 báo hệ thật **86 module / 279 bảng / 91 migration**, nhưng `erp-wecha-pos` truy cập được chỉ có **67 module / 29 migration**. → `erp-wecha-pos` có thể là snapshot cũ/một phần; bản đầy đủ ở `claude/pos-system` (ngoài 5 repo được cấp quyền). **Cần chốt repo nào là source-of-truth.**
2. **❓ Doc tiến độ cũ mâu thuẫn:** `erp-wecha-pos/docs/TIEN-DO-DU-AN.md` vẫn ghi 0/31 (stale 09/04) trong khi snapshot 16/04 báo ~đầy đủ.

---

## 7. 🔚 BÀN GIAO CHO SESSION SAU

- **Vừa làm:** Audit 5 repo (verify file thật) + dựng lại `docs/PROGRESS.md` (file này), IP scrub `<VPS_IP>`.
- **Dừng tại:** `erp-orchestration/docs/PROGRESS.md` (đã commit branch `claude/ceo-erp-reading-reporting-344rb1`).
- **Bước tiếp theo CHÍNH XÁC:** Bắt ĐỢT 1.1 — viết migration `ENABLE ROW LEVEL SECURITY` cho `orders` (mẫu ở `.claude/rules/02-multi-tenant.md` mục "Migration template enable RLS") trong repo `erp-wecha-pos`.
- **Cạm bẫy cần nhớ:** Thêm RBAC guard phải seed quyền TRƯỚC (kẻo 403). Apply migration phải backup DB + test staging trước (Rule 03/12).
- **Hệ khác cần biết:** IP thật `157.x.x.x` còn nằm trong ~29 file đã track của repo `api-google-appscript-vps` — cần scrub nếu repo đó push public.
