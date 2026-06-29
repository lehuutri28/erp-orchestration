# 🔐 SƠ ĐỒ PHÂN QUYỀN HỆ BÁN HÀNG — chờ Letri DUYỆT

> Soạn: Opus CEO ERP · ⏰ 2026-06-29 07:18 +0700
> Nguồn (ground-truth, KHÔNG đoán): grep `@RequirePermissions` từng controller bán hàng (prod code) + query `roles.permissions` (DB prod port 5433) + `docs/rbac/PERMISSION-CATALOG-v1.md`.
> ⚠️ Letri DUYỆT sơ đồ này (sửa nếu cần) **rồi Opus mới seed** (migration kiểu 0421/0425). KHÔNG tự seed.

---

## 1. TL;DR — 3 vấn đề phát hiện

| # | Vấn đề | Mức | Hệ quả |
|---|--------|-----|--------|
| **A** | **LOCKOUT**: `promotions.*`, `quotes.*`, `returns.*`, `orders.create/edit/approve/reject` — **KHÔNG vai trò nào (trừ Admin `*`) được seed** → controller có PermissionsGuard nên các role khác **403** | 🔴 | Hiện **chỉ Admin** tạo/sửa/duyệt được khuyến mãi, báo giá, trả hàng, và tạo đơn sỉ (`/orders` từ trang B2B). Manager/Quản lý CN/NV bị chặn. *(POS hằng ngày đi `pos-drafts`/`pos-sessions` riêng → KHÔNG kẹt.)* |
| **B** | **OVER-GRANT**: `gift_cards.manage`, `barcodes.manage`, `reservations.manage`, `online_orders.manage`, `product_attributes.manage` được seed **đại trà** cho gần như MỌI role (kể cả Staff, Kế toán viên) | 🟠 | Kế toán viên/Staff đang "quản lý" thẻ quà/đặt bàn/đơn online — rộng hơn cần thiết. Siết = rủi ro vỡ luồng → để Letri quyết, ưu tiên thấp. |
| **C** | Quyền nằm trong **JWT** — sau khi seed, user phải **đăng nhập lại** mới có hiệu lực (bài học mig 0421). | 🟡 | Ghi rõ khi triển khai. |

> **Ưu tiên xử lý = A (THÊM quyền — an toàn, sửa lockout).** B (siết bớt) optional. THÊM trước, đăng nhập lại.

---

## 2. Quyền TỪNG controller YÊU CẦU (ground-truth prod code)

| Tài nguyên | Controller guard | Key quyền yêu cầu |
|---|---|---|
| Đơn hàng | ✅ | `orders.` create · view · view_detail · edit · cancel · approve · reject · export_vat |
| Báo giá | ✅ | `quotes.` read · create · edit · delete · approve · reject · convert |
| Khuyến mãi | ✅ | `promotions.` view · create · edit · delete · apply |
| Trả hàng | ✅ | `returns.` read · create · edit · delete · approve · reject |
| Thẻ quà | ✅ | `gift_cards.` view · use · manage |
| Voucher | ✅ | `vouchers.` view · use · manage · delete |
| Đặt bàn | ✅ | `reservations.` view · create · manage |
| Đơn online | ✅ | `online_orders.` view · confirm · manage · sync |
| Mã vạch | ✅ | `barcodes.` view · create · manage |
| Thuộc tính SP | ✅ | `product_attributes.` view · manage |

> Mọi controller bán hàng ĐÃ gắn PermissionsGuard (tốt) — nên thiếu seed = chặn thật.

---

## 3. SƠ ĐỒ ĐỀ XUẤT — vai trò × nhóm bán hàng

**Mức:** `—` không · `👁` Xem(+xuất Excel) · `✍` Thao tác (xem+tạo+sửa/dùng) · `✅` Duyệt (✍+duyệt/từ chối/convert/confirm) · `🔑` Toàn quyền (✅+xoá/manage)

| Vai trò | Đơn hàng | Báo giá | Khuyến mãi | Trả hàng | Thẻ quà | Voucher | Đặt bàn | Đơn online | Mã vạch | Thuộc tính |
|---|---|---|---|---|---|---|---|---|---|---|
| **Admin** | 🔑 | 🔑 | 🔑 | 🔑 | 🔑 | 🔑 | 🔑 | 🔑 | 🔑 | 🔑 |
| **Giám đốc** | 🔑 | 🔑 | 🔑 | ✅ | 🔑 | 🔑 | 🔑 | 🔑 | 🔑 | 🔑 |
| **Manager** | ✅ | ✅ | ✍ | ✅ | 🔑 | 🔑 | 🔑 | ✅ | 🔑 | 🔑 |
| **Quản lý chi nhánh** | ✅ | ✅ | 👁 | ✅ | ✍(use) | ✍(use) | 🔑 | ✅ | 🔑 | ✍ |
| **Kế toán trưởng** | 👁 | 👁 | 👁 | ✅* | 👁 | 👁 | 👁 | 👁 | — | 👁 |
| **Kế toán viên** | 👁 | 👁 | 👁 | 👁 | 👁 | 👁 | 👁 | 👁 | — | 👁 |
| **Nhân viên chi nhánh** | ✍ | ✍ | 👁 | ✍ | ✍(use) | ✍(use) | ✍ | 👁+confirm | ✍ | 👁 |
| **NV Bán hàng Online** | ✍ | — | 👁 | — | ✍(use) | ✍(use) | — | 🔑 | — | 👁 |
| **NV Chăm sóc KH** | 👁 | 👁 | 👁 | 👁 | 👁 | ✍(use) | 👁 | 👁 | — | — |
| **Staff** | 👁 | — | — | — | ✍(use) | ✍(use) | ✍ | 👁 | 👁 | 👁 |

> `*` Kế toán trưởng `✅` Trả hàng = được **duyệt phiếu trả/hoàn tiền** (vì gắn bút toán TT99), nhưng KHÔNG tạo đơn bán.
> Đây là ĐỀ XUẤT mặc định — Letri sửa ô nào tuỳ thực tế (vd: cho Kế toán viên duyệt trả hàng? Staff được tạo báo giá?).

---

## 4. THAY ĐỔI so với hiện tại (việc sẽ seed sau khi duyệt)

### 4A. THÊM (sửa LOCKOUT — ưu tiên 🔴)
- **Khuyến mãi:** `promotions.view/create/edit/apply` → Giám đốc, Manager; `promotions.delete` → Giám đốc; `promotions.view` → các role còn lại.
- **Báo giá:** `quotes.read/create/edit/convert` → Giám đốc, Manager, Quản lý CN, NV chi nhánh; `quotes.approve/reject` → Giám đốc, Manager, Quản lý CN; `quotes.delete` → Giám đốc.
- **Trả hàng:** `returns.read/create/edit` → role thao tác; `returns.approve/reject` → Giám đốc, Manager, Quản lý CN, **Kế toán trưởng**; `returns.delete` → Giám đốc.
- **Đơn hàng (admin/B2B):** `orders.create/edit` → Giám đốc, Manager, Quản lý CN, NV chi nhánh, NV Online; `orders.approve/reject` → Giám đốc, Manager, Quản lý CN.
- **Voucher:** `vouchers.view` rộng; `vouchers.use` → role bán; `vouchers.manage` → Giám đốc, Manager.

### 4B. SIẾT (over-grant — optional 🟠, Letri quyết)
- Cân nhắc gỡ `gift_cards.manage` / `reservations.manage` / `barcodes.manage` khỏi **Kế toán viên** và **Staff** (giữ `use`/`view`). *Rủi ro vỡ luồng → chỉ làm nếu Letri muốn siết; mặc định GIỮ NGUYÊN để an toàn.*

---

## 5. Kế hoạch triển khai (sau khi Letri duyệt)
1. Opus soạn 1 migration `04xx_seed_sales_perms.sql` (pattern 0421/0425): chỉ **UPDATE roles.permissions** thêm key, idempotent (không trùng).
2. Apply prod port **5433** TRƯỚC (không cần build/deploy — chỉ data).
3. **Thông báo user đăng nhập lại** (quyền trong JWT).
4. C8 click-test: mỗi role thử tạo/sửa/duyệt đúng phạm vi (Rule 20 trục QUYỀN).

## 6. Cần Letri quyết
- [ ] Duyệt/sửa **ma trận mục 3** (đặc biệt: Kế toán có được duyệt trả hàng? NV Online có tạo đơn `/orders`?).
- [ ] 4B có siết over-grant không (hay giữ nguyên)?
- [ ] Có gộp seed này vào cùng đợt với việc khác không, hay seed riêng ngay.
