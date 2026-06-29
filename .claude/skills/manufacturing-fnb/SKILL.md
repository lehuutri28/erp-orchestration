# Skill: Manufacturing F&B + Máy móc CMMS

> Khi nào load: Build module nguyên liệu, BOM, sản xuất, bán + bảo trì máy móc pha chế

> ⚠️ Đồng bộ ARCHITECTURE_CONTRACT 29/5/2026: PK=uuid().primaryKey().defaultRandom(), tenant_id=uuid, TS field camelCase, soft delete thùng rác. Xem docs/ARCHITECTURE_CONTRACT.md.

---

## Domain context

CÔNG TY TNHH VUA AN TOÀN — đa nghiệp vụ:

1. **SX nguyên liệu F&B**: bột trà, syrup, bột topping, hương liệu (bán cho quán + WECHA)
2. **Bán máy pha chế**: máy ép, máy xay, máy đánh kem, blender (bán B2B)
3. **Bảo trì máy** (CMMS): contract định kỳ + on-demand

---

## Schema 7 lớp F&B

```
Lớp 1: raw_materials (nguyên liệu thô — bột, đường, hương)
   ↓ (BOM)
Lớp 2: semi_finished_products (bán thành phẩm — syrup, mix bột)
   ↓ (BOM)
Lớp 3: recipes (công thức pha chế chi tiết)
   ↓
Lớp 4: products (sản phẩm bán cho khách — có size variants)
   ↓
Lớp 5: menus (menu theo cửa hàng/franchise)
   ↓
Lớp 6: course_kits (kit nguyên liệu cho LMS)
   ↓
Lớp 7: sales_orders (đơn hàng — snapshot price)
```

```typescript
// Lớp 1: Nguyên liệu thô
export const rawMaterials = pgTable('raw_materials', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  
  sku: text('sku').notNull(),
  name: text('name').notNull(),
  category: text('category'),                   // 'powder', 'syrup', 'topping', 'flavor'
  
  unit: text('unit').notNull(),                 // 'kg', 'g', 'l', 'ml', 'pcs'
  unitCost: decimal('unit_cost', { precision: 15, scale: 4 }).notNull(),
  
  // Quản lý lô + hạn dùng
  batchTracking: boolean('batch_tracking').notNull().default(true),
  shelfLifeDays: integer('shelf_life_days'),  // Hạn sử dụng
  storageCondition: text('storage_condition'), // 'room_temp', 'refrigerated', 'frozen'
  
  minStock: decimal('min_stock', { precision: 15, scale: 4 }),
  reorderPoint: decimal('reorder_point', { precision: 15, scale: 4 }),
  
  supplierId: uuid('supplier_id').references(() => suppliers.id),
  
  ...auditFields,
}, (t) => ({
  uniqSku: uniqueIndex('rm_tenant_sku_unique').on(t.tenantId, t.sku),
}));

// Lớp 2: Bán thành phẩm (có BOM từ raw materials)
export const semiFinishedProducts = pgTable('semi_finished_products', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  
  sku: text('sku').notNull(),
  name: text('name').notNull(),
  
  // Production
  yieldQuantity: decimal('yield_quantity', { precision: 15, scale: 4 }).notNull(),  // Lượng sản xuất 1 lần
  yieldUnit: text('yield_unit').notNull(),
  prepTimeMinutes: integer('prep_time_minutes'),
  
  shelfLifeHours: integer('shelf_life_hours'),  // Bán thành phẩm thường ngắn hạn
  
  ...auditFields,
});

// BOM cho bán thành phẩm
export const sfpBom = pgTable('sfp_bom', {
  id: uuid('id').primaryKey().defaultRandom(),
  sfpId: uuid('sfp_id').notNull().references(() => semiFinishedProducts.id),
  rawMaterialId: uuid('raw_material_id').notNull().references(() => rawMaterials.id),
  quantity: decimal('quantity', { precision: 15, scale: 4 }).notNull(),
  unit: text('unit').notNull(),
  notes: text('notes'),
}, (t) => ({
  uniqBom: uniqueIndex('sfp_bom_unique').on(t.sfpId, t.rawMaterialId),
}));

// Lớp 3: Công thức pha chế
export const recipes = pgTable('recipes', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  
  code: text('code').notNull(),                 // RCP-001
  name: text('name').notNull(),                 // 'Trà sữa truyền thống'
  category: text('category'),                   // 'milk_tea', 'coffee', 'fruit_tea'
  
  servingSize: text('serving_size'),           // 'M', 'L', 'XL'
  prepTimeSeconds: integer('prep_time_seconds'),
  difficulty: text('difficulty'),               // 'easy', 'medium', 'hard'
  
  instructions: text('instructions'),           // Hướng dẫn pha chế
  videoUrl: text('video_url'),
  
  isActive: boolean('is_active').notNull().default(true),
  
  ...auditFields,
});

export const recipeIngredients = pgTable('recipe_ingredients', {
  id: uuid('id').primaryKey().defaultRandom(),
  recipeId: uuid('recipe_id').notNull().references(() => recipes.id),
  
  ingredientType: text('ingredient_type').notNull(),  // 'raw_material' | 'semi_finished'
  ingredientId: uuid('ingredient_id').notNull(),
  
  quantity: decimal('quantity', { precision: 15, scale: 4 }).notNull(),
  unit: text('unit').notNull(),
  
  isOptional: boolean('is_optional').notNull().default(false),
  notes: text('notes'),
});

// Lớp 4: Sản phẩm bán
export const products = pgTable('products', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  
  sku: text('sku').notNull(),
  name: text('name').notNull(),
  description: text('description'),
  categoryId: uuid('category_id'),
  
  type: text('type').notNull(),                 // 'beverage', 'food', 'machine', 'material', 'service'
  
  recipeId: uuid('recipe_id').references(() => recipes.id),  // Nếu là beverage có recipe
  
  vatRate: decimal('vat_rate', { precision: 4, scale: 2 }).notNull().default('8'),  // 8% cho F&B
  
  isActive: boolean('is_active').notNull().default(true),
  
  ...auditFields,
});

// Variants (size, topping)
export const productVariants = pgTable('product_variants', {
  id: uuid('id').primaryKey().defaultRandom(),
  productId: uuid('product_id').notNull().references(() => products.id),
  
  variantName: text('variant_name').notNull(),  // 'Size M', 'Size L'
  sku: text('sku').notNull(),
  
  price: decimal('price', { precision: 15, scale: 2 }).notNull(),
  costPrice: decimal('cost_price', { precision: 15, scale: 2 }),
  
  multiplier: decimal('multiplier', { precision: 4, scale: 2 }).default('1'),  // Size L = 1.3x recipe
  
  isDefault: boolean('is_default').notNull().default(false),
});
```

---

## FIFO/FEFO inventory

```typescript
// Lô hàng — track từng batch riêng
export const inventoryBatches = pgTable('inventory_batches', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  
  rawMaterialId: uuid('raw_material_id').notNull().references(() => rawMaterials.id),
  warehouseId: uuid('warehouse_id').notNull(),
  
  batchNumber: text('batch_number').notNull(),
  manufacturedDate: date('manufactured_date'),
  expiryDate: date('expiry_date'),
  
  quantityReceived: decimal('quantity_received', { precision: 15, scale: 4 }).notNull(),
  quantityRemaining: decimal('quantity_remaining', { precision: 15, scale: 4 }).notNull(),
  
  unitCost: decimal('unit_cost', { precision: 15, scale: 4 }).notNull(),
  
  supplierId: uuid('supplier_id'),
  goodsReceiptId: uuid('goods_receipt_id'),
  
  ...auditFields,
}, (t) => ({
  // FEFO query: ORDER BY expiry_date ASC
  fefoIdx: index('batches_fefo_idx').on(t.rawMaterialId, t.warehouseId, t.expiryDate),
}));

// Service: xuất kho theo FEFO (First Expiry First Out)
async issueByFEFO(rawMaterialId: string, warehouseId: string, quantity: Decimal) {
  const batches = await this.db.query.inventoryBatches.findMany({
    where: and(
      eq(inventoryBatches.raw_material_id, rawMaterialId),
      eq(inventoryBatches.warehouse_id, warehouseId),
      gt(inventoryBatches.quantity_remaining, '0'),
      gte(inventoryBatches.expiry_date, new Date()),  // Chưa hết hạn
    ),
    orderBy: asc(inventoryBatches.expiry_date),
  });
  
  let remaining = quantity;
  const issued = [];
  
  for (const batch of batches) {
    if (remaining.lte(0)) break;
    
    const issueQty = Decimal.min(remaining, new Decimal(batch.quantity_remaining));
    
    await this.db.update(inventoryBatches)
      .set({ quantity_remaining: new Decimal(batch.quantity_remaining).minus(issueQty).toString() })
      .where(eq(inventoryBatches.id, batch.id));
    
    issued.push({
      batchId: batch.id,
      batchNumber: batch.batch_number,
      quantity: issueQty.toString(),
      unitCost: batch.unit_cost,
      expiryDate: batch.expiry_date,
    });
    
    remaining = remaining.minus(issueQty);
  }
  
  if (remaining.gt(0)) {
    throw new InsufficientStockError(rawMaterialId, quantity.toNumber(), quantity.minus(remaining).toNumber());
  }
  
  return issued;
}
```

---

## Máy pha chế — sale + CMMS

```typescript
// Máy bán cho khách
export const machines = pgTable('machines', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  
  serialNumber: text('serial_number').notNull(),
  model: text('model').notNull(),
  manufacturer: text('manufacturer'),
  
  productId: uuid('product_id').references(() => products.id),  // Tham chiếu catalogue
  
  // Sale info
  soldToCustomerId: uuid('sold_to_customer_id'),
  soldAt: timestamp('sold_at', { withTimezone: true }),
  soldPrice: decimal('sold_price', { precision: 15, scale: 2 }),
  warrantyMonths: integer('warranty_months'),
  warrantyEndDate: date('warranty_end_date'),
  
  // Maintenance
  installationAddress: text('installation_address'),
  installationDate: date('installation_date'),
  nextMaintenanceDate: date('next_maintenance_date'),
  
  status: text('status').notNull().default('in_stock'),  // 'in_stock', 'sold', 'returned', 'decommissioned'
  
  ...auditFields,
}, (t) => ({
  uniqSerial: uniqueIndex('machines_serial_unique').on(t.serialNumber),
}));

// Hợp đồng bảo trì
export const maintenanceContracts = pgTable('maintenance_contracts', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  
  contractNumber: text('contract_number').notNull(),
  customerId: uuid('customer_id').notNull(),
  
  startDate: date('start_date').notNull(),
  endDate: date('end_date').notNull(),
  
  serviceLevel: text('service_level').notNull(),  // 'basic', 'standard', 'premium'
  visitsPerYear: integer('visits_per_year').notNull(),
  responseTimeHours: integer('response_time_hours'),  // SLA
  
  totalValue: decimal('total_value', { precision: 15, scale: 2 }).notNull(),
  paymentTerms: text('payment_terms'),
  
  status: text('status').notNull().default('active'),
  
  ...auditFields,
});

export const maintenanceContractMachines = pgTable('maintenance_contract_machines', {
  id: uuid('id').primaryKey().defaultRandom(),
  contractId: uuid('contract_id').notNull().references(() => maintenanceContracts.id),
  machineId: uuid('machine_id').notNull().references(() => machines.id),
});

// Yêu cầu bảo trì (work order)
export const workOrders = pgTable('work_orders', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  
  woNumber: text('wo_number').notNull(),
  machineId: uuid('machine_id').notNull().references(() => machines.id),
  contractId: uuid('contract_id').references(() => maintenanceContracts.id),
  
  type: text('type').notNull(),  // 'preventive', 'corrective', 'emergency', 'warranty'
  priority: text('priority').notNull(),  // 'low', 'medium', 'high', 'urgent'
  
  reportedBy: text('reported_by'),
  reportedAt: timestamp('reported_at', { withTimezone: true }).notNull().defaultNow(),
  description: text('description').notNull(),
  symptoms: text('symptoms'),
  
  assignedTechnicianId: uuid('assigned_technician_id'),
  scheduledAt: timestamp('scheduled_at', { withTimezone: true }),
  
  arrivedAt: timestamp('arrived_at', { withTimezone: true }),
  completedAt: timestamp('completed_at', { withTimezone: true }),
  
  diagnosis: text('diagnosis'),
  workPerformed: text('work_performed'),
  partsUsed: jsonb('parts_used'),       // [{ part_id, qty, cost }]
  
  laborHours: decimal('labor_hours', { precision: 6, scale: 2 }),
  laborCost: decimal('labor_cost', { precision: 15, scale: 2 }),
  partsCost: decimal('parts_cost', { precision: 15, scale: 2 }),
  totalCost: decimal('total_cost', { precision: 15, scale: 2 }),
  
  customerSignature: text('customer_signature'),  // Base64 image hoặc URL
  customerRating: integer('customer_rating'),     // 1-5 sao
  
  status: text('status').notNull().default('open'),
  // 'open', 'assigned', 'in_progress', 'completed', 'cancelled'
  
  ...auditFields,
}, (t) => ({
  uniqWo: uniqueIndex('wo_tenant_number_unique').on(t.tenantId, t.woNumber),
  machineIdx: index('wo_machine_idx').on(t.machineId, t.reportedAt),
}));
```

---

## Auto-trigger preventive maintenance

```typescript
// Cron daily: tạo WO bảo trì định kỳ
@Cron('0 7 * * *')
async createPreventiveMaintenanceWOs() {
  const machinesDue = await this.db.query.machines.findMany({
    where: and(
      eq(machines.status, 'sold'),
      lte(machines.next_maintenance_date, addDays(new Date(), 7)),  // 1 tuần nữa
    ),
  });
  
  for (const machine of machinesDue) {
    const contract = await this.findActiveMaintenanceContract(machine.id);
    if (!contract) continue;
    
    await this.workOrdersService.create({
      machine_id: machine.id,
      contract_id: contract.id,
      type: 'preventive',
      priority: 'medium',
      reported_by: 'system',
      description: `Bảo trì định kỳ máy ${machine.model} - ${machine.serial_number}`,
      scheduled_at: machine.next_maintenance_date,
    });
    
    // Notify customer + assigned technician
    await this.notifyService.sendMaintenanceReminder(machine);
  }
}
```

---

## Anti-patterns

- ❌ Giá bán hard-code, không snapshot vào order_items
- ❌ Inventory không track theo batch (mất control hạn dùng)
- ❌ Không có FEFO → bán hàng cận hạn cuối cùng
- ❌ Recipe không versioning → đổi recipe ảnh hưởng đơn cũ
- ❌ Work order không có SLA tracking
- ❌ Bảo trì không link với contract → không tính hết visits
- ❌ Parts cost không track per WO → không biết margin

---

**END Skill: Manufacturing F&B + CMMS**
