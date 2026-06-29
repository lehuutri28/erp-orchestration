# Skill: Kế toán Việt Nam — TT 99/2025/TT-BTC

> Khi nào load: Build module kế toán, hoá đơn, báo cáo tài chính, ghi nhận doanh thu

> ⚠️ Đồng bộ ARCHITECTURE_CONTRACT 29/5/2026: PK=uuid().primaryKey().defaultRandom(), tenant_id=uuid, TS field camelCase, soft delete thùng rác. Xem docs/ARCHITECTURE_CONTRACT.md.

---

## ⚠️ CHUẨN ÁP DỤNG

**Thông tư 99/2025/TT-BTC** — Hướng dẫn chế độ kế toán doanh nghiệp.
- Ban hành: 27/10/2025
- Hiệu lực: **01/01/2026**
- **Thay thế: TT 200/2014/TT-BTC** (cũ, KHÔNG còn áp dụng cho năm tài chính 2026 trở đi)

**KHÔNG dùng TT 200 trong code, comment, document mới.**

---

## Thay đổi quan trọng so với TT 200

### 1. Đổi tên báo cáo

| TT 200 (cũ) | TT 99 (mới) |
|---|---|
| Bảng cân đối kế toán | **Báo cáo tình hình tài chính** |
| Báo cáo kết quả hoạt động kinh doanh | **Báo cáo kết quả hoạt động toàn diện** |
| Báo cáo lưu chuyển tiền tệ | Báo cáo lưu chuyển tiền tệ (giữ nguyên tên) |
| Bản thuyết minh báo cáo tài chính | Thuyết minh báo cáo tài chính |

### 2. Hệ thống tài khoản — linh hoạt hơn

TT 99 cho phép DN **tự thiết kế chi tiết tài khoản** theo bản chất nghiệp vụ, miễn đảm bảo nguyên tắc:
- Phản ánh đúng bản chất kinh tế
- So sánh được giữa các kỳ
- Phù hợp đặc thù ngành

### 3. Tài khoản loại bỏ / bổ sung

**Loại bỏ:**
- TK 161 (Chi sự nghiệp) — không phù hợp DN tư nhân
- TK 611 (Mua hàng — chỉ kế toán giản đơn)
- TK 631 (Giá thành sản xuất — chỉ kế toán giản đơn)

**Bổ sung mới:**
- **TK 215** — Tài sản sinh học (cây trồng, vật nuôi cho DN nông nghiệp)
- **TK 332** — Phải trả cổ tức, lợi nhuận
- **TK 821 cấp 3** — Chi phí thuế TNDN tối thiểu toàn cầu (BEPS Pillar 2)

### 4. Ghi nhận doanh thu — IFRS 15

Doanh thu ghi nhận khi:
1. **Nghĩa vụ thực hiện đã hoàn thành** (Performance obligation satisfied)
2. **Quyền kiểm soát hàng/dịch vụ đã chuyển giao** cho khách

**KHÔNG phải khi:**
- Xuất hoá đơn
- Thu tiền
- Ký hợp đồng

### 5. Khái niệm mới

- **Tài sản hợp đồng** (Contract Asset): quyền nhận thanh toán đã có nhưng chưa đến hạn
- **Nợ phải trả hợp đồng** (Contract Liability): nghĩa vụ giao hàng/dịch vụ đã thu tiền trước

---

## Schema kế toán

```typescript
// modules/accounting/schemas/accounts.schema.ts
export const accounts = pgTable('accounts', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  
  code: text('code').notNull(),              // VD: '511', '511.1', '5111'
  name: text('name').notNull(),              // VD: 'Doanh thu bán hàng'
  shortName: text('short_name'),
  
  type: text('type').notNull(),              // 'asset', 'liability', 'equity', 'revenue', 'expense'
  category: text('category').notNull(),      // 'current_asset', 'fixed_asset', ...
  
  parentId: uuid('parent_id').references((): any => accounts.id),
  level: integer('level').notNull(),         // 1 = cấp 1 (3 chữ số), 2 = cấp 2, ...
  
  isDetailed: boolean('is_detailed').notNull().default(false),
  isActive: boolean('is_active').notNull().default(true),
  isSystem: boolean('is_system').notNull().default(false),  // System account, không được xoá
  
  description: text('description'),
  tt99Reference: text('tt99_reference'),    // Tham chiếu điều khoản TT 99
  
  ...auditFields,
}, (t) => ({
  uniqCode: uniqueIndex('accounts_tenant_code_unique').on(t.tenantId, t.code),
  parentIdx: index('accounts_parent_idx').on(t.tenantId, t.parentId),
}));

// Bút toán (journal_entries v1 đã drop — 0 dòng prod, xem ARCHITECTURE_CONTRACT mục 13)
export const journalEntriesV3 = pgTable('journal_entries_v3', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  
  entryNumber: text('entry_number').notNull(),  // BT-2026-00001
  entryDate: date('entry_date').notNull(),
  postedAt: timestamp('posted_at', { withTimezone: true }),
  postedBy: uuid('posted_by').references(() => users.id),
  
  description: text('description').notNull(),
  referenceType: text('reference_type'),        // 'invoice', 'payment', 'depreciation', ...
  referenceId: text('reference_id'),
  
  totalDebit: decimal('total_debit', { precision: 15, scale: 2 }).notNull(),
  totalCredit: decimal('total_credit', { precision: 15, scale: 2 }).notNull(),
  
  status: text('status').notNull().default('draft'),  // 'draft', 'posted', 'reversed'
  reversalOf: uuid('reversal_of').references((): any => journalEntriesV3.id),
  
  // Soft delete cấp A (ARCHITECTURE_CONTRACT mục 1 — bảng tài chính)
  deleteStatus: text('delete_status').default('active'),         // 'active' | 'pending_delete' | 'trashed'
  deleteRequestedBy: uuid('delete_requested_by'),
  deleteRequestedAt: timestamp('delete_requested_at'),
  deleteReason: text('delete_reason'),
  deleteApprovedBy: uuid('delete_approved_by'),
  deleteApprovedAt: timestamp('delete_approved_at'),
  deletedAt: timestamp('deleted_at'),
  deletedBy: uuid('deleted_by').references(() => users.id),
  purgeAfter: timestamp('purge_after'),  // NULL cho tài chính — không auto-purge (TT 99 10 năm)
  
  ...auditFields,
}, (t) => ({
  uniqNumber: uniqueIndex('je_tenant_number_unique').on(t.tenantId, t.entryNumber),
  dateIdx: index('je_tenant_date_idx').on(t.tenantId, t.entryDate),
  statusCheck: check('je_status_check', sql`status IN ('draft', 'posted', 'reversed')`),
}));

export const journalEntryLines = pgTable('journal_entry_lines', {
  id: uuid('id').primaryKey().defaultRandom(),
  entryId: uuid('entry_id').notNull().references(() => journalEntriesV3.id),
  
  lineNumber: integer('line_number').notNull(),
  accountId: uuid('account_id').notNull().references(() => accounts.id),
  
  debit: decimal('debit', { precision: 15, scale: 2 }).notNull().default('0'),
  credit: decimal('credit', { precision: 15, scale: 2 }).notNull().default('0'),
  
  description: text('description'),
  
  // Phân tích đa chiều
  costCenterId: uuid('cost_center_id'),
  projectId: uuid('project_id'),
  partnerId: uuid('partner_id'),
  
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => ({
  entryIdx: index('jel_entry_idx').on(t.entryId, t.lineNumber),
  accountIdx: index('jel_account_idx').on(t.accountId),
  oneOrOther: check('jel_one_or_other', sql`(debit > 0 AND credit = 0) OR (credit > 0 AND debit = 0)`),
}));
```

---

## Ghi nhận doanh thu (IFRS 15) trong code

```typescript
// orders schema bổ sung
export const orders = pgTable('orders', {
  // ... existing fields
  
  // Ghi nhận doanh thu (TT 99)
  revenue_recognized_at: timestamp('revenue_recognized_at', { withTimezone: true }),
  revenue_recognition_basis: text('revenue_recognition_basis'),  // 'on_delivery', 'on_completion', 'over_time'
  
  // Tài sản hợp đồng / Nợ phải trả hợp đồng
  contract_asset: decimal('contract_asset', { precision: 15, scale: 2 }).notNull().default('0'),
  contract_liability: decimal('contract_liability', { precision: 15, scale: 2 }).notNull().default('0'),
});

// Service: ghi nhận doanh thu khi giao hàng
async recognizeRevenue(orderId: string) {
  return this.db.transaction(async (tx) => {
    const order = await this.ordersRepo.findByIdForTenant(orderId, this.tenantId, tx);
    
    if (order.status !== 'shipped' && order.status !== 'completed') {
      throw new BadRequestException('Chỉ ghi nhận doanh thu khi đã giao hàng (TT 99 — IFRS 15)');
    }
    
    if (order.revenue_recognized_at) {
      throw new BadRequestException('Đơn hàng đã ghi nhận doanh thu rồi');
    }
    
    // Bút toán: Nợ TK 131 / Có TK 511, 3331
    const entry = await tx.insert(journalEntriesV3).values({
      tenantId: this.tenantId,
      entryNumber: await this.generateEntryNumber(tx),
      entryDate: new Date(),
      description: `Ghi nhận doanh thu đơn hàng ${order.code}`,
      referenceType: 'order',
      referenceId: order.id,
      totalDebit: order.total,
      totalCredit: order.total,
      status: 'posted',
      postedAt: new Date(),
      postedBy: this.userId,
    }).returning();
    
    await tx.insert(journalEntryLines).values([
      {
        entryId: entry[0].id,
        lineNumber: 1,
        accountId: await this.getAccountIdByCode('131'),  // Phải thu khách hàng
        debit: order.total,
        credit: '0',
        description: `Phải thu KH: ${order.customer_snapshot.name}`,
        partnerId: order.customer_id,
      },
      {
        entryId: entry[0].id,
        lineNumber: 2,
        accountId: await this.getAccountIdByCode('5111'),  // Doanh thu bán hàng
        debit: '0',
        credit: order.subtotal,
        description: 'Doanh thu bán hàng',
      },
      {
        entryId: entry[0].id,
        lineNumber: 3,
        accountId: await this.getAccountIdByCode('33311'),  // Thuế GTGT đầu ra
        debit: '0',
        credit: order.tax_amount,
        description: 'Thuế GTGT đầu ra',
      },
    ]);
    
    // Update order
    await tx.update(orders).set({
      revenue_recognized_at: new Date(),
      revenue_recognition_basis: 'on_delivery',
    }).where(eq(orders.id, orderId));
  });
}
```

---

## Báo cáo tình hình tài chính (TT 99)

```typescript
async generateBalanceSheet(asOfDate: Date, tenantId: string) {
  // Cấu trúc TT 99 — chia 3 phần lớn
  return {
    title: 'BÁO CÁO TÌNH HÌNH TÀI CHÍNH',
    asOfDate,
    
    assets: {
      currentAssets: {
        cash: await this.balanceOfAccountGroup('111-112-113'),
        receivables: await this.balanceOfAccountGroup('131-138'),
        inventory: await this.balanceOfAccountGroup('151-159'),
        contractAssets: await this.totalContractAssets(),  // MỚI theo TT 99
        otherCurrentAssets: await this.balanceOfAccountGroup('141-142-242'),
      },
      nonCurrentAssets: {
        receivables: await this.balanceOfAccountGroup('131-138', { longTerm: true }),
        fixedAssets: await this.balanceOfAccountGroup('211-217'),
        biologicalAssets: await this.balanceOfAccountCode('215'),  // MỚI TT 99
        investments: await this.balanceOfAccountGroup('221-228'),
        intangibleAssets: await this.balanceOfAccountGroup('213'),
        deferredTax: await this.balanceOfAccountCode('243'),
      },
    },
    
    liabilities: {
      currentLiabilities: {
        payables: await this.balanceOfAccountGroup('331-338'),
        contractLiabilities: await this.totalContractLiabilities(),  // MỚI TT 99
        dividendsPayable: await this.balanceOfAccountCode('332'),  // MỚI TT 99
        taxes: await this.balanceOfAccountGroup('333'),
        shortTermLoans: await this.balanceOfAccountCode('341', { shortTerm: true }),
      },
      nonCurrentLiabilities: {
        longTermLoans: await this.balanceOfAccountCode('341', { longTerm: true }),
        deferredTax: await this.balanceOfAccountCode('347'),
      },
    },
    
    equity: {
      capital: await this.balanceOfAccountCode('411'),
      retainedEarnings: await this.balanceOfAccountCode('421'),
      reserves: await this.balanceOfAccountGroup('414-418'),
    },
  };
}
```

---

## Thuế tối thiểu toàn cầu (BEPS — TK 821 cấp 3)

```typescript
// Áp dụng cho DN có doanh thu hợp nhất > 750M EUR/năm (3 trong 4 năm gần nhất)
// Nếu DN của sếp đạt ngưỡng này → phải tính top-up tax
// Hiện tại WECHA chưa đạt ngưỡng, nhưng schema ready

export const taxAdjustments = pgTable('tax_adjustments', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  fiscalYear: integer('fiscal_year').notNull(),
  
  pillar2Applicable: boolean('pillar2_applicable').notNull().default(false),
  effectiveTaxRate: decimal('effective_tax_rate', { precision: 5, scale: 4 }),
  topupTaxAmount: decimal('topup_tax_amount', { precision: 15, scale: 2 }),
  
  // Account: 821 cấp 3 (chi tiết)
  topupAccountCode: text('topup_account_code'),
  
  ...auditFields,
});
```

---

## Audit retention (TT 99)

| Loại | Retention | Note |
|---|---|---|
| Sổ kế toán chi tiết | 10 năm | TT 99 Điều 41 |
| Sổ kế toán tổng hợp | 10 năm | |
| Báo cáo tài chính | Vĩnh viễn | |
| Chứng từ kế toán | 10 năm | |
| Hoá đơn | 10 năm | TT 99 + TT 32/2025 |
| Bảng kê khai thuế | 10 năm | |

---

## CẤM 🔴

- ❌ Sửa journal entry đã `posted` → phải tạo bút toán đảo (`reversal`)
- ❌ Xoá hoặc UPDATE bảng `journal_entry_lines` đã thuộc entry posted
- ❌ Ghi nhận doanh thu khi chưa giao hàng/dịch vụ
- ❌ Dùng tài khoản TT 200 cũ (161, 611, 631)
- ❌ Float cho cột tiền — dùng `decimal(15, 2)`
- ❌ Hard delete invoice/journal — chỉ soft delete + lý do
- ❌ Tự động post bút toán mà không có review

---

**END Skill: Accounting TT 99/2025**
