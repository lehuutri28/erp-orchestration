# Skill: Franchise WECHA

> Khi nào load: Build module nhượng quyền, royalty, supply chain to partners

> ⚠️ Đồng bộ ARCHITECTURE_CONTRACT 29/5/2026: PK=uuid().primaryKey().defaultRandom(), tenant_id=uuid, TS field camelCase, soft delete thùng rác. Xem docs/ARCHITECTURE_CONTRACT.md.

---

## Domain context

WECHA = thương hiệu trà sữa nhượng quyền của VUA AN TOÀN.
- Đối tác franchise (franchisee) trả phí nhượng quyền + royalty
- Nhập nguyên liệu từ VUA AN TOÀN
- Dùng menu chuẩn + recipe chuẩn
- Báo cáo doanh thu định kỳ
- Multi-tenant: mỗi franchise = sub-tenant của tenant chính

---

## Schema

```typescript
// Đối tác franchise
export const franchisePartners = pgTable('franchise_partners', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),       // Tenant của VUA AN TOÀN (chủ thương hiệu)
  
  partnerCode: text('partner_code').notNull(), // 'WECHA-PARTNER-001'
  legalName: text('legal_name').notNull(),     // Tên pháp nhân
  tradeName: text('trade_name'),               // Tên thương mại
  taxCode: text('tax_code'),
  businessLicense: text('business_license'),
  
  contactPerson: text('contact_person').notNull(),
  contactPhone: text('contact_phone').notNull(),
  contactEmail: text('contact_email'),
  
  address: text('address'),
  city: text('city'),
  province: text('province'),
  
  // Sub-tenant ID (cho phân quyền data isolation)
  subTenantId: uuid('sub_tenant_id').notNull().unique(),
  
  status: text('status').notNull().default('prospect'),
  // 'prospect', 'contracted', 'training', 'opening', 'active', 'paused', 'terminated'
  
  ...auditFields,
}, (t) => ({
  uniqCode: uniqueIndex('partner_code_unique').on(t.tenantId, t.partnerCode),
}));

// Hợp đồng nhượng quyền
export const franchiseContracts = pgTable('franchise_contracts', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  partnerId: uuid('partner_id').notNull().references(() => franchisePartners.id),
  
  contractNumber: text('contract_number').notNull(),
  contractType: text('contract_type').notNull(), // 'master', 'single_unit', 'multi_unit'
  
  signedDate: date('signed_date').notNull(),
  startDate: date('start_date').notNull(),
  endDate: date('end_date').notNull(),
  
  // Financial terms
  initialFee: decimal('initial_fee', { precision: 15, scale: 2 }).notNull(),  // Phí nhượng quyền ban đầu
  initialFeePaid: boolean('initial_fee_paid').notNull().default(false),
  
  royaltyType: text('royalty_type').notNull(),   // 'percent_revenue', 'fixed_monthly', 'tiered'
  royaltyRate: decimal('royalty_rate', { precision: 5, scale: 4 }),  // 0.05 = 5%
  royaltyFixed: decimal('royalty_fixed', { precision: 15, scale: 2 }),
  royaltyTiers: jsonb('royalty_tiers'),
  
  marketingFeeRate: decimal('marketing_fee_rate', { precision: 5, scale: 4 }), // % doanh thu cho marketing fund
  
  // Territory
  territory: text('territory'),                   // 'Quận 7, TP.HCM'
  exclusivityRadiusKm: decimal('exclusivity_radius_km', { precision: 6, scale: 2 }),
  
  // Operational
  minPurchasePerMonth: decimal('min_purchase_per_month', { precision: 15, scale: 2 }),
  
  status: text('status').notNull().default('draft'),
  // 'draft', 'pending_signature', 'active', 'expired', 'terminated'
  
  contractPdfUrl: text('contract_pdf_url'),
  
  ...auditFields,
});

// Chi nhánh franchise (1 partner có thể nhiều chi nhánh)
export const franchiseBranches = pgTable('franchise_branches', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  partnerId: uuid('partner_id').notNull().references(() => franchisePartners.id),
  contractId: uuid('contract_id').notNull().references(() => franchiseContracts.id),
  
  branchCode: text('branch_code').notNull(),
  branchName: text('branch_name').notNull(),
  
  address: text('address').notNull(),
  city: text('city'),
  province: text('province'),
  latitude: decimal('latitude', { precision: 10, scale: 7 }),
  longitude: decimal('longitude', { precision: 10, scale: 7 }),
  
  storeSizeSqm: decimal('store_size_sqm', { precision: 8, scale: 2 }),
  seatingCapacity: integer('seating_capacity'),
  
  openedDate: date('opened_date'),
  closedDate: date('closed_date'),
  
  managerUserId: uuid('manager_user_id'),
  
  status: text('status').notNull().default('preparing'),
  // 'preparing', 'training', 'opening', 'active', 'paused', 'closed'
  
  ...auditFields,
}, (t) => ({
  uniqCode: uniqueIndex('branch_code_unique').on(t.tenantId, t.branchCode),
}));

// Báo cáo doanh thu hàng tháng từ franchise
export const franchiseRevenueReports = pgTable('franchise_revenue_reports', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  partnerId: uuid('partner_id').notNull().references(() => franchisePartners.id),
  contractId: uuid('contract_id').notNull(),
  branchId: uuid('branch_id').references(() => franchiseBranches.id),  // Null = consolidated
  
  periodYear: integer('period_year').notNull(),
  periodMonth: integer('period_month').notNull(),
  
  reportedRevenue: decimal('reported_revenue', { precision: 15, scale: 2 }).notNull(),
  auditedRevenue: decimal('audited_revenue', { precision: 15, scale: 2 }),
  
  totalOrders: integer('total_orders'),
  
  royaltyAmount: decimal('royalty_amount', { precision: 15, scale: 2 }).notNull(),
  marketingFeeAmount: decimal('marketing_fee_amount', { precision: 15, scale: 2 }),
  totalFees: decimal('total_fees', { precision: 15, scale: 2 }).notNull(),
  
  status: text('status').notNull().default('submitted'),
  // 'submitted', 'reviewing', 'approved', 'disputed', 'paid'
  
  submittedAt: timestamp('submitted_at', { withTimezone: true }),
  approvedAt: timestamp('approved_at', { withTimezone: true }),
  paidAt: timestamp('paid_at', { withTimezone: true }),
  
  notes: text('notes'),
  attachments: jsonb('attachments'),
  
  ...auditFields,
}, (t) => ({
  uniqReport: uniqueIndex('report_unique').on(t.partnerId, t.branchId, t.periodYear, t.periodMonth),
}));

// Đơn nhập nguyên liệu từ franchise → VUA AN TOÀN
export const franchiseSupplyOrders = pgTable('franchise_supply_orders', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  partnerId: uuid('partner_id').notNull().references(() => franchisePartners.id),
  branchId: uuid('branch_id').references(() => franchiseBranches.id),
  
  orderNumber: text('order_number').notNull(),
  orderedAt: timestamp('ordered_at', { withTimezone: true }).notNull().defaultNow(),
  requestedDeliveryDate: date('requested_delivery_date'),
  
  subtotal: decimal('subtotal', { precision: 15, scale: 2 }).notNull(),
  shippingFee: decimal('shipping_fee', { precision: 15, scale: 2 }).default('0'),
  total: decimal('total', { precision: 15, scale: 2 }).notNull(),
  
  paymentStatus: text('payment_status').notNull().default('unpaid'),
  // 'unpaid', 'partial', 'paid', 'credit'
  
  fulfillmentStatus: text('fulfillment_status').notNull().default('pending'),
  // 'pending', 'confirmed', 'preparing', 'shipped', 'delivered', 'cancelled'
  
  shippedAt: timestamp('shipped_at', { withTimezone: true }),
  deliveredAt: timestamp('delivered_at', { withTimezone: true }),
  
  ...auditFields,
});
```

---

## Multi-tenant strategy với franchise

```typescript
// 4 lớp defense (Rule 02) — adapt cho franchise

// 1. Tenant context có 2 cấp
interface TenantContext {
  primary_tenant_id: string;     // VUA AN TOÀN
  sub_tenant_id?: string;         // Franchise partner (nếu user thuộc franchise)
  is_franchise_user: boolean;
}

// 2. Filter helper — auto include sub_tenant nếu franchise user
export function franchiseAwareFilter<T extends { tenant_id; sub_tenant_id? }>(table: T) {
  const ctx = tenantContext.getStore();
  
  if (ctx.is_franchise_user) {
    // Franchise user chỉ thấy data của sub-tenant của mình
    return and(
      eq(table.tenant_id, ctx.primary_tenant_id),
      eq(table.sub_tenant_id, ctx.sub_tenant_id),
    );
  } else {
    // Staff VUA AN TOÀN thấy data toàn bộ tenant
    return eq(table.tenant_id, ctx.primary_tenant_id);
  }
}
```

---

## Royalty calculation

```typescript
async calculateRoyaltyForReport(reportId: string) {
  return this.db.transaction(async (tx) => {
    const report = await this.reportsRepo.findById(reportId, tx);
    const contract = await this.contractsRepo.findById(report.contract_id, tx);
    
    const revenue = new Decimal(report.audited_revenue ?? report.reported_revenue);
    
    let royalty: Decimal;
    
    switch (contract.royalty_type) {
      case 'percent_revenue':
        royalty = revenue.times(contract.royalty_rate);
        break;
      
      case 'fixed_monthly':
        royalty = new Decimal(contract.royalty_fixed);
        break;
      
      case 'tiered':
        // Tier table: [{ from: 0, to: 100M, rate: 0.05 }, { from: 100M, to: null, rate: 0.04 }]
        royalty = this.calculateTieredRoyalty(revenue, contract.royalty_tiers);
        break;
    }
    
    // Marketing fee
    const marketingFee = contract.marketing_fee_rate
      ? revenue.times(contract.marketing_fee_rate)
      : new Decimal(0);
    
    const totalFees = royalty.plus(marketingFee);
    
    await tx.update(franchiseRevenueReports).set({
      royalty_amount: royalty.toString(),
      marketing_fee_amount: marketingFee.toString(),
      total_fees: totalFees.toString(),
    }).where(eq(franchiseRevenueReports.id, reportId));
    
    return { royalty, marketingFee, totalFees };
  });
}
```

---

## Auto-detect violation (báo cáo dưới ngưỡng tối thiểu)

```typescript
@Cron('0 9 5 * *')  // Ngày 5 hàng tháng, 9h sáng
async checkMinimumPurchase() {
  const lastMonth = subMonths(new Date(), 1);
  const year = lastMonth.getFullYear();
  const month = lastMonth.getMonth() + 1;
  
  const contracts = await this.db.query.franchiseContracts.findMany({
    where: and(
      eq(franchiseContracts.status, 'active'),
      isNotNull(franchiseContracts.min_purchase_per_month),
    ),
  });
  
  for (const contract of contracts) {
    const totalPurchase = await this.db.execute(sql`
      SELECT COALESCE(SUM(total), 0) as total
      FROM franchise_supply_orders
      WHERE partner_id = ${contract.partner_id}
      AND EXTRACT(YEAR FROM ordered_at) = ${year}
      AND EXTRACT(MONTH FROM ordered_at) = ${month}
      AND fulfillment_status NOT IN ('cancelled')
    `);
    
    const purchase = new Decimal(totalPurchase[0].total);
    const min = new Decimal(contract.min_purchase_per_month);
    
    if (purchase.lt(min)) {
      const shortage = min.minus(purchase);
      
      await this.notifyService.sendBreachNotice({
        partner_id: contract.partner_id,
        type: 'min_purchase_shortage',
        period: `${month}/${year}`,
        actual: purchase.toString(),
        required: min.toString(),
        shortage: shortage.toString(),
      });
      
      await this.contractsService.recordBreach(contract.id, {
        type: 'min_purchase',
        period: `${month}/${year}`,
        details: { actual: purchase.toString(), required: min.toString() },
      });
    }
  }
}
```

---

## Anti-patterns

- ❌ Cho franchise user query cross-partner data
- ❌ Royalty calculation không có audit trail
- ❌ Báo cáo doanh thu không cho upload bằng chứng (POS export)
- ❌ Auto-deduct royalty từ tài khoản partner không có consent
- ❌ Contract terminate không cleanup sub-tenant data
- ❌ Supply order không kiểm tra min purchase contract
- ❌ Multi-branch share inventory ngầm → confusing

---

**END Skill: Franchise WECHA**
