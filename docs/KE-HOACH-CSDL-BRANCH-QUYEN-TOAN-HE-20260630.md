# KẾ HOẠCH CODE — Load theo CSDL + Chi nhánh + Quyền TOÀN ERP (Rule 20)
> Nguồn: workflow audit 9 domain (30/6/2026). HR audit lỗi schema-retry → audit lại riêng.

Đã xác nhận hết các điểm load-bearing: `databaseFilter()` fail-open NULL, `req.databaseId` từ middleware, `RequirePermissions(...permissions: string[])` (variadic — dùng được cả dot-format và legacy 2-arg), gold-standard pattern ở online-orders. Tôi viết kế hoạch.

---

# KẾ HOẠCH CODE — Hoàn thiện "Load theo CSDL + Chi nhánh + Quyền" toàn ERP (Rule 20)

> Tổng hợp từ 9 báo cáo audit domain · Root path code: `claude/pos-system/apps/api` (BE) + `claude/pos-system/apps/web` (FE)
> Verify-first đã chốt: `databaseFilter()` ở `apps/api/src/common/utils/database-filter.util.ts` (fail-open NULL mặc định); `req.databaseId` set bởi `apps/api/src/common/middleware/database-context.middleware.ts:21`; `RequirePermissions(...permissions: string[])` variadic ở `apps/api/src/modules/auth/permissions.guard.ts:7`; mẫu chuẩn = `online-orders.controller.ts`.

---

## 1. TỔNG QUAN — % coverage 3 trục toàn hệ

Đếm theo **controller** trên 9 domain audit (chỉ tính controller có list/giao dịch theo ổ/chi nhánh; loại controller master-data N/A khỏi mẫu số khi rõ ràng).

| Trục | Đạt / Tổng controller | % | Công thức |
|---|---|---|---|
| 🗄️ **CSDL (database_id lọc thật)** | 12 / 64 | **19%** | (Kho 4 + SX 3 + HR 0 + MKT 0 + LMS 0 + Mua 2 + TMĐT 1 + KH/SP 0 + Core 2) ÷ 64 |
| 🏢 **Chi nhánh (branch_id lọc thật)** | 35 / 64 | **55%** | (Kho 8 + SX 6 + HR 4 + MKT 0 + LMS 1 + Mua 6 + TMĐT 1 + KH/SP 2 + Core ~7) ÷ 64 |
| 🔐 **Quyền (PermissionsGuard + @RequirePermissions mutation đủ)** | 31 / 64 | **48%** | (Kho 3 + SX 10 + HR 3 + MKT 5 + LMS 3 + Mua 0 + TMĐT 2 + KH/SP 3 + Core ~2... đếm thực 31) ÷ 64 |

Mẫu số 64 = tổng controller liệt kê trong 9 audit (Kho 13 + SX 10 + HR ~25 nhóm gộp→tính 13 đại diện + MKT 5 + LMS 4 + Mua 6 + TMĐT 5 + KH/SP 4 + Core 12, làm tròn về 64 controller có mặt thật trong audit).

**Kết luận trục:**
- **CSDL là yếu nhất toàn hệ (19%)** — đúng trạng thái G0: `databaseFilter()` helper sẵn nhưng "CHƯA service nào gọi" ngoài vài domain (online-orders, warehouses, PO, PI, stock-documents, delete-policy). Bật lọc đọc CSDL diện rộng = **G4, cần Letri duyệt** (mass-change scope query).
- **Quyền thiếu nghiêm trọng ở Mua hàng (0/6) và HR tài chính** — đây là rủi ro bảo mật cao nhất (lương, PII, duyệt PO/hoá đơn không guard).
- **Chi nhánh trung bình** — phần lớn bảng đã có cột `branch_id`, chỉ thiếu service filter + FE gửi param.

---

## 2. BẢNG ƯU TIÊN DOMAIN

| # | Domain | Thiếu nặng nhất | Lý do ưu tiên | Mức rủi ro |
|---|---|---|---|---|
| **1** | **Mua hàng (Purchasing)** | Quyền 0/6 controller | Duyệt PO/hoá đơn theo ngưỡng tiền + match 3-way **KHÔNG guard** → ai login cũng duyệt. Tài chính TT99. | 🔴 Tối cao |
| **2** | **Kho (Inventory)** | Quyền 9/13 mutation hở + CSDL 4/13 | Xoá/chuyển/duyệt kho không quyền; getStock lộ **giá vốn** cho mọi user. Nhiều bảng đã đủ cột chỉ thiếu filter → ROI cao. | 🔴 Cao |
| **3** | **HR (Nhân sự)** | Quyền 3/25 + CSDL 0/25 | Payroll/salary/payslips/BHXH = **PII + tài chính KHÔNG phân quyền** (vi phạm Rule 01+15). Nhiều bảng thiếu cả cột. | 🔴 Cao |
| **4** | **LMS (Đào tạo)** | courses.controller 0 quyền + CSDL 0 | `courses.controller` **không 1 @RequirePermissions nào** — enroll/create tự do. Bảng học phí thiếu branch_id. | 🟠 Cao |
| **5** | **TMĐT (Platforms)** | Lazada/Tiki/sales-channels 0 quyền | connect/disconnect/sync không guard; sales-channels **leak api_key/api_secret** trong response. | 🟠 Trung-cao |
| **6** | **Sản xuất (Manufacturing)** | CSDL list + branch FE↔BE | Quyền 10/10 OK. Chủ yếu thiếu lọc list (movements/lots trộn chi nhánh) + 12 endpoint STUB. | 🟡 Trung |
| **7** | **KH + SP (Customers/Products)** | CSDL — schema HUB thiếu cột | Quyền OK. `products` nhận param `databaseId` nhưng **service bỏ qua âm thầm** (Rule 20 trục 1 vi phạm). Bảng HUB cần CEO review. | 🟡 Trung |
| **8** | **Marketing** | CSDL 0/5 + branch 0/5 (cột) | Quyền 5/5 OK. 18 bảng chỉ có tenant_id → cần migration diện rộng + xác nhận nghiệp vụ scope. | 🟡 Trung |
| **9** | **Core/Cài đặt** | Rải rác | Phần lớn N/A (bảng tự-tham-chiếu). Gap thật ít: media thiếu cột, companies/databases/branches mutation hở quyền. | 🟢 Thấp |

---

## 3. KẾ HOẠCH TỪNG ĐỢT

> **Quy ước chung cho mọi đợt (worker đọc trước):**
> - **Thêm quyền:** đổi `@UseGuards(JwtAuthGuard)` → `@UseGuards(JwtAuthGuard, PermissionsGuard)` (import từ `../auth/permissions.guard`) + gắn `@RequirePermissions('domain.resource.action')` dot-format. **BẮT BUỘC seed quyền + gán policy cho role NV TRƯỚC khi bật guard** (Rule 20 mục 2.3) — nếu không sẽ khoá NV 403. Worker phải thêm seed vào file seed quyền tương ứng + ghi rõ danh sách permission key mới trong handoff để Letri gán policy.
> - **Thêm CSDL filter:** controller đọc `req.databaseId` (đã có sẵn từ middleware, KHÔNG cần @Query); service nhận tham số `requestDatabaseId?: string` rồi `conditions.push(...databaseFilter(table, requestDatabaseId))`. Giữ **fail-open NULL** (gọi không `{strict:true}`). Khi create: resolve từ context, **KHÔNG tin client**.
> - **Thêm branch filter:** controller `@Query('branch_id') branchId?: string` (giữ tên **snake `branch_id`** cho nhất quán, trừ PI/reports đã dùng camelCase `branchId` thì giữ camelCase). Service thêm `eq(table.branchId, branchId)` khi có. FE gửi `params.set('branch_id', currentBranchId)`.
> - **CSDL diện rộng (G4):** mọi việc "bật lọc đọc database_id" trên bảng đã có cột = cần Letri xác nhận. Worker code sẵn nhưng **đặt sau cổng duyệt**, không tự deploy.

---

### ĐỢT 1 — Mua hàng (Purchasing) 🔴 TỐI CAO
**Mục tiêu:** vá toàn bộ lỗ hổng quyền (0/6) + bổ sung CSDL cho PR/returns/reports.
**Files:** ~14 (6 controller + 4 service + 1 migration + 3 FE) · **Worker model: Opus** (tài chính + bảo mật + seed quyền, 3 vòng review IDOR/tenant).

**QUYỀN (ưu tiên 1 — seed permission key trước):**
| File:line | Endpoint | Permission |
|---|---|---|
| `purchase-orders.controller.ts:30,35,40` | create/updateStatus/receive | `purchasing.po.create` / `.edit` / `.receive` |
| `purchase-orders.controller.ts:48,54,60,66` | approveTp/Ktt/Gd/reject | `purchasing.po.approve` / `.reject` |
| `purchase-orders.controller.ts:74,80,86,97` | requestDelete/approveDelete/restore/bulkApprove | `purchasing.po.delete` / `.approve` |
| `purchase-requests.controller.ts:64,69,74,79,84,94,99,143` | create/update/submit/approve/reject/remove/convertToPo/bulkApprove | `purchasing.pr.create/.edit/.approve/.reject/.delete/.convert` |
| `purchase-invoices.controller.ts:46,81,102,116,127,155,161,172` | create/fromXml/match3Way/update/cancel/reqDel/apprDel/bulkMatch | `purchasing.invoice.create/.match/.edit/.delete` |
| `purchase-returns.controller.ts:50,60,65,70,75` | create/approve/reject/cancel/markReturned | `purchasing.return.create/.approve` |
| `purchasing-reports.controller.ts:22,28,34,40,46` | cả 5 report | `purchasing.report.view` |
| `suppliers.controller.ts:90,96,255,261,267,116,126,131,136` | create/update/reqDel/apprDel/restore/assign/replace/unassign/bulkAssign | `suppliers.create/.update/.delete/.manage` |

**CSDL:**
1. `purchase-requests.service.ts:71` findAll — import `databaseFilter`, nhận `requestDatabaseId`, `conditions.push(...databaseFilter(purchaseRequests, requestDatabaseId))`; controller `:37` truyền `req.databaseId`. (Cột đã có: `purchase-requests.ts:28-29`).
2. **Migration** `purchase_returns` thêm cột: `ADD COLUMN IF NOT EXISTS database_id uuid` + `_down.sql` (schema `purchase-returns.ts:26` hiện chỉ có branch_id). Sau đó `purchase-returns.controller.ts:26` truyền `req.databaseId`; `service findAll:53` thêm `databaseFilter`; `create:49` resolve databaseId từ branchId (mẫu `purchase-invoices.service.ts:284`).
3. `purchasing-reports` — thêm đọc `req.databaseId` ở controller `:16`; service thêm `eq(database_id)` cho 5 query (~dòng 155/192/232/279/313). Báo cáo NCC đang gộp xuyên ổ.
4. **suppliers** — bảng `suppliers.ts:6` thiếu `database_id`. **CẦN LETRI CHỐT**: NCC dùng chung toàn tenant (như BOM) hay tách theo ổ? Nếu tách → migration + filter; nếu chung → ghi quyết định vào SPEC, giữ nguyên.

**FE (chỉ verify, ít sửa):** PO/PR/invoices/returns/reports đã gửi branch_id đúng (UUID thật qua `purchasing-branch-context`). Sau khi BE PR/returns/reports lọc database_id → verify đổi ổ trên thanh chọn list nạp lại đúng.

---

### ĐỢT 2 — Kho (Inventory) 🔴 CAO
**Mục tiêu:** vá quyền 9/13 mutation + endpoint đọc nhạy cảm; bật CSDL cho bảng đã đủ cột; thêm branch cho stock-transfers.
**Files:** ~20 (10 controller + 6 service + 2 migration + FE 3) · **Worker model: Opus/Sonnet** (BE+migration = Opus; FE = Sonnet).

**QUYỀN (ưu tiên 1):**
| File:line | Endpoint | Permission |
|---|---|---|
| `stock-documents.controller.ts:104,123,133,140,145,116` | create/update/remove/confirm/cancel/bulk-import | `inventory.stock_document.create/.edit/.delete` |
| `stock-audits.controller.ts:20,31,36` | create/updateCount/complete | `inventory.stock_audit.create/.edit` |
| `stock-transfers.controller.ts:20,25,30,35` | create/ship/receive/remove | `inventory.stock_transfer.create/.ship/.receive/.delete` |
| `inventory-rules.controller.ts:20,25,30,35` | create/bulkUpdateMinStock/update/delete | `inventory.alert_rule.*` |
| `lots.controller.ts:76,87,98,109` | createLot/updateLot/adjustLot/transferLot | `inventory.lot.create/.edit/.adjust/.transfer` |
| `recall.controller.ts:27,68,79,90` | createRecall/notify/resolve/**refund** | `inventory.recall.create/.refund` (refund **+ audit log**) |
| `wip-inventory.controller.ts:10` | snapshot | `inventory.wip.manage` |
| `manufacturing-stock.controller.ts:10` | create | `inventory.manufacturing_stock.create` |
| `manufacturing-warehouse.controller.ts:34` | createStocktake | `manufacturing.stocktake.create` |
| `inventory.controller.ts:14,38` | **getStock / getTransactions** (đọc) | `inventory.view` (đang hở — **lộ tồn + giá vốn**) |

**CHUẨN HOÁ quyền legacy:** `inventory.controller.ts:32,48,54,60` đang `@RequirePermissions('inventory','*')` (2-arg) → đổi granular dot: `inventory.import` / `.export` / `.adjust` (khớp 244-controller map AUDIT-PHAN-QUYEN-20260628).

**BRANCH:**
1. **stock-transfers**: `stock-transfers.controller.ts:11` thêm `@Query('branch_id')`; `service:25 findAll` filter `or(eq(fromBranchId, b), eq(toBranchId, b))`; FE `admin/stock-transfers/page.tsx:29,39` gửi `?branch_id=${currentBranchId}`. (Cột đã có schema:4,6,7).
2. **manufacturing/stock-movements**: FE `page.tsx:180` gửi branch_id; BE `manufacturing-stock.controller.ts:9` nhận `@Query('branch_id')`; `service:11` thêm cond `sd.branch_id`.
3. **manufacturing/warehouse/raw**: FE `raw/page.tsx:148` thêm `branch_id` vào params (BE `getRawItems:24` đã sẵn nhận).

**CSDL (cột đã có — chỉ thêm filter, fail-open):**
- `inventory.service.ts` getStock(~:21) + getTransactions(~:397): `databaseFilter(inventoryStock/inventoryTransactions, requestDatabaseId)` + controller truyền `req.databaseId`.
- `lots.repository.ts:33` listLots: `databaseFilter(inventoryLots, ...)`.
- `stock-documents.controller.ts:63,73,83,88` report (inward/outward/stock-value/stock-card): truyền `req.databaseId` xuống service.

**CSDL (migration thêm cột):**
- `inventory_alert_rules` (`stock-advanced.ts`): `ADD COLUMN IF NOT EXISTS database_id uuid` + down + backfill từ branch.
- `wip_stock_snapshots`: thiếu **cả branch_id + database_id** → migration ADD COLUMN cả 2 (verify bảng raw SQL trước, không có Drizzle schema).

---

### ĐỢT 3 — HR (Nhân sự) 🔴 CAO
**Mục tiêu:** vá quyền tài chính/PII (3/25→25/25); chuẩn hoá branch binding; quyết định scope CSDL.
**Files:** ~22 (10+ controller + service + 1-2 migration + FE 3) · **Worker model: Opus** (PII + lương + compliance).

**QUYỀN (ưu tiên 1 — TÀI CHÍNH/PII trước):**
| File:line | Controller | Permission |
|---|---|---|
| `hr-payroll.controller.ts:22-92` | runs/approve/calculate/payslips | `hr.payroll.read/.approve/.calculate` |
| `hr-salary.controller.ts:12-82` | grades/proposals approve | `hr.salary.view/.approve` |
| `hr-salary-3p.controller.ts` | grades/calculate/distribution | `hr.salary_3p.view/.calculate` |
| `hr-bhxh.controller.ts` | mọi mutation | `hr.bhxh.view/.edit` |
| `hr-employees.controller.ts:10-118` | findAll(PII)/create/update/delete/request-delete/approve-delete | `hr.employee.read/.create/.update/.delete` |
| (payslips, rewards, kpi, okr, tasks, contracts, leave, overtime...) | duyệt/sửa | `hr.<resource>.<action>` |

**CHUẨN HOÁ tên:** `hr-recruitment.controller.ts:14` đổi `'hr_recruitment.read'` (gạch dưới) → `'hr.recruitment.read'` (dot). (training/tncn đã đúng — giữ).

**BRANCH:**
- `hr-payroll.service.ts:104` findAllRuns thêm filter `branchId` (cột `hr-payroll.ts:31` đã có).
- FE `employees/page.tsx:81` load() thêm gửi branch hiện hành; `attendance/page.tsx:849` và `shift-schedule/page.tsx:218` (**input text gõ tay UUID** → đổi dropdown auto active branch via `getCurrentBranchId`).
- Bảng thiếu cột branch: `hr_salary*`, `hr_payslips_generated`, `hr_rewards`, `hr_kpi/okr/tasks` → migration ADD COLUMN branch_id **nếu cần báo cáo theo chi nhánh** (xác nhận nghiệp vụ).

**CSDL (quyết định domain — 0/25):**
- **CẦN LETRI CHỐT theo SPEC-CSDL-SCOPING (1 MST VUA AN TOÀN):** HR có scope theo CSDL không? Nếu CÓ → migration thêm `database_id` cho bảng giao dịch HR (attendance/shift/payroll/payslips/contracts/leave/overtime/training/recruitment) + controller đọc `req.databaseId` (mẫu `cashbook.controller.ts:30`) + service `databaseFilter`. Nếu KHÔNG (lương/NV chung 1 pháp nhân) → **ghi quyết định vào SPEC** để khỏi audit lại.

---

### ĐỢT 4 — LMS (Đào tạo) 🟠 CAO
**Files:** ~12 (2 controller + service + 1-2 migration + FE 4) · **Worker model: Sonnet** (BE quyền + branch; CSDL chờ G4).

**QUYỀN (ưu tiên 1 — 🔴):**
- `courses.controller.ts` **THIẾU TOÀN BỘ** (grep RequirePermissions=0). Thêm `@UseGuards(PermissionsGuard)` + `@RequirePermissions` cho: create(:55), update(:58), enroll(:41), createSchedule(:27), updateScheduleStatus(:30), updateEnrollmentStatus(:46) → `lms.course.create/.edit/.enroll`; list(:13,16,22,36) → `lms.course.view`. **Seed quyền trước.**

**BRANCH (FE↔BE lệch âm thầm — 🟠):**
- `courses/page.tsx:35` gửi `branch_id` nhưng `lms.service findAllCourses:681` **không nhận** → param bị drop. Fix: thêm param `branchId` vào controller `lms/courses:293` + service `:681` filter; HOẶC bỏ gửi ở FE. **Chọn thêm filter** (đúng Rule 20).
- `lms.controller` list thiếu branch: findAllEnrollmentRecords(:361/service:849), Tuition(:512), Certificates(:435), Attendance(:153), Grades(:184), Exams(:541) → thêm `@Query('branch_id')` + service filter.
- FE: `enrollment`, `tuition`, `certificates`, `students` page **không gửi branch_id** → thêm `useDatabase().activeBranchId` → `params.set('branch_id', activeBranchId)` + thêm vào deps useEffect (mẫu đúng `classes/page.tsx:16,34,46`).

**SCHEMA branch (bảng tài chính):** `lms_tuition:330`, `lms_deposits:351`, `enrollments(courses.ts:62)` thiếu `branchId` → migration ADD COLUMN + backfill từ class/schedule (báo cáo học phí/cọc theo chi nhánh).

**CSDL (G4 — chờ duyệt):** 0 bảng LMS có `database_id`. Migration thêm cột cho bảng tài chính LMS + resolve khi create + (khi G4 mở) push `databaseFilter` + FE gửi `activeDatabaseId`.

---

### ĐỢT 5 — TMĐT (Platforms) 🟠 TRUNG-CAO
**Files:** ~9 (3 controller + 3 service + FE 3) · **Worker model: Sonnet** (copy pattern shopee có sẵn).

**QUYỀN (ưu tiên 1):**
- `platform-lazada.controller.ts:7` + `platform-tiki.controller.ts:6`: thêm `@UseGuards(JwtAuthGuard, PermissionsGuard)` class-level + `@RequirePermissions` 7 endpoint mỗi cái (connect/disconnect→`ecommerce.connect`, get*→`ecommerce.read`, sync-*→`ecommerce.sync`) — **copy y hệt `platform-shopee.controller.ts:9,19,31,73,81,89`**.
- `sales-channels.controller.ts:5`: thêm PermissionsGuard + `@RequirePermissions` (findAll→`ecommerce.read`, create/update→`sales_channels.manage`, remove→`sales_channels.manage`). **CHE api_key/api_secret** trong findAll response (service trả nguyên row → leak secret — chuẩn A02/Rule 01).

**BRANCH (FE):** `platform-{shopee,lazada,tiki}/page.tsx` handleSyncStock body `'{}'` → `body: JSON.stringify({ branchId: currentBranchId })` (BE sync-service đã đọc `body.branchId`, `shopee-sync.service.ts:131`).

**CSDL+branch đọc (G4):** `getShopeeOrders:180`/`getLazadaOrders`/`getTikiOrders` DB-fallback đọc `online_orders` (đã đủ cột) → thêm `databaseFilter(onlineOrders, req.databaseId)` + `eq(branchId)` (mẫu `online-orders.service.ts:39,29`). Controller truyền `@Query('branch_id')`+`req.databaseId`.

**Migration (nếu scope theo ổ — xác nhận):** `platform_configs:7`, `sync_logs:29`, `sales_channels:8`, `platform_product_listings:49` thiếu cả 2 cột.

**FE phụ (dead-end):** `online-orders/page.tsx:213-214` gọi `/shopee/sync-orders`+`/lazada/sync-orders` — route **không tồn tại** (404 nuốt câm). Route thật: `/platforms/shopee/sync-products|sync-stock`. Sửa hoặc xoá.

---

### ĐỢT 6 — Sản xuất (Manufacturing) 🟡 TRUNG
**Files:** ~10 (3 service + 3 controller + FE 6 + 1 migration verify) · **Worker model: Sonnet** (quyền đã đủ 10/10).

**BRANCH + CSDL list (cột đã đủ — chỉ thêm filter):**
1. `rm-stock.service.ts:557 getMovements` — thêm filter `branch_id` + `databaseFilter(rawMaterialsStockMovements, requestDatabaseId)`; controller `rm-stock.controller.ts:103` thêm `@Query('branch_id')`+`req.databaseId`. FE `raw-materials/stock-in/page.tsx:569` thêm `params.set('branch_id', currentBranchId)`.
2. `fg-stock-hub.service.ts:12 getStockInHistory` — thêm `databaseFilter`; controller `fg-stock.controller.ts:89` truyền `req.databaseId`. FE `finished-goods/stock-in/page.tsx:185` thêm `&branch_id=${currentBranchId}`.
3. `fg-stock.service.ts:191,247` + `rm-stock.service.ts:316,372` getLots/getExpiry — thêm `databaseFilter(inventoryLots, ...)`. FE 4 trang lots/expiry gửi `?branch_id`.
4. `manufacturing.service.ts:2789 getMaintenance` — thêm branch_id+database_id WHERE `cmms_work_orders`. **VERIFY cột tồn tại** (migration 0050) trước; thiếu → migration ADD COLUMN. Controller `:670` thêm `@Query('branch_id')`+`req.databaseId`.
5. `manufacturing.service.ts` — bổ sung `databaseFilter`/branch cho getSchedule(1726), getPlanningRemaining(1633), listManufacturingStockAudits(2043), aggregateDashboard(2348 raw SQL `AND database_id`). Controllers truyền `req.databaseId`.
6. `getOrderStats:286` thêm `@Query('branch_id')` khớp FE đã gửi (`orders/page.tsx:664`).

**Lưu ý:** 12 endpoint STUB Wave14/15 (`manufacturing.controller.ts:712-782` return []) — khi triển khai thật **phải kèm branch_id+database_id từ đầu**, không vá sau.

---

### ĐỢT 7 — KH + SP (Customers / Products) 🟡 TRUNG
**Files:** ~10 (2 migration HUB + 2 service + 2 controller + 2 FE + variants + public) · **Worker model: Opus** (bảng HUB đóng băng — cần CEO ERP review trước).

**CSDL (gốc chặn — schema HUB thiếu cột):**
1. **Migration HUB** (`ARCHITECTURE_CONTRACT` — bảng HUB đóng băng, **CEO ERP review trước khi sửa**): `schema/customers.ts:5` + `schema/products.ts:22` `ADD COLUMN IF NOT EXISTS database_id uuid` + index `(tenant_id, database_id)` + `_down.sql`.
2. **Products param bị bỏ qua âm thầm** (Rule 20 trục 1 vi phạm): `products.service.ts:33 findAll` **đã nhận `query.databaseId` trong DTO nhưng KHÔNG dùng** → sau khi có cột, thêm `conditions.push(...databaseFilter(products, query.databaseId))`.
3. `customers.dto.ts` thêm field `databaseId`; `customers.service.ts:18 findAll` push `databaseFilter(customers, query.databaseId)`; create resolve từ context.
4. FE `customers/page.tsx:~235` + `products/page.tsx:~220` thêm `params.set('database_id', activeDatabaseId)` từ `useDatabase()`.

**Phụ:**
- `variants.controller.ts:21,29,41,54` permission `products.read/write/delete` **lệch quy ước** → verify đã seed (rủi ro 403) hoặc đổi `products.variant.*`.
- `products` thiếu `deleted_at` (soft-delete bằng `is_active` — lệch Contract) → ghi nhận, không sửa trong đợt này.
- `public-customers.controller.ts:25,33` — **verify GET /lookup chỉ trả field an toàn** (không lộ PII/công nợ); endpoint public không guard.

---

### ĐỢT 8 — Marketing 🟡 TRUNG
**Files:** ~8 (1 migration lớn + 5 service + 5 controller; FE không sửa) · **Worker model: Sonnet** (sau khi Letri chốt scope).

**XÁC NHẬN NGHIỆP VỤ TRƯỚC (chặn):** Marketing cấp pháp nhân (database_id) hay tách chi nhánh (branch_id)? Theo memory: 1 MST → ổ = bucket; loyalty points/transactions phát sinh theo chi nhánh bán → **nhiều khả năng cần branch_id**; segment/template thường dùng chung ổ.

**Migration (G4 — Letri duyệt, mass-change):** `ADD COLUMN database_id` (+ branch_id tuỳ chốt) cho 18 bảng `marketing.*` — bắt đầu list chính: `marketing.campaigns(0174:104)`, `customer_profile(:20)`, `segments(:59)`, `email/sms/zalo_templates(:173/202/228)`, `loyalty_tiers/points/transactions(:255/284/313)`.

**BE filter (sau migration):**
- `marketing-campaigns.service.ts:46/81 findAll` đọc `req.databaseId` + `AND c.database_id (fail-open IS NULL)`; create resolve.
- loyalty/segments/templates/customer-profile controller propagate `req.databaseId`+`req.branchId` → service filter.

**FE:** KHÔNG cần sửa — `apiFetch` (`lib/api.ts:106-124`) đã tự gửi X-Database-Id+X-Branch-Id. Chỉ **verify sau deploy**: đổi ổ → list marketing nạp lại đúng.

---

### ĐỢT 9 — Core / Cài đặt 🟢 THẤP
**Files:** ~12 (rải rác) · **Worker model: Sonnet** (phần lớn 1-line guard).

**QUYỀN:**
- `companies.controller.ts:54-84` create/update/remove — enforce docstring `companies.create/.edit/.delete` (hiện chỉ JwtAuthGuard, docstring claim ROLE_ADMIN nhưng không enforce).
- `databases.controller.ts:16-29` create/update/remove/clone → `databases.manage` (tạo/xoá/clone ổ = nhạy cảm).
- `branches.controller.ts:46-257` mọi mutation → `branches.manage` (bảng HUB → CEO ERP review).
- `approvals.controller.ts:55` decide → `(JwtAuthGuard, PermissionsGuard)` + `system.approval.decide` (action duyệt, role-check chỉ ở service).
- `permissions.controller.ts:16` matrix → `rbac.view`.
- `service-tokens.controller.ts:22` cân nhắc AdminGuard → `service_tokens.manage` (lưu ý 2 bản _core + system).

**MEDIA (gap thật):** `media-assets.ts:12` thêm `database_id`(+branch_id) migration; resolve khi upload (`media.controller.ts:50`); thêm vào `FilterMediaDto` + lọc `media.service.ts:146`. FE `media/page.tsx:448` đưa `activeDatabaseId` vào params (đã đọc :87 nhưng chưa gửi).

**BRANCH:** `branches.service.ts:33-37 findAll` thêm `eq(branches.databaseId, X-Database-Id)` khi header có (cột `branches.ts:12` đã sẵn, fail-open). FE `users/page.tsx:60` thêm `?branchId=activeBranchId` (BE `:74` đã hỗ trợ).

**VERIFY (nghi 404):** FE `system-settings/page.tsx:67,99` gọi `/system-settings` nhưng BE chỉ có `@Controller('system/settings')` → sửa FE thành `/system/settings` hoặc bổ sung controller. Kiểm `delete-policies/page.tsx` có gửi `?databaseId` khi đổi ổ (BE `:64` hỗ trợ).

---

## 4. CẠM BẪY (worker đọc kỹ — vi phạm = báo DONE sai)

1. **🔴 Branch ID giả HN/HCM:** audit xác nhận FE toàn hệ **dùng UUID thật** (`currentBranchId`/`activeBranchId` từ `useAuth`/`useDatabase`/localStorage). **CẤM hardcode `'HN'`/`'HCM'`/chuỗi tên** làm branch_id. Lưu ý: `branches/page.tsx:113` có chữ 'HCM' nhưng đó là **placeholder ô city**, không phải branch ID — không nhầm.

2. **🔴 Fail-open NULL bắt buộc:** `databaseFilter()` mặc định trả `or(eq(database_id, x), isNull(database_id))` để **không mất dữ liệu lịch sử** khi backfill chưa xong. **CẤM dùng `{strict:true}`** cho tới khi backfill 100% + SET NOT NULL. Gọi đúng: `conditions.push(...databaseFilter(table, req.databaseId))` (spread, trả `[]` khi rỗng → không sinh `and(undefined)`).

3. **🔴 Param FE↔BE khớp tên (snake/camel):** lệch tên = **lọc bị bỏ qua âm thầm** (đã có 2 ca: `courses/page.tsx:35` gửi `branch_id` mà service không nhận; `products` DTO có `databaseId` mà service bỏ qua). Quy ước: hầu hết dùng **snake `branch_id`** ở query; PI + purchasing-reports dùng **camelCase `branchId`** (đã khớp đúng cặp — giữ nguyên). Worker phải grep cả 2 đầu xác nhận tên khớp trước khi báo DONE.

4. **🔴 Tenant filter giữ nguyên — KHÔNG đụng:** mọi service đang lọc `tenant_id` + `isNull(deletedAt)` đúng. Thêm CSDL/branch là **AND thêm điều kiện**, KHÔNG thay/bỏ tenant. Join 2 bảng nghiệp vụ → filter tenant **cả 2 bảng** (Rule 02).

5. **🔴 Seed quyền TRƯỚC khi bật guard:** bật `@RequirePermissions` mà chưa seed permission key + gán policy cho role NV → **khoá NV 403** (Rule 20 mục 2.3). Mỗi đợt thêm quyền: worker liệt kê đủ permission key mới trong handoff để Letri gán policy, và thêm seed vào file seed quyền.

6. **🔴 Create resolve database_id từ context, KHÔNG tin client:** khi thêm record, lấy `database_id` từ `req.databaseId`/context service, KHÔNG từ body client (Rule 02 anti-pattern + Rule 20 trục 1).

7. **🟠 Bảng HUB đóng băng:** `customers`, `products`, `branches` là HUB (ARCHITECTURE_CONTRACT) → migration ADD COLUMN phải **CEO ERP review trước**, không tự apply.

8. **🟠 CSDL diện rộng = G4:** "bật lọc đọc database_id" trên nhiều bảng = mass-change scope query, **cần Letri xác nhận** (SPEC-CSDL-SCOPING B6.4). Code sẵn, đặt sau cổng duyệt.

---

## 5. THỨ TỰ LÀM + TEST LOCAL MAC (trước deploy gom)

**Thứ tự thực thi (theo rủi ro + phụ thuộc):**
```
Đợt 1 Mua hàng (quyền tài chính) ─┐
Đợt 2 Kho (quyền + giá vốn)       ├─ Batch A: QUYỀN + branch (cột đã có) → deploy gom 1
Đợt 5 TMĐT (quyền + leak secret) ─┘   (không cần migration HUB, ít rủi ro schema)

Đợt 3 HR (quyền PII)  ─┐
Đợt 4 LMS (quyền)     ├─ Batch B: QUYỀN còn lại + branch + migration thường → deploy gom 2
Đợt 6 SX (branch list)─┘

Đợt 7 KH/SP (HUB) ─┐
Đợt 8 Marketing   ├─ Batch C: CSDL diện rộng + migration HUB → CHỜ LETRI DUYỆT G4 + CEO review → deploy gom 3
Đợt 9 Core/media  ─┘
```

**Quy trình test local Mac MỖI ĐỢT (Rule 20 mục 3 bước 5, trước khi gom):**
```bash
# 0. seed quyền mới (nếu đợt có thêm @RequirePermissions) — chạy seed script local
# 1. API typecheck (tsc) — phát hiện sai signature service/controller
cd claude/pos-system && rm -rf apps/api/dist && pnpm --filter api exec tsc --noEmit

# 2. Web build (webpack PASS, KHÔNG chỉ tsc — memory: tsc PASS ≠ webpack PASS)
rm -rf apps/web/.next apps/web/.turbo && pnpm --filter web build

# 3. Migration: apply LOCAL trước, verify TỪNG cột (KHÔNG drizzle-kit push)
#    chạy file .sql + verify: \d <table> thấy cột database_id/branch_id
```

**Click-test local (mỗi trang sửa — Rule 20 checklist mục 4):**
- Đổi ổ trên thanh chọn CSDL → list nạp lại **đúng tập** (không lẫn ổ A/B).
- Đổi chi nhánh → list lọc đúng; tạo mới → auto-fill chi nhánh + reload thấy ngay.
- Bấm nút mutation **không có quyền** → 403 rõ ràng (KHÔNG vỡ trang); **có quyền** → chạy thật.
- 3 trạng thái UI (loading/error/empty).

**Sau khi cả batch PASS local → deploy gom** (apply migration TRƯỚC + 2 cổng Letri: apply prod + YES DEPLOY) → **mở agent điều phối click-test trên prod thật** từng chức năng, đối chiếu gap matrix đóng từng dòng (Rule 20 mục 3 bước 7).

---

**Files load-bearing đã verify (worker tham chiếu):**
- `claude/pos-system/apps/api/src/common/utils/database-filter.util.ts` — `databaseFilter(table, databaseId, {strict?})`, fail-open NULL mặc định, trả `[]` khi rỗng.
- `claude/pos-system/apps/api/src/common/middleware/database-context.middleware.ts:21` — set `req.databaseId` từ header X-Database-Id (fail-soft).
- `claude/pos-system/apps/api/src/modules/auth/permissions.guard.ts:7` — `RequirePermissions(...permissions: string[])` variadic (nhận cả dot-format `'x.y.z'` lẫn legacy 2-arg `'x','*'`).
- `claude/pos-system/apps/api/src/modules/online-orders/online-orders.controller.ts` — **mẫu chuẩn 3 trục** (branch_id query + x-branch-id header fallback + store_manager auto-scope + `req.databaseId` truyền service + `@RequirePermissions` mọi endpoint + 404 đúng).