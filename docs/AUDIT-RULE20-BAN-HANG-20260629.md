# AUDIT RULE-20 HỆ BÁN HÀNG — 2026-06-29 (workflow 6 nhóm)

> Tổng **65 gap**: 10 blocker, 22 high, 23 medium, 10 low. prod=aff3b5d1.

## BLOCKER (10)
| Nhóm | Trang | Chức năng | Fix |
|---|---|---|---|
| don-hang | orders/page.tsx + orders.dto.ts | load/filter - CSDL axis | Thêm @IsOptional() @IsUUID() database_id?: string vào OrderQueryDto. Trong findAll service thêm: if (query.database_id) conditions.push(databaseFilter(orders, query.database_id)). |
| don-hang | sales/pending-approval/page.tsx | load/filter - CSDL axis | Import useDatabase(), lấy activeDatabaseId, append &database_id=${activeDatabaseId} vào qs. Cần fix OrderQueryDto (gap#1) trước. |
| don-hang | online-orders/page.tsx | load/filter - CSDL axis | Import useDatabase(), thêm database_id: activeDatabaseId vào fetch params. Hoặc FE set header x-database-id nhất quán với mọi request. |
| bao-gia | quotes/new/page.tsx + quotes/[id]/edit/page.tsx | create | Đồng bộ FE-BE: BE service.ts:90 đổi thành `(item.discount??0)/100 * item.unitPrice * item.quantity` để nhận %. Cập nhật DTO comment cho rõ đơn vị %. |
| b2b | b2b/page.tsx | filter | Move paymentStatus, date_from, date_to, branch_id to server-side query params; re-fetch on change. |
| b2b | b2b/[id]/page.tsx | action | Change cancel to call DELETE /orders/:id to use the orders.cancel permission guard. |
| b2b | b2b/new/page.tsx | create | FE convert before send: discount: unitPrice*quantity*i.discount/100 (VND). Or add discountType to DTO and handle in service. |
| km-voucher-the | promotions/page.tsx | load | Apply migration to align prod column names with code; remove silent catch-return-empty workaround. |
| dashboard-report-attr | admin/sales/reports/sales-by-channel | filter | Change dep array to [branchId, startDate, endDate] so any filter change auto-reloads chart and KPI. |
| dashboard-report-attr | admin/sales/barcodes | create | Replace with product search combobox (GET /admin/sales/products?q=...) returning name+sku; store productId UUID internally. |

## HIGH (22)
| Nhóm | Trang | Chức năng | Fix |
|---|---|---|---|
| don-hang | sales/pending-approval/page.tsx | load/filter - Branch axis | Import useDatabase(), lấy activeBranchId, append &branch_id=${activeBranchId} vào qs. BE service.ts:1519 đã có if (query.branch_id). |
| don-hang | orders/page.tsx | filter - status & search không gửi BE | Trong buildParams() thêm: if (statusFilter!=='all') params.set('status', statusFilter) và if (search.trim()) params.set('search', search.trim()). Trigger refetch khi filter thay đổ |
| don-hang | online-orders/page.tsx | load - branch guard chặn trang | Dùng activeBranchId từ useDatabase() (line 51/76). Bỏ guard cứng, load tất cả khi không có branch. |
| don-hang | sales/process/page.tsx | action - stage links 404 | Đổi href sang /admin/orders hoặc /admin/online-orders tùy stage, kèm ?status=... để pre-filter. |
| bao-gia | quotes/new/page.tsx | create | Thêm `@IsOptional() @IsEnum(['draft','sent']) status?:string` vào CreateQuoteDto và service.ts đọc `dto.status??'draft'`. Hoặc tạo endpoint POST /quotes/:id/send riêng. |
| bao-gia | quotes/page.tsx | filter | Đổi FE page.tsx:137 thành `params.set('branchId', activeBranchId)`. |
| bao-gia | quotes/[id]/edit/page.tsx | edit | Thêm `@IsOptional() @IsUUID() branchId?:string` vào UpdateQuoteDto và xử lý trong service update(). |
| b2b | b2b/page.tsx | export | Fetch all matching rows via GET /orders?order_type=wholesale&limit=9999 before export, or wire a dedicated bulk-export endpoint. |
| b2b | b2b/page.tsx | axis-csdl | Add database_id to OrderQueryDto, apply databaseFilter in findAll(), pass activeDatabaseId from FE useDatabase() hook. |
| b2b | b2b/[id]/page.tsx | action | Wire VAT button to POST /orders/:id/vat-export. Hide when user lacks orders.export_vat. Mark print as not-yet-implemented. |
| b2b | b2b/[id]/page.tsx | edit | Add edit drawer for editable fields (receiver, delivery date, notes, shipping fee). Gate on orders.edit. Call PATCH /orders/:id. |
| km-voucher-the | promotions/page.tsx | delete | Add deleted_at+deleted_by to table; set in softDelete(); filter isNull(deleted_at) in findAll(). Add thùng rác/restore per CONTRACT mục 1. |
| km-voucher-the | vouchers/campaigns/page.tsx | filter | Pass filter params to BE; add server-side isActive/date filter in listCampaigns(); add name search param. |
| km-voucher-the | vouchers/campaigns/page.tsx | edit | Add edit button per row; build modal with name/value/maxDiscount/minOrder/endsAt/isActive fields; wire to PATCH endpoint. |
| km-voucher-the | gift-cards/page.tsx | filter | Add PENDING to STATUS_BADGE map and to status filter dropdown. |
| datban-trahang | reservations | axis-branch | Đọc currentBranchId từ auth context (như returns/page.tsx:163) append &branch_id=... vào load() + stats. |
| datban-trahang | returns | filter | Xoá option approved khỏi STATUS_OPTIONS dòng 69 và STATUS map dòng 63. |
| dashboard-report-attr | admin/sales/attributes | edit | Add edit modal for groups (name, slug, sortOrder) and values (value, code, hexColor). Wire to existing PATCH endpoints. |
| dashboard-report-attr | admin/sales/attributes | delete | Add DELETE BE endpoints for groups and values (soft delete + tenant guard). Add delete buttons in FE with GlassConfirmDialog. |
| dashboard-report-attr | admin/sales/barcodes | edit | Add edit modal for barcodeType/notes/isPrimary fields; barcode string requires delete+recreate to preserve GS1 integrity. |
| dashboard-report-attr | admin/sales/barcodes | delete | Add DELETE /admin/sales/barcodes/:id BE endpoint with tenant guard. Add delete button in FE row with confirmation dialog. |
| dashboard-report-attr | admin/sales/dashboard | filter | Add requiresApproval=true query param to orders BE, or add GET /orders/pending-approval count endpoint. |

## MEDIUM (23)
| Nhóm | Trang | Chức năng | Fix |
|---|---|---|---|
| don-hang | orders/page.tsx | action - cancel thiếu permission check | Wrap nút: {hasPermission('orders.cancel') && status!=='cancelled' && <CancelButton />}. |
| don-hang | sales/pending-approval/page.tsx | action - approve/reject thiếu permission check | Wrap: {hasPermission('orders.approve') && <ApproveButton />} và {hasPermission('orders.reject') && <RejectButton />}. |
| don-hang | sales/process/page.tsx | filter - stats thiếu CSDL axis | Append &database_id=${activeDatabaseId} vào stats query. |
| bao-gia | quotes/[id]/page.tsx + quotes/page.tsx | delete | Đổi FE thành `canDelete = ['draft','rejected'].includes(status)` tại [id]/page.tsx:184 và page.tsx:528. |
| bao-gia | quotes/[id]/page.tsx | axis-permission | Đọc permissions từ auth context. Bổ sung điều kiện: `canApprove = status==='sent' && hasPermission('quotes.approve')` tương tự cho các nút còn lại. |
| b2b | b2b/[id]/page.tsx | axis-permission | Wrap VAT export button with orders.export_vat permission check using usePermissions hook. |
| km-voucher-the | promotions/page.tsx | axis-csdl | Add database_id param in FE load() and eq(promotions.databaseId, databaseId) condition in service. |
| km-voucher-the | promotions/page.tsx | axis-branch | Add optional branch_id filter in load() + findAll(), or document promotions are tenant-wide. |
| km-voucher-the | vouchers/campaigns/page.tsx | axis-csdl | Add database_id param in load() and filter in listCampaigns(). |
| km-voucher-the | vouchers/campaigns/page.tsx | axis-branch | Add branch_id filter in FE load() and BE listCampaigns() if campaigns are branch-scoped. |
| km-voucher-the | vouchers/codes/page.tsx | uistate | BE getCampaignCodes() should join customers table and return customerName; FE renders name. |
| km-voucher-the | gift-cards/page.tsx | action | Add Recharge button on cards; build amount+note modal; wire to POST /gift-cards/:id/recharge. |
| km-voucher-the | gift-cards/page.tsx | axis-csdl | Add database_id param in FE load(); add database_id filter in controller list(). |
| km-voucher-the | gift-cards/page.tsx | axis-branch | Add optional branch_id filter in FE load() and BE list(). |
| km-voucher-the | vouchers/campaigns/page.tsx + gift-cards/page.tsx | export | Add CSV/Excel BOM UTF-8 export to both pages exporting filtered results. |
| km-voucher-the | vouchers/campaigns/page.tsx | axis-permission | Read user permissions from auth context; conditionally hide Create/Generate/Delete buttons per vouchers.manage + vouchers.delete. |
| datban-trahang | reservations | load | Thêm isNull(commerceReservations.deletedAt) vào getStats() giống findAll() dòng 26. |
| datban-trahang | reservations | action | Thêm nút Nhận cọc (hiện khi !depositPaid và active/confirmed) gọi POST confirm-deposit. |
| datban-trahang | returns | delete | Thêm nút Xoá (pending/rejected) với promptDialog lý do, gọi DELETE /admin/sales/returns/{id}. |
| dashboard-report-attr | admin/sales/dashboard | load | Add separate fetch for GET /orders?limit=10&sort=createdAt:desc (no status filter) to populate recentOrders. |
| dashboard-report-attr | admin/sales/dashboard | export | Wire to GET /reports/export-excel?branch_id=&from=&to= (reports.controller.ts:170) for proper multi-sheet Excel export. |
| dashboard-report-attr | admin/commerce | load | Use meta.total from BE response, or add GET /inventory/low-stock/count endpoint returning integer total. |
| dashboard-report-attr | admin/commerce | load | Add GET /admin/sales/promotions/stats endpoint returning isActive count. Avoid fetching full list for a KPI card. |

## LOW (10)
| Nhóm | Trang | Chức năng | Fix |
|---|---|---|---|
| don-hang | online-orders/page.tsx | export - gọi lại API thừa | Dùng orders state để build Excel. Nếu cần full data thì gọi với limit=9999. |
| km-voucher-the | vouchers/codes/page.tsx | filter | Add code search input; pass q param to getCampaignCodes(); add ILIKE filter on vouchers.code in BE. |
| km-voucher-the | gift-cards/page.tsx | uistate | Add breadcrumb: Home → Ban hang → The qua tang. |
| datban-trahang | reservations | load | FE normalize: Array.isArray(d)?d:(d?.data??[]) hoặc controller wrap {data:result}. |
| datban-trahang | returns | axis-permission | Ẩn nút khi user thiếu returns.approve/reject từ useAuth() permissions. |
| dashboard-report-attr | admin/commerce | load | BE getDashboard should return {data:{...}} per CONTRACT §6. FE reads res.data unconditionally. |
| dashboard-report-attr | admin/sales/reports/by-region | action | Verify /orders/sales-by-region exists in orders.controller.ts with @RequirePermissions guard. Add if missing. |
| dashboard-report-attr | admin/sales/attributes | axis-permission | Seed product_attributes.view and product_attributes.manage; assign to appropriate roles. |
| dashboard-report-attr | admin/sales/barcodes | axis-permission | Seed barcodes.view, barcodes.create, barcodes.manage; assign to appropriate roles. |
| dashboard-report-attr | admin/sales/reports/sales-by-channel | load | Initialize branchId with currentBranchId in useState, or merge into single useEffect([currentBranchId, startDate, endDate]). |
