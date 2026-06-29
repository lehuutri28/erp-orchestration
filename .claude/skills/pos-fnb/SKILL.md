# Skill: POS F&B (WECHA)

> Khi nào load: Build POS terminal, e-invoice từ máy tính tiền, ca làm việc

> ⚠️ Đồng bộ ARCHITECTURE_CONTRACT 29/5/2026: PK=uuid().primaryKey().defaultRandom(), tenant_id=uuid, TS field camelCase, soft delete thùng rác. Xem docs/ARCHITECTURE_CONTRACT.md.

---

## Domain context

POS quán trà sữa WECHA — đặc thù:
- Recipe-based pricing (tự tính cost theo BOM)
- Size variants (M/L/XL với multiplier)
- Topping & customization (đường, đá, topping)
- E-invoice từ máy tính tiền (NĐ 70/2025 + TT 32/2025)
- Ca làm việc + két thu (cash drawer)
- Multi-payment (cash + transfer + voucher)

---

## Schema POS

```typescript
// POS terminal — đăng ký với CQT
export const posTerminals = pgTable('pos_terminals', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  
  terminalCode: text('terminal_code').notNull(),  // 'WECHA_POS_001'
  storeId: uuid('store_id').notNull(),            // FK to stores
  deviceName: text('device_name'),
  
  // CQT registration (TT 32/2025)
  cqtRegistered: boolean('cqt_registered').notNull().default(false),
  cqtRegisteredAt: timestamp('cqt_registered_at', { withTimezone: true }),
  cqtTerminalId: text('cqt_terminal_id'),         // ID từ CQT
  
  // E-invoice config
  einvoiceSerial: text('einvoice_serial'),         // Ký hiệu hoá đơn
  einvoiceTemplate: text('einvoice_template'),     // Mẫu hoá đơn
  
  status: text('status').notNull().default('active'),
  lastSeenAt: timestamp('last_seen_at', { withTimezone: true }),
  
  ...auditFields,
}, (t) => ({
  uniqCode: uniqueIndex('pos_terminal_code_unique').on(t.tenantId, t.terminalCode),
}));

// Ca làm việc
export const posShifts = pgTable('pos_shifts', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  
  terminalId: uuid('terminal_id').notNull().references(() => posTerminals.id),
  cashierId: uuid('cashier_id').notNull(),         // User ID
  
  shiftNumber: text('shift_number').notNull(),
  
  openedAt: timestamp('opened_at', { withTimezone: true }).notNull(),
  openingCash: decimal('opening_cash', { precision: 15, scale: 2 }).notNull(),
  
  closedAt: timestamp('closed_at', { withTimezone: true }),
  closingCash: decimal('closing_cash', { precision: 15, scale: 2 }),
  expectedCash: decimal('expected_cash', { precision: 15, scale: 2 }),
  cashDifference: decimal('cash_difference', { precision: 15, scale: 2 }),
  
  totalOrders: integer('total_orders').notNull().default(0),
  totalRevenue: decimal('total_revenue', { precision: 15, scale: 2 }).notNull().default('0'),
  totalCash: decimal('total_cash', { precision: 15, scale: 2 }).notNull().default('0'),
  totalTransfer: decimal('total_transfer', { precision: 15, scale: 2 }).notNull().default('0'),
  totalVoucher: decimal('total_voucher', { precision: 15, scale: 2 }).notNull().default('0'),
  
  status: text('status').notNull().default('open'),  // 'open', 'closed', 'reconciled'
  notes: text('notes'),
  
  ...auditFields,
});

// POS order (extend orders table)
export const posOrders = pgTable('pos_orders', {
  id: uuid('id').primaryKey().defaultRandom(),
  orderId: uuid('order_id').notNull().references(() => orders.id),
  
  terminalId: uuid('terminal_id').notNull().references(() => posTerminals.id),
  shiftId: uuid('shift_id').notNull().references(() => posShifts.id),
  cashierId: uuid('cashier_id').notNull(),
  
  tableNumber: text('table_number'),               // Số bàn (nếu dine-in)
  serviceType: text('service_type').notNull(),     // 'takeaway', 'dine_in', 'delivery'
  
  // E-invoice
  einvoiceId: uuid('einvoice_id'),                 // FK to e_invoices
  einvoiceIssuedAt: timestamp('einvoice_issued_at', { withTimezone: true }),
  
  ...auditFields,
});

// Order item với customization
export const orderItemCustomizations = pgTable('order_item_customizations', {
  id: uuid('id').primaryKey().defaultRandom(),
  orderItemId: uuid('order_item_id').notNull().references(() => orderItems.id),
  
  type: text('type').notNull(),                     // 'sugar', 'ice', 'topping', 'note'
  value: text('value').notNull(),                   // '50%', 'less', 'thach den', ...
  additionalPrice: decimal('additional_price', { precision: 15, scale: 2 }).default('0'),
});
```

---

## Recipe-based pricing

```typescript
async calculatePosItemPrice(productId: string, variantId: string, customizations: Customization[]) {
  // 1. Get base price từ variant
  const variant = await this.productsRepo.findVariant(variantId);
  let price = new Decimal(variant.price);
  
  // 2. Add customization prices (topping, extra shot)
  for (const cust of customizations) {
    if (cust.type === 'topping') {
      const toppingPrice = await this.getToppingPrice(cust.value);
      price = price.plus(toppingPrice);
    }
    // Sugar/ice không phụ thu
  }
  
  return price.toString();
}

// Calculate cost (cho profit margin report)
async calculatePosItemCost(productId: string, variantId: string) {
  const product = await this.productsRepo.findById(productId);
  if (!product.recipe_id) return new Decimal(0);
  
  const recipe = await this.recipesRepo.findWithIngredients(product.recipe_id);
  const variant = await this.productsRepo.findVariant(variantId);
  const multiplier = new Decimal(variant.multiplier ?? 1);
  
  let totalCost = new Decimal(0);
  
  for (const ingredient of recipe.ingredients) {
    const adjustedQty = new Decimal(ingredient.quantity).times(multiplier);
    
    let unitCost: Decimal;
    if (ingredient.ingredient_type === 'raw_material') {
      const rm = await this.rawMaterialsRepo.findById(ingredient.ingredient_id);
      unitCost = new Decimal(rm.unit_cost);
    } else {
      // Semi-finished — tính đệ quy hoặc cache
      unitCost = await this.calculateSfpCost(ingredient.ingredient_id);
    }
    
    totalCost = totalCost.plus(adjustedQty.times(unitCost));
  }
  
  return totalCost;
}
```

---

## Tạo order POS + auto-issue e-invoice

```typescript
async createPosOrder(input: CreatePosOrderInput, cashier: AuthUser) {
  return this.db.transaction(async (tx) => {
    // 1. Verify shift đang mở
    const shift = await this.shiftsRepo.findOpenShift(input.terminal_id, tx);
    if (!shift) {
      throw new BadRequestException('Chưa mở ca. Mở ca trước khi tạo đơn.');
    }
    
    if (shift.cashier_id !== cashier.id) {
      throw new ForbiddenException('Ca đang được mở bởi nhân viên khác');
    }
    
    // 2. Calculate prices
    const orderItems = await Promise.all(input.items.map(async (item) => {
      const price = await this.calculatePosItemPrice(item.product_id, item.variant_id, item.customizations);
      const cost = await this.calculatePosItemCost(item.product_id, item.variant_id);
      
      return {
        ...item,
        price_at_order: price,
        cost_at_order: cost.toString(),
        subtotal: new Decimal(price).times(item.quantity).toString(),
      };
    }));
    
    const subtotal = orderItems.reduce((sum, i) => sum.plus(i.subtotal), new Decimal(0));
    const taxAmount = subtotal.times(0.08);  // 8% VAT cho F&B
    const total = subtotal.plus(taxAmount).minus(input.discount_amount ?? 0);
    
    // 3. Create base order
    const [order] = await tx.insert(orders).values({
      tenant_id: this.tenantId,
      type: 'pos',
      code: await this.generatePosOrderCode(tx),
      customer_id: input.customer_id,
      customer_snapshot: input.customer_id ? await this.snapshotCustomer(input.customer_id) : null,
      status: 'paid',  // POS thường thanh toán ngay
      subtotal: subtotal.toString(),
      tax_amount: taxAmount.toString(),
      discount_amount: input.discount_amount ?? '0',
      total: total.toString(),
      currency: 'VND',
      payment_method: input.payment_method,
      created_by: cashier.id,
    }).returning();
    
    // 4. Insert items + customizations
    for (const item of orderItems) {
      const [orderItem] = await tx.insert(orderItems).values({
        order_id: order.id,
        ...item,
      }).returning();
      
      if (item.customizations?.length > 0) {
        await tx.insert(orderItemCustomizations).values(
          item.customizations.map(c => ({ order_item_id: orderItem.id, ...c }))
        );
      }
    }
    
    // 5. Create POS-specific record
    await tx.insert(posOrders).values({
      order_id: order.id,
      terminal_id: input.terminal_id,
      shift_id: shift.id,
      cashier_id: cashier.id,
      table_number: input.table_number,
      service_type: input.service_type,
    });
    
    // 6. Update shift totals
    await tx.update(posShifts).set({
      total_orders: sql`${posShifts.total_orders} + 1`,
      total_revenue: sql`${posShifts.total_revenue} + ${total.toString()}::decimal`,
      [`total_${input.payment_method}`]: sql`${posShifts[`total_${input.payment_method}`]} + ${total.toString()}::decimal`,
    }).where(eq(posShifts.id, shift.id));
    
    // 7. Deduct inventory (auto via recipe BOM)
    await this.inventoryService.deductFromRecipe(orderItems, tx);
    
    // 8. Outbox event for async e-invoice issuance
    await tx.insert(outboxEvents).values({
      aggregate_type: 'PosOrder',
      aggregate_id: order.id,
      event_type: 'pos.order.completed',
      payload: { order_id: order.id, terminal_id: input.terminal_id },
    });
    
    return order;
  });
}

// Background worker: issue e-invoice
@OnEvent('pos.order.completed')
async issueEInvoiceForPosOrder(event: { order_id: string; terminal_id: string }) {
  // Theo NĐ 70/2025 — DN có doanh thu > 1 tỷ phải dùng HĐĐT
  // Theo TT 32/2025 — POS phải push lên CQT trong 1 ngày
  
  const order = await this.ordersRepo.findById(event.order_id);
  const terminal = await this.terminalsRepo.findById(event.terminal_id);
  
  const invoicePayload = {
    invoice_type: 'pos',  // Hoá đơn từ máy tính tiền
    terminal_id: terminal.cqt_terminal_id,
    serial: terminal.einvoice_serial,
    template: terminal.einvoice_template,
    
    seller: {
      tax_code: '0313334177',
      name: 'CÔNG TY TNHH VUA AN TOÀN',
      address: 'Bà Điểm, TP.HCM',
    },
    buyer: order.customer_snapshot ? {
      tax_code: order.customer_snapshot.tax_code,
      name: order.customer_snapshot.name,
      address: order.customer_snapshot.address,
    } : null,  // B2C có thể không có buyer info
    
    items: await this.formatItemsForEInvoice(order.id),
    
    payment_method: order.payment_method,
    subtotal: order.subtotal,
    tax_amount: order.tax_amount,
    discount_amount: order.discount_amount,
    total: order.total,
  };
  
  // Call MISA meInvoice API
  const response = await this.misaClient.issueInvoice(invoicePayload);
  
  // Save
  const [eInvoice] = await this.db.insert(eInvoices).values({
    order_id: order.id,
    invoice_number: response.invoice_number,
    invoice_code: response.invoice_code,
    serial: response.serial,
    issued_at: new Date(),
    cqt_code: response.cqt_code,
    misa_id: response.id,
    qr_code: response.qr_code,
    pdf_url: response.pdf_url,
    status: 'issued',
    raw_response: response,
  }).returning();
  
  // Link to POS order
  await this.db.update(posOrders).set({
    einvoice_id: eInvoice.id,
    einvoice_issued_at: new Date(),
  }).where(eq(posOrders.order_id, order.id));
  
  // Print receipt + e-invoice QR
  await this.printService.printReceipt(order.id, eInvoice.id);
}
```

---

## Đóng ca + reconcile

```typescript
async closeShift(shiftId: string, input: CloseShiftInput, cashier: AuthUser) {
  return this.db.transaction(async (tx) => {
    const shift = await this.shiftsRepo.findById(shiftId, tx);
    
    if (shift.status !== 'open') {
      throw new BadRequestException('Ca đã đóng');
    }
    if (shift.cashier_id !== cashier.id) {
      throw new ForbiddenException('Không phải ca của bạn');
    }
    
    // Tính expected cash
    const expectedCash = new Decimal(shift.opening_cash).plus(shift.total_cash);
    const actualCash = new Decimal(input.actual_cash);
    const difference = actualCash.minus(expectedCash);
    
    await tx.update(posShifts).set({
      closed_at: new Date(),
      closing_cash: actualCash.toString(),
      expected_cash: expectedCash.toString(),
      cash_difference: difference.toString(),
      status: 'closed',
      notes: input.notes,
    }).where(eq(posShifts.id, shiftId));
    
    // Alert nếu chênh lệch lớn
    if (difference.abs().gt(50000)) {  // > 50k chênh
      await this.alertManager(`Ca ${shift.shift_number} chênh ${difference.toString()} VND`);
    }
    
    // Generate Z-report (báo cáo tổng hợp ca)
    const report = await this.generateZReport(shiftId, tx);
    
    return { shift, report };
  });
}
```

---

## Anti-patterns

- ❌ Tính cost từ price (sai logic — cost từ BOM)
- ❌ Cho tạo order không có shift mở
- ❌ E-invoice issue đồng bộ trong request (block UX, fail thì mất đơn)
- ❌ Hard-code VAT 10% (F&B là 8%, có loại 5%)
- ❌ POS terminal không đăng ký CQT vẫn issue được hoá đơn
- ❌ Đóng ca không tính chênh lệch
- ❌ Recipe variant multiplier không áp dụng → cost sai

---

**END Skill: POS F&B**
