# 🗄️ KẾ HOẠCH HOÀN THIỆN CSDL SCOPING TOÀN HỆ — chờ Letri DUYỆT

> Soạn: Opus CEO ERP · ⏰ 2026-06-29 08:00 +0700
> Nền: `docs/SPEC-CSDL-SCOPING-20260621.md` (thiết kế + 5 quyết định D1-D5 chốt 22/6) — bản này là **LỘ TRÌNH THỰC THI cập nhật theo ground-truth 29/6**.
> Ground-truth: query DB prod (port 5433) + workflow audit 5 domain (236 findings, **121 GAP**, mọi finding có file:method).
> 🔴 Mục tiêu Letri: (a) **BẮT BUỘC chọn sổ khi đăng nhập** + (b) **NHỚ lựa chọn lần sau** + (c) **mọi trang/nút load đúng theo CSDL** + (d) **không thiếu sót**.

---

## 1. HIỆN TRẠNG (đã verify thật)

| Tầng | Trạng thái | Bằng chứng |
|---|---|---|
| G0 — Middleware đọc `X-Database-Id`→`req.databaseId` | ✅ XONG | `app.module.ts:537` + `common/middleware/database-context.middleware.ts` (fail-soft) |
| FE gửi header mọi request | ✅ XONG | `apps/web/src/lib/api.ts:123` (đọc localStorage `erp_active_database`) |
| G1 — Cột `database_id` trên bảng | ✅ ~XONG | **55/446 bảng** có cột (tài chính/thương mại/kho/SX) |
| G2 — GHI `database_id` khi tạo mới | ⚠️ **DỞ (27 method thiếu)** | orders/cashbook ghi đúng (0% NULL); kho/SX/trả hàng/journal/shipments/payments **chưa ghi** (NULL 100%) |
| G3 — Backfill dữ liệu cũ | ⚠️ DỞ nhưng NHẸ | NULL: inventory_transactions 29, returns 9, manufacturing_orders 6, stock_documents 4, stock_audits 3, quotes 2, purchase_orders 1 (tổng <60 dòng) |
| G4 — LỌC khi đọc/báo cáo | ⚠️ **DỞ (90 method thiếu)** | 22 service có dùng `databaseFilter` nhưng KHÔNG đủ method (vd orders.findAll/findOne/getStats vẫn thiếu, finance-aggregates 11 BCTC thiếu) |
| Cổng bắt buộc chọn CSDL | ❌ **THIẾU** | `layout.tsx:195` chỉ **banner mềm** có nút ✕ dismiss → qua được mà không chọn |
| Nhớ lựa chọn (server) | ❌ **HỞ** | FE chỉ đọc localStorage; `getProfile`/login KHÔNG trả `default_database_id` → máy mới = "Tất Cả" |
| Re-fetch khi đổi CSDL | ⚠️ DỞ | nhiều trang dashboard thiếu `activeDatabaseId` trong `useEffect` deps → đổi ổ không nạp lại |
| G7 — TaxDeclarationsService + ReconciliationService | ❌ **CHƯA CÓ** | `find` không thấy file service (bảng có sẵn) |

**Bối cảnh (quyết định 22/6):** 1 MST (0313334177) → 5 "ổ" = bucket nội bộ (quản trị vs thuế). Chỉ "VUA AN TOÀN 2026" active. **Dữ liệu rất ít → hoàn thiện toàn bộ rủi ro THẤP.** BOM/master KHÔNG scope. `journal_entries_v3` giữ `scope_id` + thêm lọc `database_id` (D3-A).

---

## 2. NGUYÊN TẮC RÀNG BUỘC AGENT (🔴 BẮT BUỘC — mọi agent đọc trước khi code)

> Copy nguyên khối này vào ĐẦU mỗi brief agent. Vi phạm = reject, không commit.

1. **READ (lọc):** mọi method LIST/REPORT/FINDONE thêm `databaseFilter(table, req.databaseId)` vào mảng `conditions` (helper `common/utils/database-filter.util.ts`). **Fail-open** giai đoạn này: `dbId` rỗng → `[]` (không thêm điều kiện); else `or(eq(database_id,dbId), isNull(database_id))`. KHÔNG đổi sang strict cho tới đợt NOT NULL.
2. **WRITE (ghi):** mọi method CREATE/INSERT set `database_id = resolvedDatabaseId` lấy từ `DatabaseContextService.resolve(req.databaseId, branchId)` — **TUYỆT ĐỐI KHÔNG tin `dto.databaseId` từ client** (rule 02). Bảng con (items/lines) kế thừa từ cha.
3. **Truyền context:** controller method phải lấy `req.databaseId` (qua `@Req()` hoặc decorator `@DatabaseId()` nếu có) và truyền xuống service. KHÔNG đọc lại header trong service.
4. **KHÔNG đụng master:** `products, customers, suppliers, categories, units, tax_rates, accounting_accounts, cashbook_reasons, expense_items` — chỉ lọc `tenant_id`. **BOM/định mức (manufacturing master) = DÙNG CHUNG, KHÔNG scope** (D2). `manufacturing_orders` (đơn SX) = CÓ scope.
5. **Sổ cái:** `journal_entries_v3` GIỮ `scopeId` (cây tổ chức) + THÊM lọc `database_id` (cách ly ổ) — 2 trục độc lập (D3-A). `journal_lines` lọc qua JOIN cha hoặc cột denorm nếu đọc thẳng.
6. **Tenant + soft-delete:** giữ NGUYÊN `tenant_id` filter + `isNull(deletedAt)` — chỉ THÊM databaseFilter, không bỏ điều kiện cũ.
7. **Shape API:** KHÔNG đổi shape response (giữ `{data,meta}` hoặc mảng phẳng hiện có) — đổi shape = vỡ FE (bài học 27/6).
8. **Test cách ly:** mỗi service sửa phải có ≥1 test "ổ A không thấy chứng từ ổ B" (404, không 403) — mẫu B7 trong SPEC.
9. **Build sạch:** `rm -rf .next .turbo` rồi `pnpm --filter api build` + `pnpm --filter web build` PASS trước báo DONE (tsc PASS ≠ webpack PASS).
10. **Cạm bẫy:** `req.user.sub` (không phải .id); `db.execute()`→`.rows`; route `:id` SAU route literal; KHÔNG @RequirePermissions key chưa seed; reviewer Sonnet soát đối kháng (IDOR/tenant/database) TRƯỚC commit.

---

## 3. KẾ HOẠCH THEO ĐỢT (có CỔNG KIỂM mỗi đợt)

### 🥇 ĐỢT 1 — FE: CỔNG CỨNG + NHỚ LỰA CHỌN (Letri ưu tiên #1) — 1 agent FE (Sonnet) + reviewer
**Mục tiêu:** đăng nhập = bắt buộc chọn CSDL, nhớ lần sau, đổi ổ là nạp lại.
- **1.1 Cổng cứng (Hard Guard Modal):** thay banner mềm `layout.tsx:195` bằng **modal CHẶN** — chưa chọn CSDL thì không vào được app (trừ trang `/admin/databases` để chọn + super-admin được chọn "Tất Cả"). Bỏ nút ✕ dismiss. Thiết kế sẵn trong SPEC mục "Hard Guard" (dòng 410-650).
- **1.2 Nhớ qua server:** `auth.service.getProfile` (auth.service.ts:686) + login response **SELECT + trả `default_database_id`**; FE `auth-context` sau login: nếu localStorage rỗng → set CSDL = `user.default_database_id`. Khi user chọn ổ mới → **PATCH lên server** (set `users.default_database_id`) để nhớ đa-thiết-bị.
- **1.3 Re-fetch:** thêm `activeDatabaseId` vào `useEffect` deps các trang dashboard/report (đổi ổ → tự nạp lại). (api.ts đã gửi header sẵn → chỉ cần trigger refetch.)
- **CỔNG KIỂM:** đăng nhập user chưa có default → modal hiện, không bỏ qua được; chọn ổ → reload trang vẫn nhớ; đổi ổ → số liệu trang đổi theo.
- **Rủi ro:** THẤP (FE-only, không migration). Deploy độc lập được.

### 🥈 ĐỢT 2 — WRITE (G2): GHI `database_id` khi tạo mới — 3 agent domain song song + reviewer mỗi domain
**27 method** (resolve từ context, KHÔNG tin client). Sau đó **backfill (G3)** + verify 0 NULL.
- **2A Tài chính (2):** `journal-entries.create`, `purchase-invoices.createFromXml`.
- **2B Thương mại (3):** `shipments.create`, `order-payments.addPayment`, `payments.createPayment`.
- **2C Kho/SX (22):** inventory(createTransaction/importStock/exportStock/adjustStock), stock-audits.complete, stock-transfers.create, goods-receipts(create/createFromPo), purchase-requests(create/convertToPo), rfqs(create/convertToPo/createMixedPo), manufacturing(completeOrder/reportScrap/createMrpRun), fg-stock(stockIn/transferToSales), fg-stock-hub.stockInHub, rm-stock(stockIn/stockOut/stockInBatch).
- **2D Backfill (migration 0430):** UPDATE `database_id` cho ~7 bảng còn NULL theo `branch_id→branches.database_id`, fallback ổ active đầu. Idempotent `WHERE database_id IS NULL`. (Dữ liệu <60 dòng → 1 lần chạy.)
- **CỔNG KIỂM:** tạo mới mỗi loại chứng từ → cột `database_id` đúng ổ; sau backfill `SELECT count(*) WHERE database_id IS NULL` = 0 mọi bảng.
- **Rủi ro:** THẤP-TRUNG (chỉ ghi thêm cột, không đổi nghiệp vụ). Apply 0430 TRƯỚC deploy code.

### 🥉 ĐỢT 3 — READ (G4): LỌC list/report/findOne — 4 agent domain song song + reviewer
**90 method**, fail-open. ⚠️ **ĐỔI KẾT QUẢ BÁO CÁO/DANH SÁCH** → 🔴 **Letri xác nhận trước khi live + test cách ly đa-ổ**.
- **3A Tài chính (24):** cashbook(getSummary/getTaxSummary), journal-entries.findAll, einvoice.dashboard, purchase-invoices(findOne/listTrash), bank-statements.findOne, **finance-aggregates 11 BCTC** (getReportPreview/getGeneralLedger/getCashbookV3/getDashboardKpi/getPaymentRequests/getEinvoiceList/getRevenueCostTrend/getTopCustomers/getFinanceAlerts/getCostStructure/getCostByUnit/getFinancialRatios), payment-requests(getSummary/findOne/listTrash). *(Sổ cái: giữ scopeId + thêm database_id.)*
- **3B Thương mại (13):** orders(findAll/findOne/getStats/exportToExcel), quotes.getQuote, returns.findOne, online-orders(findOne/getStats), reservations.findOne, shipments.findAll, order-payments.getPaymentsForOrder, payments.findAllTransactions, refunds.getRefund. *(shipments/order-payments/payments hiện chưa inject DatabaseContext → wire module.)*
- **3C Kho/SX (41):** inventory(getStock/getTransactions), stock-documents(findOne/reportInwardSummary/reportOutwardSummary/reportStockValue/reportStockCard/dashboardKpi), stock-audits.findOne, stock-transfers(findAll/findOne), goods-receipts(findAll/findOne/listTrash), purchase-orders(findOne/listTrash), purchase-requests(findAll/findOne/listTrash), rfqs(findAll/findOne/listTrash), **manufacturing 13** (findOneOrder/getOrderStats/getDashboard/getNmvatReport/getPlanningRemaining/getSchedule/findOneOrderFull/checkBomForQuantity/listMrpRuns/getPlanningForecast/getPlanningWeekOrders), fg-stock(getLots/getExpiry), fg-stock-hub(getStockInHistory/getPurchaseHistory), rm-stock(getLots/getExpiry/getMovements/getMovementDetail).
- **3D Báo cáo (14):** reports(getAlerts/getInventoryDashboard/listDailyReports/getDailyReport), purchasing-reports(bySupplier/byProduct/leadTime/threeWayVariance/qcPassRate), reports-export BCTC(exportB01a/exportB02/exportB03/exportS03a/exportSoCai).
- **CỔNG KIỂM:** test cách ly đa-ổ PASS (tạo chứng từ ổ A+B, đọc với X-Database-Id=A → chỉ thấy A); C8 QA xem báo cáo từng ổ đúng; chọn "Tất Cả" (super-admin) → gộp đủ.
- **Rủi ro:** TRUNG (đổi số báo cáo) → **fail-open** + test kỹ. Có thể bật theo domain (3A tài chính trước — đúng ưu tiên thuế).

### 🏅 ĐỢT 4 — KHOÁ CHẶT (G6) — sau khi 0 NULL + test ổn
- Migration SET NOT NULL các bảng lõi (orders→cashbook→einvoices→journal_entries_v3) + đổi `databaseFilter` sang **strict** (`eq` thay `or-isNull`). RLS `database_id` (như tenant rule 02 lớp 4) — Wave sau. 🔴 Letri xác nhận 100% trước.

### 🎖️ ĐỢT 5 — BUILD 2 SERVICE THIẾU (G7)
- `TaxDeclarationsService` (ETL gom einvoices/purchase_invoices/bank_transactions ĐÃ scope → tờ khai GTGT mỗi ổ) + `ReconciliationService` (đối soát bank_statements theo ổ). **Cần để khai thuế đúng** (compliance TT99 + NĐ70).

---

## 4. TEST CÁCH LY ĐA-Ổ (B7 — BẮT BUỘC mỗi đợt READ)
```
it('Ổ A KHÔNG thấy chứng từ ổ B (cùng tenant)') → GET /einvoices/:id với X-Database-Id=B → 404
it('BCTC ổ A không gộp số ổ B') → tạo JE 2 ổ, generateBalanceSheet ổ A → tổng chỉ ổ A
it('Tạo chứng từ ổ A bằng context ổ B → từ chối/đúng ổ A')
it('Thiếu X-Database-Id (fail-open) → thấy hàng NULL cũ, KHÔNG thấy hàng ổ khác')
```

## 5. 🔴 QUYẾT ĐỊNH CẦN LETRI
1. **Thứ tự đợt:** đề xuất **ĐỢT 1 (FE cổng cứng) LÀM TRƯỚC** (đúng điều sếp nói nhất, rủi ro thấp, deploy độc lập) → rồi ĐỢT 2 (write+backfill) → ĐỢT 3 (read, có cổng xác nhận). Sếp đồng ý?
2. **Cổng cứng:** super-admin có được chọn "Tất Cả" (gộp đa ổ) không, hay MỌI user bắt buộc 1 ổ? (ảnh hưởng báo cáo gộp + RBAC `databases.cross_read`).
3. **Khi nào bật ĐỢT 3 read** (đổi số báo cáo) — ngay sau write+backfill, hay chờ sếp xem thử từng ổ?
4. **NOT NULL (Đợt 4):** làm sớm (dữ liệu ít) hay để fail-open chạy một thời gian?

## 6. CẠM BẪY (nhắc agent)
- Migration tiếp theo = **0430** (0429 = seed quyền bán hàng đang chờ apply). Apply prod port **5433** TRƯỚC deploy code.
- safe-deploy KHÔNG tự chạy migration. Branch `n8n/005-wecha-pos-zalo-keypos`, code `claude/pos-system`.
- `orders` đã import databaseFilter nhưng chỉ dùng 1 chỗ (line 1349) — agent PHẢI verify từng method, đừng tưởng "đã có là đủ".
- Đổi shape API = vỡ FE. Wire module (shipments/order-payments/payments) cần inject DatabaseContextService trước.
