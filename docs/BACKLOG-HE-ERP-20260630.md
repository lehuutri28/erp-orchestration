# 📋 BACKLOG HỆ ERP — việc CEO ERP MOBILE code tiếp (30/6/2026)

> **Nguồn:** Opus desktop audit map 470 trang admin (grep deterministic: `apiFetch`/`fetch`/`redirect`/"đang phát triển" trên file thật — KHÔNG đoán). Hệ **bán hàng đã hoàn thiện** (LIVE prod `831c1bee`).
> **Quy trình:** Mobile code từ backlog này → push `erp-wecha-pos` (branch `n8n/005`) → Opus desktop pull+verify+reviewer → Letri **deploy gom 1 lần/ngày**.
> 🔴 **Ràng buộc mọi task:** verify-first (đọc BE controller trước khi code, không bịa endpoint/key) · Rule 20 đủ 3 trục (CSDL `database_id` → chi nhánh `branch_id` → quyền `@RequirePermissions`) · 3-state UI · reload-after-action · KHÔNG hardcode/mock trong production.

## Tổng quan
- 470 trang admin (ngoài bán hàng) — **91% FULL (427)** · DUP 23 · HUB 6 · **STUB 13 + PARTIAL 1 = 14 trang cần code** · 8 trang FULL thiếu nút xuất.
- Nợ kỹ thuật **3 cụm**: (A) Tài chính/compliance · (B) HR lương/BHXH · (C) SaaS POS-V4 dashboard.

---

## 🏆 TOP 10 ƯU TIÊN (làm theo thứ tự)

| # | Trang | Loại | Việc cần làm | Bằng chứng |
|---|---|---|---|---|
| 1 | **finance/misa-export** | PARTIAL | Verify 8 endpoint `/api/excel-export/*` có thật trên BE; thiếu → build. **Xuất MISA AMIS — cột sai = phạt thuế** | `page.tsx:218` raw fetch + `:221,244` fallback "đang phát triển" |
| 2 | **finance/debts/doi-chieu** | STUB | Wire đối chiếu công nợ KH/NCC gọi BE (bỏ `DC_LIST`/`DC_DETAIL_TRANS` mock) | `page.tsx:39,50` hardcode, no apiFetch |
| 3 | **daily-reports** | STUB | Wire 4 tab Kho/LMS/Marketing/TaiChinh + 4 tab Wave2/3 gọi BE (CEO mở ra đang rỗng) | `tabs/TabKho.tsx:9` "Đang triển khai" |
| 4 | **finance/tax-tndn** (XML) | FULL-gap | Wire xuất XML ký số gửi CQT — **compliance TNDN** | `page.tsx:356` "Wave 18.E chưa ký số" |
| 5 | **hr/bhxh/declarations + reports** | STUB×2 | Wire tờ khai + báo cáo BHXH gọi BE (bỏ hardcode rate) — compliance lao động. Gộp 1 đợt | `declarations:7` `BHXH_RATES=[]` |
| 6 | **hr/salary/position + competency** | STUB×2 | Wire bậc lương chức danh + lương năng lực gọi BE (lương 3P). Gộp 1 đợt | `position:10` `VUNG_LUONG=[]` |
| 7 | **inventory/lots/recall/new** (SMS) | FULL-gap | Nối gateway SMS/email thu hồi lô — **recall HACCP** | `page.tsx:122` "gateway chưa kết nối" |
| 8 | **products/attributes + barcodes** | STUB×2 | Wire CRUD thuộc tính SP + danh sách/gen barcode gọi BE. Gộp 1 đợt | `attributes:153` "đang hoàn thiện" |
| 9 | **saas/apps-script-google/{analytics,alerts,expiry,feedback}** | STUB×4 | Wire cohort/funnel + alert share-key + license expiry + feedback gọi BE (chống gian lận key). Gộp cụm | `expiry:23` `MOCK_ROWS` |
| 10 | **finance/report-center/custom** | STUB | Wire Lưu/Chạy/Xuất custom report builder (để sau — 14 báo cáo chuẩn đã FULL) | `page.tsx:183` "Wave 19" |

## 8 nút xuất/feature phụ thiếu (page chạy, không gấp)
finance/notes-bctc-b09 (Word/PDF) · report-center/so-quy (Excel) · manufacturing/nmvat-report (Excel) · system-settings (theme/font) · databases (1 nút) · media (1 nút) · + tax-tndn XML & recall SMS (đã ở TOP 10).

## ⚠️ Cần Letri quyết trước (không tự làm)
- VAT export (b2b) → tích hợp MISA meInvoice push thật (giờ chỉ kiểm điều kiện)
- reservations "Chuyển đơn bán" → tạo order thật (saga)
- Bảo mật: đổi pass DB · confirm 2 repo private

---
*Cập nhật bởi Opus CEO ERP desktop sau khi audit map 470 trang. Mobile pick theo TOP 10. Mỗi việc xong → ghi PROGRESS + push.*
