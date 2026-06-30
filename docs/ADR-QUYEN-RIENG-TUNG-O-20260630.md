# ADR — QUYỀN RIÊNG TỪNG Ổ (PER-DATABASE RBAC) — kiểu MISA

> **Trạng thái:** PROPOSED (chỉ thiết kế — CHƯA code/migration/build)
> **Ngày:** 2026-06-30 20:45 (+07)
> **Tác giả:** Opus CEO ERP (worker chiến lược)
> **Repo:** `claude/pos-system` · branch `n8n/005-wecha-pos-zalo-keypos`
> **Bối cảnh:** Letri muốn user có quyền KHÁC NHAU ở mỗi ổ (CSDL) — vd full ở ổ A, chỉ xem ở ổ B — giống MISA.
> **Mức độ bảo mật:** 🔴 CRITICAL — đụng access control xuyên schema → guard → JWT → RLS → UI.
> **Nguyên tắc chủ đạo:** ADDITIVE + BACKWARD-COMPATIBLE. Không phá hệ đang chạy LIVE. Mỗi phase deploy độc lập, fail-open an toàn.

---

## PHẦN A — AUDIT HIỆN TRẠNG (verify-first, trích file:dòng)

### A1. Mô hình quyền hiện tại = USER × ROLE × BRANCH (KHÔNG có chiều Ổ)

`apps/api/src/database/schema/roles.ts`:
- Dòng 8-17 `roles`: `permissions jsonb` (mảng `["products.view","inventory.*"]`), `isSystem`, `isFactoryOnly`. **Permission gắn vào ROLE, không gắn ổ.**
- Dòng 21-28 `userRoles`: PK = `(userId, roleId, branchId)` + cột `scope varchar(20) default 'branch'` (`'all'` = tổng, `'branch'` = chi nhánh). **Chiều scope hiện tại là CHI NHÁNH, KHÔNG phải Ổ.**

→ **Kết luận:** Hệ thống hiện CHƯA có khái niệm "quyền theo ổ". Quyền của user là hằng số trên toàn hệ thống (union các role), chỉ giới hạn theo `branchId` ở vài chỗ.

### A2. JWT mang sẵn toàn bộ roles+permissions — guard KHÔNG query DB mỗi request

`apps/api/src/common/types.ts` dòng 1-12 `JwtPayload`:
```
sub, email, tenantId, roles: [{ roleId, roleName, branchId, scope, permissions[] }]
```
→ **KHÔNG có `databaseId` trong roles.** Permission là mảng phẳng theo role, không phân theo ổ.

`apps/api/src/modules/auth/jwt.strategy.ts` dòng 16-18: `validate(payload) { return payload; }` — **trả nguyên payload, KHÔNG re-fetch DB.** Mọi quyền đọc từ JWT.

`apps/api/src/modules/auth/auth.service.ts`:
- Dòng 50-62 `getUserRoles()`: join `userRoles → roles`, select `branchId, scope, permissions`. **Không đụng tới database/ổ.**
- Dòng 380-395 (`login`) + 580-597 (`refresh`): build `payload.roles` từ `getUserRoles()`. JWT = ảnh chụp quyền lúc đăng nhập.
- Dòng 405-407: `flatPerms = union(roles.permissions)` → trả `user.permissions` root (flatten toàn cục, KHÔNG theo ổ).

### A3. PermissionsGuard check trên union TOÀN CỤC (không theo ổ)

`apps/api/src/modules/auth/permissions.guard.ts`:
- Dòng 7-8 `RequirePermissions(...perms)` decorator.
- Dòng 32-37: `userPermissions = user.roles.flatMap(r => r.permissions)` → check `requiredPermissions.some(p => hasPermission(userPermissions, p))`. **OR logic, union TẤT CẢ role bất kể ổ/chi nhánh nào.**
- Dòng 41-67 `BranchGuard` (`@RequireBranch`): chỉ check `branchId` từ header `x-branch-id`, scope `'all'` hoặc `'*'` thì bỏ qua. **Vẫn không có chiều ổ.**

`apps/api/src/common/permissions.ts` dòng 707-717 `hasPermission()`: wildcard `*` / `module.*` / exact. Đây là hàm dùng chung BE.

### A4. FE flatten permission theo BRANCH+SCOPE — KHÔNG theo ổ

`apps/web/src/contexts/auth-context.tsx`:
- Dòng 57-63 `hasPermission()`: wildcard giống BE.
- Dòng 164-189 `allPermissions` (useMemo): **flatten dựa trên `currentBranchId` + `scope`** (dòng 175-183). Filter role: scope `all`/`group` → global; scope `branch` → khớp `currentBranchId`. **KHÔNG có biến nào là `currentDatabaseId`.** Đổi ổ KHÔNG đổi permission set.
- Dòng 191-194 `checkPermission` + dòng 320 expose `hasPermission` cho 112 consumer.

### A5. "Quyền vào ổ" hiện được SUY GIÁN TIẾP qua chi nhánh (không có bảng quyền-ổ)

`apps/web/src/components/database-select-guard.tsx`:
- Dòng 37 `isSuperAdmin = allPermissions.includes('*')`.
- Dòng 42-44 `allowedBranchIds = set(user.roles.map(r => r.branchId))`.
- Dòng 81-92 `permittedDatabaseIds`: super-admin → `null` (thấy hết); còn lại → **set các `databaseId` của chi nhánh mà user có role** (qua `branches.databaseId`). → "Quyền vào ổ X" = "user có role ở ≥1 chi nhánh thuộc ổ X".
- Dòng 97-100 `visibleDatabases`: FAIL-OPEN — `permittedDatabaseIds === null || size === 0` → hiện HẾT (tránh lockout).
- Dòng 130-133: lọc chi nhánh theo `allowedBranchIds`, fail-open nếu rỗng.

→ **Đây chính là điểm cần nâng cấp:** quyền-ổ hiện là phái sinh nhị phân (vào được / không vào được), KHÔNG có khái niệm "vào được nhưng quyền khác nhau giữa các ổ".

### A6. Ma trận quyền hiện tại = ROLE × MODULE (2 chiều, không có ổ)

`apps/web/src/app/(dashboard)/admin/permissions/matrix/page.tsx`:
- Dòng 27 `roles` state, dòng 36 fetch `/roles` + `/roles/permissions-list`.
- Dòng 44-50 `hasRolePerm(role, permKey)`: đọc `role.permissions[]`.
- Dòng 161-270: render bảng **Module (hàng) × Role (cột)**. **Chỉ 2 chiều. Không có chiều ổ.** Đây là VIEW read-only; sửa quyền ở `/admin/roles`.

### A7. Hạ tầng CSDL-scoping ĐÃ CÓ SẴN (tái dùng được)

- `apps/api/src/common/middleware/database-context.middleware.ts` dòng 16-24: đọc header `X-Database-Id` (fallback `?database_id=`) → set `req.databaseId`. **G0: chỉ propagate, CHƯA lọc query nào** (dòng 11-13).
- `apps/api/src/common/utils/database-filter.util.ts` dòng 15-23 `databaseFilter(table, databaseId, {strict})`: fail-open (`OR database_id IS NULL`) → strict sau backfill. **CHƯA service nào gọi (G4)** (dòng 12-14).
- `apps/web/src/lib/api.ts` dòng 106-124: mọi `apiFetch` tự đính `X-Database-Id` (từ `localStorage 'erp_active_database'`) + `X-Branch-Id`. → **FE đã gửi ổ active mỗi request rồi.**
- `apps/api/src/database/schema/users.ts` dòng 14: `defaultDatabaseId uuid('default_database_id')` — ĐÃ CÓ.
- `apps/web/src/components/database-select-guard.tsx` dòng 169-186: PATCH `/users/me/default-database` lưu ổ mặc định cross-device.

### A8. Quan hệ Ổ ↔ Chi nhánh ↔ Entity

- `databases.ts` dòng 6-23 `databases`: `companyId` (nullable, mig 0082), `databaseType` (`commerce|education|manufacturing`), `year`.
- `branches.ts` dòng 12 `databaseId` (legacy single-FK, nullable) + `isFactory`, `branchType`.
- `branch-databases.ts` dòng 12-26 `branchDatabases` (M2M): PK `(branchId, databaseId)` + `linkRole` (`primary|tax_only|non_invoice`). → **1 chi nhánh ghi vào NHIỀU ổ.**
- `entity-scopes.ts` dòng 6-17 `entityScopes`: node org-hierarchy (`company|independent_branch|dependent_branch`) + `accountingScheme`. (Không trực tiếp liên quan quyền user, nhưng là chiều pháp nhân.)

### A9. ⚠️ TRẠNG THÁI RLS (verify-first — KHÁC giả định)

- 24 migration `.sql` CÓ `ENABLE ROW LEVEL SECURITY` + `CREATE POLICY` cho `tenant_id` (dùng `current_setting('app.current_tenant_id', true)`). VD `0070_customer_province_and_branches.sql` dòng 79-81.
- **NHƯNG:** grep toàn bộ TS (`common`, `database`, `modules/_core`) **KHÔNG tìm thấy** chỗ nào gọi `set_config('app.current_tenant_id', ...)` hay `SET LOCAL app.current_tenant_id` per request.
- → **Kết luận quan trọng:** App kết nối Postgres bằng role **owner/superuser** (RLS có `ENABLE` nhưng KHÔNG `FORCE` → owner bypass). RLS hiện **DORMANT** ở tầng app — chỉ là defense-in-depth "ngủ", chưa thực thi. Việc bật RLS thực sự (cả tenant lẫn database) là một dự án riêng cần set session-var per request + đổi DB role.

> 🔴 Hệ quả thiết kế: **KHÔNG được coi RLS là lớp chặn đang hoạt động.** Lớp chặn THẬT hiện tại = Guard (app layer). RLS cho `database_id` đưa vào Phase 4 như defense cuối, kèm điều kiện tiên quyết "bật được RLS tenant trước".

---

## PHẦN B — THIẾT KẾ (QUYẾT ĐỊNH KIẾN TRÚC)

### B0. Option đã cân nhắc (chọn có lý do)

| # | Phương án | Mô tả | Pros | Cons | Quyết định |
|---|---|---|---|---|---|
| 1 | **Nhân role theo ổ** | Mỗi ổ tạo bản sao role riêng | Không đổi schema | Bùng nổ role (N ổ × M role), khó quản lý, không phải "override" | ❌ |
| 2 | **Thêm `databaseId` vào `userRoles`** | Mở rộng bảng gán hiện có | Ít bảng mới | Phá PK hiện tại `(userId,roleId,branchId)`, đổi semantic cột đang LIVE → rủi ro cao | ❌ |
| 3 | **Bảng MỚI `user_database_roles` (override theo ổ) + fallback role toàn cục** | Bảng phụ chỉ chứa override; user không có override → dùng role hiện tại | Backward-compat tuyệt đối, đúng mô hình MISA (override per-ổ), bảng cũ bất biến | Resolution phức tạp hơn 1 chút | ✅ **CHỌN** |
| 4 | **ABAC/policy engine (CASL)** | Viết lại RBAC thành ABAC | Linh hoạt nhất | Over-engineer, viết lại 244 controller, rủi ro khổng lồ cho hệ LIVE | ❌ (để Wave sau nếu cần) |

**Lý do chọn Option 3:** Đúng yêu cầu Letri ("override per-ổ kiểu MISA"), giữ nguyên `roles`+`userRoles` đang LIVE (không phá PK, không đổi semantic), thêm 1 bảng phụ ADDITIVE. Khi user KHÔNG có dòng override cho ổ → tự động fallback về quyền toàn cục hiện tại → hệ đang chạy KHÔNG vỡ.

### B1. SCHEMA MỚI — bảng `user_database_roles` (override quyền theo ổ)

> ⚠️ Phác thảo Drizzle — KHÔNG phải migration thật. Migration SQL viết ở phase implement.

```
user_database_roles
─────────────────────────────────────────────
id            uuid  PK defaultRandom
tenantId      uuid  notNull  → tenants.id          (multi-tenant)
userId        uuid  notNull  → users.id
databaseId    uuid  notNull  → databases.id        (CHIỀU MỚI: ổ)
roleId        uuid  nullable → roles.id            (gán role có sẵn cho ổ này)
extraPermissions  jsonb default []                 (grant thêm quyền lẻ cho ổ này)
deniedPermissions jsonb default []                 (CHẶN quyền cụ thể ở ổ này — ưu tiên cao nhất)
accessLevel   varchar(20) default 'custom'         ('full' | 'readonly' | 'none' | 'custom')
isActive      boolean default true
createdAt / createdBy / updatedAt / deletedAt      (audit + soft-delete theo Contract)
─────────────────────────────────────────────
UNIQUE (tenantId, userId, databaseId, roleId)       — 1 user có thể gán nhiều role/ổ
INDEX  (tenantId, userId)                            — resolution lookup
INDEX  (tenantId, databaseId)                        — audit theo ổ
PARTIAL INDEX WHERE deletedAt IS NULL
```

**Ngữ nghĩa:**
- `accessLevel='readonly'` = shortcut "chỉ xem" cho ổ → resolver chỉ giữ các quyền `*.view`/`*.read` (xem B2.4).
- `accessLevel='none'` = chặn hoàn toàn ổ này (user không vào được — phủ định cả quyền toàn cục).
- `accessLevel='custom'` = dùng `roleId` + `extraPermissions` − `deniedPermissions`.
- `accessLevel='full'` = `['*']` cho ổ đó.
- **`deniedPermissions` thắng tất cả** (deny-by-default cho field nhạy cảm: `products.edit_cost`, `hr_salary.*`, `reports.view_finance`...).

> Backward-compat tuyệt đối: bảng rỗng = hệ chạy y như hiện tại (resolver luôn fallback role toàn cục).

### B2. RESOLUTION — quyền HIỆU LỰC theo ổ đang chọn

Hàm trung tâm (BE), gọi `resolveEffectivePermissions(userId, databaseId)`:

```
1. globalPerms = union(userRoles → roles.permissions)        // = hành vi HIỆN TẠI
2. overrides = user_database_roles WHERE userId & databaseId & isActive & not deleted
3. NẾU overrides rỗng:
       → return globalPerms                                   // FALLBACK — không vỡ hệ cũ
4. NẾU bất kỳ override.accessLevel='none':
       → return []                                            // chặn ổ
5. base =
       - 'full'     → ['*']
       - 'readonly' → globalPerms.filter(p => là view/read) ∪ overrideViewPerms
       - 'custom'   → union(override.roleId→roles.permissions) ∪ extraPermissions
       - (nếu trộn nhiều dòng → union tất cả base)
6. effective = base − deniedPermissions (deny thắng)          // áp deny SAU cùng
7. return effective
```

**Quy tắc vàng:** override **THAY THẾ** globalPerms cho ổ đó (không cộng dồn mù quáng) — để "full ổ A, chỉ xem ổ B" hoạt động đúng (ổ B không kế thừa full của global). `extraPermissions` mới là phần cộng thêm có chủ đích.

> Quyết định: **override = replace, extra = add, denied = subtract.** Đơn giản, dự đoán được, đúng MISA.

### B3. JWT / CONTEXT — ổ active drive permission set thế nào

**Vấn đề:** JWT hiện mang `roles[]` cố định (ảnh chụp lúc login). Nếu nhúng toàn bộ map ổ×quyền vào JWT → token phình to + stale khi admin đổi quyền.

**Option JWT:**

| Cách | Mô tả | Pros | Cons |
|---|---|---|---|
| 3a | **Guard re-resolve theo `req.databaseId` mỗi request** (query `user_database_roles` + cache Redis 60s) | Luôn đúng, đổi quyền áp ngay (≤60s), token nhỏ | Thêm 1 query/request (giảm bằng cache) |
| 3b | Nhúng map `{databaseId: perms[]}` vào JWT | Không query | Token phình (N ổ), stale tới khi refresh, lộ cấu trúc quyền |
| 3c | Re-issue JWT mỗi lần đổi ổ | Token gọn theo ổ | FE phải re-login-flow khi đổi ổ, phức tạp, cross-tab khó |

**CHỌN 3a — Guard re-resolve + cache Redis.** Lý do: đúng triết lý hiện tại (JWT có sẵn nhưng guard vẫn là điểm chốt), token KHÔNG đổi shape (backward-compat), admin đổi quyền áp dụng gần tức thì. Cache key: `perm:eff:{userId}:{databaseId}` TTL 60s, invalidate khi sửa `user_database_roles`/`roles`/`userRoles`.

**Cơ chế:**
- `PermissionsGuard` đọc `req.databaseId` (đã có từ `DatabaseContextMiddleware`).
- Nếu có `databaseId` → gọi `resolveEffectivePermissions(user.sub, databaseId)` (Redis-cached) thay cho `user.roles.flatMap(...)`.
- Nếu KHÔNG có `databaseId` (login, n8n, endpoint không scope ổ) → **giữ nguyên** logic cũ (union toàn cục) → fail-open backward-compat.
- Super-admin (`*`) → bypass resolution (luôn full).

> JWT vẫn giữ `roles[]` cũ để: (a) backward-compat token đang lưu ở client, (b) fallback khi Redis/DB lỗi (degrade về quyền toàn cục — fail-open, KHÔNG khóa NV).

**FE:** `auth-context.tsx` cần thêm `currentDatabaseId` + khi đổi ổ → re-fetch `/auth/me/effective-permissions?databaseId=X` (endpoint mới) → cập nhật `allPermissions`. Hoặc đơn giản hơn: BE trả `effectivePermissions` trong response của mọi call (qua `meta`) — nhưng cách sạch là 1 endpoint riêng gọi khi đổi ổ (giống đổi branch hiện tại).

### B4. MIGRATION ADDITIVE (chỉ ADD — phác thảo, KHÔNG SQL thật)

| MIG | Nội dung | Rủi ro | Reversible |
|---|---|---|---|
| 0432 | `CREATE TABLE IF NOT EXISTS user_database_roles (...)` + indexes | LOW (bảng mới, không đụng bảng cũ) | DROP TABLE |
| 0433 | (tùy chọn) seed: KHÔNG seed gì — bảng rỗng = hành vi cũ | NONE | — |
| (sau, Phase 4) | RLS `database_id` cho bảng nghiệp vụ — CHỈ khi RLS tenant đã bật thật (xem A9) | HIGH | DROP POLICY |

**Quy tắc:** mỗi migration có `_down.sql`. Bảng `user_database_roles` KHÔNG phải bảng HUB → nhưng vì là access-control nên VẪN báo Letri review trước apply. KHÔNG `drizzle-kit push` prod (memory rule). Verify schema prod port 5433 trước.

### B5. UI — MA TRẬN 3 CHIỀU (user × ổ × quyền/vai trò)

**Vấn đề:** ma trận hiện 2D (module × role). Thêm chiều ổ → 3D không hiển thị phẳng được. **Giải pháp: tách thành 2 màn, KHÔNG nhồi 3D vào 1 bảng.**

**Màn 1 — Tổng quan "User × Ổ" (`/admin/permissions/database-access`):**
- Bảng: hàng = User, cột = Ổ (database). Mỗi ô hiển thị badge `accessLevel`: 🟢 Full · 🔵 Chỉ xem · 🟡 Tùy chỉnh · ⬜ Mặc định (fallback role) · 🔴 Chặn.
- Click ô → mở panel chi tiết (Màn 2).

**Màn 2 — Panel "Quyền của User X trong Ổ Y":**
- Dropdown `accessLevel` (Full / Chỉ xem / Tùy chỉnh / Chặn / Mặc-định-theo-role).
- Khi `Tùy chỉnh`: chọn role gán cho ổ + checkbox `extraPermissions` (tái dùng `PERMISSION_MODULES` ở `permissions.ts`) + checkbox `deniedPermissions` (nhóm field nhạy cảm: giá vốn, lương, sổ quỹ, công nợ — highlight đỏ).
- Preview "quyền hiệu lực" (gọi `resolveEffectivePermissions` ở FE-readonly) để admin thấy kết quả trước khi lưu.

**Màn ma trận cũ (`matrix/page.tsx`):** giữ nguyên (role × module toàn cục) + thêm 1 dropdown "Xem theo ổ" (optional) để lọc preview — KHÔNG bắt buộc đổi.

> Quyết định: 2 màn riêng > nhồi 3D. Dễ dùng, đúng UX MISA (MISA cũng tách "phân quyền theo dữ liệu kế toán").

### B6. RLS `database_id` — TÍCH HỢP (Phase 4, có điều kiện tiên quyết)

Theo A9, RLS hiện DORMANT. Thiết kế RLS database_id chỉ kích hoạt SAU khi:
1. App đổi sang DB role non-superuser HOẶC `FORCE ROW LEVEL SECURITY`.
2. App `set_config('app.current_database_id', req.databaseId, true)` per request (transaction-scoped) — pattern y hệt `app.current_tenant_id` đã có trong 24 migration.
3. Backfill `database_id` 100% (đã có mig 0430 `backfill_database_id` — verify) + SET NOT NULL.

Policy mẫu (phác thảo, KHÔNG apply):
```
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;  -- (FORCE khi sẵn sàng)
CREATE POLICY database_scope_orders ON orders
  USING (database_id::text = current_setting('app.current_database_id', true));
```
> RLS là **defense cuối**, KHÔNG thay Guard. Guard (B2/B3) vẫn là lớp chặn chính. Đưa RLS vào sau cùng để không rủi ro hệ LIVE.

### B7. PHÂN BIỆT 2 KHÁI NIỆM (tránh nhầm — quan trọng)

- **"Quyền VÀO ổ"** (visibility): user thấy/chọn được ổ nào ở `DatabaseSelectGuard`. Hiện suy gián tiếp qua chi nhánh (A5). Bảng mới cho phép tường minh hóa (`accessLevel='none'` chặn hẳn, hoặc có dòng override = được vào).
- **"Quyền LÀM GÌ trong ổ"** (action permissions): full/readonly/custom — đây là phần mới chính.

Bảng `user_database_roles` giải quyết cả 2: có dòng (active, ≠none) → vào được + quyền theo resolver; `none` → chặn; không có dòng → fallback (vào được nếu có chi nhánh thuộc ổ, quyền = toàn cục).

---

## PHẦN C — LỘ TRÌNH PHASE (mỗi phase deploy ĐỘC LẬP, an toàn)

| Phase | Việc | Deploy độc lập? | Tác động hệ LIVE | Model gợi ý |
|---|---|---|---|---|
| **P1 — Schema** | Migration 0432 tạo `user_database_roles` (rỗng) + Drizzle schema + repository CRUD. Không guard nào đọc. | ✅ Bảng rỗng = no-op. | KHÔNG | Sonnet (BE/migration) |
| **P2 — Resolution + Guard** | `resolveEffectivePermissions()` + Redis cache + sửa `PermissionsGuard` đọc `req.databaseId` (fallback toàn cục nếu rỗng/lỗi). Endpoint `/auth/me/effective-permissions?databaseId=`. | ✅ Bảng vẫn rỗng → resolver luôn fallback → hành vi y cũ. | KHÔNG (fail-open) | **Opus review** + Sonnet code |
| **P3 — UI quản trị** | Màn "User × Ổ" + panel chi tiết + preview. Admin bắt đầu gán override THẬT. | ✅ Chỉ admin dùng; FE đọc resolver mới. | Bắt đầu có hiệu lực khi admin gán | Sonnet (FE) |
| **P3.5 — FE context** | `auth-context` thêm `currentDatabaseId` + re-fetch effective perms khi đổi ổ. Ẩn/hiện nút theo quyền-ổ. | ✅ Fallback nếu endpoint lỗi. | UI ẩn nút đúng theo ổ | Sonnet (FE) |
| **P4 — RLS database_id** | CHỈ khi RLS tenant bật thật (set session-var + DB role). Policy `database_id` + `set_config` per request. | ⚠️ Phụ thuộc tiền đề A9 — dự án riêng | Defense cuối | **Opus** (security) |

**Thứ tự bắt buộc:** P1 → P2 → P3 → P3.5 → (P4 sau, có điều kiện). P2 phải Opus review đối kháng trước commit (IDOR/cross-database/fallback).

---

## PHẦN D — RỦI RO BẢO MẬT + CÁCH TEST

### D1. Rủi ro & mitigation

| # | Rủi ro | Mức | Mitigation |
|---|---|---|---|
| R1 | **Cross-database leak**: user full ổ A đọc/sửa data ổ B qua đổi `X-Database-Id` | 🔴 | Guard resolve theo `databaseId` server-side; service PHẢI filter `databaseFilter()` (G4). KHÔNG tin client tự khai quyền. Test cross-db bắt buộc. |
| R2 | **Fail-open lockout ngược**: resolver lỗi → fallback toàn cục → user thấy nhiều hơn quyền-ổ | 🟠 | Fail-open CÓ CHỦ ĐÍCH (tránh khóa NV), NHƯNG `deniedPermissions` field nhạy cảm vẫn áp ở tầng service riêng (defense). Log mọi fallback để audit. |
| R3 | **Header spoof `X-Database-Id`**: user gửi ổ không có quyền | 🔴 | Resolver trả `[]`/`none` nếu user không có override hợp lệ + chi nhánh không thuộc ổ → guard từ chối. `databaseId` KHÔNG tự cấp quyền, chỉ là context. |
| R4 | **Stale permission**: admin thu hồi quyền nhưng JWT/cache còn | 🟠 | Cache TTL 60s + invalidate on write. JWT chỉ là fallback degrade, không phải nguồn chân lý cho quyền-ổ. |
| R5 | **Privilege escalation qua `extraPermissions`**: admin ổ tự cấp `*` | 🟠 | Chỉ user có `roles.edit`/`*` mới sửa `user_database_roles`. Audit log mọi thay đổi (Rule 01/15). Chặn tự-cấp `*` trừ super-admin. |
| R6 | **RLS dormant tạo cảm giác an toàn giả** (A9) | 🔴 | Tài liệu hóa rõ: lớp chặn THẬT = Guard. KHÔNG dựa RLS cho tới P4. |
| R7 | **Bùng nổ override khó quản** | 🟢 | `accessLevel` shortcut (full/readonly/none) giảm 90% nhu cầu custom từng quyền. |

### D2. Test bắt buộc (trước mỗi phase deploy)

1. **Backward-compat (P1/P2):** bảng `user_database_roles` rỗng → mọi user quyền y hệt trước (snapshot `/auth/me` trước/sau). Reload prod verify.
2. **Cross-database (P2):** user full ổ A, KHÔNG override ổ B, ổ B không có chi nhánh user → set `X-Database-Id=B` → guard từ chối mutation ổ B (403/404). Verify DB không đổi.
3. **Override replace (P2):** user role full toàn cục + override `readonly` ổ B → trong ổ B chỉ pass `*.view`, fail `*.edit/create/delete`. Trong ổ A vẫn full.
4. **Denied thắng (P2):** override `full` ổ A + `deniedPermissions=['products.edit_cost']` → ổ A pass mọi thứ TRỪ xem/sửa giá vốn.
5. **accessLevel=none (P2):** override `none` ổ C → user KHÔNG vào được ổ C dù có chi nhánh thuộc ổ C.
6. **Fail-open (P2):** kill Redis/DB resolver → degrade về quyền toàn cục, KHÔNG khóa NV (log fallback).
7. **Cache invalidation (P2):** admin đổi override → ≤60s áp dụng (hoặc tức thì nếu invalidate).
8. **UI ẩn nút theo ổ (P3.5):** đổi ổ → nút Sửa/Xóa ẩn/hiện đúng theo quyền-ổ; bấm nút không quyền → 403 rõ ràng, không vỡ trang.
9. **Multi-tenant (mọi phase):** override ổ tenant A KHÔNG ảnh hưởng tenant B; mọi query filter `tenantId`.

---

## PHẦN E — TASK DECOMPOSITION (cho worker tier thấp)

```
## P1 — Schema (sau khi Letri duyệt ADR)
- [ ] Sonnet: tạo apps/api/src/database/schema/user-database-roles.ts (Drizzle, theo B1)
- [ ] Sonnet: migration 0432_user_database_roles.sql + _down.sql (ADD COLUMN/TABLE IF NOT EXISTS)
- [ ] Sonnet: repository CRUD user_database_roles (filter tenant + soft-delete)
- [ ] Haiku: export type + thêm vào schema/index.ts

## P2 — Resolution + Guard (Opus review TRƯỚC commit)
- [ ] Sonnet: resolveEffectivePermissions(userId, databaseId) + Redis cache (key perm:eff:{u}:{db} TTL 60s)
- [ ] Sonnet: sửa PermissionsGuard đọc req.databaseId → resolver, fallback toàn cục nếu rỗng/lỗi
- [ ] Sonnet: endpoint GET /auth/me/effective-permissions?databaseId=
- [ ] Sonnet: invalidate cache khi sửa user_database_roles / roles / userRoles
- [ ] Opus: review đối kháng IDOR/cross-database/fallback + viết test cross-db
- [ ] Sonnet: integration test D2.1–D2.9

## P3 — UI quản trị
- [ ] Sonnet: trang /admin/permissions/database-access (bảng User × Ổ + badge accessLevel)
- [ ] Sonnet: panel chi tiết (accessLevel dropdown + extra/denied checkbox + preview)
- [ ] Haiku: i18n text vi

## P3.5 — FE context
- [ ] Sonnet: auth-context thêm currentDatabaseId + re-fetch effective perms khi đổi ổ
- [ ] Sonnet: rà nút Sửa/Xóa ẩn/hiện theo quyền-ổ (rule 20 — từng nút × 3 trục)

## P4 — RLS (CHỈ khi RLS tenant bật thật — dự án riêng)
- [ ] Opus: thiết kế bật RLS tenant (set_config per request + DB role) TRƯỚC
- [ ] Opus: policy database_id + test
```

---

## PHẦN F — CẦN CEO (LETRI) DUYỆT

1. **Mô hình override (B2):** "override = REPLACE quyền toàn cục cho ổ đó, extra = cộng thêm, denied = trừ" — đúng ý "full ổ A, chỉ xem ổ B"? (Khác với "cộng dồn".)
2. **accessLevel shortcut (B1):** dùng 5 mức (full/readonly/none/custom/mặc-định) — hợp lý cho NV vận hành không rành quyền lẻ?
3. **JWT cách 3a (B3):** guard re-resolve + Redis cache 60s (token KHÔNG đổi) — chấp nhận thêm 1 query cache/request?
4. **RLS (A9 + B6):** RLS hiện DORMANT (app chạy superuser, không set session-var). Bật RLS thật là dự án riêng — để P4 sau, KHÔNG gộp đợt này. Đồng ý?
5. **Bảng access-control mới** → có muốn review từng cột trước khi P1 apply không (theo luật bảng nhạy cảm)?

---

**END ADR — chỉ thiết kế, chưa code. Apply theo phase sau khi Letri duyệt PHẦN F.**
