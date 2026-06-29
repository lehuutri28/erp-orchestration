# ARCHITECTURE CONTRACT — HIẾN PHÁP HỆ THỐNG ERP PASSION LINK

> **Bộ quy ước BẮT BUỘC** cho mọi session code về sau. Đây là "hiến pháp" — mọi C, mọi worker, mọi PR phải tuân thủ.
> **Ngày ban hành:** 2026-05-29 | **Owner:** Lê Hữu Trí | **Cơ sở:** `ARCHITECTURE_AUDIT.md` + `AUDIT_VERIFY.md` + `DB_VERIFY.md` (400 bảng, 9 domain, data prod thật).
> **Trạng thái:** PRODUCTION-BINDING. Vi phạm = BLOCK merge.
>
> **Phạm vi:** Code MỚI tuân thủ 100% NGAY. Code CŨ refactor theo wave (mục 13).

---

## 0. NGUYÊN TẮC TỐI THƯỢNG

1. Mâu thuẫn giữa contract này và thói quen cũ → **contract thắng**.
2. Không chắc → đọc contract + hỏi, KHÔNG đoán.
3. Mọi bảng mới phải qua **CHECKLIST mục 12** trước khi commit.
4. 4 bảng HUB (`orders`, `products`, `customers`, `branches`) → **review riêng** (mục 11).

---

## 1. SOFT DELETE + THÙNG RÁC + DUYỆT XOÁ 🔴

### LUẬT
**Mọi bảng nghiệp vụ BẮT BUỘC soft delete (vào THÙNG RÁC, khôi phục được). CẤM hard-delete trực tiếp. 3 quyền TÁCH RIÊNG: `sửa` ≠ `xoá` ≠ `duyệt xoá`. Mọi query list mặc định filter `WHERE deleted_at IS NULL`.**

### Phân 2 cấp bảng (Letri chốt 29/5: tài chính = cấp A)

| Cấp | Bảng | Luồng xoá | Auto-purge |
|---|---|---|---|
| **A. Quan trọng (+ Tài chính)** | orders, products, customers, suppliers, hr_contracts + **tài chính**: journal_entries_v3, cashbook, einvoices, purchase_invoices, tax_*, payment_*, bank_* | Nhân viên bấm xoá → **chờ xếp duyệt** → duyệt → vào thùng rác → khôi phục được | 30 ngày — **TRỪ bảng tài chính/chứng từ: KHÔNG auto-purge** (TT 99 lưu 10 năm, chỉ admin purge thủ công + audit log) |
| **B. Thường** | categories, units, tags, danh mục nhỏ... | Nhân viên (có quyền xoá) bấm → vào thùng rác THẲNG (không cần duyệt) | 30 ngày |

> 📌 Tài chính xử lý GIỐNG cấp A (cùng luồng duyệt xoá + thùng rác + khôi phục). Khác biệt DUY NHẤT: cờ `purge_after = NULL` (không tự xoá sau 30 ngày — tuân TT 99/2025 giữ 10 năm). Đây KHÔNG phải cấp riêng, chỉ là cờ retention trên nhóm bảng tài chính.

### Schema chuẩn

**Bảng cấp B (thường) — tối thiểu:**
```typescript
deleted_at: timestamp('deleted_at'),                          // null=active, có giá trị=trong thùng rác
deleted_by: uuid('deleted_by').references(() => users.id),    // ai xoá
purge_after: timestamp('purge_after'),                        // mốc tự xoá vĩnh viễn (=deleted_at+30d)
```

**Bảng cấp A + C (cần duyệt) — thêm:**
```typescript
delete_status: text('delete_status').default('active'),       // 'active' | 'pending_delete' | 'trashed'
delete_requested_by: uuid('delete_requested_by'),             // NV yêu cầu xoá
delete_requested_at: timestamp('delete_requested_at'),
delete_reason: text('delete_reason'),                         // lý do xoá
delete_approved_by: uuid('delete_approved_by'),               // xếp duyệt
delete_approved_at: timestamp('delete_approved_at'),
// + deleted_at, deleted_by, purge_after như trên (purge_after = NULL cho cấp C tài chính)
```

### Luồng nghiệp vụ
```
[Cấp A]   NV bấm Xoá → delete_status='pending_delete' (CHƯA mất, hiện badge "Chờ duyệt xoá")
          → Xếp (quyền *.delete.approve) duyệt → delete_status='trashed' + deleted_at=now
          → purge_after = now+30d (bảng thường) HOẶC NULL (bảng tài chính — không tự xoá)
[Cấp B]   NV (quyền *.delete) bấm Xoá → deleted_at=now + purge_after=now+30d (vào thùng rác thẳng)
THÙNG RÁC → nút KHÔI PHỤC (quyền *.restore): deleted_at=null, delete_status='active', purge_after=null
CRON ngày → record có purge_after < now() → xoá vĩnh viễn (bảng tài chính purge_after=NULL → giữ mãi)
```

### Permission tách (CONTRACT mục 9)
`<resource>.update` | `<resource>.delete` | `<resource>.delete.approve` | `<resource>.restore`

### Code ĐÚNG
```typescript
// Query list — luôn ẩn record trong thùng rác
.where(and(eq(orders.tenant_id, tenantId), isNull(orders.deleted_at)))  // ✅

// Cấp B — xoá thẳng vào thùng rác
await db.update(units).set({ deleted_at: now, deleted_by: user.id, purge_after: add30d(now) })  // ✅

// Cấp A — yêu cầu xoá (chờ duyệt)
await db.update(orders).set({ delete_status: 'pending_delete', delete_requested_by: user.id, delete_reason: dto.reason });  // ✅
// Xếp duyệt
await db.update(orders).set({ delete_status: 'trashed', deleted_at: now, purge_after: add30d(now), delete_approved_by: user.id });  // ✅

// Khôi phục
await db.update(orders).set({ deleted_at: null, delete_status: 'active', purge_after: null });  // ✅
```

### Code SAI
```typescript
await db.delete(orders).where(eq(orders.id, id));  // ❌ hard-delete mất vĩnh viễn
// ❌ purge bảng tài chính (cấp C) sau 30 ngày → vi phạm TT 99/2025
// ❌ gộp quyền sửa+xoá làm 1 → NV sửa được là xoá được
```

### Ngoại lệ (KHÔNG cần soft delete)
- Log/audit: `audit_logs`, `event_log`, `*_audit`, `*_logs`
- Lookup global: `tax_rates`, `vn_provinces/districts/wards`, `currencies`
- Pivot M:N: `user_branches`, `user_roles`, `product_allowed_*`
- Bảng con cascade theo parent: `order_items`, `journal_lines`

> ⚠️ Data prod (DB_VERIFY): orders/products/customers/suppliers/hr_contracts **đang thiếu** deleted_at → refactor CW1 (mục 13).
> 📦 CW1 triển khai: cấp B trước (đơn giản) → cấp A (thêm duyệt) → cấp C (tài chính, không purge).

---

## 2. PRIMARY KEY 🔴

### LUẬT
**Mọi bảng MỚI dùng `uuid('id').primaryKey().defaultRandom()`. Thứ tự chain thống nhất: `.primaryKey()` TRƯỚC `.defaultRandom()`. CẤM serial/bigserial. CẤM cuid2 cho bảng mới.**

### Code ĐÚNG
```typescript
id: uuid('id').primaryKey().defaultRandom(),  // ✅ chuẩn duy nhất
```

### Code SAI
```typescript
id: uuid('id').defaultRandom().primaryKey(),           // ❌ sai thứ tự chain
id: serial('id').primaryKey(),                          // ❌ auto-increment
id: text('id').primaryKey().$defaultFn(() => createId()), // ❌ cuid2 (bảng mới)
```

### Ngoại lệ được phép
- **7 bảng outlier hiện có GIỮ NGUYÊN, KHÔNG sửa:** `hr_contract_files`, `hr_contract_signatures`, `hr_contract_versions` (text+cuid2) + 4 bảng marketing (`marketing_campaign_sends/recipients/events/approvals` text). Sửa = rủi ro vỡ FK, không đáng.
- Pivot M:N dùng composite PK: `primaryKey({ columns: [t.userId, t.branchId] })` — hợp lệ.

---

## 3. MULTI-TENANT 🔴

### LUẬT
**Mọi bảng nghiệp vụ BẮT BUỘC `tenant_id uuid notNull`. Mọi query (select/update/delete) lọc `tenant_id`. POST/PATCH override `tenant_id` từ JWT, KHÔNG tin client body.**

### Code ĐÚNG
```typescript
tenant_id: uuid('tenant_id').notNull().references(() => tenants.id),  // ✅

// Query
.where(and(eq(orders.tenant_id, user.tenantId), isNull(orders.deleted_at)))  // ✅

// Create — override từ JWT
await db.insert(orders).values({ ...dto, tenant_id: user.tenantId });  // ✅
```

### Code SAI
```typescript
await db.insert(orders).values(dto);  // ❌ tin tenant_id từ client
.where(eq(orders.id, id))             // ❌ thiếu tenant_id → IDOR cross-tenant
```

### Ngoại lệ được phép
- Master: `tenants`
- Lookup global: `tax_rates`, `vn_provinces/districts/wards`, `currencies`
- Bảng con scope qua parent: `order_items`, `journal_lines` (parent đã có tenant_id)
- Pivot M:N

> ⚠️ `support_tickets` + `ticket_*` đang thiếu tenant_id (lỗ hổng IDOR — AUDIT_VERIFY mục 2) → thêm Wave 4 (mục 13).

---

## 4. MULTI-BRANCH 🟠

### LUẬT
**Bảng nào cần BÁO CÁO hoặc PHÂN QUYỀN theo chi nhánh thì BẮT BUỘC `branch_id uuid`. Quy tắc quyết định: "Sếp chi nhánh có cần xem/sửa riêng dữ liệu chi nhánh mình không?" → CÓ thì thêm branch_id.**

### Code ĐÚNG
```typescript
branch_id: uuid('branch_id').notNull().references(() => branches.id),  // bắt buộc theo CN
// hoặc nullable nếu có data toàn tenant lẫn theo CN:
branch_id: uuid('branch_id').references(() => branches.id),
```

### 5 bảng nhóm CAO PHẢI thêm branch_id (Wave 4):
`customer_debt_transactions`, `supplier_debt_transactions`, `payment_transactions`, `hr_payslips` (+`hr_payslips_generated`), `hr_kpi_tasks` (+`hr_kpi_employee_results`)

### Ngoại lệ được phép (KHÔNG cần branch_id)
- Master toàn tenant: `customers` (đã có `customer_branches` M:N), `products`, `suppliers`
- Dùng `scope_id → entity_scopes` thay branch_id raw: `journal_entries_v3`, `report_snapshots` (pattern phân cấp công ty/chi nhánh — KHÔNG phải thiếu)
- Config/lookup cấp tenant

---

## 5. AUDIT FIELDS 🟠 (🔴 cho tài chính)

### LUẬT
**Mọi bảng nghiệp vụ có đủ `created_at`, `updated_at` (auto `$onUpdate`), `created_by`. Bảng tài chính/chứng từ BẮT BUỘC `created_by` (TT 99/2025 — lưu 10 năm).**

### Code ĐÚNG
```typescript
created_at: timestamp('created_at').notNull().defaultNow(),                    // ✅
updated_at: timestamp('updated_at').notNull().defaultNow().$onUpdate(() => new Date()), // ✅ auto
created_by: uuid('created_by').references(() => users.id),                     // ✅
```

### Code SAI
```typescript
updated_at: timestamp('updated_at'),  // ❌ thiếu .$onUpdate → không tự cập nhật
// Bảng cashbook/invoice/journal mà KHÔNG có created_by → ❌ vi phạm TT 99/2025
```

### Bảng tài chính BẮT BUỘC created_by
`cashbook_entries`, `journal_entries_v3`, `journal_lines`, `payment_transactions`, `purchase_invoices`, `einvoices`, `bank_transactions`, `closing_entries`, `employee_advances`, `customer_debt_transactions`, `supplier_debt_transactions`.

---

## 6. RESPONSE API 🔴

### LUẬT
**MỌI endpoint list trả `{ data: T[], meta: {...} }`. MỌI endpoint single trả `{ data: T }`. CẤM trả mảng trần, CẤM shape tự chế.**

### Shape chuẩn
```typescript
// LIST
{
  data: T[],
  meta: {
    total?: number,        // tổng record (offset pagination)
    nextCursor?: string,   // cursor trang sau (null nếu hết) — cho cursor pagination
    hasMore: boolean,      // còn trang sau không
    limit: number,         // số record/trang
  }
}

// SINGLE
{ data: T }

// ERROR (RFC 7807 — đã có GlobalExceptionFilter)
{ error: { code, title, status, detail, requestId } }
```

### Code ĐÚNG
```typescript
async findAll(query): Promise<{ data: Order[]; meta: ListMeta }> {
  const rows = await ...;
  return { data: rows, meta: { hasMore, nextCursor, limit } };  // ✅
}
async findOne(id): Promise<{ data: Order }> {
  return { data: order };  // ✅
}
```

### Code SAI
```typescript
return rows;                              // ❌ mảng trần
return { items: rows, total };            // ❌ shape tự chế (phải là data/meta)
return { data, total, page, limit };      // ❌ total/limit phải nằm trong meta
```

### Áp dụng
- **Code mới: 100% NGAY.** FE wire 1 contract duy nhất, hết `normalizeList()` đoán shape.
- Code cũ (~200 endpoint với 6 shape) refactor Wave 2 (mục 13).

---

## 7. PAGINATION 🟠

### LUẬT
**Bảng lớn (orders, customers, products, order_items, inventory_transactions, journal_lines, audit_logs) BẮT BUỘC cursor-based. Bảng nhỏ (<10k rows, config, lookup) cho phép offset.**

### Cursor format chuẩn
```typescript
// Cursor = base64(id của record cuối trang trước) hoặc {created_at}_{id}
// Query
const limit = Math.min(query.limit ?? 20, 100);  // cap 100
const items = await db.select().from(orders)
  .where(and(
    eq(orders.tenant_id, tenantId),
    isNull(orders.deleted_at),
    query.cursor ? lt(orders.id, decodeCursor(query.cursor)) : undefined,
  ))
  .orderBy(desc(orders.created_at), desc(orders.id))
  .limit(limit + 1);  // +1 để biết hasMore

const hasMore = items.length > limit;
const data = hasMore ? items.slice(0, -1) : items;
return { data, meta: { nextCursor: hasMore ? encodeCursor(data.at(-1).id) : null, hasMore, limit } };
```

### FE dùng
```typescript
// Lần đầu: GET /orders?limit=20
// Trang sau: GET /orders?limit=20&cursor={meta.nextCursor}
// Hết khi meta.hasMore === false
```

### Code SAI
```typescript
.limit(pageSize).offset(page * pageSize)  // ❌ offset cho bảng lớn → chậm + lệch khi insert
const limit = query.limit;                // ❌ không cap → client xin 1 triệu record
```

### Ngoại lệ
Offset OK cho: danh mục, đơn vị, cấu hình, vai trò, bảng <10k rows ít thay đổi.

---

## 8. MÃ CHỨNG TỪ 🔴

### LUẬT
**1 quy luật DUY NHẤT: `{PREFIX}-{YYYYMMDD}-{seq}`. seq sinh từ PostgreSQL SEQUENCE. CẤM `Math.random()`. CẤM `COUNT(*)+1`. Mỗi loại chứng từ 1 prefix DUY NHẤT (hết trùng).**

### Code ĐÚNG
```typescript
// Tạo sequence per tenant+prefix+ngày (migration)
// CREATE SEQUENCE IF NOT EXISTS seq_order_20260529;
const seq = await db.execute(sql`SELECT nextval('seq_order_' || ${dateKey})`);
const code = `HD-${dateStr}-${String(seq).padStart(4, '0')}`;  // ✅ HD-20260529-0001
```

### Code SAI
```typescript
const code = `WO-${date}-${Math.floor(Math.random()*10000)}`;  // ❌ random → trùng khi concurrent
const count = await db.select({c: count()}).from(orders);
const code = `HD${count+1}`;                                    // ❌ COUNT+1 → race condition
```

### Bảng MAP PREFIX chuẩn hoá (BẮT BUỘC tuân theo)

| Loại chứng từ | Prefix CŨ | **Prefix CHUẨN** | Ghi chú |
|---|---|---|---|
| Đơn POS | HD | **HD** | giữ |
| Đơn online | OL | **OL** | giữ |
| Đề nghị mua | PR | **PR** | giữ |
| **Yêu cầu thanh toán** | PR ⚠️ | **PMT** | 🔴 ĐỔI (hết trùng Đề nghị mua) |
| Đơn mua NCC | PO | **PO** | giữ |
| Yêu cầu báo giá | RFQ | **RFQ** | giữ |
| Nhập kho | NK | **NK** | giữ |
| Xuất kho | XK | **XK** | giữ |
| Chuyển kho | CK | **CK** | giữ |
| Kiểm kho | KK | **KK** | giữ |
| Lệnh sản xuất | MO | **MO** | giữ |
| Work Order SX | WO | **WO** | bỏ Math.random → sequence |
| Nhập kho TP (SX) | PN-SX | **PN** | bỏ random → sequence |
| Xuất NVL (SX) | XI-SX | **XI** | bỏ random → sequence |
| Trả hàng mua | TLR | **TLR** | giữ |
| Chứng chỉ LMS | PL | **PL** | giữ (đã dùng PG sequence) |
| Nhân viên | NV | **NV** | giữ |
| Hoá đơn DV (DMS) | SI | **SI** | giữ |
| Lô tồn kho | (theo SP) | **LOT** | chuẩn hoá `LOT-{SP}-{YYYYMM}-{seq}` |

> 🔴 **Quan trọng:** "PR" CHỈ còn nghĩa "Đề nghị mua". "Yêu cầu thanh toán" đổi sang **PMT**. Refactor Wave 3.

---

## 9. RBAC PERMISSION 🟠

### LUẬT
**Format thống nhất `resource.action` — dấu CHẤM ngăn cấp. resource nhiều từ dùng camelCase hoặc gộp. CẤM dùng `_` làm separator cấp.**

### Code ĐÚNG
```typescript
@RequirePermissions('customers.view')          // ✅
@RequirePermissions('customerLms.deposits.create')  // ✅ camelCase cho resource nhiều từ
@RequirePermissions('products.*')              // ✅ wildcard
```

### Code SAI
```typescript
@RequirePermissions('customer_lms.deposits.create')  // ❌ _ làm separator
@RequirePermissions('audit_log.view')                // ❌ → 'auditLog.view'
@RequirePermissions('system.audit_log.view')         // ❌ trùng nghĩa, 2 format
```

### Quy ước
- Cấp: `resource.action` hoặc `module.resource.action`
- action chuẩn: `view`, `create`, `update`, `delete`, `export`, `approve`, `*`
- Code cũ format `_` → refactor Wave 4 (cùng tenant fix).

---

## 10. NAMING 🟠

### LUẬT
**Bảng DB: `snake_case` số NHIỀU. TS field: `camelCase` ↔ DB column `snake_case`. File schema: `kebab-case.ts`.**

### Code ĐÚNG
```typescript
export const orderItems = pgTable('order_items', {  // ✅ snake_case số nhiều
  tenantId: uuid('tenant_id'),                       // ✅ camelCase ↔ snake_case
  createdAt: timestamp('created_at'),
});
```

### Code SAI
```typescript
pgTable('OrderItem', {...})   // ❌ PascalCase + số ít
pgTable('order_item', {...})  // ❌ số ít (trừ singleton/config/log)
tenant_id: uuid('tenant_id')  // ❌ TS field phải camelCase: tenantId
```

### Ngoại lệ được phép (số ít)
Bảng singleton/config/log: `finance_config`, `fiscal_year`, `inventory_stock`, `event_log`, `customer_timeline`, `loyalty_config`. (~14% bảng — chấp nhận.)

---

## 11. 🧊 BẢNG HUB ĐÓNG BĂNG

### LUẬT
**4 bảng sau có >10 domain phụ thuộc qua FK. MỌI thay đổi schema (thêm/xoá/đổi cột, đổi type, đổi FK) PHẢI có review riêng + thông báo cross-team TRƯỚC khi migration.**

| Bảng HUB | Domain phụ thuộc | Rủi ro khi sửa |
|---|---|---|
| **`orders`** | Bán hàng, Kế toán (payments/einvoice), Kho (stock_documents), Marketing (ads_leads), CRM, Returns, Vouchers, Warranties | 🔴 RẤT CAO — hub trung tâm |
| **`products`** | Kho, Sản xuất (boms/mo), Mua hàng (mrp), Bán hàng (order_items), LMS | 🔴 RẤT CAO |
| **`customers`** | CRM, Bán hàng, LMS (enrollments/deposits), Kế toán (einvoice), Marketing, Chatbot | 🔴 RẤT CAO — FK từ >10 bảng |
| **`branches`** | TẤT CẢ domain | 🔴 RẤT CAO |

### Quy trình sửa bảng HUB
1. Viết đề xuất thay đổi → file `docs/HUB_CHANGE_<bảng>_<ngày>.md`
2. Liệt kê mọi bảng/service phụ thuộc bị ảnh hưởng
3. CEO ERP review + thông báo các C liên quan
4. Chỉ migration sau khi xác nhận không vỡ FK domain khác
5. Ưu tiên expand-contract (thêm cột mới, không sửa/xoá cột cũ ngay)

### Code SAI
```sql
ALTER TABLE orders DROP COLUMN customer_id;   -- ❌ vỡ FK 5+ domain, KHÔNG review
ALTER TABLE products ALTER COLUMN price TYPE integer;  -- ❌ đổi type hub, không thông báo
```

---

## 12. ✅ CHECKLIST BẢNG MỚI (tick trước khi commit)

Mỗi bảng nghiệp vụ mới, dev tự kiểm:

```
□ PK: uuid('id').primaryKey().defaultRandom()  (đúng thứ tự chain)
□ tenant_id: uuid notNull + references(tenants.id)  — trừ ngoại lệ mục 3
□ branch_id: nếu cần báo cáo/phân quyền theo chi nhánh (mục 4)
□ deleted_at: timestamp nullable  — trừ ngoại lệ mục 1
□ created_at: notNull().defaultNow()
□ updated_at: notNull().defaultNow().$onUpdate(() => new Date())
□ created_by: references(users.id)  — BẮT BUỘC nếu là bảng tài chính/chứng từ
□ Tên bảng: snake_case số nhiều
□ TS field camelCase ↔ DB column snake_case
□ Index: tenant_id + (tenant_id, created_at DESC) + FK columns
□ Partial index: WHERE deleted_at IS NULL (cho query active)
□ FK có onDelete rõ ràng (restrict/cascade/set null)
□ Migration có file _down.sql (reversible)
□ Nếu là chứng từ: mã theo {PREFIX}-{YYYYMMDD}-{seq} qua PG sequence
□ Service: query list filter tenant_id + deleted_at
□ API: response shape { data, meta }
□ Pagination: cursor-based nếu bảng lớn
□ Permission: format resource.action (dấu chấm)
□ KHÔNG sửa 4 bảng HUB mà chưa review (mục 11)
```

---

## 13. 📅 KẾ HOẠCH ÁP DỤNG

### Code MỚI: tuân thủ 100% NGAY (từ 29/5/2026)
Mọi bảng/endpoint mới qua CHECKLIST mục 12. PR vi phạm → BLOCK merge.

### Code CŨ: refactor theo 4 wave (thứ tự ưu tiên)

| Wave | Nội dung | Phạm vi | Ưu tiên | Owner gợi ý |
|---|---|---|---|---|
| **CW1** | **Soft delete** — thêm `deleted_at` cho bảng nghiệp vụ thiếu (bắt đầu orders/products/customers/suppliers/hr_contracts) + sửa query filter | ~30-50 bảng | 🔴 CAO (đang mất data vĩnh viễn + migration fail) | C1/C4/C9/C16 theo domain |
| **CW2** | **Response API** — chuẩn hoá `{data, meta}` cho ~200 endpoint + sửa FE bỏ `normalizeList` | ~200 endpoint + ~100 page | 🔴 CAO (FE liên tục vỡ) | Mỗi C domain mình |
| **CW3** | **Mã chứng từ** — đổi Payment Request PR→PMT + bỏ Math.random (WO/PN/XI) → PG sequence + chuẩn format `{PREFIX}-{YYYYMMDD}-{seq}` | ~20 loại chứng từ | 🟠 TRUNG (PR trùng gây nhầm + random trùng mã) | C17 (mua hàng) + C4 (thanh toán) + C10 (SX) |
| **CW4** | **Branch/Tenant** — thêm tenant_id cho support_tickets (lỗ hổng IDOR) + branch_id cho 5 bảng nhóm CAO + chuẩn RBAC `_`→`.` | support_* + 5 bảng + permissions | 🟠 TRUNG (nền franchise + bảo mật trước onboard) | C5/C7 (support) + C4/C9 (branch) |

### Việc dọn ngay (data prod đã verify rỗng/không tồn tại)
| Việc | Cơ sở | Owner |
|---|---|---|
| Drop `journal_entries` v1 | 0 dòng + 0 code dùng (DB_VERIFY Q2) | C4 |
| Xoá code `promotionsLegacy` (KHÔNG cần drop table — không tồn tại prod) | DB_VERIFY Q3 = NULL | C1 |

### Quy tắc refactor (mọi wave)
- Expand-contract: thêm cột mới trước, sửa code, drop cột cũ sau (rule 03)
- Bảng HUB → review riêng (mục 11)
- Mỗi migration có `.down.sql`
- Verify schema prod TRƯỚC khi viết migration (bài học 28/5: 3 migration fail vì giả định sai deleted_at)

---

## 14. THỰC THI

- File này là **chuẩn tham chiếu** cho mọi BRIEF/NHAC từ CEO ERP.
- Mỗi BRIEF Sprint mới BẮT BUỘC ghi: "Tuân thủ ARCHITECTURE_CONTRACT mục X".
- Code review (C8 + reviewer) check theo CHECKLIST mục 12.
- Cập nhật contract → cần Letri duyệt + ghi lịch sử thay đổi cuối file.

---

## LỊCH SỬ THAY ĐỔI

| Version | Ngày | Nội dung |
|---|---|---|
| 1.0 | 2026-05-29 | Ban hành đầu tiên — 10 luật + bảng HUB + checklist + 4 wave. Cơ sở: audit 400 bảng + 10 quyết định Letri chốt. |

---

> **Đây là hiến pháp. Code mới tuân thủ ngay. Code cũ theo wave. Vi phạm = BLOCK merge.**
> File ghi tại `docs/` gốc (Letri đọc được) + copy `claude/pos-system/docs/` (14 C tham chiếu).
