# PROGRESS — Tiến độ hệ ERP WECHA / Vua An Toàn

> **Xương sống tiến độ + onboarding.** File này là nguồn tiến độ chính thức (kể cả mobile clone về xem).
>
> **Phiên bản:** 2.0 (sửa lớn — audit đúng trunk) | **Cập nhật:** 2026-06-29 | **Owner:** Lê Hữu Trí | **Người ghi:** CEO ERP (đọc & báo cáo)
>
> 🔒 **0 secret** — IP VPS để `<VPS_IP>`, không pass DB/token/key. Mobile cần IP thật để deploy → lấy từ phiên local, KHÔNG để GitHub.

---

## 0. ONBOARDING (đọc đầu tiên)

```bash
# Code thật của hệ ERP nằm ở TRUNK này (KHÔNG phải main):
git clone -b n8n/005-wecha-pos-zalo-keypos git@github.com:lehuutri28/erp-wecha-pos.git
# + đọc file này (docs/PROGRESS.md, repo erp-orchestration) để biết tiến độ
```

> ⚠️ **TRUNK THẬT = branch `n8n/005-wecha-pos-zalo-keypos`** (sửa hằng ngày). Branch `main` ĐÃ LẠC HẬU XA (~67 module cũ) — đừng audit `main`.
> ⛔ **KHÔNG đọc `api-google-appscript-vps/orchestrator-ceo/` (C1–C18):** đó là **kế hoạch lịch sử**, KHÔNG phải code deployed → đọc sẽ hiểu sai tiến độ.
> 📌 Số liệu dưới đây verify bằng `git ls-tree / git grep` trên `origin/n8n/005-wecha-pos-zalo-keypos` ngày 2026-06-29 (📖 đọc / 🔍 grep / 🖥 lệnh).

---

## 1. HIỆN TRẠNG TRUNK `n8n/005` (số thật)

| Chỉ số | Giá trị | Verify |
|---|---|---|
| Module backend (NestJS) | **169** | 🔍 `ls-tree apps/api/src/modules` |
| Controller | **244** | 🔍 đếm `*.controller.ts` |
| Schema file | **210** | 🔍 `ls-tree database/schema` |
| Migration | **~431** (mã tới `0431`, 659 file .sql) | 🔍 đếm `.sql` |
| Trang web (Next.js) | **565** | 🔍 đếm `page.tsx` |
| n8n workflow | **40** | 🔍 `ls-tree n8n` |

> 📌 So với báo cáo cũ (audit nhầm `main`): 67→**169** module, 29→**~431** migration, 176→**565** trang. Hệ lớn hơn nhiều và nhiều gap đã được vá.

---

## 2. TRẠNG THÁI CÁC GAP PHI-CHỨC-NĂNG (trên trunk)

| Hạng mục | Trạng thái | Công thức / Verify |
|---|---|---|
| RLS Postgres (Rule 02) | ⚠️ đang làm | 🔍 **24 migration** có `ROW LEVEL SECURITY` (trước = 0) |
| Soft-delete (Contract mục 1) | ⚠️ đang làm | 🔍 **73/210 schema = 35%** có `deletedAt` |
| RBAC `@RequirePermissions` (Rule 20) | ⚠️ đang làm | 🔍 **88/244 controller = 36%**; commit hôm nay đang fix thêm 65 gap Rule-20 |
| helmet (Rule 01 A05) | ✅ có | 🔍 `main.ts` có `helmet` |
| Password hash | ⚠️ chưa đủ | 🔍 vẫn **bcrypt** (argon2 = 0 file); chưa 2FA TOTP (otplib = 0) |
| Test (Rule 06) | ❌ chưa đủ | 🔍 **21 spec / 244 controller ≈ 9%**; **0 E2E/Playwright** |
| CI/CD | ❌ chưa có | 🔍 `.github/workflows` = 0 |

> 📐 Bảo mật/đa-tenant đã khá hơn nhiều so với báo cáo cũ, nhưng RLS/soft-delete/RBAC **mới phủ ~35%**, còn lại đang cuốn chiếu (ĐỢT 3 đang chạy hôm nay).

---

## 3. CÁI GÌ CHƯA DEPLOY

> Theo chính commit mới nhất trên trunk (29/6):

1. **ĐỢT 3 sales read-filter + 5 worker fix 65 gap Rule-20 + bulk** — commit `173a57b7` tự ghi *"CHƯA build/review/deploy"*.
2. **Migration `0430` (backfill) + `0431`** (chuẩn hoá cột 46 bảng giao dịch) — *"CHỜ apply"* lên DB.
3. **42 branch `feat/*` chưa merge** vào trunk/main (admin-service-tokens, approval-engine-bpm, audit-log-immutable, bank-reconciliation, inventory-lots-fefo, loyalty-voucher… mỗi nhánh +200–600 commit). → cần xác nhận: đã gom vào `n8n/005` chưa hay còn rời.
4. **Toàn bộ `webphache`** (tích hợp phache.com.vn): Phase A code+test xong nhưng A.4 wrap chưa apply, `.env` placeholder, chatbot AI 0 file → **0% deployed**.

> ❓ **Branch production thực tế:** chưa xác định chính xác (không vào được server). Trunk `n8n/005` là mới nhất nhưng WIP cuối "chưa deploy". Cần Letri xác nhận điểm deploy.

---

## 4. KẾ HOẠCH CODE TIẾP THEO

### 🔴 ĐỢT 1 — Đóng nốt nền tảng đang cuốn chiếu (trên trunk n8n/005)
1. **RBAC Rule-20:** phủ nốt 156/244 controller chưa có `@RequirePermissions` (hiện 88/244). *Cạm bẫy: seed quyền + gán policy TRƯỚC, kẻo 403.*
2. **RLS + soft-delete:** mở rộng từ ~35% lên đủ 20 bảng nghiệp vụ ưu tiên + 210 schema.
3. **Apply migration `0430/0431`** (backup DB + test staging trước — Rule 03/12).
4. **Build + review + deploy ĐỢT 3** (đang WIP).

### 🟠 ĐỢT 2 — Gom nợ tích hợp
5. Merge/đối soát **42 branch feat** về trunk (tránh trôi mỗi nhánh +200 commit).
6. **webphache:** apply A.4 wrap + nhúng JS/CSS + code chatbot AI.

### 🟡 ĐỢT 3 — Hardening
7. **CI/CD** (lint+typecheck+test+security-scan) → tăng test (hiện ~9%, 0 E2E) → **argon2id + 2FA TOTP** → Outbox pattern.

---

## 5. CÁC REPO KHÁC (tóm tắt)

| Repo | Vai trò | Trạng thái |
|---|---|---|
| **erp-orchestration** (repo này) | Hiến pháp + 21 rule + 16 skill + Contract | ✅ governance đủ, 0 code (cố ý) |
| **api-google-appscript-vps** | Điều phối CEO con + docs lịch sử (C1–C18) | ⚠️ KHÔNG phải code deployed — **đừng dùng để đo tiến độ** |
| **webphache** | Tích hợp phache.com.vn | ⚠️ Phase A xong, chưa deploy |
| **c10-erp-wecha** | — | ❌ rỗng (0 commit) |

---

## 6. 🔚 BÀN GIAO CHO SESSION SAU

- **Vừa làm:** Phát hiện trunk thật là `n8n/005-wecha-pos-zalo-keypos` (main đã chết); audit lại số thật (169 module/~431 mig/565 trang); sửa `PROGRESS.md` v2.0.
- **Dừng tại:** `erp-orchestration/docs/PROGRESS.md` (commit branch `claude/ceo-erp-reading-reporting-344rb1`).
- **Bước tiếp theo CHÍNH XÁC:** clone trunk `n8n/005`, chạy audit đầy đủ 169 module → đối chiếu RBAC 156 controller còn thiếu quyền (danh sách = 244 controller trừ 88 đã có `@RequirePermissions`).
- **Cạm bẫy:** ĐỪNG audit `main` (lạc hậu). ĐỪNG đọc orchestrator-ceo C1–C18 (kế hoạch lịch sử). Apply mig phải backup + staging trước.
- **Hệ khác cần biết:** mig `0430/0431` CHỜ apply; ĐỢT 3 sales WIP chưa build/deploy; 42 branch feat chưa rõ đã gom trunk chưa.
