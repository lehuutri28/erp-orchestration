---
name: database-migration
description: Tạo migration phức tạp, expand-contract pattern, RLS migration, data migration với millions of rows.
when_to_use:
  - Đổi schema bảng đang có data production
  - Migrate data
  - Bật RLS cho bảng mới
  - Restructure schema
---

# Database Migration Skill

## 1. Quy trình migration an toàn

```
1. PLAN
   □ Document goal + before/after schema
   □ Estimate migration time với production data size
   □ Identify breaking changes

2. WRITE
   □ Forward migration (.sql)
   □ Down migration (.down.sql)
   □ Update MIGRATION_REGISTRY.md

3. TEST
   □ Run on local DB clone
   □ Run on staging
   □ Verify rollback works
   □ Time it (estimated downtime)

4. REVIEW
   □ PR with breaking change detection
   □ Approver check checklist

5. DEPLOY
   □ Backup DB
   □ Maintenance window (nếu cần)
   □ Run migration
   □ Verify health
   □ Smoke test
   □ Keep backup 30 ngày
```

## 2. Expand-Contract pattern

Đổi schema không downtime cần 3 deploy:

### Đổi tên column

```sql
-- DEPLOY 1: 0070_add_full_name.sql
ALTER TABLE users ADD COLUMN full_name TEXT;
UPDATE users SET full_name = first_name || ' ' || last_name;

-- Code: ghi vào CẢ first_name+last_name VÀ full_name
-- Code: đọc full_name nếu có, else fallback first_name+last_name

-- DEPLOY 2 (sau khi DEPLOY 1 stable): 0071_make_full_name_required.sql
ALTER TABLE users ALTER COLUMN full_name SET NOT NULL;
-- Code: chỉ dùng full_name

-- DEPLOY 3 (sau 1 tuần): 0072_drop_old_name_columns.sql
ALTER TABLE users DROP COLUMN first_name;
ALTER TABLE users DROP COLUMN last_name;
```

### Đổi type column (vd: INT → BIGINT)

```sql
-- DEPLOY 1
ALTER TABLE orders ADD COLUMN total_v2 BIGINT;
UPDATE orders SET total_v2 = total::BIGINT;
-- Trigger để sync 2 cột:
CREATE TRIGGER sync_total BEFORE INSERT OR UPDATE ON orders
FOR EACH ROW EXECUTE FUNCTION sync_total_v1_v2();

-- DEPLOY 2: code đọc từ total_v2
-- DEPLOY 3: drop total, rename total_v2 → total
```

## 3. Data migration cho large tables

```sql
-- Migration cho bảng > 10M rows — chunked
DO $$
DECLARE
  batch_size INT := 10000;
  rows_updated INT;
BEGIN
  LOOP
    UPDATE orders
    SET status = 'completed'
    WHERE id IN (
      SELECT id FROM orders
      WHERE status = 'shipped' AND shipped_at < NOW() - INTERVAL '7 days'
      LIMIT batch_size
    );
    
    GET DIAGNOSTICS rows_updated = ROW_COUNT;
    EXIT WHEN rows_updated = 0;
    
    COMMIT;  -- Release locks
    PERFORM pg_sleep(0.5);  -- Reduce load
  END LOOP;
END $$;
```

## 4. RLS migration template

```sql
-- 0080_enable_rls_orders.sql

-- Enable RLS
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders FORCE ROW LEVEL SECURITY;

-- Policy: tenant isolation
CREATE POLICY tenant_isolation_orders ON orders

> 📎 **Cross-ref:** RLS policy + service_role bypass canonical xem [rules/02-multi-tenant.md](../../rules/02-multi-tenant.md#lớp-4-rls)
  USING (tenant_id = current_setting('app.current_tenant', true))
  WITH CHECK (tenant_id = current_setting('app.current_tenant', true));

-- Policy: service role bypass
CREATE POLICY service_role_bypass_orders ON orders
  TO service_role
  USING (true)
  WITH CHECK (true);

-- Down: 0080_enable_rls_orders.down.sql
DROP POLICY IF EXISTS tenant_isolation_orders ON orders;
DROP POLICY IF EXISTS service_role_bypass_orders ON orders;
ALTER TABLE orders DISABLE ROW LEVEL SECURITY;
```

## 5. CẤM trong migration 🔴

- DROP TABLE without backup
- DROP COLUMN có data quan trọng (làm expand-contract)
- ALTER COLUMN ... TYPE trực tiếp (lock table)
- Tạo index không CONCURRENTLY trên bảng lớn
- Migration không có .down.sql
- Sửa migration đã merge

## 6. CONCURRENTLY index (cho production)

```sql
-- ❌ Lock bảng
CREATE INDEX orders_status_idx ON orders (status);

-- ✅ Không lock, chậm hơn nhưng safe
CREATE INDEX CONCURRENTLY orders_status_idx ON orders (status);
```

**Lưu ý:** CONCURRENTLY không chạy được trong transaction. Tách ra migration riêng.

## 7. Test migration trước khi merge

```bash
# Clone production DB local
pg_dump -h prod.wecha.vn -U readonly erp | psql erp_test

# Run migration
DATABASE_URL=postgres://localhost/erp_test pnpm db:migrate

# Verify schema
pnpm db:diff

# Test rollback
pnpm db:migrate:down
pnpm db:diff  # Should show NO diff với original
```
