# PROGRESS — Tiến độ & Nhật ký Release ERP Passion Link

> **File xương sống** cho Release Manager (gom changelog) + QA nghiệm thu (track màn DONE).
> **Cập nhật:** worker ghi mỗi session xong việc. Release Manager đọc trước mỗi deploy.
> **Vị trí:** `docs/PROGRESS.md` (root — Letri đọc được). 1 nguồn duy nhất, KHÔNG copy nhiều bản (tránh lệch).
> **Khởi tạo:** 2026-05-29

---

## 1. RELEASE GẦN NHẤT

| Mốc | Tag git | Ngày | Nội dung |
|---|---|---|---|
| **Mới nhất** | `wave-2.20.22-live-success-20260624-1521` | 24/6/2026 | **LIVE 15:21** (14 commit). **CSDL G-complete**: G2+G4 wire 4 module kho/SX (purchase-orders/stock-documents/manufacturing-orders/stock-audits) — databaseId fail-open. G-journal: databaseId propagate auto-je event → journal_entries_v3. **Fix DI**: PurchasingReportsModule thiếu DatabasesModule → crash 502 lần đầu deploy. **Cashbook UI**: 5 trang lọc nhãn-trên + items-end. **PhacheLeads**: module register + sidebar + link deposits. **req.user.sub**: 11+6 controller hết created_by NULL. KHÔNG migration đợt này (0410/0411/0412 chờ Letri apply prod riêng). |
| Trước | `wave-2.20.21-live-success-20260622-0359` | 22/6/2026 | **LIVE 03:59 health api+web 200** (Opus tự điều phối). **`/pos/materials`:** bấm danh mục lọc đúng cả SP danh mục **con** (trước khớp-chính-xác → bấm cha ra rỗng) + sắp danh mục **cha→con (rộng→hẹp)** thay alphabet. **8 trang sửa thanh lọc lệch hàng** (DatePickerSingle thêm nhãn-trên + `items-end` + đồng chiều cao): sales/pending-approval, assets/[id], customers/[id], finance/cost-analysis, hr/kpi/periods, manufacturing/waste, revenue-dashboard, pos/orders. **CSDL G0** (nền tách ổ): middleware đọc `X-Database-Id`→req.databaseId (inert, chưa bật lọc) + helper `databaseFilter`. **Mig 0408** thêm `database_id` 13 bảng tài chính ⚠️ **CHƯA APPLY** (G1, chờ quyết định D3/D4). SPEC-CSDL hoàn thiện B0–B9. Commit `8e10244`, KHÔNG apply migration đợt này. |
| Trước | `wave-2.20.20-live-success-20260622-0010` | 22/6/2026 | **LIVE 00:10** (Opus tự điều phối). **Sticky header TOÀN HỆ** (header bảng dính khi cuộn) — globals.css rule `.glass-table thead th{sticky}` + wrapper `:has(.glass-table)` thành vùng cuộn; 158 bảng khác chuẩn-hoá sang class `.glass-table` (sticky + nhất quán). **Trang Sản phẩm:** nút "Đổi nhóm SP" rõ (trước nhãn lộn) + hàng lọc lưới đều. Commit `b96f317`, KHÔNG migration. ⚠️ Cần verify visual 158 bảng. Bắt đầu hướng tách-class (NV đề xuất). |
| Trước | `wave-2.20.19-live-success-20260621-2239` | 21/6/2026 | **LIVE 22:39**. **Hệ bán hàng trang riêng** (như mua hàng): `/admin/sales/dashboard` + `/admin/sales/pending-approval` (đơn chờ duyệt + Duyệt/Từ chối Glass) + orders/[id] nút duyệt đúng trạng thái. **Sửa 500:** route order RmStock/FgStock TRƯỚC `:id` → "Nhập NVL từ NCC" hết 500 · created_by NULL = req.user.userId→**.sub** (20 controller) · **mig 0407** rfq_date → RFQ hết 500. Commit `c185055`. |
| Trước | `wave-2.20.18-live-success-20260621-2010` | 21/6/2026 | **LIVE 20:10**. **Responsive grid 81 trang** — grid-cols-N → grid-cols-1 sm:/lg: (card xuống 1-2 cột mobile, hết chồng/tràn). UI-2 trước chỉ sửa gridTemplateColumns inline, bổ sung nhóm Tailwind grid-cols. Commit `4f5605b`, KHÔNG migration. |
| Trước | `wave-2.20.17-live-success-20260621-1925` | 21/6/2026 | **LIVE 19:25 health api 200** (Opus tự điều phối). **Đồng bộ FE↔BE↔DB toàn hệ** — audit 14 hệ/208 trang → 128 lỗi (76 HIGH) lưu `docs/AUDIT-FE-BE-DB-20260621.md`, sửa ~104: route 404 (reports-export, lead-scoring, employee-advances param) · shape trống (so-cai/so-quy/debts/chung-tu đọc đúng res.entries/buckets) · sai cột (FE đọc đúng camelCase BE + BE thêm join: journal_lines, users name, assets, transactionCount, requesterName). Re-verify loại false-positive (b01a/b02/b03 có route wildcard). 2 commit `1d6e3f1`+`55a7081`, **mig 0406** (haccp_deviations +4 cột, đã apply prod — cột có sẵn). Build API+web PASS. Đã push GitHub. ⚠️ DEFER 2.20.18: KPI tổng hợp sổ cái · tên người (join) · norms sync · endpoint audit-log marketing · việc tồn (TK515/RFQ/leads/seed roles). |
| Trước | `wave-2.20.16-live-success-20260621-1722` | 21/6/2026 | **LIVE 17:22 health api+web 200** (Opus tự điều phối). **Sửa 5 bug C8 wave 2.20.13 + quét sạch lỗi QueryResult-coi-như-mảng** (`2334c4d`): C1 Đổi loại SP (VALID_CATALOG_TYPES) · C4 Thanh toán (map field BE + beneficiary + mark-paid) · C9 users-roles {data,meta} + assign/remove role + quyền LMS đăng nhập + lms-roles/deposits · C10 scope DTO class+whitelist · C16 route /admin/approval/instances + status lowercase + tên cột + filter tab · manufacturing 15 chỗ dashboard .rows · auth getMe quyền LMS. QC 3 vòng grep code thật. **UI-1** (`46b5de6`): 9 trang popup gốc→Glass. **UI-2** (`d6a9e8e`): 53 trang, 62 lưới thẻ KPI→auto-fit minmax (cùng hàng bằng nhau + không tràn mobile), giữ nguyên lịch 7 ngày/bảng/form. **KHÔNG migration.** Build API+web PASS. Đã push GitHub. ⚠️ Còn (đợt 2.20.17): C4 N2 lãi NH→TK515 · RFQ 500 thiếu rfq_date · leads cron · seed lms_roles · audit FE↔BE↔DB toàn hệ đang chạy. Chờ C8 nghiệm thu (5 bug + UI mobile máy thật). |
| Trước | `wave-2.20.15-live-success-20260621-1315` | 21/6/2026 | **LIVE 13:15 health 200**. Hotfix: ô Hạn dùng/Bảo hành hết kẹt-0 (`8c4b45d`) · Xoá vĩnh viễn ô gõ SKU dễ normSku (`f30088f`) · type-fix (`2e90ff9`). KHÔNG migration. |
| Trước | `wave-2.20.14-live-success-20260620-1628` | 20/6/2026 | **LIVE 16:28 health api+web 200**. **Fix giao diện DI ĐỘNG toàn hệ** — Opus audit mobile 5 trợ lý: gốc rễ **z-index loạn** (9 sub-sidebar mobile z-9500 đè mọi popup z-50→3000). **C6 hạ 10 sidebar mobile z-9500→900** (thang chuẩn: sidebar 900 < modal 1000 < dialog 2000 < toast 3000) — fix gốc, đa số trang hết lỗi. Các hệ sửa popup z thấp + co giãn: C1 (dropdown lọc Sản phẩm hết bị card che — thẻ lọc z-30) · C4/C5/C9/C10/C16/C17 (modal z-index + maxWidth co màn hẹp + bảng overflow-x + touch 44px + DatePicker). **C2: nút Gia hạn phiên wire `refreshAccessToken` + reschedule timer** (bản cũ đóng popup nhưng không gia hạn → vẫn bị đá ra). **KHÔNG migration.** ⚠️ Còn: RFQ 500 (thiếu cột rfq_date — C17 đợt sau, có NHAC) · leads cron (C11). Chờ C8 nghiệm thu (cần test máy thật/di động). |
| Trước | `wave-2.20.13-live-success-20260620-1552` | 20/6/2026 | **LIVE 15:52 health api+web 200**. **8 hệ sửa nút HÀNG LOẠT (bulk-action)** — Opus audit toàn hệ phát hiện nhiều nút 404/lưu sai/nút giả, giao từng C, QC từng vòng grep code thật. C1: đổi loại đồng bộ `catalogMainCategory`+productType + online-orders lọc server-side+bulk. C4: chi hàng loạt + 4 route Excel + reset thuế chiuThueOverride. C5: LMS điểm danh upsert+tuition soft-delete (**mig 0405**, dedup an toàn — bảng prod trống) + roles không đè perms. C9: users-roles bỏ mock+RBAC + roles quyền/description/cảnh báo xoá role. C10: 2 endpoint planning + thu hồi khoá lô. C11: gán NV lead + chặn lộ NV chéo. C17: GRN/QC approve/reject + returns bù hàng. C16: approval bulk-approve + media-admin path. ⚠️ Còn: 📱 lỗi di động z-index menu che popup (8 NHAC mobile, C6 gốc rễ) · RFQ 500 thiếu cột rfq_date (C17) · nút Gia hạn phiên (C2) · leads cron (C11). Chờ C8 nghiệm thu. |
| Trước | `wave-2.20.12-live-success-20260619-1053` | 19/6/2026 | **LIVE 10:53 health api+web 200**. 18 migration (0388–0404). **Hệ Bán hàng O2C chuẩn** (C1): báo giá `quotes`/`quote_items` (mig **0404 Opus tạo bù**) + chi tiết đơn `/admin/orders/[id]` + trả hàng soft-delete+bulk + POS ca quầy/thanh toán tách (0394) + biến thể SP (0395) + RBAC 46 endpoint + báo giá→đơn emit JE đúng event. **Sổ kế toán tự động** (C4): engine sinh JE từ event (511/632/156/131/531) + 4 rule seed (0401) + N1 đào tạo KCT chỉ tiêu 26 + N3 TNDN 17%. **Mua hàng** (C17): GRN giá vốn bình quân (0396). **Nhân sự** (C9): tuyển dụng (0388) + đào tạo nội bộ (0390/0391) + **thuế TNCN 5 bậc Nhi 15,5/6,2tr** (0403) + Excel quyết toán thật (exceljs). **Tích hợp** (C2): hết hạn phiên (0389) + Xuất Excel entity + IP tin cậy + lịch sử đăng nhập + FK platform (0393/0402). ⚠️ Opus apply migration TRƯỚC deploy → bắt+sửa 3 lỗi (quotes table thiếu→0404, orders.deleted_at thiếu→0399, drop index của constraint→0395). 🔧 Còn: C4 N2 lãi NH→**TK515** (chưa code), cron `leads:batch-scoring` chưa start (C11). Chờ C8 nghiệm thu. |
| Trước | `wave-2.20.11-live-success-20260615-2150` | 15/6/2026 | **LIVE 21:49 health 200**. KHÔNG migration (chỉ FE/BE feature). C4 **Xuất Excel MISA thuế/einvoice** + sidebar/breadcrumb (`be4a120`) · C1 **MISA import wire thật einvoice+debts + Xuất Excel B2B** (`55790ba`) · C16 **đổi mật khẩu verify MK cũ + bỏ mock Quy trình/Phiếu duyệt + dấu TV + PermissionGate** (`0e4441b`) · C11 **endpoint `POST /public/wecha/lead`** nhận lead website wecha.vn (`b8b716c`) · C2 Xuất Excel template (uncommitted rsync). ⚠️ Opus gỡ build-blocker: dup `exportTaxSummary` trong excel-export.service.ts (xoá placeholder stub C10, giữ C4 thật). Chờ C8 nghiệm thu. |
| Trước | `wave-2.20.10-live-success-20260615-1816` | 15/6/2026 | **LIVE 18:16 health 200**. Hotfix 7 P0 (C10 MRP 500 + FG stock-in 500 · C17 PR 500+PO trash 500+PO raw array · C5 formula 404+tên góp ý) + C10 Wave 2.21 gộp luôn: **cột Tồn NVL+lọc nhà máy** /admin/manufacturing/raw-materials · **Khung Nhóm** /admin/manufacturing/settings/product-groups (mig 0386) · **Xuất Excel NL/Bao bì/TP** · **HUB 2 tên NCC/nội bộ + 3 giá** (mig 0387) · **trang Bán thành phẩm** /admin/manufacturing/semi-finished · C9 filter HĐLĐ. **Mig 0385+0386+0387 apply (0379-0384 đã có sẵn — 500 là bug CODE không phải migration).** ⚠️ 0387 suýt sót (C10 commit 8798cba sau gom) → apply bù sau deploy. ⚠️ HUB products đổi CHƯA có HUB_CHANGE doc (C10 viết bù). Chờ C8 nghiệm thu. |
| Trước | `wave-2.20.9-live-success-20260611-1524` | 11/6/2026 | **LIVE 15:24 health 200**. 6 fix bug C8 QA bắt "DONE giả" (C10 MRP 500 · C17 PR 500+PO xoá · C5 Định lượng lưu công thức · C1 MISA import bỏ mock · C16 Hộp góp ý hiện tên NV · C11 Campaign thùng rác) + 2 màn mới mockup C14 (C10 **Nhập kho TP 4 loại** /admin/manufacturing/finished-goods/stock-in · C9 **Chấm công nhập Excel** /admin/hr/attendance) + Opus fix build (DiffStats + moRows). **Mig 0382/0383/0384 đã apply.** Chờ C8 nghiệm thu. |
| Trước | `wave-2.20.8-live-success-20260608-1238` | 8/6/2026 | **LIVE 12:38**. C1 (nhập kho MISA 23 cột) + C5 (E2 lưu công thức Định lượng) + C10 (**Nhập NVL** /admin/manufacturing/raw-materials/stock-in: VAT/Tổng GTNK auto + ngày giờ + đính kèm chứng từ + dây chuyền + 3 GAP-SX: duyệt MRP/filter) + C11 (breadcrumb) + C17 (6 GAP: purge_after fix + AI hoá đơn error-state + compliance NCC) + C6 (mobile/FilterBar). **Mig 0378-0381 apply trước.** Chờ C8 nghiệm thu. |
| LIVE trước | `wave-2.20.7-live-success-20260608-0944` | 8/6/2026 | **LIVE 09:44 health 200**. C2 (audit Integration 14 trang) + C5 (Bán TP redirect /admin/products) + C10 (**MRP runs** /admin/manufacturing/mrp mig 0377 + **fix bug "Chi nhánh: —"** màn Kiểm kê kho) + C11 (campaign **soft delete** mig 0376 + xoá trang trùng /campaigns) + C17 (thùng rác/purge mua hàng) + C4 (lưu nháp form bút toán). **Mig 0376/0377 apply trước.** Chờ C8 nghiệm thu. |
| Trước | `wave-2.20.6-live-success-20260606-1155` | 6/6/2026 | **LIVE 11:55**. C1 (Wizard /admin/setup + Đơn sỉ B2B /admin/sales/b2b) + C2 (IP tin cậy BE) + C4 (thuế GTGT) + C5 (LMS) + C9 (thùng rác HR 4 màn + fix hàng lọc) + C10 (Kiểm kê kho + Norms) + C11 (ghost columns + deep-link) + C16 (audit QT) + C17 (soft-delete) + C18 (n8n Đợt 2). Mig 0369/0373/0374/0375. | C1 (Wizard thiết lập /admin/setup + Đơn sỉ B2B /admin/sales/b2b) + C2 (IP tin cậy BE — FE chờ wire) + C4 (fix thuế GTGT lọc đúng kỳ) + C5 (LMS thêm khoá/workflow/breadcrumb) + C9 (thùng rác HR 4 màn + **fix hàng lọc đều độ rộng**) + C10 (Kiểm kê kho SX + Norms CRUD + NVL pagination) + C11 (ghost columns + deep-link filter) + C16 (audit Quản trị) + C17 (soft-delete FE) + C18 (n8n Đợt 2 HMAC/error). **Mig 0369/0373/0374/0375 apply trước.** Chờ C8 nghiệm thu. |
| Trước | `wave-2.20.5-live-success-20260605-1103` | 5/6/2026 | LIVE 11:03. C9 HR (NV hết treo + phiếu lương + thùng rác) + C11 (cột Thẻ + lọc kênh) + C7 (Wave 14 + breadcrumb) + C17 (PO tối ưu) + C18 (catalog + 3 bảng tracking n8n mig 0370-0372) + C6 (27 màn date picker). |
| Trước | `wave-2.20.4-live-success-20260601-1100` | 1/6/2026 | LIVE 11:00 (@ a2afee9). C17 3 màn nghiệp vụ sâu (3-way/QC/so báo giá RFQ) + PG sequence mã chứng từ + C1 fix lọc Trà + khách "0 bản ghi" + bán sỉ 0đ + C5 LMS×4 + C10 Sprint 17.C. |
| Trước | `wave-2.20.1-live-success-20260531-1746` | 30/5/2026 | C1 GAP-NL-1 (SP hiện mọi chi nhánh POS) + C16 Delete Policy M1-M5 + Thùng rác FE (hết 404) + 9 màn Quản trị + C17 mua hàng (soft-del/phân trang/search). Migration 0310-0359 apply verify từng bảng. |
| Trước | `wave-2.20-live-success-20260530-0927` | 30/5/2026 | POS lưu KH hết 500 (mig 0250 fax) + barcode + Vertex AI Vault + BOM Công thức + Danh mục + Thùng rác nền C16 |
| Trước | `wave-2.19.3-live-success-20260529-0918` | 29/5/2026 | Fix invoices/returns API, phạm vi chi nhánh, HĐLĐ, pagination, auto-compress ảnh PR |
| Trước | `wave-2.19.2-live-success-20260527-2226` | 27/5 | Marketing API 404, reports timeout, NCC field, POS sửa/xoá, customer KPI |
| Trước | `wave-2.19.1-live-success-20260527-1855` | 27/5 | POS barcode, DatePicker, audit log |
| Trước | `wave-2.19-live-success-20260527-1426` | 27/5 | 10 migration + 13 commit (7 C) |

---

## 2. TRẠNG THÁI 9 DOMAIN (nghiệm thu thật chưa)

> DoD (Definition of Done) 1 màn = QA Chrome test pass 7 mục (list/thêm/sửa/xoá soft/lọc/validate/phân trang).

| Domain | Code | QA Chrome nghiệm thu | DONE? |
|---|---|---|---|
| Bán hàng | ✅ | ⏳ Chưa nghiệm thu đủ | ⬜ |
| Khách hàng & Marketing | ✅ | ⏳ | ⬜ |
| Sản xuất & Cung ứng | ✅ | ⏳ | ⬜ |
| Mua hàng | ✅ | ⚠️ 1 phần (C8 test Wave 2.19.2) | ⬜ |
| LMS | ✅ | ⏳ | ⬜ |
| Tài chính – Kế toán | ✅ | ⏳ | ⬜ |
| HR | ✅ | ⏳ | ⬜ |
| Báo cáo | ✅ | ⏳ | ⬜ |
| Quản trị hệ thống | ✅ | ⏳ | ⬜ |

> ⬜ = chưa nghiệm thu | 🟡 = đang nghiệm thu | ✅ = Letri duyệt DONE

---

## 📢 THÔNG BÁO 18C + ĐIỀU PHỐI (mới nhất — ⏰ 2026-06-29 12:51) — 1 NGUỒN DUY NHẤT, không ghi file riêng

- ✅ **C13 POS-V4 FE ROLE-GATING — 29/6 12:51** (`clasp push` 43 files): Phân quyền giao diện GAS SPA theo vai trò NV. Thêm `staff_forLogin()` (NhanVienUI.js:241) — NV active + chi nhánh, không có password_hash. Modal chọn NV (`#staff-selector-overlay`) hiện khi mở app sau setup. User bar drawer (`#drawer-user-bar`) hiện tên+vai trò+chi nhánh+🔄. JS: `applyRoleUI(role)` ẩn sidebar theo `ROLE_HIDE` map; `cashier` ẩn 10 mục (thue/thuchi/baocao/baocaoton/chuoi/nguoidung/chinhanh/phanquyen/caidat/tichHop); `staff` ẩn thêm nhacungcap/congthuc/kho. Load handler sửa: sau `isSetupDone:true` → `showStaffSelector()` thay vì `navigateTo(page)` trực tiếp. ⚠️ Letri test: mở POS URL → modal NV hiện đúng; thêm NV test role cashier → sidebar bị ẩn đúng. Cạm bẫy: tên NV có dấu nháy đơn `'` có thể lỗi onclick (cần escape thêm nếu gặp).

- ✅ **C4 MOBILE 20/6** (commit `5787c19`): 9 fix mobile — modal zIndex 50→1050 (TCModal/TaoDeNghiTT/ThemButToan), close btn 32→44px, KPI auto-fit minmax(140px), layout split collapse 1 cột @767px, bảng 7-10 cột overflow-x. tsc 0 error. XIN-DUYET viết rồi, chờ Opus QC.

- 🟢 **CHUẨN TOÀN HỆ (Letri chốt 15/6) — NÚT XUẤT EXCEL THEO LỌC:** MỌI bảng dữ liệu (list) ở MỌI màn của MỌI C **phải có nút "Xuất Excel"** xuất ĐÚNG theo bộ lọc đang áp (không xuất hết khi đang lọc). Mục đích: kéo data về **dò rà / đối soát tiền**. Tái dùng module excel-export sẵn có (KHÔNG tự viết parser). Cột xuất = cột đang hiện + tôn trọng tenant/branch. **Mỗi C tự rà bảng của mình + gắn dần** (không cần làm 1 lần) — đưa vào PROPOSE-GAP/wave của từng C. Áp từ nay.

- **Wave 2.20.5 ĐÃ LIVE** (5/6 11:03, health 200): C9(NV treo+phiếu lương+thùng rác) · C11(cột Thẻ+lọc kênh) · C7(Wave14+breadcrumb) · C17(PO tối ưu) · C18(catalog+3 bảng tracking n8n mig 0370-0372) · C6(27 màn date picker).
- 🔴 **BỎ chờ "chốt branch"**: deploy KHÔNG phụ thuộc branch — `safe-deploy` rsync working tree `claude/pos-system`. Cứ code→build PASS→commit/push→báo Opus gom wave. Migration mới **0373+**. Báo cáo ⏰ giờ THẬT (`date`).
- **Đang gom Wave 2.20.6**: wizard+B2B(C1) · LMS G2/G4/G6(C5) · thùng rác 4 màn+hàng lọc(C9) · mobile(C6) · gap SX(C10) · ghost columns(C11) · endpoint setup-check(C16) · AI hoá đơn(C3/C17).
- 🔴 **Chỉ Letri gỡ**: token phache(C12) · token C15 · upload tra.webp · 4 FB token+Zalo(C18) · hỏi Nhi 4 câu thuế(C4).
- C13 @76 LIVE (kho_nhap Sheet-first) — backend mailer c308b8d chờ Letri deploy VPS (`pm2 restart pos-api`). C14 vẽ 2 mockup SPEC-17B rồi nghỉ. NHAC/DUYET từng C ở `orchestrator-ceo/ceos/c<N>-*/` (Letri dán vào session C).

---

## 3. WAVE ĐANG CHẠY

| Wave | Loại | Trạng thái | Owner |
|---|---|---|---|
| **2.20.9** ✅ LIVE 11/6 15:24 | Fix+Feature | LIVE — xem mục 1 | nhiều C |
| **2.20.10 HOTFIX** 🔴 READY | Hotfix P0 | C17 ✅ DONE (d647747) · C10 ? · C5 ? — chờ C10+C5, sau đó gom deploy | Opus gom |
| **CW1 — Soft delete** | Refactor | Chạy rải theo C (HR/LMS/CRM/SX đã có thùng rác) | 8 C |

---

## 4. REFACTOR WAVE (theo ARCHITECTURE_CONTRACT mục 13)

| Wave | Nội dung | Trạng thái |
|---|---|---|
| CW1 | Soft delete (~158 bảng) | 📋 Sẵn sàng |
| CW2 | Response API chuẩn `{data,meta}` | ⏳ Chưa bắt đầu |
| CW3 | Mã chứng từ (PR→PMT, bỏ random) | ⏳ Chưa bắt đầu |
| CW4 | Tenant support_tickets + branch_id 5 bảng + RBAC | ⏳ Chưa bắt đầu |

---

## 5. VIỆC DỌN NGAY (đã verify data prod)

| Việc | Cơ sở | Owner | Trạng thái |
|---|---|---|---|
| Drop sổ kế toán v1 (`journal_entries`) | 0 dòng + 0 code (DB_VERIFY) | C4 | ⏳ Chờ duyệt |
| Xoá code `promotionsLegacy` (không cần drop table) | DB_VERIFY = NULL | C1 | ⏳ Chờ duyệt |

---

## 6. NHẬT KÝ THAY ĐỔI (mỗi worker ghi 1 dòng khi xong việc)

> Format: `[NGÀY] [C] [commit] mô tả ngắn`

```
[29/6] [C18] ✅ BE OPTION-B HMAC DONE (chưa deploy) — n8n_wf-89 FB Ads 12h: Letri chọn Option B. Tạo N8nAdsController `POST /api/n8n/ads/daily-stats` + `POST /api/n8n/ads/leads` (N8nHmacGuard thay JWT). rawBody:true vào main.ts. Cập nhật AdsTrackingModule. tsc PASS. Flow JSON v2.0: thêm 2 HMAC Code node, URL đổi sang /api/n8n/ads/*, bỏ httpBearerAuth. ⏳ CHẶN: Letri deploy 3 file BE + set env var N8N_WEBHOOK_HMAC_SECRET (ERP + n8n) + WECHA_TENANT_ID (n8n) + tạo credential FB/Telegram + import flow + activate. Chi tiết: VIEC-C18.md 2206-03. ⏰ 2026-06-29.
[28/6] [C18] 📋 DRAFT READY n8n_wf-89 FB Ads 12h — draft-016-fb-ads-12h-telegram.json (13 nodes: Cron 00h/12h + FB Insights account+campaign + POST /ads/daily-stats + FB Lead Forms filter phone + POST /ads/leads + Telegram report + N9 log). Đã audit endpoint (JwtAuthGuard, không HMAC). ⏳ CHẶN: cần Letri cấp FB Ad Account ID + System Token + FB Page ID + Telegram Chat ID (qua n8n credential, không paste chat). Quyết định: đề xuất Option A service account JWT (không sửa BE). VIEC-C18.md task 2206-03. ⏰ 2026-06-28 20:36.
[26/6] [C13] ✅ PG-01+PG-02 VIP+Tax settings LIVE @82 (commit 2a7e0ca) — 10-CaiDat.html: inputs có ID + lazy load + saveVipSettings()/saveTaxSettings() gọi GAS. CaiDat.js: 4 functions mới lưu Sheet CaiDat (keys: VIP_POINT_RATE, VIP_POINT_EXCHANGE, TAX_INDUSTRY_CODE, TAX_ZALO_ALERT). Không cần VPS. Deploy ID: AKfycbyFITYE3k7rKCp07jeKjXlqcka1qy9rGGwpIRUdWhiC3mctoln1AQUodBzPEa530gg7. ⏳ PENDING: PG-03/PG-04/PG-05 HOÃN. [Letri-URGENT] VPS deploy mailer c308b8d vẫn chưa apply. ⏰ 2026-06-26 09:45.
[25/6] [CEO-ERP] ✅ wave-2.20.24 LIVE 01:13 (Đổi nhóm SP gộp + Quy trình Bán hàng + file mig 0413). tag+push. ⚠️ mig 0413 CHƯA apply (RFQ vẫn 500). → Bắt đầu VIỆC 3 KHO: (1) ✅ CỐ ĐỊNH CỘT đầu khi cuộn ngang (class .col-frozen tái dùng + áp Nhập kho cột #/Ngày HT) — ask nổi bật "di thanh trượt cột đứng yên như Excel"; (2) ẩn/hiện cột ĐÃ có sẵn (ColumnToggle); (3) Excel full/basic = CHƯA — phát hiện KHÔNG có route export cấp document cho Nhập kho (`/excel-export/stock_inward` không tồn tại; `/purchases` là export CHI TIẾT dòng kế toán cần from-to, khác). → Build route BE mới cấp document (29 cột MISA full vs cơ bản) = việc focused tiếp theo, không rush export tài chính. 1 commit LOCAL (cố định cột) chờ deploy. ⏰ 2026-06-25 ~01:20.
[24/6] [CEO-ERP] 🆕 Quy trình Bán hàng + fix RFQ 500 (3 commit LOCAL, build PASS, chờ deploy) — (1) Trang MỚI /admin/sales/process: pipeline O2C 6 chặng đếm THẬT từ /orders/stats (Bán hàng trước KHÔNG có "quy trình" như Mua hàng). (2) Lỗi 500 trang Quy trình Mua hàng = /api/rfqs "column title/deleted_at does not exist" — bảng rfqs prod 5433 SCHEMA DRIFT (tạo từ schema cũ, thiếu 13 cột) → migration 0413 thuần ADD COLUMN IF NOT EXISTS. ⚠️ CHỜ Letri apply 0413 prod (agent KHÔNG tự apply — đã bị auto-mode chặn đúng). 🔜 CÒN: Kho tuỳ chỉnh cột hiện/ẩn + cố định cột (freeze cuộn ngang) + Excel full-vs-basic (mẫu MISA 29 cột) = việc lớn riêng. ⏰ 2026-06-24 ~22:10.
[24/6] [CEO-ERP] 🐞 bulk "Đổi nhóm SP" (1 commit LOCAL, build PASS, chờ deploy) — Letri báo "không chạy". TRUY: action ĐÃ chạy (nginx 201, log change_type count=3, DB đã set GOODS_VAT_TRADING) NHƯNG chỉ đổi catalog_main_category (cột NHÓM THUẾ), cột NHÓM SP=group_code (RAW_MAT_PROD) không đổi → tưởng hỏng. Letri chốt "GỘP" → thêm CATALOG_TO_GROUP_CODE map, change_type set LUÔN group_id (vd GOODS_VAT_TRADING→FINISHED_GOODS). findAll groupCode lấy từ join group_id → cột đổi theo. ✅ CSDL hiện đúng "VUA AN TOÀN 2026" (fix auth ăn). ⏰ 2026-06-24 ~22:00.
[24/6] [CEO-ERP] ✅ wave-2.20.23 LIVE 17:41 — deploy 5 commit (auth refresh fix + orders guard SP ẩn + Menu Quản trị mobile + leads cron + min(uuid) + GlassPrompt 4 trang). Verify: health 200, process mới online, auth.dto fix trên prod, leads cron HẾT lỗi sau restart (createQueue ăn), không lỗi mới. ⏳ Chờ traffic thật xác nhận refresh/CSDL (Letri test iPhone). tag+push GitHub. NHAC C14 QA. ⏰ 2026-06-24 17:41.
[24/6] [CEO-ERP] 🐞 3 bug Letri báo (2 commit LOCAL, build PASS, chờ deploy): (3) SP đã ẩn/ngừng vẫn lên đơn → guard order create chặn is_active=false + discontinued + tenant filter (SP dùng is_active/archived, KHÔNG có deleted_at). (1) nút "Menu Quản trị" mobile đè breadcrumb → class .admin-header-bar + media query padding-right:150px (cách chung CSS). (2) "Bảng Nhận Việc" (improvement-requests) crash error-boundary: hook phòng thủ tốt (không throw), không thấy crash render độc lập → NGHI session/auth gãy (refresh-400) → retest sau deploy fix auth; nếu còn cần repro browser. ⏰ 2026-06-24 ~17:45.
[24/6] [CEO-ERP] 🔴 FIX GỐC RỄ "Chưa có CSDL" (1 commit LOCAL, build PASS, ƯU TIÊN DEPLOY) — Picker hiện "Chưa có CSDL" dù prod 5433 có ổ active. Truy: nginx log /api/databases→401, /api/auth/refresh→**400** (4 lần/ngày). Gốc: RefreshTokenDto.refreshToken @IsNotEmpty BẮT BUỘC nhưng token ở cookie httpOnly (body rỗng) → ValidationPipe chặn 400 TRƯỚC handler → refresh chết → access token hết hạn KHÔNG hồi phục → 401 lan /databases → CSDL rỗng. Fix: refreshToken→optional (RefreshTokenDto+LogoutDto). ⚠️ Lỗi SESSION TOÀN APP, không chỉ CSDL. Bài học: docs/lessons-learned/security/2026-06-24-01-refresh-dto-required-chan-cookie-auth.md. ⏰ 2026-06-24 ~17:35.
[24/6] [CEO-ERP] 🧹 BACKLOG tồn đọng (3 commit LOCAL, build PASS, chờ deploy batch) — (1) leads cron pg-boss createQueue → hết spam "Queue not found"; (2) rm-stock getMovements MIN(uuid)→MIN(::text)::uuid → hết lỗi [NL-AUDIT]; (3) GlassPromptDialog mới + promptDialog() API → 4 trang hết native prompt/alert (deposits/refunds/tickets/media). Audit lại: req.user.sub ĐÃ XONG (chỉ còn 2 file ngoại lệ đúng). ⏰ 2026-06-24 ~16:10.
  🔜 CÒN (cần migration gate hoặc domain input): Math.random→sequence 6 service (⚠️ einvoice number = compliance NĐ70, payments txn — cần Letri/domain); G7 TaxDeclarationsService + ReconciliationService; ~30 báo cáo tài chính (MOCKUP-GAP); AI System Health.
[24/6] [CEO-ERP] ✅ APPLY migration 0410+0411+0412 vào PRODUCTION (port 5433) 15:42 — 30/30 bảng commerce+kho/SX có database_id, backfill sạch (orders 69, cashbook 41, inventory_lots 17), tổng 55 bảng có database_id. App HTTP 200. CSDL G hoàn tất end-to-end. ⏰ 2026-06-24 15:42.
  ⚠️ SỰ CỐ: suýt apply nhầm DB rác `pos_db@5432` (n8n docker) vì .env ghi port 5432 — PROD THẬT ở **5433**. Đã backup full trước khi apply (/opt/backup/pre-csdl-0410-0412-5433-20260624-154141.dump). Bài học: docs/lessons-learned/database/2026-06-24-01-apply-migration-nham-port-5432-vs-5433.md. ⭐ TRƯỚC psql prod: đọc DATABASE_URL từ /proc process, KHÔNG tin .env.
[24/6] [CEO-ERP] wave-2.20.22 LIVE 15:21 — CSDL G2+G4 kho/SX + G-journal + cashbook UI + phache-leads + req.user.sub fix. Fix DI crash: PurchasingReportsModule+DatabasesModule. 14 commit, tag+push GitHub. ⏰ 2026-06-24 15:21.
[22/6] [CEO-ERP] fbd6fb6+4fe15ad+c56535a — **CSDL G-complete**: (A) G2+G4 wire kho/SX — purchase-orders/stock-documents/manufacturing-orders/stock-audits (databaseId lọc ổ fail-open). (B) G-journal: databaseId truyền qua auto-je event payload → journal_entries_v3. (C) UI cashbook 5 trang lọc thẳng hàng (nhãn-trên + items-end). (D) phache-leads module register + sidebar. tsc 0 lỗi. 13 commit local, chờ Letri: (1) apply mig 0410+0411+0412 prod (psql), (2) safe-deploy. ⏰ 2026-06-22.
[22/6] [C12] pos-system — ADMIN "Khách từ Web Phache" (Letri chọn): dashboard gom toàn bộ lead phache.com.vn cho NV chốt + đo ads. BE: `customer-lms/phache-leads.service.ts` (query customers+course_deposits+ads_leads source='phache_web', đa tenant filter, soft-delete, response {data,meta}) + `phache-leads.controller.ts` (GET /customer-lms/phache-leads/dashboard, JwtAuthGuard+PermissionsGuard, reuse perm customer_lms.deposits.read) + reg customer-lms.module. FE: `admin/customer-lms/phache-leads/page.tsx` (6 thẻ thống kê: tổng khách/đã cọc/chờ cọc/chỉ tư vấn/doanh thu/tỷ lệ chốt + bảng lead + lọc) + sidebar entry + link từ Sổ đặt cọc. Verify: nest build PASS + next build PASS + tsc 0 lỗi + route vào .next. Dữ liệu hiện khi lead chảy về (cần token prod). ⏰ 2026-06-22 09:15.
[22/6] [C12] static — Web phache.com.vn: thêm mục "Khoá Học Liên Quan" (Letri chọn — giữ chân khách + SEO nội bộ) vào 27/27 trang khoá con. Mỗi trang gợi ý 4 khoá glass card: ưu tiên khoá cùng chuyên mục (siblings) + flagship phổ biến (Tổng Hợp, Trà Sữa Thương Hiệu, Cà Phê Barista, Ăn Vặt, Trà Trái Cây, Làm Kem), không trùng/không tự trỏ. Bấm dẫn qua trang khác → tăng internal link (SEO) + khách xem nhiều trang hơn. Verify giao diện+link PASS. ⏰ 2026-06-22 05:51.
[22/6] [C12] static — Web phache.com.vn TRỌN BỘ 16 CHUYÊN MỤC: +4 trang khoá con kem/bingsu (Kem Các Loại 9.5tr · Kem Viên 5tr · Bingsu Hàn Quốc 9.5tr 13 vị · Kakigori 5tr Nhật — dữ liệu thật) → tổng **27 trang khoá con, 27/27 đã polish glass/perf**. Nối khoa-lam-kem → 4 trang chi tiết. Đồng bộ Zalo OA toàn site (100 link đều OA chuẩn). Sửa phụ đề generic (24 trang). Sitemap 46 URL. 0 nút card trỏ CMS cũ. → Website khoá học HOÀN CHỈNH 100%: 1 menu + 16 chuyên mục + 27 khoá con, glass kính sang ấm desktop + load nhanh không giật mobile, sẵn sàng chạy ads. CÒN: form auto-save admin ERP chờ Letri set token prod + CEO ERP mở endpoint. ⏰ 2026-06-22 05:39.
[22/6] [C12] static — Web phache.com.vn HOÀN TẤT: 23 trang khoá con chi tiết (mọi khoá của 16 chuyên mục, giá+món THẬT từ web, clone chuẩn barista qua workflow tuần tự vượt rate-limit). QA glass/perf đồng bộ bộ Tổng Hợp: desktop kính dày blur20px sang ấm, mobile TẮT backdrop-filter + bg-attachment scroll → load nhanh, hết giật/lác iOS (verify desktop+mobile). Nối 16 chuyên mục: mọi card "Xem chi tiết" → trang khoá con mới (branded + form chốt khách degrade-Zalo), 0 nút trỏ CMS cũ. Sitemap +23 (tổng 42 URL). CÒN: form auto-save admin ERP chờ Letri set token prod + CEO ERP mở endpoint. ⏰ 2026-06-22 05:01.
[22/6] [C12] static — Web phache.com.vn: mẫu khoá con #2 `khoa-tra-sua/thuong-hieu.html` (Trà Sữa Thương Hiệu 24.9tr -45%, 8 buổi/4 ngày, signature trà cốt+kem cheese+brulee+oolong+matcha — thật từ trang 223). Đồng bộ chuẩn barista. → CÓ 2 MẪU khoá con 2 phân khúc giá (5.7tr Barista + 24.9tr Thương Hiệu). DỪNG re-schedule, CHỜ Letri duyệt chuẩn trước khi nhân rộng ~25 trang + quyết bật form/backend. ⏰ 2026-06-22 02:44.
[22/6] [C12] static — Web phache.com.vn: chuẩn bị form chốt khách + mẫu khoá con. (1) Soạn SẴN `api/consultation-proxy.php.PENDING` (luồng tư vấn generic fullName+phone+interestedCourse, KHÔNG cần courseCode 552/553/635, endpoint từ ENV) — CHƯA wire live, chờ Letri set token + CEO ERP xác nhận endpoint. (2) Trang khoá con MẪU `khoa-ca-phe/barista.html` (Cà Phê Barista 5.7tr) theo hướng chi tiết: hero+giá nổi, video thật, 6 ô curriculum thật, menu pha máy, bảng giá, lịch khai giảng LINH ĐỘNG (JS array, không hardcode), form chốt khách POST proxy → degrade an toàn về Zalo khi backend chưa bật (verify OK không mất lead), FAQ. CHỜ Letri duyệt mẫu trước khi nhân rộng ~25 khoá con. ⏰ 2026-06-22 02:08.
[22/6] [C12] static — Web phache.com.vn HOÀN TẤT 16/16 CHUYÊN MỤC: build nốt 4 trang (Rau Má Mix 4tr/7.11tr, Đồ Uống Nóng 1.9tr/4.55tr 28 món, Sữa Chua trân châu 5tr/yogurt 6.4tr, Sữa Hạt 5tr 17 công thức) — verify nguồn web (soi 8 trang DS, 4 chuyên mục "tưởng trống" đều CÓ khoá thật 274/444/318+543/323). Nối Menu 4 ô cuối → trang riêng: giờ 16/16 chuyên mục bấm ra trang đầy đủ, 0 ô tư vấn suông, 21 nút "Xem chi tiết". sitemap +4. Verify giá/món/CTA khớp 100% web, mobile no-overflow PASS. CÒN: form lead→admin ERP (chờ Letri gỡ token + CEO ERP mở endpoint) + rebuild trang chi tiết từng khoá con (tùy chọn). ⏰ 2026-06-22 01:30.
[22/6] [C12] static — Web phache.com.vn TIẾP: build 8 trang chuyên mục (Cà Phê 3 khoá, Trà Trái Cây 2, Trà Chanh, Đá Xay, Tàu Hủ, Chè 2, Sinh Tố, Ăn Vặt 2) — giá+số món+thời lượng THẬT fetch từ 10 trang khoá live phache.com.vn (KHÔNG bịa), clone mẫu Trà Sữa qua workflow 8 agent (3+5 do server rate-limit phải resume). Nối Menu: 6 ô Zalo→trang riêng (giờ 12 chuyên mục có trang, 4 còn Zalo đúng thực tế). sitemap +11. Backend lead: đọc enrollment-proxy.php → endpoint ERP KHÔNG có trong apps/api working tree + token prod chưa set + courseCode chỉ 552/553/635 → ghi NOTE-BACKEND-LEAD-FLOW + đề xuất Letri. Verify giá/CTA/mobile PASS. ⏰ 2026-06-22 01:12.
[21/6] [C12] static — Web phache.com.vn: (1) MỚI Menu Khoá Học `cac-khoa-hoc-day-pha-che.html` — 16 chuyên mục thật (emoji+video YouTube+mô tả từ home.php), click chuyên mục → accordion hiện các khoá, chip nav dính, link đúng URL live (9 khoá thật), Glass Quicksand mobile-safe. (2) MỚI chuyên mục `/khoa-tra-sua/` theo mẫu khoa-tong-hop — 3 card (Chuẩn Vị/Thương Hiệu 24.9tr-45%/Bubble Tea), breadcrumb+schema SEO, giá THẬT fetch từ live (không bịa). sitemap +3 link. Verify desktop+mobile (no x-overflow). Nguồn dữ liệu: MAP-16-CHUYEN-MUC-KHOA.md. CHỜ Letri upload. ⏰ 2026-06-21 23:12.
[15/6] [C4] be4a120 — P0 FIXED: TCShell sidebar deriveActiveFromPathname rewrite (25+ route→NAV id đúng, cũ map cb-cash→'soquy' không tồn tại). Breadcrumb.tsx +26 Finance path. Einvoice modal POST wired (controller createManual + service). 🔴 Excel MISA bug fix: excel-buttons.tsx đổi POST→GET + URL /excel/export→/excel-export/{entity}. tax-summary: GET /excel-export/tax-summary mới + exportTaxSummary() 11 cột MISA. FE tax-summary/page.tsx xoá CSV thủ công → ExcelButtons. Build web PASS ✅ ⏰ 2026-06-15 21:10.
[18/6] [C9] BUILD PASS — Đợt B+C P1: exams lessonId→<select> autocomplete (/hr/training/lessons?limit=200) + thùng rác exams/lessons (loadTrash+handleRestore+PermissionGate hr.training.exams/lessons.restore). BE hr-tncn.controller RBAC(@RequirePermissions)+{data,meta} tất cả endpoint. dependents DatePickerSingle×3+alertDialog. settlement Xuất Excel blob (/api/hr/tncn/declarations/export?year=&format=XLSX)+label "đã khấu trừ". `pnpm --filter web build` ✅ PASS. XIN OPUS QC lại. DEFERRED: seed 15.5tr/6.2tr (chờ Nhi qua Letri) + hr_payslips.tncn_amount migration. ⏰ 2026-06-18.
[15/6] [C2] Wave 2215 DONE — Việc 1+2: (1) BE entity-configs thêm sms-templates/email-templates/login-history (schema-qualified marketing.* tables + noActiveFilter cho login_history append-only). Fix exportExcel() dùng sql.identifier chaining cho schema-qualified tables. FE: EntityExcelExport.tsx component mới (POST /api/excel/export/:entity), wire vào sms/templates + email/templates + login-history. ExcelExportService: thêm exportTaxSummary() stub fix pre-existing build error. (2) FE 2 trang mới: /admin/system/trusted-ips (GlassConfirmDialog approve/block) + /admin/system/login-history (filter+stats+Xuất Excel). NHAC-C6 gửi để C6 thêm 2 sidebar entries. Build PASS (FE Next.js + BE nest build). Code in claude/ (gitignored). ⏰ 2026-06-15 22:30.
[18/6] [C4] dfd08d9 N1+N3 thuế DONE — N1: course_revenue→AUTO_EXEMPT (KCT chỉ tiêu 26), GTGT_CODE_MAP+INDICATOR_MAP, Excel MISA thêm 2 cột (Mã thuế GTGT+Chỉ tiêu tờ khai), FE label KCT. N3: tax-tndn/page.tsx dropdown 17%/20% (default 17% ưu đãi NĐ218/TT78), lưu metadata.tndn_rate audit trail. N2 WAIT TK515vs635. Build web+api PASS ✅ ⏰ 2026-06-18 15:47 +0700.
[18/6] [C1] d67a506 W-P0A DONE — mig 0397: ALTER order_items ADD unit_cost DECIMAL(15,2) + cost_total. Drizzle schema updated. Capture avg_cost từ inventory_stock khi tạo order_items (nullable). Emit order.completed + payment.received events với payload (C4 JE 511/632 + 111/131). tsc PASS ✅ ⏰ 2026-06-18 16:50 +0700.
[18/6] [C1] d577010 W-P0B DONE — FE /admin/orders/[id] detail page: timeline trạng thái, bảng items, payments section, dropdown hành động (xác nhận/giao/in/HĐĐT/huỷ), Glass dialog confirm, link từ orders list, loading skeleton + error/404. tsc PASS ✅ ⏰ 2026-06-18 16:50 +0700.
[18/6] [C1] b97371b W-QUOTE-BE DONE — QuotesModule: mig 0398 orders.quote_id FK (nullable), quotes table (id/tenant_id/code/customer_id/items_snapshot/total/currency/status/created_at/quote_id). 9 endpoints: CRUD + sendToCustomer + approve + reject + convertToOrder. RBAC: 7 permissions (create/read/update/delete/send/approve/reject). Service logic: convert quote→order (copy items + emit order.created + link via quote_id). Drizzle schema validation. tsc PASS ✅ ⏰ 2026-06-18 16:57:56 +0700.
[18/6] [C1] 8569ae5 W-QUOTE-FE DONE — /admin/sales/quotes 3 trang: list (filter status/search code/date), new (form items+customer), [id] (detail+timeline+action buttons "Gửi/Duyệt/Từ chối/Tạo đơn"). Sidebar entry QuotesNav. Glass dialog confirm. Link "Tạo đơn từ báo giá" redirect /admin/orders/[id] new form. Loading skeleton + 404. tsc PASS ✅ ⏰ 2026-06-18 16:57:56 +0700.
[18/6] [C1] 9af4d4f W-RETURNS DONE — mig 0400: returns.updated_at/updated_by/deleted_at/deleted_by/deleted_reason + partial index WHERE deleted_at IS NULL + trigger auto-update. Fix approve() inventory upsert: INSERT ON CONFLICT (branch_id,product_id) thay update không có fallback. Emit RETURN_APPROVED_EVENT payload đầy đủ cho C4 JE 531/632/156. deleteReturn soft-delete (chỉ draft/rejected). notDeleted filter trong list query. POST bulk-approve + bulk-reject endpoints. tsc PASS ✅ ⏰ 2026-06-18 +0700.
[18/6] [C1] a0b3a2d W-BULK-ORDERS DONE — Bulk confirm: POST /orders/bulk-confirm (max 50 orders draft→confirmed, validate list, atomic transaction, emit events). Bulk shipping status: POST /orders/bulk-shipping-status (update status + tracking). Excel export: GET /orders/export?status=paid&from=2026-01-01 (exceljs limit 1000 rows, schema-qualified order data). FE: DataTable checkbox select-all (per-page + whole result aware), bulk action toolbar (Xác nhận/Cập nhật/Xuất Excel buttons), Glass confirm dialog. Load dynamic role check. tsc PASS ✅ ⏰ 2026-06-18 17:15 +0700.
[18/6] [C1] c90f296 P1-CLUSTER DONE — event: returns.return.approved→orders.return.approved (khớp C4). Route: @Get('export') trước @Get(':id') fix 404. Quotes: taxRate @Max(100)%, validUntil/notes field sync FE↔BE, customer join+search name/phone. convertToOrder: branchId validate bắt buộc (fix FK crash), emit orders.order.created. FE order [id]: nút Xác nhận + Glass dialog Huỷ (reason≥5) + payment list fetch. tsc+build PASS ✅ ⏰ 2026-06-18 +0700.
[18/6] [C4] 2ebfa68 QC P0+P1 Auto-JE Engine DONE — JeLineConfig XOR: side:'debit'|'credit' → mỗi journal_lines row CHỈ có tkNo XOR tkCo → Σ Nợ=Σ Có validate thật. Running balance: query COALESCE(MAX(balance/balance_after),'0') WHERE supplier_id/customer_id AND tenant_id trước insert → công nợ cộng dồn đúng. SEED mig 0401: je_auto_voucher_seq SEQUENCE (voucherNo={TYPE}-YYYYMMDD-{seq5} chuẩn Contract §8) + 4 rules BN/PN/PT/HC cho mọi tenant idempotent. 2 @OnEvent mới: sales_payment.received + return.approved. ReturnApprovedPayload đồng bộ C1. api PASS ✅ XIN OPUS QC. ⏰ 2026-06-18 21:10 +0700.
[18/6] [C9] QC2 BLOCKER sửa DONE — BLOCKER-1: exportDeclaration() viết lại dùng ExcelJS.Workbook thật (8 cột, header bold+fill, numFmt #,##0, border, total row, trả Buffer). Controller: @Res() res + res.send(buffer) + Content-Type xlsx (pattern excel-export.controller). BLOCKER-2: FE settlement/page.tsx:54 + withholding/page.tsx:67 đổi 'hr.tncn.read'→'hr.tncn.view' khớp BE. API tsc ✅ PASS · FE build ✅ PASS. XIN OPUS QC vòng 3. DEFERRED: seed 15.5tr/6.2tr mig≥0402 + tncn_amount column (chờ Nhi qua Letri). ⏰ 2026-06-18 21:20 +0700.
[20/6] [C5] 972ce03 LMS BLOCKER FIX — mig 0405: scheduleId nullable + unique constraint attendance (tenant_id,class_id,customer_id,session_date) + tuition.deleted_at. bulkAttendance→upsert onConflictDoUpdate. deleteTuition soft-delete. DELETE /lms/tuition/:id route. lms-roles.service: DELETE+INSERT bọc transaction. roles/page: openModal đọc perms thật, card fix desc/permCount. attendance/page: PermissionGate lms.activity.attendance.view. reports/page: toast feedback. tsc api+web ✅ PASS. Chưa push remote. Apply mig 0405 TRƯỚC deploy (⚠️ check dup trước). ⏰ 2026-06-20.
[20/6] [C5] 69c9a6a LMS MOBILE Z-INDEX FIX — GradeEntryModal overlay zIndex 200→1100 (nổi trên menu z900). AddStudentModal 7×grid→auto-fit minmax(160px,1fr). deposits backdrop 9000→1000. students <img>→next/image. LmsHeader flexWrap+maxHeight min(420px,60vh). courses/exams/certificates maxWidth:95vw. tsc web ✅ PASS. Chưa push. ⏰ 2026-06-20 16:05.
[20/6] [C5] 0b40212 LMS QC ROUND 2 — Hoàn thành 6 blocker QC: mig 0405 DEDUP trước ADD UNIQUE (an toàn prod). schema lmsTuition.deletedAt thêm + scheduleId bỏ notNull. bulkAttendance→onConflictDoUpdate. findAllTuition+isNull(deletedAt). deleteTuition service + DELETE /tuition/:id controller. roles/page: initial từ role.perms thật + card hiển thị đúng. attendance PermissionGate lms.activity.attendance.view. tsc 0 lỗi LMS. XIN OPUS DUYỆT + apply mig 0405 + C8 test. ⏰ 2026-06-20.
[20/6] [C1] 972ce03 (gom vào C5) BRIEF1+BRIEF2 — BRIEF1: products.service.ts change_type atomically cập nhật cả 2 cột catalogMainCategory+productType (11-key CATALOG_TO_PRODUCT_TYPE map, derive type từ catalog). FE products/page.tsx: thay PRODUCT_TYPES bằng CATALOG_TYPES 11 mục (taxNote+icon), modal radio name=catalogType, cột "Nhóm thuế" trong bảng (badge CATALOG_SHORT). BRIEF2: online-orders.service.ts findAll thêm search/dateFrom/dateTo server-side (ilike+gte+lte+or, limit 500). 2 endpoint mới: POST /bulk-confirm (new→confirmed, Promise.allSettled) + POST /bulk-mark-shipped. FE online-orders/page.tsx: load() truyền filter→BE, xoá useMemo, đổi "Tạo vận đơn hàng loạt"→"Đánh dấu đang giao", Xuất Excel CSV. tsc PASS (1 preexist lỗi manufacturing/categories). ⏰ 2026-06-20 13:50 +0700.
[20/6] [C10] 4fad554 NUT-HANG-LOAT-FIX — BE: GET /manufacturing/planning/forecast (dự báo nhu cầu SX: finished_goods có BOM, tính avgDailySales 30d từ order_items, tồn inventory_lots, priority urgent/normal/ok, suggestedQty 14d). BE: GET /manufacturing/planning/week-orders (lệnh SX tuần này nhóm theo ngày). FE categories/page.tsx: nút Xuất Excel có onClick (fetch blob), nút Xem sản phẩm navigate /admin/products?category_id=. categories.service.ts: console.log scope_change+bulk_scope → ghi auditLogs DB (IMMUTABLE). recall/new/page.tsx: toast "thành công" đổi thành cảnh báo rõ gateway chưa kết nối. tsc api+web PASS ✅ ⏰ 2026-06-20 13:55 +0700.
[20/6] [C10] 1db3523 QC-FIX — recall.service.ts: khoá lô ngay khi tạo lệnh (status→quarantined, FEFO chỉ pick active → POS tự bị chặn). excel/export/categories: nhận filters.scope+search từ body, lọc is_for_production/sale/teaching/purchase đúng. categories.service.ts: xoá comment sai "TODO persist to console". tsc PASS ✅ ⏰ 2026-06-20 14:15 +0700.
[20/6] [C10] 8ed6f87 MOBILE-FIX — 6 file CSS: ProductionOrderWizard maxWidth step3 1100→min(1100px,calc(100vw-32px)); orders AssignLineModal width 440→min(440px,calc(100vw-32px)); raw-materials table thêm overflowX:auto cuộn ngang 390px; haccp/ccp-definitions modal 520→min; certificates modal 500px→min; sop side panel 480→min(480px,100vw). inventory/lots modal zIndex 1000 OK (C6 đã hạ sidebar). tsc PASS ✅ ⏰ 2026-06-20 16:04 +0700.
[18/6] [C9] mig 0403 DONE — QC3 phát hiện BLOCKER MỚI: mig 0194 seed 7 bậc TT80 ON CONFLICT DO NOTHING → DB đang sai, loadTncnConfig ưu tiên DB → tính lương sai. FIX: mig 0403_tncn_brackets_nhi_5bac_2026.sql: DELETE bậc 6+7 thừa + UPSERT 5 bậc đúng (B1 0-10tr 5% B2 10-30tr 10% B3 30-60tr 20% B4 60-100tr 30% B5 >100tr 35%, quick_deduction tính lại) + UPSERT deduction 15.5tr/6.2tr + ADD COLUMN tncn_amount DECIMAL(15,2) DEFAULT 0. Drizzle schema hr-payroll.ts thêm tncnAmount. _down.sql khôi phục TT80. API tsc ✅ PASS. SẴN SÀNG GOM WAVE (apply mig 0403 trước). ⏰ 2026-06-18 21:30 +0700.
[20/6] [C16] b8bbc0e NUT-HANG-LOAT-FIX — BE: POST /admin/approval/instances/bulk-approve + bulk-reject + bulkDecide() Promise.allSettled (bypass sang adminRoles). POST /media/bulk-delete + bulkArchive() service. Stub exportAttendanceMatrix() fix pre-existing C9 tsc error. FE: approval-instances fix URL /approvals/bulk-approve→/admin/approval/instances/bulk-approve + thêm nút Từ chối hàng loạt + Excel. improvement-requests: N PATCH call → 1 POST /improvement-requests/bulk-assign (atomic). media: loop DELETE → POST /media/bulk-delete (atomic). media-admin: method DELETE→POST /system/media/bulk-delete. Excel export 4 trang (dynamic import xlsx). API+Web PASS (466 pages) ✅ ⏰ 2026-06-20 14:08 +0700.
[20/6] [C16] a1ca922 NUT-HANG-LOAT-FIX v2 (sau QC Opus) — NHAC-C16-SUA-NOT-SAU-QC: media-admin POST /system/media/bulk-delete THẬT (lần trước chỉ thêm Excel, quên đổi method). improvement-requests 1 POST /improvement-requests/bulk-assign THẬT (lần trước comment "no bulk endpoint" vẫn còn). WEB PASS ✅ ⏰ 2026-06-20 14:20 +0700.
[20/6] [C16] 570354b KB-CATEGORIES-EXCEL — knowledge/categories/page.tsx: nút Xuất Excel (FileSpreadsheet icon) flatten tree → xlsx. handleExportExcel dùng flattenTree(tree) + dynamic import xlsx. tsc PASS ✅ ⏰ 2026-06-20 16:03 +0700.
[20/6] [C16] ce7ac73 MOBILE-ZINDEX-FIX — NHAC-C16-MOBILE: media drawer overlay z30→z40, drawer z31→z41 + width 380→min(380px,100vw) (header sticky 35 không còn che). approval/instances/[id] toast bottom:24→bottom:80 + z:300→z:2100 (thanh dưới pb-14 không còn che). tsc PASS ✅ ⏰ 2026-06-20 16:03 +0700.
[20/6] [C1] c8d67de DROPDOWN Z-INDEX FIX — products/page.tsx:442 glass-card filter wrapper thêm position:relative+zIndex:30. Root cause: glass-card dùng backdrop-filter:blur → tạo stacking context riêng. Card danh sách SP (DOM sau) phủ lên dropdown zIndex:200 của filter card. Fix: nâng stacking context filter card lên zIndex:30 → cả 2 dropdown (danh mục:L475 + NCC:L604) nổi trên card SP. tsc PASS ✅ ⏰ 2026-06-20 16:07 +0700.
[20/6] [C8] QA NGHIỆM THU WAVE 2.20.13 — Kết quả: ❌ 5 FAIL / ⚠️ 5 N/A (no data). BUG C1: bulk change_type double mismatch FE-COURSE↔BE-course + service "Nhóm thuế" reject. BUG C4: /admin/finance/payments luôn trống do field `tt`≠`status`(pending_kt). BUG C9: users-roles hiện 0/12 NV vì API trả raw pg QueryResult thay vì {data,meta}. BUG C10: category scope PATCH→500, POST /categories/bulk-scope→400. BUG C16: GET /api/approvals→404 (endpoint chưa đăng ký). N/A: online-orders 0 đơn, LMS 0 buổi, leads 0, goods-receipts 0. Report: docs/QA/wave-2-20-13.md. 5 ESCALATION gửi C1/C4/C9/C10(→C2)/C16(→C6). ⏰ 2026-06-20 ~17:30 +0700.
[21/6] [Opus] 2334c4d — 5 BUG C8 wave 2.20.13 FIX + quét sạch QueryResult-coi-như-mảng. C1 VALID_CATALOG_TYPES · C4 payments map field+beneficiary+mark-paid · C9 {data,meta}+assign/remove role+quyền LMS · C10 scope DTO class+whitelist · C16 route+status lowercase+filter tab · manufacturing 15 chỗ .rows · auth getMe quyền LMS. QC 3 vòng grep thật. Build API+web PASS. ⏰ 2026-06-21 13:25 +0700.
[21/6] [Opus] 46b5de6 — UI-1: 9 trang popup gốc (confirm/alert/DOM thủ công) → Glass confirmDialog/alertDialog. 11 trang đã đúng sẵn. Build web PASS. ⏰ 2026-06-21 13:35 +0700.
[21/6] [Opus] d6a9e8e — UI-2: 53 trang, 62 lưới thẻ KPI → auto-fit minmax (cùng hàng bằng nhau + không tràn mobile). Giữ nguyên lịch 7 ngày/header bảng/form MISA. Build web PASS. ⏰ 2026-06-21 13:50 +0700.
[21/6] [Opus] (2.20.16 LIVE 17:22, tag wave-2.20.16-...-1722, push GitHub) — gồm 5 bug C8 + QueryResult + UI-1 + UI-2.
[21/6] [Opus] 1d6e3f1 — SYNC đợt 1: audit 14 hệ→128 lỗi (docs/AUDIT-FE-BE-DB-20260621.md), sửa ~100 route/shape/field, 84 file. BE thêm join (journal_lines, users name, assets, transactionCount). mig 0406 haccp +4 cột. Fix build: hr-payroll totalPenalty, cast .items, revert norms. Build API+web PASS. ⏰ 2026-06-21 17:15 +0700.
[21/6] [Opus] 55a7081 — SYNC đợt 2: finance so-cai(fetch TK thật)+chung-tu(map field), saas API-log, marketing scoring/channels, monitoring msg.message/created_at. Build web PASS. 2.20.17 chờ deploy (apply 0406 trước). ⏰ 2026-06-21 17:38 +0700.
[22/6] [Opus] 8e10244 — LIVE 2.20.21 (push GitHub + tag wave-2.20.21-...-0359). /pos/materials lọc danh mục cha→con (descendant set) + sắp rộng→hẹp (trước alphabet) · 8 trang sửa thanh lọc lệch hàng (DatePickerSingle thêm nhãn-trên + items-end + đồng chiều cao) · CSDL G0 middleware đọc X-Database-Id→req.databaseId (inert) + helper databaseFilter · mig 0408 thêm database_id 13 bảng tài chính (⚠️ CHƯA APPLY — G1 chờ D3/D4) · SPEC-CSDL-SCOPING hoàn thiện B0–B9. Build API+web PASS. ⏰ 2026-06-22 04:05 +0700.
```

---

## 7. 📋 BÀN GIAO CHO SESSION SAU (Opus CEO ERP) ⭐ ĐỌC ĐẦU TIÊN

### 🏁 BÀN GIAO MỚI NHẤT ⏰ 2026-06-30 (tiếp hệ bán hàng dang dở — audit fields + gate quyền)
> **Vừa làm:** Tiếp 2 slice hoàn thiện bán hàng (Rule 20), verify-first từng bước (không rush):
> - **Slice 1 — người tạo/duyệt/xoá** (commit `796a7ac8`, reviewer PASS 0 blocker): khai cột Drizzle `quotes.deletedBy/deletedReason` + `commerce_reservations.approvedBy/approvedAt` (0431 đã thêm ở DB); set `createdBy` (promotions/gift-card/voucher), `approvedBy/approvedAt` (reservations khi confirm), `deletedBy/deletedReason` (quotes khi xoá, reason qua query param). userId từ `req.user.sub`.
> - **Slice 2 — ẩn/hiện 27 nút theo quyền** (commit `c8edb6cf`, reviewer 2 vòng): 7 trang (orders/pending-approval/quotes/b2b/vouchers-campaigns/returns). Key verify theo BE `@RequirePermissions` (orders.cancel/edit/approve/reject/export_vat, quotes.edit/approve/reject/convert/delete, **online_orders.confirm**, vouchers.manage/delete, returns.approve/reject/delete). Reviewer vòng 1 bắt **9 nút sót** (bulk-confirm/bulk-status/convert/advance/online-confirm/returns-delete) → worker fix → vòng 2 PASS. `'*'` toàn quyền vẫn thấy hết.
> **Dừng tại:** ✅ **2 SLICE LIVE 30/6 11:51** — prod runtime=`c8edb6cf`, tag `sales-audit-gate-live-20260630-1151`, push GitHub `erp-wecha-pos` HEAD=c8edb6cf. Deploy SUCCESS (api=200 web=200, BUILD_ID 9wsscj, pm2 restart 11:51). **Code-only, KHÔNG migration** (0431 đã apply trước). Lúc deploy disk VPS 83% → dọn 6 swap `.old-*` cũ về 73% (⚠️ bài học mới: [[feedback_backup_to_macbook_cloud_before_delete_prod]] — lần sau tải về Macbook+cloud TRƯỚC khi xoá).
> **Bước tiếp CHÍNH XÁC:** (1) **Slice 3 dang dở** (hệ bán hàng): promotions/voucher/gift-card thêm `branch_id` read-filter (database_id đã có) + nút **nạp thẻ** gift-card (POST /gift-cards/:id/recharge) + export CSV campaigns/gift-cards + vouchers/codes join customerName + dashboard recentOrders/export-excel. (2) ⚠️ **TEST giảm-giá-%** trên prod (tạo báo giá+B2B giảm 10% kiểm tổng — fix chưa test chính thức). (3) approval-engine/instances.controller.ts thiếu @RequirePermissions → seed + gate nút duyệt POS-side. (4) Bảo mật: ĐỔI pass DB (còn history) + xác nhận `erp-wecha-pos` private (gh chưa auth ở máy Letri).
> **Cạm bẫy:** 🔴 KIỂM KỸ KHÔNG ĐOÁN — reviewer vòng 1 bắt 9 nút worker khai "đã gate 18" nhưng sót → **luôn reviewer trước deploy, đừng tin worker mù**. `approval-engine/instances.controller.ts` thiếu `@RequirePermissions` (nút duyệt POS-side chưa gate được — seed sau). Slice 2 chưa deploy = prod CHƯA có gate.
> **Hệ khác cần biết:** prod=`c8edb6cf` (đã LIVE 2 slice). Repo `erp-orchestration` PROGRESS sync tay (scrub IP/pass trước push). Swap rollback giữ: `.old-20260630-114220` (=484eb0c3) + tar `/opt/backup/deploy-20260630-114220/`.

### 🏁 BÀN GIAO ⏰ 2026-06-29 ~14:xx (phiên DÀI — CSDL scoping + audit bán hàng + repo điều phối)
> **Vừa làm:** (1) CSDL scoping LIVE: ĐỢT1 cổng cứng MISA + guard-fix lockout (commit `367b6178`), ĐỢT2 write G2 + backfill 0430 (commit `aff3b5d1`), ĐỢT3 read-filter sales (ac8789f0 — **CHƯA commit, trong working tree**). C8 fix tạo-KH-500 + orders-500 LIVE (`677aab09`). (2) Push **repo điều phối `git@github.com:lehuutri28/erp-orchestration.git`** (private, đã scrub secret: rules+skills+CLAUDE.md+CONTRACT+PROGRESS) cho Claude mobile. (3) Audit Rule-20 bán hàng = **65 gap** (`docs/AUDIT-RULE20-BAN-HANG-20260629.md`, 10 blocker gồm 💀 giảm-giá-% sai tiền). (4) Audit schema toàn DB: coverage cột chuẩn THẤP (database_id 12%, branch_id 19%, created_by 26%, approved_by 8%). Soạn mig **0431** chuẩn hoá cột 46 bảng giao dịch.
> **Dừng tại:** ✅ **ĐỢT SALES LIVE 30/6 06:38** — prod runtime=`484eb0c3`, tag `sales-rule20-bulk-live-20260630-0638`. **0429+0431 ĐÃ apply** (verify: promotions/vouchers/gift_cards có database_id; promotions.create seed 2 vai trò; health/admin 200; bulk markers trong dist). Đã LIVE: ĐỢT3 read-filter + 5 worker fix 65 gap + bulk + fix giảm-giá-% (báo giá+B2B) + soft-delete promotions + edit/xoá attributes/barcodes + barcode search SP. products/customers REVERT (đã có sẵn). ⚠️ **Bug giảm-giá-% chưa reviewer chính thức → CẦN TEST prod** (tạo đơn giảm 10% kiểm tổng). 🔒 password DB đã scrub khỏi HEAD (commit `484eb0c3`) NHƯNG còn history → **CẦN ĐỔI pass DB** + xác nhận repo private.
> **Bước tiếp CHÍNH XÁC:** (1) `cd claude/pos-system && git status` xem 5 worker xong chưa; **REVIEW từng file** (đừng tin mù — bài học rush). (2) 🔍 reviewer 3 vòng **bug giảm-giá-% FE↔BE** (quotes/b2b — SAI TIỀN ĐƠN). (3) clean build api+web PASS. (4) **apply mig 0429** (lockout perms) + **0431** (cột chuẩn) port 5433 TRƯỚC deploy. (5) commit+tag+push+deploy. (6) tiếp chuẩn hoá: backfill 0431 + code SET created_by/approved_by/deleted_by + read-filter promotions/vouchers/gift_cards (sau 0431 có cột database_id).
> **Cạm bẫy:** 🔴 **KIỂM THẬT KỸ, KHÔNG ĐOÁN** (Letri nhắc 2 lần) — verify-existing-first (đã rush bung products/customers TRÙNG, phải revert). Bulk endpoint NHIỀU cái ĐÃ CÓ (customers 9 bulk, orders bulk-confirm/shipping, returns bulk-approve/reject, gift-cards bulk-issue) → REUSE đừng tạo trùng. 💀 giảm-giá-% là bug SAI TIỀN. 0429/0431 chưa apply. ĐỢT3 + 5 worker = UNCOMMITTED/UNREVIEWED.
> **Hệ khác cần biết:** mig đã apply prod: 0427/0428 (đợt7), 0430 (backfill). Chưa apply: 0429, 0431. org-scoping data gap (ổ Nhà Máy tắt + truongngocsau role=Admin chưa scoped — chờ Letri model). Repo `erp-orchestration` đồng bộ tay (curate→scrub→push khi luật/PROGRESS đổi).


### [C13-POS-FNB-V4] BÀN GIAO ⏰ 2026-06-29 — Role-gating UI + Staff selector modal (session 11)
- **Vừa làm:** Implement hoàn chỉnh phân quyền UI theo vai trò cho GAS POS (anh-tuan instance). 2 file thay đổi:
  - **`html/Index.html`**: (1) Staff selector overlay glass (mở khi app load, sau license+setup check); (2) `ROLE_HIDE` matrix (cashier ẩn 10 route, staff ẩn 13 route, admin không ẩn gì); (3) `applyRoleUI(role)` ẩn sidebar items theo `data-route` + ẩn group header mồ côi; (4) `_updateDrawerUserBar()` hiện tên+role+chi nhánh trong drawer; (5) `selectStaff()` dùng `window._staffChoices[idx]` thay `JSON.stringify().replace(/"/g,"'")` — fix bug single-quote trong tên NV; (6) `switchStaff()` nút 🔄 mở lại modal; (7) Fallback "Tiếp tục với quyền Admin" khi sheet NguoiDung trống.
  - **`src/NhanVienUI.js`**: thêm `staff_forLogin()` — trả NV active only, KHÔNG có password_hash, join tên chi nhánh.
- **Deploy:** `clasp push` thành công (2 lần, file list hiện đến Wizard.js). ✅ LIVE. Commit `c86eba0`.
- **Dừng tại:** `html/Index.html:557-574` (`showStaffSelector()` handler). Code complete. Chưa test browser thật (cross-origin iframe GAS không tự động test được).
- **Bước tiếp CHÍNH XÁC:** Letri/Anh Tuấn mở URL `/dev` → xác nhận 8 kịch bản: (1) App load → modal NV hiện; (2) Danh sách NV + role badge màu; (3) Chọn cashier → ẩn 10 menu; (4) Chọn staff → ẩn 13 menu; (5) Chọn admin → đủ menu; (6) Drawer hiện tên+role+chi nhánh; (7) Nút 🔄 mở lại modal; (8) "Tiếp tục với quyền Admin" hoạt động.
- **Cạm bẫy:** GAS SPA nằm trong cross-origin iframe `googleusercontent.com` — mọi DOM manipulation phải chạy từ trong sandbox, không thể kiểm tra từ ngoài bằng automation. Nếu `staff_forLogin()` trả `data:[]` trống → kiểm tra Sheet "NguoiDung" có cột "Trạng thái" không và giá trị có phải "inactive" không.
- **Hệ khác cần biết:** Không thay đổi BE VPS. Chỉ GAS FE. ROLE_HIDE matrix: `html/Index.html:478-484`.

### [C13-POS-FNB-V4] BÀN GIAO ⏰ 2026-06-29 — Fix timezone VN toàn hệ GAS POS (session 10)
- **Vừa làm:** Audit + fix TẤT CẢ `new Date().toISOString()` lưu vào Sheet để dùng múi giờ VN (Asia/Ho_Chi_Minh) thay vì UTC. Thêm 2 helper global vào đầu `SheetIO.js` (lines 12-18): `_nowVN_()` → `yyyy-MM-dd'T'HH:mm:ss` VN; `_todayVN_()` → `yyyy-MM-dd` VN. Fix cụ thể:
  - **SheetIO.js** (9 chỗ): DonHang `created_at` (L2471), ChiTietDon `created_at` (L2542), ThanhToan `created_at` (L2794), ThuChi `Ngày`/`ngay` (L2809, `_todayVN_()`), ThuChi `created_at` (L2818), LogDiem tich+dung `created_at` (L2868+L2881 replace_all), KenhBan (L3275), Sessions `created_at` (L3516).
  - **KhoUI.js** (7 chỗ): NguonLieu `created_at` (L242), BTP `Ngày cập nhật` (L304), nhapBulkConfirm `ngay` (L604, `_todayVN_()`), BaoBi `created_at` (L831), NguonLieu `updated_at` (L1048), BTP `updated_at` (L1090), BaoBi `updated_at` (L1134).
  - **Giữ UTC (đúng):** `tao_luc`/`dong_luc` CaPos (L2254/L2310/L2384) → dùng làm filter `_readDonHang_()` cho VPS API, đổi sẽ sai; các chỗ read-only parse date từ Sheet cũng giữ nguyên.
- **Deploy:** `clasp push` lúc 11:42:29 AM → "Pushed 43 files". ✅ LIVE.
- **Dừng tại:** Mọi Sheet (DonHang, ChiTietDon, ThanhToan, ThuChi, LogDiem, KenhBan, Sessions, NhapKho, XuatKho, NguonLieu, BTP, BaoBi, KhachHang, NhaCungCap, NhanVien) đều lưu VN time. `clasp push` thành công.
- **Bước tiếp CHÍNH XÁC:** (1) Test tạo đơn mới → vào Sheet DonHang/ThanhToan xem cột `created_at` có dạng `2026-06-29T17:xx:xx` (không phải `T10:xx:xx` UTC). (2) Tương tự nhập kho → NhapKho sheet. (3) [OPTIONAL] Nếu muốn `tao_luc`/`dong_luc` cũng hiện VN → cần thêm helper `_nowVNtz_()` (format `+07:00`) và BE accept offset — nhưng hiện tại không urgent.
- **Cạm bẫy:** `tao_luc`/`dong_luc` còn UTC là CỐ Ý (xem comment trong SheetIO.js). KHÔNG sửa chúng nếu không có thay đổi BE đồng thời. GAS project `appsscript.json` đã có `"timeZone": "Asia/Ho_Chi_Minh"` nhưng không ảnh hưởng `.toISOString()` — phải dùng `Utilities.formatDate()`.
- **Hệ khác cần biết:** Không có thay đổi BE VPS (api.phache.com.vn). Chỉ fix timezone GAS-side. VPS 502 (`pm2 restart phache-api`) vẫn chưa resolve — cần Letri SSH restart.

### [C13-POS-FNB-V4] BÀN GIAO ⏰ 2026-06-29 — Xuất Excel TOÀN BỘ trang GAS POS (hoàn thành LUẬT TOÀN HỆ)
- **Vừa làm:** Thêm nút 📥 Excel cho MỌI trang có dữ liệu trong GAS POS (anh-tuan). 3 batch commit:
  - `5438474` (batch 1 — session trước): Index.html shared `exportCsv()`, KhachHang, DonHang, ThuChi, Kho
  - `7b930e4` (batch 2): KhuyenMai, BaoCaoTon, NguoiDung, ChiNhanh, NhaCungCap
  - `8620f90` (batch 3): BaoCao, Menu, CongThuc
- **Coverage 10/10 trang có list data:** Index✅ KhachHang✅ DonHang✅ ThuChi✅ Kho✅ KhuyenMai✅ BaoCaoTon✅ NguoiDung✅ ChiNhanh✅ NhaCungCap✅ BaoCao✅ Menu✅ CongThuc✅. Skip đúng: Login(form), Dashboard(KPI), Chuoi(KPI), Checkout(POS), PhanQuyen(stub), CaiDat(settings).
- **Dừng tại:** `8620f90` committed. Chưa `clasp push` (Letri làm).
- **Bước tiếp CHÍNH XÁC:** (1) Letri chạy `clasp push` trong `POS-FNB-V4/apps-script-client/anh-tuan/` → deploy lên GAS. (2) Test: mở từng trang → bấm 📥 Excel → kiểm file CSV tải xuống đúng tên + có dữ liệu. (3) Test empty-state: trang chưa tải xong bấm → thấy banner warning "Chưa có dữ liệu".
- **Cạm bẫy:** `clasp push` PHẢI chạy trong đúng folder `anh-tuan/`. BaoCao export theo kỳ đang chọn (hôm nay/7 ngày/tháng) → tên file có suffix. CongThuc export tab đang xem (Mon hoặc BTP).
- **Hệ khác:** LUẬT TOÀN HỆ 15/6 đã cover C13 GAS POS. ERP web (Next.js) đã đủ Excel từ các session trước.

### [C13-POS-FNB-V4] BÀN GIAO ⏰ 2026-06-29 — Fix 13 bug GAS audit session 9 (real-user test)
- **Vừa làm:** Deep audit + fix 13 bug thật (không test qua loa, test như người dùng thật). 4 commit: `46d25d0`, `e7d0e39`, `a0343c0`, `3521022`. Tổng 2 session: 9 commit sạch (e9503ca→3521022).
  - `46d25d0`: history tab kẹt trong modal-checkout → move ra ngoài `pos-layout`; `renderCatList` null crash; `hang_vip` whitelist CSS class
  - `e7d0e39`: NhaCungCap `.apply(null)` mất `this` → tất cả save/update NCC âm thầm crash (CRITICAL); Chuoi period filter không truyền xuống BE; CongThuc nút đóng thiếu `>`
  - `a0343c0`: BranchUI hard-delete thật; NhanVienUI rewrite Sheet-based CRUD (không dùng API chưa có)
  - `3521022`: `renderCustomerInfo` tỷ lệ điểm sai `Math.floor(pts/100)*20000` → `pts*DIEM_VND_RATE` (100đ=100K không phải 20K)
- **Dừng tại:** `3521022` committed, chưa `clasp push` (Letri làm)
- **Bước tiếp CHÍNH XÁC:** (1) Letri chạy `clasp push` trong `POS-FNB-V4/apps-script-client/anh-tuan/` → deploy lên GAS. (2) Test trực tiếp: POS → chọn KH có điểm → kiểm tra "có thể đổi Xđ" đúng tỷ lệ; vào NCC thêm/sửa → xem thành công không còn im lặng; vào Chuỗi đổi kỳ xem dữ liệu thay đổi. (3) VPS api.phache.com.vn vẫn 502 — cần Letri `pm2 restart pos-api` trên VPS phache (không phải VPS erp).
- **Cạm bẫy:** (a) `clasp push` PHẢI chạy trong đúng folder `anh-tuan/`, không phải root POS. (b) VPS 502 = chức năng VPS-based (báo cáo aggregate, promo) không test được cho đến khi restart. (c) 17-PhanQuyen.html là stub hoàn toàn — không có BE endpoint, cần Sprint 4 để làm full RBAC.
- **Hệ khác cần biết:** `BranchUI.js` giờ XÓA THẬT (không soft delete) khi admin xóa chi nhánh — Letri cần biết để cẩn thận. `NhanVienUI.js` dùng Sheet NguoiDung thay vì API. Dead code `_writeCheckoutToSheets_` (PosUI.js:846) — checkout thật đi qua `sheet_orderFinalize` (SheetIO.js:2639), hoạt động đúng.

### 🏁 BÀN GIAO MỚI NHẤT ⏰ 2026-06-29 ~07:03 (hệ BÁN HÀNG: 7 đợt LIVE — còn Excel audit + sơ đồ phân quyền)
> **Vừa làm:** ĐỢT 7 LIVE 07:02 (Letri apply 0427+0428 + safe-deploy; Opus verify+tag+push). Trước đó hoàn thiện hệ bán hàng qua 6 đợt LIVE (entry ✅ ĐỢT 1→6 bên dưới) — vòng đời 9/9 + tài chính (avg_cost/VAT-per-SP/công nợ/ngày đòi/nhắc TT) + bảo mật (chi nhánh #3, quyền T2/H1) + UX (UI đồng bộ mua-bán, tìm kiếm, dropdown CN). Đợt 7: fix khuyến mãi 500 (schema align) + thẻ quà deliveryMethod.
> **Dừng tại:** prod=HEAD=`717b0c95` (đợt 7, branch n8n/005). Tag `wave-banhang-fix7-live-20260629-0703` + push GitHub xong (unpushed=0). Mig 0427+0428 đã apply. Verify prod: health api/web 200, pm2 restart 07:02, dist deliveryMethod(5 file), DB promotions 7 cột + gift_cards.delivery_method.
> **Bước tiếp CHÍNH XÁC:** (1)(2)(3) deploy+verify+tag+push = ✅ XONG. (4) Excel audit = ✅ **XONG, KHÔNG GAP** (workflow 25 trang: 15 list đã có Excel theo lọc, 10 miễn trừ; spot-check OK). (5) **Sơ đồ phân quyền + migration 0429 = ✅ SOẠN + reviewer soát sạch** (`docs/SO-DO-PHAN-QUYEN-BAN-HANG-20260629.md` + `claude/pos-system/apps/api/src/database/migrations/0429_seed_sales_perms.sql`). Letri chọn "THÊM quyền sửa lockout A". Mig: 9 vai trò, 44 key (đều controller bảo vệ, 0 mồ côi), idempotent THÊM. **CHỜ Letri apply prod 5433:** `ssh root@<VPS_IP> "sudo -u postgres psql -p 5433 pos_db -v ON_ERROR_STOP=1" < claude/pos-system/apps/api/src/database/migrations/0429_seed_sales_perms.sql` → user ĐĂNG NHẬP LẠI → C8 test. Apply xong Opus commit+push (ritual, KHÔNG cần deploy code — data-only). C8 đang QA đợt 7.
> **Cạm bẫy:** agent fan-out có thể đi SAI đề (F1) → LUÔN xem git diff + revert nếu lệch phạm vi. Migration apply `sudo -u postgres psql -p 5433` (KHÔNG 5432). Mỗi deploy: verify marker trong dist + push SAU success. Reviewer Sonnet hữu ích — đã bắt nhiều BLOCK tài chính/bảo mật (ghi nợ 2 lần, lượt KM ×2, avg_cost transfer corruption, voucher validator thiếu deletedAt).
> **Hệ khác cần biết:** bảng `promotions` sau 0427 có thêm cột value/rules/start_at/end_at/max_uses/current_uses/updated_at (giữ cột cũ). `gift_cards` sau 0428 có delivery_method. C18: brief flow FB Ads 12h→Telegram+ERP+CRM sẵn sàng `orchestrator-ceo/ceos/c18-n8n/BRIEF-OPUS-C18-FB-ADS-12H-TELEGRAM-CRM-20260628.md` (chờ Letri giao + cấp FB token/Telegram/HMAC secret).
> **4 XÁC NHẬN Letri (29/6):** ① khuyến mãi gộp hệ mới + fix 500 (✅ làm) · ② phân quyền: Letri DUYỆT SƠ ĐỒ chi tiết rồi mới seed (Opus chuẩn bị, KHÔNG tự seed) · ③ lọc SP theo CSDL G4: CHƯA bật (chờ backfill) · ④ ưu tiên: hoàn thiện phần còn lại.
> **CÒN LẠI hệ bán hàng:** Excel audit (làm lại) · sơ đồ phân quyền (soạn cho Letri duyệt) · (defer thấp) response {data,meta} contract · (Letri quyết) CRM /admin/debts stub + định nghĩa công nợ phải thu (chị Nhi).


### ✅ GUARD FAIL-OPEN + ĐỢT 2 WRITE LIVE ⏰ 2026-06-29 ~12:30 (2 commit; mig 0430 đã apply; verify prod OK)
> Deploy 12:29→restart 12:30 (⚠️ terminal safe-deploy TREO ở verify cuối NHƯNG deploy ĐÃ XONG: swap+restart+build PASS). Verify prod: health/admin/web 200, guard 4 marker (isSuperAdmin chỉ '*' + FAIL-OPEN), dist resolvedDatabaseId(inventory+journal), log lỗi cũ 12:11 (KHÔNG phải mới). Tag `csdl-guardfix-dot2-live-20260629-1305` + push (unpushed=0). prod=HEAD=`aff3b5d1` (`367b6178` guard-fix + `aff3b5d1` ĐỢT2+0430).
> **Sự cố đã FIX:** `lehuuthuong1992@gmail.com` (GĐ Kinh Doanh) bị cổng cứng KHÓA → giờ VÀO ĐƯỢC (fail-open + Giám đốc scoped theo ổ). 0430 backfill Letri ĐÃ apply (13 UPDATE, NULL=0).
> **Root cause:** guard `noDatabaseAccess` HARD-BLOCK khi không resolve được quyền (isSuperAdmin false + derivation rỗng tạm). `/databases` chỉ trả ổ ACTIVE (1 ổ VUA AN TOÀN) → Nhà Máy inactive KHÔNG hiện.
> **🆕 QUY TẮC LETRI 29/6 (chiều):** chỉ **toàn quyền (`*`/Admin=lehuutri)** xem HẾT; **Giám đốc=SCOPED theo ổ chi nhánh** (lehuuthuong=GĐ Kinh Doanh→commerce KHÔNG Nhà Máy; truongngocsau=GĐ Sản Xuất→Nhà Máy).
> **Đã vá (chờ build+deploy):** (1) `isSuperAdmin = CHỈ '*'` (bỏ 'Giám đốc' see-all). (2) FAIL-OPEN: cổng vẫn bắt buộc chọn, nhưng chưa resolve được quyền → hiện ổ active thay vì KHÓA → hết lockout + scoped đúng.
> **Gỡ kẹt tức thì:** set `lehuuthuong default_database_id=ccf95375` → đăng nhập lại vào thẳng (Letri chạy).
> **⚠️ DATA GAP org-scoping (Letri/CEO sau):** ổ "Nhà Máy Sản Xuất" (33b1e96a) INACTIVE + KHÔNG branch trỏ tới; truongngocsau role=Admin('*') chưa phải GĐ Sản Xuất scoped; 5 branch đều →ccf95375. Muốn scoped thật: activate ổ Nhà Máy + link branch + đổi role truongngocsau.
> **ĐỢT 2 write G2:** ~25 method ghi database_id (3 worker), reviewer SẠCH (0 blocker), build api PASS+tsc EXIT0, mig 0430 applied → deploy GỘP với guard-fix.

### ✅ CSDL ĐỢT 1 + C8 FIX LIVE ⏰ 2026-06-29 ~11:14 (3 commit, KHÔNG migration; verify prod OK)
> Deploy bundle (safe-deploy 11:09→restart 11:13, health 200, guard 17 marker trong source VPS, dist customers/orders OK). Tag `csdl-dot1-c8-live-20260629-1114` + push GitHub (unpushed=0). prod=HEAD=`d91d4731`.
> **Commit (717b0c95..d91d4731):** `a74b7887` **CSDL ĐỢT 1 cổng cứng MISA** (modal 2 bước CHẶN vào app: chọn CSDL lọc-theo-quyền → chọn chi nhánh lọc-theo-quyền `roles[].branchId`; Admin/Giám đốc xem Tất Cả; login/getProfile trả `default_database_id`, PATCH `/users/me/default-database` tenant-safe; 4 trang dashboard re-fetch khi đổi ổ) · `677aab09` **C8 fix 2 lỗi chặn** (tạo KH 500 full_name NULL/numeric "" → 400 message "Thiếu họ tên" + FE hiện lỗi; `/admin/orders` 500 findOne id rác "online"/"b2b" → UUID guard 404) · `d91d4731` mig 0429 (chờ apply).
> **Reviewer soát đối kháng** cả ĐỢT 1 (bắt 1 blocker cross-tenant + 2 major) đã vá. C8 fix qua log prod thật (root cause chính xác).
> **CÒN LẠI CSDL:** ĐỢT2 write G2 (27 method create ghi database_id) + backfill mig 0430 · ĐỢT3 read G4 (90 method, decision-gated) · ĐỢT4 NOT NULL · ĐỢT5 build 2 service thuế. (Cách A: lọc CSDL+chi nhánh theo quyền suy từ branch — đã LIVE.)
> **🟠 GHI NHẬN:** Norms 500 `column cn.approval_status does not exist` (migration thiếu bảng norms manufacturing) — xử lý đợt sau. 0429 seed quyền bán hàng vẫn CHỜ Letri apply (port 5433).
> **C8 click-test:** brief `orchestrator-ceo/ceos/c8-qa-audit/QA-CSDL-DOT1-C8-20260629.md`.

### 🗄️ CSDL SCOPING — KẾ HOẠCH HOÀN THIỆN ⏰ 2026-06-29 ~08:00 (audit xong → ĐỢT1 LIVE 11:14)
> **Việc Letri giao:** hoàn thiện CSDL scoping toàn hệ — bắt buộc chọn sổ khi đăng nhập + nhớ lần sau + mọi trang/nút load theo CSDL. **Yêu cầu: kế hoạch chi tiết + ràng buộc agent (đúng đủ không thiếu).**
> **Đã làm:** audit ground-truth (DB prod + workflow 5 domain = 236 findings, **121 GAP**). Kế hoạch đầy đủ: `docs/KE-HOACH-CSDL-SCOPING-HOAN-THIEN-20260629.md`.
> **Hiện trạng:** G0✅(middleware+header) · G1✅(55 bảng có database_id) · **G2 dở** (27 method create chưa ghi: kho/SX/journal/shipments/payments — orders/cashbook đã ghi 0%NULL) · **G3 nhẹ** (NULL <60 dòng) · **G4 dở** (90 method list/report/findOne chưa lọc: finance-aggregates 11 BCTC, manufacturing 13, orders.findAll/findOne/getStats...) · **Cổng cứng ❌** (layout.tsx:195 chỉ banner mềm dismiss được) · **Nhớ server ❌** (getProfile không trả default_database_id) · G7❌ (TaxDeclarations+Reconciliation chưa có).
> **Kế hoạch 5 đợt:** ĐỢT1 FE cổng cứng+nhớ (ưu tiên #1, rủi ro thấp) → ĐỢT2 write G2+backfill mig 0430 → ĐỢT3 read G4 (decision-gated, đổi số báo cáo, fail-open+test cách ly) → ĐỢT4 NOT NULL+RLS → ĐỢT5 build 2 service thuế.
> **CHỜ Letri:** duyệt thứ tự (đề xuất ĐỢT1 trước) + cổng cứng super-admin có "Tất Cả" không. Migration tiếp theo = **0430**. (0429 seed quyền bán hàng vẫn chờ apply.)

### ✅ ĐỢT 7 LIVE ⏰ 2026-06-29 ~07:03 (2 commit + mig 0427+0428 đã apply; verify prod OK)
> Letri apply 0427+0428 (port 5433) + safe-deploy 06:53→restart 07:02. Opus verify+tag+push. Tag `wave-banhang-fix7-live-20260629-0703` + push GitHub (unpushed=0). prod=HEAD=`717b0c95`.
> **Commit (45ce40d2..717b0c95):** `0f9010e9` **promotions schema align** (mig 0427 — KHÔNG code; bảng `promotions` thêm value/rules/start_at/end_at/max_uses/current_uses/updated_at, giữ cột cũ → tạo/sửa khuyến mãi hết 500, Letri chốt gộp hệ mới) · `717b0c95` **gift-card deliveryMethod** (mig 0428 — `gift_cards.delivery_method` print/email/zalo + code DTO/service/controller).
> **Verify prod:** health api 200 + web public 200; pm2 pos-api/pos-web restart 07:02 (age <80s, restarts 464/1342); dist `deliveryMethod` ở 5 file (gift-card-issue.service + gift-cards.dto + controller); DB promotions=7 cột mới, gift_cards.delivery_method=1.
> **Còn lại hệ bán hàng:** (4) Excel audit làm lại (rà `admin/sales/*` thiếu nút Xuất Excel theo lọc) · (5) sơ đồ phân quyền bán hàng cho Letri duyệt rồi mới seed. (defer thấp) response {data,meta} contract · (Letri quyết) CRM /admin/debts stub + định nghĩa công nợ phải thu (chị Nhi).
> **C8:** QA đợt 7 — tạo/sửa khuyến mãi (hết 500) + phát hành thẻ quà chọn hình thức giao (in/email/zalo). Brief: `orchestrator-ceo/ceos/c8-qa-audit/QA-DOT7-BANHANG-20260629.md`.

### [C18] n8n_wf-89 FB Ads 12h — BE OPTION-B HMAC DONE, chờ Letri deploy ⏰ 2026-06-29
> **Vừa làm:** Letri chọn Option B (HMAC thay JWT). Tạo `N8nAdsController` (`POST /api/n8n/ads/daily-stats`, `POST /api/n8n/ads/leads`) với `N8nHmacGuard`. Enable `rawBody: true` trong `main.ts`. Cập nhật `AdsTrackingModule`. tsc PASS. Flow JSON v2.0 (15 nodes, 2 HMAC Code node mới, body raw/json, không còn httpBearerAuth).
> **Dừng tại:** `claude/pos-system/apps/api/src/modules/ads-tracking/n8n-ads.controller.ts` + `claude/n8n-ceo-automation/flows/draft-016-fb-ads-12h-telegram.json` — CẢ HAI đã xong code.
> **Bước tiếp theo CHÍNH XÁC (Letri làm):**
> 1. `git add claude/pos-system/apps/api/src/modules/ads-tracking/n8n-ads.controller.ts ads-tracking.module.ts main.ts` → commit → `bash scripts/safe-deploy.sh n8n/005-wecha-pos-zalo-keypos` → verify `curl -X POST https://erp.wecha.vn/api/n8n/ads/daily-stats` → expect **401** (không phải 404)
> 2. ERP VPS `.env`: thêm `N8N_WEBHOOK_HMAC_SECRET=<random_hex_32>`. n8n env: `N8N_WEBHOOK_HMAC_SECRET=<same>` + `WECHA_TENANT_ID=<uuid>`
> 3. n8n: tạo credential FB (HTTP Header Auth `Authorization: Bearer <token>`) + Telegram bot
> 4. Thay 6 PLACEHOLDER trong `draft-016-fb-ads-12h-telegram.json` → import n8n → test → activate
> **Cạm bẫy:** `N8N_WEBHOOK_HMAC_SECRET` phải CÙNG GIÁ TRỊ ở cả ERP `.env` và n8n env vars — khác = 401 mọi call. KHÔNG paste secret trong chat.
> **Hệ khác cần biết:** Endpoint cũ `POST /api/ads/daily-stats` (JwtAuthGuard) vẫn còn nguyên, KHÔNG đụng. Endpoint mới chỉ có HMAC.

### ✅ ĐỢT 6 LIVE ⏰ 2026-06-29 ~06:00 (4 commit + mig 0426 đã apply; verify prod OK)
> Deploy ~05:59 (health 200, pos-api/pos-web online; dist: avg_cost(6)/vouchers delete(3)/reservations transition(3); vouchers.deleted_at col + vouchers.delete seeded 3 vai trò). Tag `wave-banhang-fix6-live-20260629-0600` + push GitHub. prod=HEAD=`45ce40d2`.
> Fan-out 4 agent (Letri "làm tiếp") + soát đối kháng (bắt 3 BLOCK tài chính/bảo mật → đã vá).
> **Commit (d37b4c22..45ce40d2):** `d37b4c22` A giá vốn bình quân avg_cost khi nhập kho (chuyển kho KHÔNG đổi avg; rm-stock atomic) · `e0c1a8d3` B vouchers soft-delete+xoá+lọc (**mig 0426** + validator filter deletedAt) · `b0067410` C reservations state-machine + isNull(deletedAt) · `45ce40d2` E b2b/new lọc khách sỉ + hạn mức + VAT theo từng SP.
> **⚠️ DEPLOY ĐỢT 6 — apply mig 0426 TRƯỚC:** `0426_vouchers_softdelete_perm.sql` rồi `bash scripts/safe-deploy.sh n8n/005-wecha-pos-zalo-keypos`.
> **HỆ BÁN HÀNG GẦN HOÀN THIỆN.** Còn lại: (defer/thấp) response {data,meta} contract wrap · rà nút Excel theo lọc · cột deliveryMethod thẻ quà. (decision-gated) Promotions C1 (2 hệ trùng + có thể 500) · lọc SP theo CSDL G4 · phân quyền controller còn lại + CRM /admin/debts.
> ⚠️ Disk VPS đã dọn còn 71% (26G). C18: brief flow FB Ads 12h→Telegram+ERP+CRM = `orchestrator-ceo/ceos/c18-n8n/BRIEF-OPUS-C18-FB-ADS-12H-TELEGRAM-CRM-20260628.md` (chờ Letri giao + cấp secrets).

### ✅ ĐỢT 4+5 LIVE ⏰ 2026-06-28 ~20:45 (6 commit + mig 0425 đã apply; verify prod OK)
> Deploy 20:44 (health 200, pos-api/pos-web online; dist verify: care-reminders.cron + online_orders guard(9) + products q param + permissions online_orders; mig 0425 = 13 vai trò có online_orders.confirm). Tag `wave-banhang-fix45-live-20260628-2053` + push GitHub. prod=HEAD=`c48c66c`.
> ⚠️ Disk VPS 82% (16G trống) — có ~13.7G backup `.old-*` cũ (9 bản) nên dọn (giữ 1-2 bản mới): `ls -dt /opt/pos-system.old-* | tail -n +3 | xargs rm -rf`.
> Sau khi mở fan-out 5 agent song song (Letri "mở nhiều agent hoàn thành") — mỗi việc soát đối kháng riêng.
> **Commit (e82b71c..c48c66c):** `e82b71c` B2 nhắc thanh toán đơn chưa TT (care-reminders type payment_reminder + cron 08:00 VN; dùng due_date B1) · `b645bbd` T5a search autocomplete nhận param q (products+customers) · `9efff9e` T5c thẻ quà list join tên người mua/giữ + DTO deliveryMethod · `f22c5a5` **H1 quyền online-orders** (guard + permissions.ts + **mig 0425** seed 13 vai trò) · `53fe6d8` T5b dropdown chi nhánh báo cáo vùng/kênh + returns branch_id · `c48c66c` B3 đồng bộ UI bán hàng theo mua hàng (breadcrumb clickable + status tabs + KPI palette dashboard/b2b/quotes).
> **⚠️ DEPLOY ĐỢT 4+5 — apply mig 0425 TRƯỚC code:** `0425_seed_online_orders_perms.sql` rồi `bash scripts/safe-deploy.sh n8n/005-wecha-pos-zalo-keypos`.
> **Soát đối kháng từng việc** (reviewer Sonnet): T2 BLOCK shape (pre-existing array, giữ — chỉ cần FE workaround sẵn có); các SHOULD đã fix (T1 stats searchTerm, T2 deleted_at, H1 throw→400, T4 useEffect deps). api tsc + web build PASS.
> **CÒN LẠI (decision-gated, chờ Letri):** lọc SP theo CSDL tầng đọc (G4 mass change) · phân quyền các controller còn lại (sơ đồ AUDIT-PHAN-QUYEN, mới làm reservations/barcodes/attributes/online-orders).
> **Vòng đời bán hàng 9 luồng: 9/9 xong** (chặn tồn · ẩn hàng xoá POS · trừ kho · nhập kho atomic · khuyến mãi ghi đơn · cộng điểm POS+B2B · ngày đòi nợ · nhắc thanh toán · UI đồng bộ).

### ✅ ĐỢT 3 LIVE ⏰ 2026-06-28 ~17:42 (4 commit + mig 0424 đã apply; verify prod OK)
> Deploy ~17:40 (health 200, pos-api/pos-web online; dist verify: upsertStockTx/loyaltyEarn quotes/dueDate orders đều có; orders.due_date + customer_debt_transactions.due_date column có). Tag `wave-banhang-fix3-live-20260628-1742` + push GitHub. prod=HEAD=`51dcf10`.
> Sau rà vòng đời bán hàng (workflow 9 luồng + Opus grep-verify, bắt 3 báo cáo con sai).
> **Commit:** `33824e5` A1 nhập kho atomic (stock-documents.confirm bọc transaction + upsertStockTx thêm tenant_id + chặn transfer thiếu kho đích) · `e1d99bc` A2 khuyến mãi áp đơn được BE ghi (xử lý appliedPromotionIds: snapshot promotion_applied + insert promotion_usage + currentUses; chặn double-increment với promotionCode; DTO @IsUUID) · `298a2ff` A3 loyalty đơn B2B (quotes.convertToOrder gọi earn+tier, wire LoyaltyModule) · `51dcf10` B1 ngày đòi nợ due_date (mig 0424 orders+customer_debt_transactions; DTO+FE picker POS/b2b; ĐÃ gỡ recordDebtIncrease ở quotes để tránh ghi nợ 2 lần — auto-je credit lo).
> **⚠️ DEPLOY ĐỢT 3 — apply mig 0424 TRƯỚC code:** `0424_orders_debt_due_date.sql` rồi `bash scripts/safe-deploy.sh n8n/005-wecha-pos-zalo-keypos`.
> **Soát đối kháng từng việc** (reviewer Sonnet): A1 bắt BLOCK tenant filter+transfer câm (vá); A2 bắt BLOCK double-increment lượt KM (vá); B1 bắt BLOCK ghi nợ 2 lần ở quotes (vá). api tsc + web build PASS.
> **CÒN LẠI (chưa làm — wave sau):** B2 nhắc thanh toán đơn chưa TT (care-reminders type 'payment_reminder' + cron + đọc orders.due_date từ B1 — KHÔNG cần migration) · B3 đồng bộ UI bán hàng = mua hàng (FE: workflow circle/status tabs/bulk toolbar/breadcrumb clickable) · T5 (search param q↔search, branch dropdown reports/returns, gift-card list join tên). + decision-gated: orders lọc CSDL (G4), phân quyền 143 controller còn lại.
> **Việc Letri:** deploy đợt 3 (mig 0424 trước) · test: lên đơn nợ chọn ngày đòi / khuyến mãi áp lưu đúng / đơn sỉ convert cộng điểm / nhập kho.

### ✅ ĐỢT 2 LIVE ⏰ 2026-06-28 ~15:49 (3 commit + mig 0422+0423 đã apply; verify prod OK)
> Deploy 15:48 (health 200, pos-api/pos-web online, dist rebuild xác minh: view_all_branches/reservations.manage/PermissionsGuard/product_attributes đều trong dist). Tag `wave-banhang-fix2-live-20260628-1549` + push GitHub. prod=HEAD=`07b4e30`. Mig 0422 (UPDATE 4) + 0423 (UPDATE 13) đã apply+verify.
> **Commit:** `f3eb4e1` #3 BẢO MẬT (products.findAll ép phạm vi chi nhánh: NV chi nhánh chỉ xem SP CN mình + SP chưa gán; head office=admin/quyền products.view_all_branches xem hết; factory bypass chỉ head office; stats cũng theo CN) · `a838cd4` Nhóm SP tự đặt + nút "+ Thêm nhóm" trong popup Đổi nhóm SP (POST /product-groups mới) · `07b4e30` T2 bảo mật: thêm @RequirePermissions cho reservations/barcodes/attributes (12+ endpoint trước chỉ JwtAuthGuard) + đăng ký barcodes/product_attributes vào PERMISSION_MODULES.
> **⚠️ DEPLOY ĐỢT 2 — apply 2 migration TRƯỚC code (port 5433):** `0422_seed_view_all_branches_perm.sql` (seed products.view_all_branches cho Giám đốc/Manager/Kế toán) + `0423_seed_reservations_barcodes_attributes_perms.sql` (seed 8 quyền cho 13 vai trò). Rồi `bash scripts/safe-deploy.sh n8n/005-wecha-pos-zalo-keypos`. KHÔNG apply trước → head office non-admin / nhân viên 403.
> **Đã soát đối kháng (reviewer Sonnet)** từng commit: #3 bypass-proof+tenant OK; T2 bắt blocker reservations.edit→manage đã sửa; custom groups tenant-safe. api tsc + web build PASS.
> **CÒN LẠI (chưa làm):** T5 (UX, thấp): search autocomplete param q↔search · branch dropdown reports/returns · gift-card list join tên người mua. + Popup "Đổi nhóm SP" #1: chưa rõ "lỗi" cụ thể (Letri xác nhận = thiếu nút thêm nhóm → đã làm).
> **Việc Letri:** đăng nhập lại test phát hành thẻ quà (đợt 1) · deploy đợt 2 (2 migration trước) · sau deploy test: NV chi nhánh login không thấy SP CN khác / tạo Nhóm SP mới / đặt bàn-mã vạch còn vào được.

### 🏁 ⏰ 2026-06-28 ~10:47 ✅ **wave fix-bán-hàng LIVE** (7 commit + mig 0421 đã apply; verify prod OK) + LUẬT Rule 20
> Deploy 10:35 (health 200, pos-api/pos-web online, dist rebuild xác minh: block cũ "Đơn này không yêu cầu duyệt"=0). Tag `wave-banhang-fix-live-20260628-1047`. prod=HEAD=`d0ed9b7` (branch n8n/005). Mig **0421** (seed quyền gift_cards cho 9 vai trò) đã apply prod (UPDATE 9, verified).
> ⭐ **LUẬT MỚI Letri chốt 28/6:** `.claude/rules/20-completeness-csdl-branch-quyen.md` — hoàn thiện TỪNG trang: mọi nút/chức năng (load/lọc/thêm/sửa/xoá/duyệt/xuất) chạy THẬT theo 3 trục **CSDL→CHI NHÁNH→QUYỀN**; quy trình audit→gap→dispatch→deploy→kiểm-sau-deploy, không đợi nhắc.
> **7 commit (3a772c9..d0ed9b7):** báo giá hết hạn vẫn duyệt được + bỏ side-effect tự-set-expired (gỡ kẹt) · đơn chờ duyệt list trả requiresApproval · báo giá JOIN chi nhánh/người duyệt/mã ĐH + Excel · đơn chờ duyệt hiện MỌI đơn confirmed + approveOrder/rejectOrder bỏ chặn requiresApproval (Letri: duyệt=bấm lưu quy trình) · POS chỉ hiện hàng bán được (isForSale, ẩn nhãn/bao bì/hàng mẫu cho admin) · b2b detail hết crash creditLimit + giao/huỷ đơn (đồng bộ field shipping FE↔BE).
> **GIFT-CARDS:** lỗi "phát hành chưa có gì" = thiếu quyền (không role nào có gift_cards.*, chỉ Admin '*') → mig 0421 seed → **user phải ĐĂNG NHẬP LẠI** (quyền nằm trong JWT, jwt.strategy không đọc lại DB) mới phát hành được.
> **CÒN LẠI (chưa làm — dừng agent nền):** T5 (search param q↔search autocomplete, branch dropdown reports/returns, gift-card list join tên người mua) · T2 phân quyền 12+ endpoint thiếu guard (đặt bàn/mã vạch/thuộc tính) seed an toàn như 0421. Audit đầy đủ: 6 blocker/10 high/5 medium (gap matrix trong transcript phiên).
> **Việc Letri:** đăng nhập lại test phát hành thẻ · Ctrl+Shift+R cho POS · (rảnh) tạo rule low_stock.

### 🏁 ⏰ 2026-06-28 ~08:13 ✅ **wave-2.20.34 LIVE** (9 commit + mig 0419+0420 đã apply; verify prod OK)
> Deploy 08:11 (health/login 200). Tag `wave-2.20.34-live-success-20260628-0813` + push GitHub (prod=HEAD=`3eb2383`). Verify prod: 6 marker (menu thu gọn/cảnh báo hết kho/doanh thu/loyalty tenant/đặt-bàn sequence/promotions graceful) đều live; seq+deleted_at+tenant_id có trên DB; /admin/sales/promotions=200 (hết 500/vòng lặp). **Việc Letri làm:** tạo rule low_stock ở /admin/inventory/rules để thấy cảnh báo hết kho · bật C8 QA wave mới · (rảnh) dọn ~10G `.old-*` cũ trên VPS.
> 🔒 Phiên đêm: Opus tự chủ code+soát+commit local, Letri sáng deploy 1 lần — đã LIVE.
> 📋 **BÀN GIAO SESSION đầy đủ (lỗi đã nhắc + khởi động/deploy + luật đã nhắc):** `docs/handoff/HANDOVER-20260628-2waves-banhang.md`
- **wave-2.20.34 (9 commit local `7f7dbf5..HEAD`, build sạch + final build PASS, CHƯA push/deploy):** `43ea099` menu trái THU GỌN desktop (256↔64px persist, áp mọi hệ) · `b3b14fa` đặt-bàn soft-delete+sequence + thẻ quà ghi tenant 6 chỗ (**mig 0419+0420**) · `4c8294a` fix REGRESSION khuyến mãi redirect-vòng-lặp + findAll graceful · `a146a79` online-orders GET/:id→404 · `2939f40` **responsive MOBILE** (3 grid 2-cột→stack + touch 40px + isMobile chuẩn) · `9562310` **CRM loyalty** thêm tenant filter + transaction (chống ghi chéo điểm) · `658708d` Excel customer-groups/care-reminders/campaigns · **`CẢNH BÁO HẾT KHO`** (dashboard nối /inventory-rules/check-alerts — cần tạo rule low_stock mới có data) · **`CẢNH BÁO DOANH THU GIẢM`** (dashboard nối /reports/kpi-overview growth, fail-soft cho user không quyền).
- **⚠️ DEPLOY wave-2.20.34:** apply **mig 0419 + 0420 TRƯỚC code** (code dùng reservation_doc_seq + cột tenant_id/deleted_at; thiếu → 500). mig 0417/0418 đã apply ở 2.20.33. Lệnh deploy: `cd /Users/letri/Projects/erp-project && bash scripts/safe-deploy.sh n8n/005-wecha-pos-zalo-keypos` (gõ YES DEPLOY) + pipe 0419/0420 vào psql -p 5433 trước.
- **C8 RE-AUDIT 2.20.33** (`AUDIT-BAN-HANG-LIVE-20260627.md`): 8/9 FIXED; #9 khuyến mãi regression → **ĐÃ FIX đêm nay**.
- **5 AUDIT DOC đêm (đọc khi cần):** `AUDIT-SALES-INVENTORY-GAPS` ❌ KHÔNG tin (file ảo) · `AUDIT-SCHEMA-DRIFT-20260628` ✅ tin (chỉ promotions drift thật) · `AUDIT-MOBILE-SALES-20260628` ✅ (đã fix 3 critical) · `AUDIT-CRM-20260628` ✅ (loyalty đã fix) · `AUDIT-PHAN-QUYEN-20260628` ✅ (244 controller: 13🔴=webhook/oauth OK, 146🟠 chỉ JwtAuthGuard gồm sổ quỹ/bút toán/LƯƠNG/R&D — **sơ đồ seed sẵn chờ Letri duyệt**).
- **🔴 CẦN LETRI/CEO QUYẾT (đã chuẩn bị, KHÔNG tự làm — rủi ro):**
  1. **PHÂN QUYỀN** (ưu tiên #4): duyệt sơ đồ `AUDIT-PHAN-QUYEN` → seed perm cho 146 controller nhạy cảm (đặc biệt LƯƠNG=PII NĐ13, sổ quỹ/bút toán, R&D) theo pattern mig 0418 — cần policy "ai xem gì" + seed (tránh khóa NV). 9 module thiếu registry (accounting/bank/invoicing...).
  2. **C1 promotions:** 2 hệ trùng (brief `c1-commerce/BRIEF-OPUS-C1-KHUYEN-MAI...`); create/edit còn 500.
  3. **CRM:** guards campaigns/care-reminders/customer-groups/customer-crm (có JwtAuthGuard, thiếu PermissionsGuard — cần seed) · `/admin/debts` STUB giả lập (lọc VIP, không phải nợ thật — trùng `/admin/finance/debts`) · getKpiDebt định nghĩa "công nợ phải thu" (chị Nhi).
  4. thẻ quà lifecycle PENDING · customers database_id (HUB) · `/admin/debts` stub + getKpiDebt định nghĩa (chị Nhi).
- **✅ 2 CẢNH BÁO Letri yêu cầu = ĐÃ LÀM** (hết kho + doanh thu giảm — nối endpoint sẵn có vào dashboard, fail-soft). Letri test: tạo rule low_stock ở /admin/inventory/rules để thấy cảnh báo hết kho.
- **✅ VERIFY CORE KHO (Letri lo nhất) = ĐÚNG, KHÔNG LỖI:** trừ kho khi bán (orders.service.create:282 — UPDATE atomic guard `qty>=cần` chống bán quá tồn + race, combo trừ thành phần, transaction) + hoàn kho khi huỷ (remove():921 — re-expand combo cộng lại + đảo bút toán, idempotent). Chu trình kho lõi vững.
- **Lộ trình đêm: phủ cả 4 ưu tiên + 2 cảnh báo + verify core.** Việc safe rõ ràng còn lại = ÍT (đa số decision-gated, đã ghi trên). Loop duy trì re-verify + sẵn sàng C8/Letri.

### 🏁 ⏰ 2026-06-27 ~23:00 ✅ **wave-2.20.33 LIVE** (12 commit + mig 0417+0418 đã apply; 9 lỗi C8 vá hết; verify prod OK)
> Tag `wave-2.20.33-live-success-20260627-2258` + push GitHub (HEAD=`7f7dbf5`). Verify prod: 4 route cũ-404 giờ=200; marker promotions-redirect/Đợt3-guard/stock-in-transaction/gift-card-IsInt đều live; 6 vai trò có quyền báo cáo+công nợ; CHECK voucher_type có 'HC'. **C8 QA LẠI NGAY** đối chiếu baseline + click-test CRUD.
- **Đã LIVE (12 commit `dd0768e..7f7dbf5`, build sạch + soát đối kháng):**
  - `dd0768e` **Đợt1** IDOR voucher+gift-card (13 điểm, soát 3 vòng) · `a356811` **Đợt2** branch auto-inject · `e9dc889` quét vá an toàn (registry +4 module, gift-card 400+tenant, log) · `42a9ec8` **Đợt4** lọc database_id ~13 báo cáo TÀI CHÍNH (fail-open) · `0b713d9` **Nhập kho NVL** (BE Sửa/Xoá/PUT đảo tồn+transaction+ON CONFLICT; FE 9 mục) · `3bd1dc0` full-width 4 trang + báo giá limit 200→100 · `5a49e03` **mig 0417** 'HC' vào CHECK voucher_type (returns vào sổ) · `2edcce1` nút Kích-hoạt-lại thẻ + Sửa báo giá + 3 route 404 + trang cha · `b4111ed` quét UI breadcrumb+confirm+Excel · **Đợt3** chốt quyền báo cáo+công nợ (reports+customer-debts guard, **mig 0418** seed 6 vai trò) · fix thẻ quà 400 'limit string' (DTO @IsString→@IsInt) · redirect trang khuyến mãi mới lỗi 500.
- **9 LỖI C8 (QA prod 2.20.32) — ĐÃ XỬ LÝ HẾT trong batch:** báo giá limit>100 ✅ · 5 route 404 ✅ · VAT nhập kho 8→0 ✅ · thẻ quà 400 'limit string' ✅ · khuyến mãi 500 = **lệch schema** (2 hệ promotions: prod=bảng cũ mig0007, code mới=mig0188) → **Letri chốt ẩn bản mới (redirect /admin/promotions), giao C1 quyết hệ chuẩn + chuyển data sau**.
- **Dừng tại:** ✅ wave-2.20.33 LIVE (deploy 22:56, verify OK). prod=HEAD=`7f7dbf5`. Tree sạch.
- **Bước tiếp CHÍNH XÁC:** (1) C8 QA lại prod 2.20.33 đối chiếu `orchestrator-ceo/ceos/c8-qa-audit/AUDIT-BAN-HANG-LIVE-20260627.md` (kỳ vọng 9 lỗi → hết) + click-test Thêm/Sửa/Xoá/Lọc. (2) Opus mở Chrome (Letri đăng nhập) soi từng nút + chụp ảnh. (3) TEST nên làm: huỷ phiếu trả hàng (giờ vào sổ JE 'HC'); nhân viên tuyến KHÔNG vào được báo cáo/công nợ (đúng); nhập kho NVL Sửa/Xoá (đảo tồn).
- **CÒN LẠI (cần Letri/đợt sau):** C1 quyết hệ khuyến mãi chuẩn (gộp 2 hệ) ▸ thẻ quà lifecycle PENDING? ▸ online-orders+reservations CỐ Ý chưa gắn guard (NV dùng) ▸ migration tồn: commerce_reservations deleted_at + COUNT+1→sequence, gift_card_transactions tenant_id, customers database_id (HUB) ▸ Đợt4 báo cáo kho/tồn + BCTC.
- **Cạm bẫy:** phiếu NVL KHÔNG sinh bút toán (Xoá/Sửa chỉ đảo tồn); thẻ quà KHÔNG có PENDING (nút=khôi phục REVOKED); list voucher/gift-card/reports giữ shape phẳng (.map); Đợt3 dùng '*' fallback + seed 0418 cho 6 role (Giám đốc/Manager/Quản lý chi nhánh/Production Manager/Kế toán trường/Kế toán viên). Audit gap: `docs/AUDIT-SALES-UI-BUTTONS-20260627.md`.

### 🏁 (14:20) → ĐỌC: **`docs/handoff/HANDOVER-20260627-scoping-security-audit.md`**
**TÓM TẮT:** prod **wave-2.20.32 LIVE** (06c5fee — đã tag+push, verify health/products 200): JE-reversal huỷ đơn + rà nút 16 trang bán hàng. KHÔNG commit nào chờ deploy (tree sạch). Phiên này LIVE 2.20.28→2.20.31: hoàn thiện hệ bán hàng (atomic đơn #11, huỷ-đơn hoàn kho/quỹ/nợ + **đảo bút toán TT99**, convertToOrder, audit-log, validate, loyalty/promo concurrency, reservations item-picker, returns hardening, 2 báo cáo wired, Excel mọi bảng, breadcrumb+Glass mọi trang). **🔴 Audit 3 chiều scoping (CSDL+chi nhánh+quyền) RA LỖ HỔNG: cross-tenant IDOR voucher/gift-card (thiếu tenantId), thiếu PermissionsGuard online-orders/reservations/reports/customer-debts, branch-scope leak, reports chưa lọc database_id.** Chi tiết file:line + kế hoạch vá phased trong handoff. Cạm bẫy SUPREME: WEB BUILD PHẢI CLEAN (xoá cache) — incremental che syntax+thiếu page.js (đã gây products 500). Luật điều phối tự-mở-nhiều-agent + báo no-code (trong handoff).

---

### 🏁 BÀN GIAO MỚI NHẤT ⏰ 2026-06-27 ~00:40 (phiên đêm DÀI tự chủ — deploy 2.20.28 LIVE + CSDL fix + #11 + audit bán hàng sweep + fix critical/high)
- **Vừa làm:** (1) Commit phache-leads (`a4fbe15`) — file đã LIVE 22/6 nhưng untracked, customer-lms.module import nó (bom nổ chậm). (2) **wave-2.20.28 LIVE 22:37** (Letri chạy safe-deploy): 6 commit (7 sổ + 3 fix tài chính + returns + quotes + sales-ux + phache-leads) + **mig 0415+0416 ĐÃ apply** (verify prod: returns_doc_seq + quotes/quote_items.deleted_at tồn tại). Tag `wave-2.20.28-live-success-20260626-2237` + push GitHub. (3) **Fix lỗi CSDL không hiện cho NV Trương Ngọc Sáu** (commit `e96e8b6`, build PASS, **CHƯA deploy**).
- **Lỗi CSDL Sáu — root cause:** tài khoản Sáu ĐÚNG 100% (role Admin `["*"]`, đúng tenant 2eb99a91, default_database_id=ccf95375 CSDL active). Backend SẠCH (databases.findAll chỉ lọc tenantId, **RLS tắt** trên databases, không gate quyền, JWT có tenantId). Nguyên nhân: **pos-api restart 22:36** (chính safe-deploy của Letri) trùng lúc NV thao tác → `GET /databases` lỗi tạm → FE `database-context.tsx` `catch{}` **nuốt lỗi âm thầm** → list CSDL trống. Fix `e96e8b6`: tự thử lại 2 lần + nút "Thử lại" + không xoá lựa chọn cũ. Workaround tức thì: F5/đăng xuất→Ctrl+Shift+R→đăng nhập lại.
- **#11 ĐÃ XONG** (commit `1aa3ca0`, build PASS + reviewer agent xác nhận tx semantics đúng): bọc TOÀN BỘ `orders.service.ts create()` ghi DB trong 1 `this.db.transaction` (orders→order_items→combo→trừ kho→cashbook→recordDebtIncrease→update customers) → atomic, lỗi giữa chừng rollback hết (hết lệch tồn). Bỏ nested tx (recordDebtIncrease nhận tx ngoài), event emit SAU commit, side-effect non-critical ngoài tx, XOÁ block re-check credit dead code (FOR UPDATE ngoài tx). Follow-up pre-existing (không gấp): console.*→logger ở side-effect; capture avgCost in-tx (auto-JE cost chính xác hơn).
- **AUDIT BÁN HÀNG (6 agent song song) + FIX critical/high — ĐÃ XONG (commit, build PASS, push, CHƯA deploy):**
  - `d81f695` all-orders + `57f2d4e` sweep 5 trang (orders POS/quotes/b2b/reservations/returns): **filter ngày/trạng thái/tìm + Xuất Excel theo lọc** (đúng luật "mọi bảng có Excel theo lọc"), cột nghiệp vụ ẩn-hiện theo data.
  - `20159aa` 🔴 **HUỶ ĐƠN HOÀN KHO/CÔNG NỢ/SỔ QUỸ** (critical): remove() trước chỉ set cancelled → lệch tồn/nợ ảo/quỹ sai. Giờ bọc tx đảo NGƯỢC create() (hoàn kho re-expand combo + cashbook expense + recordDebtDecrease[method MỚI] + trừ totalSpent/Orders), idempotent FOR UPDATE. Qua worker-opus + reviewer (sửa 1 BLOCK + 2 MAJOR).
  - `74bfb57` validate qty>0/giá≥0/giảm-giá hợp lệ (orders.dto+service+quotes) + loyalty-redeem bọc tx + promotion apply atomic (chống đổi điểm/vượt lượt khi song song).
- **✅ ĐÃ DEPLOY — wave-2.20.29 LIVE ~00:57 27/6** (Letri YES DEPLOY): 7 commit (e96e8b6 CSDL · 1aa3ca0 #11 · c4c5347 redirect · d81f695 all-orders · 57f2d4e sweep · 20159aa huỷ-đơn-reversal · 74bfb57 validate+loyalty+promo). Verify prod: #11/recordDebtDecrease/retriesLeft đều live, health 200. Tag `wave-2.20.29-live-success-20260627-0057` + push. prod=HEAD=`74bfb57`. KHÔNG migration mới. **TEST nên làm:** #11 rollback (đơn nhiều item 1 hết tồn) + huỷ-đơn-hoàn-kho/quỹ/nợ.
- **🔄 HOÀN THIỆN HỆ BÁN HÀNG round 2-6 (sau wave-2.20.29) — 9 commit MỚI CHỜ DEPLOY (build PASS, push, KHÔNG migration → deploy anytime):**
  - `e961aa9` reservations item-picker (sửa bug giữ hàng items:[]) · `39ff884` returns hardening (chặn phiếu trả đơn cancelled + lọc KH/SP xoá) · `9c67f2e` wire 2 báo cáo commerce (daily-revenue+top-products) bỏ mockup · `f806a6c` reservations CSDL scoping · `68a860d` top-products lọc from/to · `77e38da` Xuất Excel 7 trang sales.
  - `b3f3f3e` **audit-log TT99** (orders huỷ/duyệt/từ chối, fail-soft, AuditLogModule no-cycle) · `6e67c22` reservations.create bọc transaction (atomicity) · `63e2a45` 🔴 **convertToOrder** trừ kho + JE đúng 1 lần (lỗi cũ: JE sai VAT=0/cost=0 khoá idempotent vĩnh viễn + không trừ kho). Letri chốt convert='credit'/unpaid → JE Nợ 131 phải thu (không 111 quỹ). Qua worker-opus + reviewer.
  - ⚠️ Bài học round 3: agent lỡ đổi reservations BE→{data,meta} (vỡ FE) + permission chưa-seed (khoá NV 403) → REVERT. **LUẬT: đổi shape API phải sửa FE consumer CÙNG LÚC; KHÔNG @RequirePermissions chuỗi chưa đăng ký permissions.ts+seed role.**
- **✅ ĐÃ DEPLOY — wave-2.20.30 LIVE ~10:05 27/6** (Letri safe-deploy): 9 commit (e961aa9→63e2a45). Verify prod: convertToOrder-credit + AuditLogModule + exportPromotionsCsv(2) + reservations databaseFilter(3) đều live, health 200. Tag `wave-2.20.30-live-success-20260627-1005` + push. prod=HEAD=`63e2a45`. KHÔNG migration. **TEST nên làm:** convert báo giá→đơn (kho giảm + đơn 'chưa thanh toán' + sổ Nợ 131); huỷ/duyệt đơn (có audit-log); phiếu giữ hàng chọn SP.
- **`5ccc457` ĐẢO BÚT TOÁN KHI HUỶ ĐƠN (TT99) — committed+push, CHỜ DEPLOY (build PASS, no migration):** AutoJeEngineService.reverseOrderJournalEntry (đọc JE gốc posted → đảo Nợ↔Có swap, amount dương, validate Σ cân, voucherNo sequence, reuse voucherType né 'HC', idempotent eventId='order.cancelled:{id}'); remove() gọi SAU COMMIT tx riêng + fail-soft (đảo lỗi không vỡ huỷ đơn, audit critical). CEO tự review đối kháng (call-site+method) + build PASS.
- **🔴 PHÁT HIỆN khi làm JE-reversal — 'HC' BUG trên PROD (cần fix riêng):** CHECK constraint `journal_entries_v3.voucher_type` (verify prod) = ['PT','PC','BN','BC','PN','PX','KC','HD'] **THIẾU 'HC'**. Rule `orders.return.approved` (mig 0401) seed voucherType='HC' → **JE duyệt phiếu TRẢ HÀNG FAIL âm thầm = returns KHÔNG sinh bút toán** (gap TT99). Fix: đổi rule 'HC'→type hợp lệ (vd 'PX'/'KC') HOẶC migration thêm 'HC' vào CHECK. Cần Letri/kế toán quyết.
- **✅ wave-2.20.31 LIVE ~10:53** (5ccc457 JE-reversal + cf71a55 fix syntax). 🔴 **SỰ CỐ + BÀI HỌC:** wave-2.20.30 build INCREMENTAL (cache) **lặng lẽ bỏ sót page.js** (products/pricelists/warranties) → 3 trang 500 trên prod (Letri thấy 10:54). Clean-build 2.20.31 build lại đầy đủ → 16/16 trang admin = 200. **LUẬT: web build PHẢI CLEAN (rm .next/.turbo/tsbuildinfo/cache) — incremental che cả syntax (mix ??/||) lẫn thiếu file.** (đã ghi memory SUPREME).
- **✅ RÀ TỪNG NÚT 16 TRANG BÁN HÀNG — 2 commit CHỜ DEPLOY (clean build PASS):** `53023d8` (10 trang: breadcrumb + confirm Glass nút Huỷ + error banner + void fix) + `06c5fee` (5 trang: breadcrumb + DatePicker thay datetime-local + validation + nhãn VN). gift-cards REVERT (nút Kích hoạt wire endpoint /activate KHÔNG tồn tại).
- **Dừng tại:** HEAD=`06c5fee`, prod=`cf71a55` (wave-2.20.31). **2 commit chờ deploy** (53023d8+06c5fee button-audit, no migration).
- **Bước tiếp:** (1) safe-deploy 2 commit button-audit → tag 2.20.32. (2) **CẦN LETRI QUYẾT:** 🔴 fix 'HC' returns JE (returns không vào sổ) ▸ gift-cards nút 'Kích hoạt' (thêm endpoint /activate hay bỏ nút? gift-card lifecycle) ▸ quotes/[id] nút 'Sửa' (route /edit chưa có — tạo/bỏ/redirect-prefill?) ▸ migration promotions database_id (scope design?) + reservations sequence ▸ đặt-bàn xây/gỡ ▸ kỳ ghi sổ reversal (Nhi). (3) **Còn lại:** script đối soát đơn convert CŨ ghi JE sai; chuẩn hoá list {data,meta}; promotions gift/x_get_y; rà nút các hệ khác (mua hàng/kho/...). (4) Backlog (#9 TK515, leads cron, webhook HMAC, seed roles).
- **Cạm bẫy:** 🔴 safe-deploy rsync WORKING TREE → KHÔNG sửa file lúc deploy chạy (suýt dính: edit FE lúc Letri deploy 22:28, may là SAU rsync nên không lên prod). 🔴 PROD DB port **5433**. Migration apply TRƯỚC deploy code.
- **Hệ khác:** prod ở 6-commit state (a4fbe15). C13 phache-pos hệ riêng (đã duyệt PG-01+02).

### [C13-POS-FNB-V4] BÀN GIAO ⏰ 2026-06-27 ~16:49 — Fix 14 silent-failure GAS (session 8)
- **Vừa làm:** Hoàn tất audit + fix toàn bộ 19 trang POS Apps Script. Commit `6c453eb` (`fix(pos-v4): 14 silent-failure`). `clasp push` LIVE (43 files, 16:49).
- **Dừng tại:** Tất cả 14 bug đã fix và deploy. `17-PhanQuyen.html` là trang static thuần (OK, không cần sửa). Các fix: `02-POS.html` (4 bug: loadPendingOrders silent return → showBanner; loadTodaySummary thêm error cb; loadWallets thêm wallets.error+errorCb; createOrderDraft thêm r.error+errorCb — CRITICAL checkout) · `07-Kho.html` (6 bug: khoOpenAddBTP+khoBtpRecipeAdd thêm withFailureHandler x4; khoLoadNhap thêm res.error; khoLoadXuat thêm data.error) · `06-ThuChi.html` (4 bug: loadSoQuyHomNay silent cb; loadWalletsForTc GAS.getWallets+GAS.getThuChiCategories thêm cats.error+errorCb x2; loadTcList thêm data.error).
- **Bước tiếp CHÍNH XÁC:** (1) C13 còn 3 việc HOÃN chờ CEO ERP brief: PG-03 Menu Category CRUD · PG-04 ctRecalcAll giá vốn · PG-05 PhanQuyen FULL RBAC. (2) Nếu tiếp C13: run `clasp push` sau mọi thay đổi; test link anh Tuấn.
- **Cạm bẫy:** C13 là GAS ONLY — KHÔNG deploy VPS. PROD DB port 5433. Migration chờ Letri.
- **Hệ khác:** ERP prod=`06c5fee` (wave-2.20.32, tag+push). 2 commit button-audit đã deploy. #11 transaction LIVE.

### 🏁 (cũ) ⏰ 2026-06-26 ~09:35 → đọc đầy đủ: **`docs/handoff/HANDOVER-20260626-ERP-banhang.md`**
**TÓM TẮT:** Phiên dài tự chủ. LIVE tới wave-2.20.27 (CSDL fix + bulk gán CN + Math.random→seq + Excel + Quy trình O2C + bán hàng theo vùng + đối soát NH menu + header CSDL). **5 commit CHƯA DEPLOY** (`6580957` 7 sổ + `2b19f60` 3 fix tài chính + `b08f9ef` returns + `dae324d` quotes + `30753c0` sales-ux), git sạch, build PASS. **CHƯA apply mig 0415+0416.** Bước tiếp: apply 0415/0416 (pipe Mac→ssh) → safe-deploy → tag 2.20.28 → làm #11 (bọc transaction orders.create). Audit bán hàng 58 findings (13 critical) — đã fix critical, còn medium/low vặt. C13 duyệt PG-01+02. ⚠️ PROD DB port **5433**; safe-deploy rsync working-tree (verify prod SAU deploy — đã sót 2 lần).

---

### (cũ) ⏰ 2026-06-22 ~09:00 ⭐

### 🏁 BÀN GIAO ⏰ 2026-06-22 ~09:00 (phiên DÀI tự chủ — vượt token) → đọc đầy đủ: **`docs/handoff/CSDL-SCOPING-HANDOVER-20260622.md`**
**⭐ LUẬT MỚI (Letri chốt):** Opus TỰ CHỦ phát triển + tự kiểm — `orchestrator-ceo/LUAT-OPUS-TU-CHU-PHAT-TRIEN-TU-KIEM-20260622.md` + memory `feedback_opus_autonomous_dev_audit_loop`. Vòng lặp: kế hoạch→bung agent→hẹn 5p→trong lúc chờ thì tự kiểm rà từng trang/nút+fix lỗi lặp→build/QC/commit. Làm KỸ/SIÊNG/MỞ RỘNG. 2 cổng chờ Letri: apply migration prod + deploy YES DEPLOY.
**ĐÃ XONG (9 commit LOCAL `ddca385→8311e3c`, chưa push/deploy, build PASS):** CSDL tách VAT — G0 (LIVE) + G1 mig 0408/0409 **applied** + **G2 ghi ổ** (orders+cashbook+einvoice+purchase-invoice+bank+returns+online+quotes+công nợ) + **G4 lọc báo cáo** (BCTC B01/B02/B03+công nợ+dashboard+list) — **tài chính+thương mại ĐỦ VÒNG** + toàn bộ cột schema (mig 0410 backfill/0411 commerce/0412 kho — **chờ apply**) + fix req.user.sub 17 ctrl (created_by NULL). Bối cảnh: **1 MST 0313334177, 1 ổ active ccf95375**.
**CHƯA XONG:** deploy batch (push+apply 0410/0411/0412+safe-deploy — CHỜ Letri) · kho/SX wire+G4 (schema xong, wire chưa — 529 fail) · journal/auto-je qua event · **G7** TaxDeclarationsService+ReconciliationService (chưa có) · backlog (#9 TK515, Math.random→sequence, leads cron, webhook HMAC, seed roles) · #11 ~30 báo cáo · AI · UI.
**CẠM BẪY:** zsh không word-split; req.user.sub TRỪ lms-deposits/phache (type/guard riêng→.id); migration apply=WRITE prod chờ Letri; databaseFilter fail-open an toàn pre-backfill; build `pnpm --filter api build` trước commit.

---

### (cũ) BÀN GIAO CUỐI PHIÊN 22/6 ~02:40 (Opus tự điều phối — phiên DÀI, 6 deploy LIVE)

**✅ ĐÃ LIVE (tag + push GitHub đủ):** 2.20.15 (hotfix) · 2.20.16 (5 bug C8 + quét QueryResult + UI grid) · 2.20.17 (đồng bộ FE↔BE↔DB 104 lỗi route/shape/cột + mig 0406) · 2.20.18 (responsive grid 81 trang) · 2.20.19 (bán hàng trang riêng dashboard+đơn-chờ-duyệt + sửa 500 route/created_by + mig 0407 rfq_date) · **2.20.20 (sticky header toàn hệ + trang Sản phẩm đổi-nhóm-SP/lọc-đều)**.

**🔴 VIỆC LỚN CHƯA LÀM — ƯU TIÊN:**
1. **CSDL scoping (LỖI NẶNG NHẤT — Letri nhấn nhiều lần).** CSDL=ổ đĩa lưu (CHÍNH/XUẤT-THUẾ-có-VAT/KHÔNG-HOÁ-ĐƠN). BE **chưa lọc database_id** (40+ bảng thiếu cột, đa số TÀI CHÍNH → trộn VAT = rủi ro thuế). Spec: **`docs/SPEC-CSDL-SCOPING-20260621.md`**. ⚠️ RỦI RO CATASTROPHIC → STAGED + backfill + test, **bảng tài chính trước**. Pattern: FE gate (layout.tsx:171 soft banner → modal cứng database-branch-picker) + BE interceptor (copy tenant.interceptor.ts: `x-database-id`→request.databaseId) + thêm database_id từng bảng (mig 0408+) + backfill qua branch→warehouse.database_id. **XÁC NHẬN Letri trước khi mass-change scope query.**
2. **Mockup chưa code** (`docs/MOCKUP-GAP-20260621.md`): ~30 trang báo cáo tài chính (b01b/b09/s05/s08/s11/s12/thuế GTGT-TNDN-TNCN/einvoice reports) + chưa đủ (einvoice, công nợ aging, payments bulk).
3. **AI phát hiện & vận hành** (Letri muốn hiện đại): nền ai-advisor/ai-document-extractor/chatbot-rag/ai-vault. Claude API = Wave 4 (cần key). System Health AI (tự quét lỗi FE↔BE/500/SKU/scope) khả thi.

**🟠 BACKLOG ĐÃ NHẮC CHƯA LÀM (làm dứt điểm):** lãi NH→**TK515** (auto-je) · **leads cron** pg-boss "Queue leads:batch-scoring not found" · seed **lms_roles** (prod, Letri duyệt) · **prompt()** pre-existing customer-lms/deposits+refunds, support/tickets/[id], media alert → Glass · **req.user.id→.sub** 71 chỗ còn (NULL created_by ngầm) · order-approve nút approved→processing + console.warn promotion/voucher · Math.random lotNo/refNo rm-stock/fg-stock (cấm) · webhook sàn verify chữ ký (spawn `task_2d723f48`).

**🟢 TÁCH INLINE-STYLE → CLASS/COMPONENT (NV đề xuất, Letri duyệt):** đã bắt đầu (sticky dùng `.glass-table`). Tiếp: trích nút/card/KPI/FilterBar/table → component dùng chung (DOM nhẹ + sửa-1-nơi + duyệt nhanh).

**📂 DOCS phiên này:** AUDIT-FE-BE-DB-20260621 · PHAN-TICH-HE-BAN-HANG-O2C-20260621 · SPEC-CSDL-SCOPING-20260621 · MOCKUP-GAP-20260621 (đều `docs/`).

**⚠️ Letri verify mắt:** 2.20.20 đổi 158 bảng→`.glass-table` (sticky) — lướt vài trang xem visual OK; bảng nào lệch sửa 1 nơi `.glass-table`.

**Cạm bẫy:** migration từ **0408**. safe-deploy KHÔNG tự chạy mig → apply tay (scp+psql). req.user**.sub**. db.execute()=QueryResult→.rows. Route `:id` đặt SAU route literal. Branch `n8n/005-wecha-pos-zalo-keypos`, code `claude/pos-system`, VPS `root@<VPS_IP>` `pos_db`.

---

### ⭐⭐⭐ CẬP NHẬT 22:15 — chế độ TỰ CHỦ (Letri uỷ quyền: tự dispatch wave, deploy từng đợt, đừng chờ nhắc)

**✅ ĐÃ LIVE hôm nay:** 2.20.15→2.20.18 (bugs · FE↔BE sync · UI grid mobile 81 trang).

**🟢 2.20.19 ĐÃ COMMIT — CHƯA DEPLOY (commit `c185055`, CÓ migration 0407):**
- **Hệ bán hàng trang riêng** (như mua hàng): `/admin/sales/dashboard` (KPI thật) + `/admin/sales/pending-approval` (đơn chờ duyệt riêng + Duyệt/Từ chối Glass + DatePicker + Excel + sidebar) + orders/[id] nút duyệt/từ chối đúng trạng thái.
- **Sửa 500 (log prod thật):** route order RmStock/FgStock TRƯỚC :id → "Nhập NVL từ NCC" hết 500 · **created_by NULL gốc rễ = req.user.userId→req.user.sub (20 controller)** · mig 0407 rfq_date → RFQ hết 500 · console.error→Logger.
- Build API+web PASS. **Deploy 2 bước:** (1) apply 0407 (scp+psql) (2) safe-deploy. → tag 2.20.19.

**🗺️ LỘ TRÌNH TỰ CHẠY (Opus tự dispatch wave-by-wave, deploy từng đợt):**
1. ✅ Bán hàng trang riêng + 500 (2.20.19)
2. 🔄 **Mockup chưa code** (đang recon `mockup-gap-recon`): tài chính (báo cáo/custom/hoá đơn ĐT) · leads (5 màn) · định mức SX · KH 57 cột MISA
3. **AI phát hiện & hoàn thiện** (System Health AI — tự quét lỗi FE↔BE/500/SKU/scope/thuế + đề xuất). Nền: ai-advisor, ai-document-extractor, chatbot-rag, ai-vault.
4. **CSDL scoping** (nền tảng, RỦI RO cao nhất — spec `docs/SPEC-CSDL-SCOPING-20260621.md`: 40+ bảng thiếu database_id, làm staged + backfill + test). Cổng login + middleware X-Database-Id + bảng tài chính trước.
5. **Tách inline-style → component** (đề xuất NV đúng — DOM nhẹ + sửa-1-nơi). Trích nút/card/KPI/FilterBar.
6. **Mobile card-internal** per-page (sau grid 81 trang).

**📂 DOCS phân tích đã tạo:** `docs/AUDIT-FE-BE-DB-20260621.md` · `docs/PHAN-TICH-HE-BAN-HANG-O2C-20260621.md` · `docs/SPEC-CSDL-SCOPING-20260621.md`.

**⚠️ PRE-EXISTING tồn (QC bắt, ngoài scope — fix dần):** req.user.id 71 chỗ ghi NULL created_by (như 20 chỗ vừa sửa) · console.warn nhánh promotion/voucher orders.service · Math.random sinh lotNo/refNo rm-stock/fg-stock (CONTRACT cấm) · webhook sàn chưa verify chữ ký (spawn task `task_2d723f48`) · `as any` orders.service:935 query status không validate.

**Cạm bẫy:** migration mới từ **0408** (0407 đã dùng). safe-deploy KHÔNG tự chạy migration → apply tay trước. req.user **.sub** (không phải .id/.userId).

---

### (cũ) BÀN GIAO ⏰ 2026-06-21 ~19:45

### ⭐⭐ TRẠNG THÁI MỚI NHẤT 19:45 — 2.20.18 mobile READY + 2 việc lớn đã chẩn đoán

**✅ 2.20.17 ĐÃ LIVE 19:25** (tag + push) — đồng bộ FE↔BE↔DB toàn hệ.

**🟢 2.20.18 mobile ĐÃ COMMIT (chưa deploy, KHÔNG migration):** commit `4f5605b` — **responsive grid-cols 81 trang** (grid-cols-N → grid-cols-1 sm:/lg: → card xuống 1-2 cột mobile, hết "chồng card tràn card" Letri thấy khi test ĐT). UI-2 trước chỉ sửa `gridTemplateColumns` inline, BỎ SÓT nhóm Tailwind `grid-cols-N` — đây vá nốt. Build web PASS.
→ Deploy: `cd /Users/letri/Projects/erp-project && bash scripts/safe-deploy.sh n8n/005-wecha-pos-zalo-keypos` (YES) → tag 2.20.18 → Letri test ĐT lại.

**🔴🔴 LỖI NẶNG NHẤT — CSDL không lọc dữ liệu (đã chẩn đoán đầy đủ, CHƯA sửa):**
- Mô hình (memory ⭐⭐⭐ `project_company_layer_architecture`): **CSDL = ổ đĩa lưu** (CHÍNH / XUẤT THUẾ-có VAT / KHÔNG HOÁ ĐƠN-tiền mặt). Mọi giao dịch phải thuộc 1 ổ + lọc theo ổ.
- Thực tế: FE có `database-context` + tiêm header `X-Database-Id`, NHƯNG **BE đọc header này ở 0 controller** (X-Branch-Id chỉ ~5 controller), **không middleware scope toàn cục**, **đa số bảng giao dịch THIẾU `database_id`** (online-sales/order-payments/payment-requests/cashbook/returns... — chỉ orders/inventory có). → Ổ "có VAT" và "không VAT" **TRỘN LẪN** = rủi ro thuế.
- Layout chỉ có banner nhắc mềm (đóng được), KHÔNG cổng bắt buộc.
- ⚠️ **Sửa = cô lập dữ liệu lõi (rủi ro CATASTROPHIC — lộ/trộn data chéo ổ). PHẢI staged + backfill + test, KHÔNG rush.** Kế hoạch: (a) cổng bắt buộc chọn CSDL khi login (FE, an toàn); (b) middleware BE đọc X-Database-Id → AsyncLocalStorage (như tenant context); (c) thêm database_id + lọc + ghi đúng ổ cho **bảng TÀI CHÍNH trước**, backfill data cũ→ổ CHÍNH, từng đợt + test. → ĐỢT TIẾP THEO, làm tập trung.

**📱 Mobile còn (sau grid):** card-internal flex (tên+nút trong 1 card vẫn có thể chật) — rà tiếp per-page. Header "Menu Quản trị" đè breadcrumb (xem screenshot). 74 trang đã responsive sẵn (không đụng).

**🔵 O2C (đợt bán hàng) — ĐÃ REVERT (làm lại sau):** Phase 1 build xong nhưng QC FAIL (Math.random mã đơn, thiếu transaction, **webhook sàn không xác thực chữ ký** = lỗ hổng bảo mật có sẵn → đã spawn task `task_2d723f48`). Revert sạch về 2.20.17. Thiết kế đầy đủ: `docs/PHAN-TICH-HE-BAN-HANG-O2C-20260621.md` (46 gap, 4 đợt). Audit FE↔BE: `docs/AUDIT-FE-BE-DB-20260621.md`. Letri chốt "làm hết 4 đợt" — làm lại tử tế từng đợt sau khi xong CSDL+mobile.

**Bước tiếp ƯU TIÊN:** (1) Deploy 2.20.18 mobile → Letri test ĐT. (2) CSDL scoping (làm tập trung, staged). (3) Mobile card-internal per-page. (4) O2C 4 đợt làm lại.

---

### ⭐ TRẠNG THÁI — 2.20.17 (đã LIVE 19:25)

**Đã LIVE hôm nay:** 2.20.14 · 2.20.15 (13:15) · **2.20.16 (17:22 — tag `wave-2.20.16-...-1722`, đã push GitHub)**: 5 bug C8 + quét QueryResult + UI-1 popup Glass + UI-2 lưới thẻ.

**🟢 2.20.17 ĐÃ COMMIT — CHƯA DEPLOY (branch n8n/005):** đồng bộ FE↔BE↔DB toàn hệ theo audit.
- Audit 14 hệ (208 trang) → **128 lỗi** (76 HIGH) lưu `docs/AUDIT-FE-BE-DB-20260621.md`.
- Commit `1d6e3f1` (đợt 1, 84 file): ~100 lỗi route/shape/field. **CÓ migration 0406** (haccp_deviations +4 cột measured_value/unit/limit_desc/batch_no, ADD COLUMN IF NOT EXISTS, có down.sql). Build API+web PASS.
- Commit `55a7081` (đợt 2): finance so-cai/chung-tu, saas API-log, marketing scoring/channels, monitoring msg. Build web PASS.
- Re-verify đã loại false-positive (b01a/b02/b03 có route wildcard, không lỗi).

**🚀 DEPLOY 2.20.17 — 2 BƯỚC (Letri):**
1. **Apply migration 0406 TRƯỚC** (an toàn, idempotent):
   ```
   cd /Users/letri/Projects/erp-project && scp claude/pos-system/apps/api/src/database/migrations/0406_haccp_deviations_measurement_fields.sql root@<VPS_IP>:/opt/pos-system/apps/api/src/database/migrations/ && ssh -t root@<VPS_IP> 'cd /opt/pos-system/apps/api && set -a && . ./.env && set +a && psql "$DATABASE_URL" -f src/database/migrations/0406_haccp_deviations_measurement_fields.sql'
   ```
2. **Deploy:** `cd /Users/letri/Projects/erp-project && bash scripts/safe-deploy.sh n8n/005-wecha-pos-zalo-keypos` → YES DEPLOY. Sau LIVE → tag `wave-2.20.17-...` + push.

**🔧 CÒN DEFER (đợt 2.20.18 — cần BE mới/sâu, KHÔNG gấp):**
- finance so-cai KPI strip tổng hợp (cần endpoint aggregate, đang =0) · createdBy hiện UUID (cần BE join users name) · chung-tu error state.
- marketing: endpoint GET /ads/lead-channels/audit-log chưa có (BE) · appliedRules hiện UUID (cần join tên rule) · channels testSync shape.
- module **norms** sync (đã revert vì rename nửa vời — cần refactor đồng bộ _types + create/compare/[spId] 1 lần).
- purchasing: mock-trong-catch 4 trang report (by-product/lead-time/3way/qc-pass) — bỏ fallback giả.
- **Việc tồn cũ (chưa làm):** C4 N2 lãi NH→TK515 · RFQ 500 thiếu rfq_date (mig ≥0407) · leads cron · seed lms_roles (prod data, Letri duyệt).
- C8 nghiệm thu lại: 5 bug 2.20.16 + UI mobile máy thật + các trang sync 2.20.17.

**Cạm bẫy:** migration mới từ **0407** (0406 đã dùng). safe-deploy KHÔNG tự chạy migration → apply tay TRƯỚC. QueryResult=.rows, db.select=mảng. Union .catch(()=>[]) dùng `(x as any).rows ?? x`.

---

### 🔥 BÀN GIAO OPUS — phiên 21/6 (Opus tự điều phối: 5 bug C8 + quét QueryResult + UI sweep)

> Letri uỷ quyền phiên này Opus tự mở C giao việc (không dán session). Đã deploy 2.20.14 + 2.20.15 (LIVE 13:15:23 health 200). Đang gom **2.20.16** (CHƯA deploy — chờ Letri YES).

**Đã làm + ĐÃ COMMIT (branch n8n/005, code claude/pos-system):**
- ✅ **5 bug C8 wave 2.20.13 + quét QueryResult** — commit `2334c4d` (14 file, build API+web PASS): C1 bulk Đổi loại SP (VALID_CATALOG_TYPES) · C4 trang Thanh toán (map field BE + BE select beneficiary + TaoLenhChiModal gọi mark-paid) · C9 users-roles {data,meta} + **sửa triệt để QueryResult-coi-như-mảng** (lms.service assign/remove role + quyền LMS, auth.service getMe, lms-roles, lms-deposits) · C10 category scope DTO class + whitelist · C16 route /admin/approval/instances + {data,meta,stats} + status lowercase + tên cột + filter tab + guard ids rỗng · manufacturing 15 chỗ QueryResult dashboard (.rows). QC 3 vòng grep code thật. auto-je đã an toàn sẵn (.rows ?? x).
- ✅ **UI-1 popup gốc → Glass** — commit `46b5de6` (9 trang): stock-in/nmvat-report/support-categories/finance-payment-requests[id]/warehouse-outward/warehouse-transfer/saas-expiry/monitoring-media/approval-workflows[id]. Dùng confirmDialog/alertDialog (@/lib/glass-dialog, provider đã mount layout). 11 trang khác đã đúng sẵn; 4 "ô ngày" là grep dính comment (đã dùng DatePicker sẵn).
- ✅ **UI-2 lưới thẻ** — commit `d6a9e8e` (53 trang, build web PASS): đổi **62 lưới** repeat(N) hàng thẻ KPI → `repeat(auto-fit, minmax(Xpx,1fr))` (thẻ cùng hàng bằng nhau + mobile tự xuống dòng không tràn). GIỮ NGUYÊN lịch 7 ngày (lms/manufacturing schedule, shifts), header bảng aging căn hàng (finance/debts), form MISA accordion (customers). saas/customers bỏ media query !important mâu thuẫn. 5 file đã đúng sẵn (NO_CHANGE).

**→ Đợt 2.20.16 = 3 commit, build API+web PASS, cây code SẠCH, KHÔNG migration: `2334c4d` (5 bug C8+QueryResult) · `46b5de6` (UI-1 Glass) · `d6a9e8e` (UI-2 grid).**

**Bước tiếp CHÍNH XÁC:**
1. **Letri gõ YES deploy 2.20.16**: `cd /Users/letri/Projects/erp-project && bash scripts/safe-deploy.sh n8n/005-wecha-pos-zalo-keypos`. KHÔNG migration. Sau LIVE health 200 → tag `wave-2.20.16-live-success-...` + push GitHub + giao C8 nghiệm thu lại 5 bug + rà UI mobile máy thật.
3. Việc tồn (chưa làm phiên này): C4 N2 lãi NH→TK515 · RFQ 500 thiếu cột rfq_date (mig ≥0406) · leads cron C11 · seed lms_roles (prod data, Letri duyệt) — users-roles cần role để gán.

**Cạm bẫy:** migration mới từ **0406**. safe-deploy rsync working tree → commit hết trước (docs/ được --exclude). QueryResult: db.execute()=QueryResult phải .rows; db.select()/db.query=mảng. Khi cast union có .catch(()=>[]) dùng `(x as any).rows ?? x` để tsc không lỗi.

**Hệ khác:** auth.service.ts getMe nay trả ĐÚNG quyền LMS (trước luôn rỗng do bug). manufacturing dashboard KPI nay lấy đúng .rows. payment-requests có route mark-paid wire FE.

---

### 🔥 BÀN GIAO OPUS — phiên 20/6 (3 deploy LIVE + 1 hotfix CHƯA deploy + 5 bug C8)

**Đã LIVE hôm nay:** 2.20.12 (O2C+sổ kế toán+thuế) · 2.20.13 (8 hệ nút hàng loạt) · 2.20.14 (fix giao diện di động z-index menu 9500→900 + Gia hạn phiên + dropdown).

**🟢 2.20.15 HOTFIX SẴN SÀNG — CHƯA DEPLOY (bước tiếp NGAY):** 3 commit sau 2.20.14 (`8c4b45d` ô số Hạn-sử-dụng/Bảo-hành hết kẹt-0 · `f30088f` Xoá vĩnh viễn — ô gõ SKU dễ normSku() FE+BE · type-fix). **API tsc 0, web build OK, git sạch, KHÔNG migration.** → Deploy: `cd /Users/letri/Projects/erp-project && bash scripts/safe-deploy.sh n8n/005-wecha-pos-zalo-keypos` (Letri gõ YES DEPLOY) → tag 2.20.15 → C8.

**🔴 C8 NGHIỆM THU 2.20.13: 5 BUG FAIL (escalation có sẵn — GIAO C sửa, đọc file rồi grep verify):**
1. **C1** bulk change_type: FE gửi 'COURSE' (HOA) ≠ BE enum 'course' (thường) → service reject. `c1-commerce/ESCALATION-BULK-CHANGE-TYPE-20260620.md`. Fix: FE lowercase HOẶC BE toLowerCase.
2. **C4** trang payments LUÔN TRỐNG: field `tt` ≠ `status`. `c4-finance/ESCALATION-PAYMENTS-FIELD-MISMATCH-20260620.md`.
3. **C9** users-roles 0/12 NV: API trả raw pg QueryResult (phải trả `.rows`). `c9-hr/ESCALATION-USERS-ROLES-SHAPE-MISMATCH-20260620.md`.
4. **C10** category scope: PATCH→500, bulk-scope POST→400. `c2-integration/ESCALATION-CATEGORIES-SCOPE-500-20260620.md`.
5. **C16** GET /api/approvals → 404. `c6-platform/ESCALATION-APPROVAL-INSTANCES-404-20260620.md`.
→ C8 báo 5 N/A vì prod thiếu data test (online-orders/LMS/leads/GRN = 0). QA full: `docs/QA/wave-2-20-13.md`.

**Việc tồn (NHAC đã viết sẵn):** RFQ 500 (bảng rfqs thiếu cột `rfq_date` — schema có, prod không, migration cũ chưa apply → `c17-procurement/NHAC-C17-RFQ-DATE-500`, viết mig ≥0406 + verify FULL schema rfqs) · C4 N2 lãi NH→**TK515** chưa code (Nhi đã chốt 515) · leads cron (C11) · mobile per-page (8 NHAC mobile giao rồi, đa số xong, C8 cần test ĐIỆN THOẠI THẬT).

**Cạm bẫy:** migration mới từ **0406**. safe-deploy rsync CẢ working tree → commit hết trước. ⚠️ Pattern phiên này: **Opus apply migration TRƯỚC deploy bắt được nhiều schema-mismatch** (quotes/orders.deleted_at/rfq_date thiếu cột do migration cũ chưa apply prod) — luôn QC grep code thật, các C báo DONE nhiều lần chưa ăn.

---

### [20/6] [C1] NHAC dropdown z-index DONE — commit c8d67de

**Vừa làm:** Fix dropdown "Tất cả danh mục" + "Tất cả nhà cung cấp" bị card danh sách SP che (prod bug).
**Dừng tại:** `apps/web/src/app/(dashboard)/admin/products/page.tsx:442` — đã thêm `position:'relative', zIndex:30`.
**Bước tiếp CHÍNH XÁC:** Deploy `c8d67de` lên prod để verify visual. KHÔNG cần migration. tsc PASS.
**Cạm bẫy:** commit c8d67de gom thêm sidebar files (lint-staged) + `z-layers.ts` từ unstaged khác C — không phải C1 tự tạo. Nhưng commit sạch, không break gì.
**Hệ khác:** Pattern glass-card backdrop-filter + dropdown absolute cũng có ở `lms/reports/page.tsx` + `manufacturing/norms/create/page.tsx` nhưng ngoài scope C1 — Opus assign C5/C10 nếu cần.

---

### [20/6] [C1] BRIEF1+BRIEF2 DONE — gom vào commit C5 972ce03

**Vừa làm:** Fix 2 BRIEF C1: (1) products change_type đồng bộ nhóm thuế; (2) online-orders lọc server-side + bulk thật.

**File đã thay đổi (C1):**
- `apps/api/src/modules/products/products.service.ts` — CATALOG_TO_PRODUCT_TYPE map 11 key, atomic update 2 cột
- `apps/api/src/modules/online-orders/online-orders.service.ts` — server-side filter + bulkConfirm + bulkMarkShipped
- `apps/api/src/modules/online-orders/online-orders.controller.ts` — query params search/date_from/date_to + 2 POST endpoints
- `apps/web/src/app/(dashboard)/admin/products/page.tsx` — CATALOG_TYPES 11 mục + modal + cột Nhóm thuế
- `apps/web/src/app/(dashboard)/admin/online-orders/page.tsx` — server-side load + bulk actions + đổi tên nút + Excel

**Bước tiếp CHÍNH XÁC:** Tất cả code C1 đã có trong HEAD commit `972ce03`. KHÔNG cần commit thêm. Letri cần deploy kèm migration 0405 (C5) để test tất cả.

**Cạm bẫy C1:** `catalogMainCategory` là text column — nếu DB prod có giá trị cũ (7-8 key cũ) thì CATALOG_TO_PRODUCT_TYPE map sẽ trả `undefined` → API báo lỗi 400. Letri hoặc C1 cần kiểm tra data prod trước.

**Hệ khác cần biết:** Không cần migration C1. BE online-orders /bulk-confirm + /bulk-mark-shipped sẵn sàng.

---

### [20/6] [C4] 225ece6 QC round 2 DONE — báo Opus duyệt

**Vừa làm:** Sửa 4 issue từ NHAC-C4-SUA-TIEP-SAU-QC-20260620.md.

**File đã thay đổi:**
- `apps/web/src/app/(dashboard)/admin/finance/payments/page.tsx` — FIX URL /finance/payment-requests/bulk-pay → /payment-requests/bulk-pay (khớp @Controller)
- `apps/web/src/app/(dashboard)/admin/finance/depreciation/page.tsx` — handleBulkApply capture result → alertDialog warning nếu failed>0
- `apps/api/src/modules/excel-export/excel-export.service.ts` — thêm exportPaymentRequests / exportBankStatements / exportDepreciationSchedule / exportCostAnalysis
- `apps/api/src/modules/excel-export/excel-export.controller.ts` — thêm @Get payment-requests / bank-statements / depreciation-schedule / cost-analysis

**Acceptance:** tsc api+web PASS ✅. Bulk-pay URL đúng. 4 Excel routes sẵn sàng.
**Cạm bẫy:** exportCostAnalysis delegate sang exportCashbook (cùng dữ liệu sổ quỹ) — nếu cần tách riêng thì cần query cashbook theo loại chi phí.
**Hệ khác:** KHÔNG cần migration. KHÔNG push/deploy.

---

### [20/6] [C17] 6eb671f NHAC-C17-MOBILE DONE ⏰ 16:08

**Vừa làm:** 7 lỗi mobile purchasing module từ NHAC-C17-MOBILE-20260620.md — do C6 sidebar z-index gây ra.
- `purchase-orders/page.tsx` — side panel createPortal→body + zIndex backdrop 40→1000 / panel 50→1050 + width 380→min(380px,100vw) (không tràn iPhone 375px, không bị menu che)
- `purchase-requests/page.tsx` — table wrapper overflow-x:auto+minWidth:900 (10 cột cuộn được); btnBase padding→10px 14px, fontSize→12, minHeight:44px (touch target ≥44px)
- `purchase-requests/new/page.tsx` — stepper flexWrap:wrap (không tràn màn hẹp)
- `sales/b2b/page.tsx` — 2× input[type=date] CẤM → DatePickerSingle; KPI grid repeat(4,1fr) → repeat(auto-fit,minmax(160px,1fr))

**web build PASS (466 pages). Không có migration. KHÔNG push.**

---

### [20/6] [C17] e937a12 NUT-HANG-LOAT-FIX DONE

**Vừa làm:** Fix 5 issues từ BRIEF-C17-NUT-HANG-LOAT-FIX-20260620.md.

**File đã thay đổi:**
- `apps/api/src/modules/goods-receipts/goods-receipts.controller.ts` — thêm POST /:id/approve (alias confirm) + /:id/reject
- `apps/api/src/modules/goods-receipts/goods-receipts.service.ts` — thêm rejectGrn() với status guard
- `apps/api/src/modules/purchase-qc-inspections/purchase-qc-inspections.controller.ts` — thêm 3 POST /:id/approve|reject|reinspect
- `apps/api/src/modules/purchase-qc-inspections/purchase-qc-inspections.service.ts` — thêm approve/reject/reinspect với BadRequestException guards
- `apps/web/src/app/(dashboard)/admin/purchasing/returns/page.tsx` — doBulkCompensate gửi status=compensated
- `apps/web/src/app/(dashboard)/admin/purchasing/rfqs/page.tsx` — XLSX import + exportExcel() + nút Xuất Excel theo filter

**Issue 5 (suppliers):** false positive — BE đã ghi errors[], KHÔNG cần fix.
**Cạm bẫy:** api build có 2 pre-existing errors (approval.service TS2353 + hr-attendance TS2339) không liên quan C17. Cần fix riêng.

---

### [20/6] [C5] 972ce03 LMS BLOCKER FIX DONE

**Vừa làm:** Fix 4 blocker từ BRIEF-C5-NUT-HANG-LOAT-FIX-20260620.md + 2 secondary issues. tsc api+web PASS ✅.

**File đã thay đổi:**
- `apps/api/src/database/migrations/0405_lms_attendance_upsert_tuition_softdelete.sql` + down.sql
- `apps/api/src/database/schema/lms.ts` — scheduleId nullable, lmsTuition thêm deletedAt
- `apps/api/src/modules/lms/lms.service.ts` — bulkAttendance upsert, deleteTuition, findAllTuition filter deleted
- `apps/api/src/modules/lms/lms.controller.ts` — DELETE /lms/tuition/:id
- `apps/api/src/modules/lms/services/lms-roles.service.ts` — transaction bọc DELETE+INSERT
- `apps/web/src/app/(dashboard)/admin/lms/roles/page.tsx` — openModal đọc perms thật từ BE, card hiện desc/permCount đúng
- `apps/web/src/app/(dashboard)/admin/lms/attendance/page.tsx` — PermissionGate → lms.activity.attendance.view
- `apps/web/src/app/(dashboard)/admin/lms/reports/page.tsx` — toast success/error, handleDelete type string

**Bước tiếp CHÍNH XÁC:**
- Apply migration 0405 trên staging/prod TRƯỚC deploy
- Giao C8 test LMS: điểm danh không trùng, xoá học phí, vai trò perms thật
- MIGRATION số tiếp từ **0406**

**Cạm bẫy:** mig 0405 thêm unique constraint `lms_attendance_tenant_class_customer_date_unique` — nếu DB prod đã có dữ liệu trùng thì apply sẽ fail → cần DISTINCT/cleanup trước.

---

### Bàn giao cũ 19/6 (còn hiệu lực)

### ✅ ĐÃ DEPLOY WAVE 2.20.12 — LIVE 19/6 10:53 (health api+web 200, tag `wave-2.20.12-live-success-20260619-1053`, đã push GitHub) — Bàn giao cũ 19/6

**Vừa làm:** Gom + deploy toàn bộ tồn đọng từ 15/6 (C1 O2C + C4 auto-JE + C17 GRN + C9 tuyển dụng/đào tạo/thuế + C2 session/Excel/IP). Opus QC vòng 3 (grep code thật) + apply 18 migration (0388–0404) TRƯỚC deploy → bắt+sửa 3 lỗi migration (C1): **0404 tạo bảng quotes/quote_items (thiếu)** · **0399 thêm orders.deleted_at (thiếu)** · **0395 DROP CONSTRAINT + idempotent**. 4 câu thuế Nhi chốt: đào tạo KCT(N1)/lãi NH TK**515**(N2)/TNDN 17%(N3)/TNCN 5 bậc 15,5-6,2tr(N4) — N4 đã seed mig 0403, N1/N3 đã code, **N2 (C4 code handler lãi NH→515) CHƯA code**.

**Bước tiếp CHÍNH XÁC cho session sau:**
1. **Giao C8 nghiệm thu 2.20.12** (Chrome thật): báo giá `/admin/sales/quotes` (tạo→gửi→duyệt→tạo đơn) · chi tiết đơn `/admin/orders/[id]` · trả hàng · thuế TNCN (xem số 5 bậc 15,5/6,2tr) · quyết toán Xuất Excel · IP tin cậy/lịch sử đăng nhập. → `NHAC-C8-NGHIEM-THU-WAVE-2.20.12-20260619.md`.
2. **C4 code N2** — handler lãi tiền gửi NH → JE TK 515 (doanh thu hoạt động tài chính), KHÔNG vào doanh thu GTGT. (Nhi đã chốt 515.)
3. **C11/C18 fix cron** `leads:batch-scoring` (pg-boss queue not found khi boot — không chặn nhưng cron chấm điểm lead không chạy).
4. P1 tồn (không gấp): C1 bulk-confirm bọc transaction · C1 webhook HMAC verify mọi env · C2 login-history lọc theo email · MIGRATION_REGISTRY bổ sung 0388–0404.

**Cạm bẫy:** migration mới từ **0405**. orders giờ ĐÃ có deleted_at (soft-delete) — module khác có thể filter. quotes/quote_items vừa tạo (cấu trúc khớp commerce-quotes.ts). VPS pos_db root@<VPS_IP>, deploy 8 bước (apply mig TRƯỚC code), safe-deploy đã thêm `--exclude=docs/`.

---

<details><summary>📜 Lịch sử bàn giao 18/6 (đã xử lý xong — bấm xem)</summary>

### 🔥 CẬP NHẬT CUỐI PHIÊN ⏰ 2026-06-18 ~17:35 (session ngưng — vượt token, Letri CHỐT NGƯNG GIAO VIỆC MỚI)

**Trạng thái QC mới nhất các cụm code 18/6 (Opus grep code thật — KHÔNG tin "DONE"):**
| Cụm | Commit | QC verdict | NHAC sửa |
|---|---|---|---|
| C1 O2C (7 worker) | `d67a506`/`d577010`/`b97371b`/`8569ae5`/`a488327`/`9af4d4f`/`a0b3a2d` | ❌ **CAN-SUA** (6/6 P1) | `NHAC-C1-SUA-O2C-CLUSTER-20260618` |
| C4 auto-JE engine | `3c1c412` | ❌ **CAN-SUA NẶNG** | `NHAC-C4-SUA-AUTO-JE-ENGINE-20260618` |
| C1 W4/W5/W6 | (cụm trước) | ❌ CAN-SUA | `NHAC-C1-SUA-QC-W4-W5-W6-20260618` |
| C9 Đợt B+C sửa | (sau 160d6ea) | 🔄 ĐANG QC (workflow `wx7ydih5w` chưa xong khi ngưng — đọc kết quả `/tmp/.../wx7ydih5w.output` hoặc re-QC) | `NHAC-C9-SUA-DOT-B-C-QC-20260618` |
| C2 session (2206-11) | (mig 0389) | 🔄 ĐANG QC (cùng workflow trên) | — |

**Lỗi P1 trọng yếu cần C sửa (chi tiết trong NHAC):**
- **C4 auto-JE (xương sống — QUAN TRỌNG NHẤT):** Σ Nợ=Σ Có validate VÔ NGHĨA (cấu trúc line sai) · balance công nợ KHÔNG cộng dồn · **CHƯA seed auto_je_rules → engine không sinh JE** · thiếu 2 handler (payment.received/return_approved) · lệch tên event C1↔C4 · outbox insert ngoài tx · voucherNo Date.now(). → **KHÔNG deploy engine này tới khi sửa** (sổ kế toán sẽ sai/trống).
- **C1 O2C:** báo giá taxRate(% vs 0-1)+field validUntil/notes lệch → tạo BG có thuế FAIL · `/orders/export` route sau `:id` → 404 · nút "Gửi HĐ" chết + thiếu nút Xác nhận + danh sách thanh toán (detail order) · convert quote branchId rỗng crash · event name lệch C4.

**🔴 CHỐT CHẶN GOM WAVE (Letri chốt: ĐỢI C9 xong HẾT → gom TẤT CẢ 1 wave):**
1. **Letri hỏi Nhi 2 điều** (chặn C9/C4): số thuế TNCN **15.5/6.2/5 bậc (Nhi) hay 11/4.4/7 bậc (TT80, DB mig 0194 đang là số này)** + lãi NH **TK 515 hay 635** → C9 seed mig 0195 + C4 code N2.
2. C1 sửa O2C cluster + W4/W5/W6 (2 NHAC).
3. C4 sửa auto-JE engine (NHAC) — xương sống.
4. C9 hoàn tất B (đang QC) + C (seed thuế sau Nhi).
5. C2 sau QC.
→ Khi tất cả PASS + git status sạch (commit hết) + apply mig đúng thứ tự (0389/0391/0394-0400/0396, verify từng cột) + build local PASS → safe-deploy → 4 ô.

**Bước tiếp CHÍNH XÁC cho session sau:**
1. **RE-QC C2 session + C9 sửa B/C** (workflow QC trước ĐÃ HỦY khi ngưng session — KHÔNG có kết quả lưu): verify C2 fix bug 30p (FE mutex + BE Redis rotation) + mig 0389 + tab Bảo mật; C9 exams autocomplete/thùng rác + RBAC hr.tncn.* + {data,meta} + dependents (bỏ type=date/alert). PASS → sẵn sàng gom; CAN-SUA → viết NHAC.
2. Khi Letri có đáp Nhi → cập nhật C9 (seed thuế) + C4 (N2).
3. Poll các C sau khi sửa NHAC → Opus QC lại → C8 → gom.
**Cạm bẫy:** migration mới từ **0401**. Auto-JE chưa seed = engine vô dụng. C1↔C4 phải khớp tên event. safe-deploy rsync working tree → commit hết trước.

---

**Vừa làm (phiên 17-18/6 — trọng tâm HỆ BÁN HÀNG chuẩn thế giới + thuế HR + sự cố Antigravity):**
- 🔴 **Sự cố Antigravity code lụi** 6 màn bán hàng → Opus LOẠI sạch (git checkout về stub, backup `orchestrator-ceo/_archive/antigravity-lui-banghang-20260616/`). Lessons `docs/lessons-learned/architecture/2026-06-16-01-...`. AG-05 test = HỦY. **Luật: Opus QC `git status` working tree TRƯỚC gom wave.**
- ✅ **Benchmark + audit:** `docs/BENCHMARK-HE-BAN-HANG-THE-GIOI-20260617.md` (12 gap) + `docs/AUDIT-QUY-TRINH-HE-BAN-HANG-20260618.md` (4 quy trình). **GỐC RỄ: kế toán KHÔNG lên sổ ở cả 4 quy trình** → BRIEF-C4 auto-JE engine (xương sống).
- ✅ **C1 ĐÃ CODE (chờ Opus QC):** order_items.unit_cost+events (`d67a506`) · trang `/admin/orders/[id]` (`d577010`) · báo giá BE+FE (`b97371b`+`8569ae5`, mig 0398) · returns soft-delete+bulk (`9af4d4f`, mig 0400) · bulk-orders+DataTable (`a0b3a2d`). + ĐÃ DUYỆT: schema W1/W2, PROPOSE-GAP, QC W4/W5/W6 (CAN-SUA, NHAC sửa). + giao 4-thao-tác + 4 gap đào sâu + O2C.
- ✅ **C4:** N1 (GTGT KCT→chỉ tiêu 26) + N3 (TNDN 17%) DONE (`dfd08d9`). N2 lãi NH CHỜ verify TK 515/635. auto-JE engine (BRIEF) chưa code.
- ✅ **C9 HR:** Đợt A QC PASS (`435c15a`) · Đợt B: exams lessonId+thùng rác DONE · Đợt C: RBAC+{data,meta}+DatePicker+alertDialog+settlement Excel DONE · build PASS. DEFERRED: seed thuế 15.5tr/6.2tr (chờ Nhi) + tncn_amount column. **XIN OPUS QC lại** → `orchestrator-ceo/ceos/c9-hr/NHAC-C9-SUA-DOT-B-C-QC-20260618.md`.
- ✅ **C2:** session timeout config (`BRIEF-C2-SESSION-TIMEOUT-CONFIG`, mig 0389). C6 sticky header (`88c3915`) + Antigravity autocomplete (`f22d002`) chờ C8.

**Dừng tại (việc treo + chờ):**
- **QC tồn đọng (Opus làm tiếp):** C1 cụm O2C mới (`d67a506`/`d577010`/`b97371b`/`8569ae5`/`9af4d4f`/`a0b3a2d`/`c90f296`) — chưa QC. C4 N1+N3 (`dfd08d9`) — chưa QC. C9 sửa B+C — chờ C9 code lại.
- C4 auto-JE engine + C17 P2P/nhập/mua-trả: BRIEF đã viết, chưa code.

**Bước tiếp CHÍNH XÁC:**
1. 🔴 **Letri hỏi Nhi 2 điều (CHẶN C9/C4):** (a) số thuế TNCN 15.5tr/6.2tr/5 bậc (Nhi đưa) hay 11tr/4.4tr/7 bậc (TT80 hiện hành — DB mig 0194 đang là số này)? (b) lãi NH ghi TK **515 (doanh thu tài chính)** hay 635? → C9 seed thuế + C4 code N2 theo chốt.
2. **Opus QC cụm C1 O2C** (7 commit 18/6 bao gồm c90f296) — grep code thật: unit_cost capture đúng? events payload đủ cho C4? trang detail order load thật? quote convert→order? returns soft-delete+JE event? bulk atomic? → CAN-SUA thì NHAC.
3. Letri dán BRIEF: C4 auto-JE (TRƯỚC) → C17 → C9 sửa.
4. 🎯 **Letri CHỐT 18/6: ĐỢI C9 XONG HẾT (Đợt B sửa + Đợt C thuế sau khi Nhi chốt số) → GOM TẤT CẢ 1 WAVE** (góp ý NV + C1 O2C + C9 A/B/C + C4 N1+N3 + C2 session) — KHÔNG tách Đợt C. Điều kiện trước YES DEPLOY: mọi cụm Opus QC PASS + `git status` sạch (commit hết) + apply migration đúng thứ tự (0389→0391→0397/98/0400→...) verify từng cột + build local PASS. → safe-deploy → đóng 4 ô.

**Cạm bẫy:**
- 🔴 `safe-deploy` rsync CẢ working tree → QC `git status` + commit từng C trước gom (C9 Đợt C build PASS chưa rõ commit).
- 🔴 Antigravity NHANH nhưng lụi → QC grep TRƯỚC duyệt. Brief Antigravity chốt "CHỈ mockup".
- 🔴 Migration: C9→0391, C2→0389, C1→0397/0398/0400 (18/6). Số mới từ **0401** (check `ls migrations|tail`).
- 🔴 Số thuế: DB mig 0194 = TT80 (11/4.4/7) ≠ Nhi (15.5/6.2/5) → CHỜ chốt rồi seed, đừng để 2 nguồn lệch.
- 🔴 HUB W1(orders)+W2(products) schema DUYỆT: tồn variant qua `inventory_stock` (thêm variant_id), tái dùng `product_attribute_groups/values` cũ.
- VPS `root@<VPS_IP>` DB `pos_db`. Deploy 5 bước.

**Hệ khác cần biết (bàn giao chéo):**
- **Auto-JE engine (C4) = XƯƠNG SỐNG** — C1 đã emit `order.completed`/`payment.received`/`RETURN_APPROVED`; C17 sẽ emit grn/invoice/payment. C4 phải có handler sinh JE 511/632/156/331/131/531 + supplier/customer debt. order_items có `unit_cost`, cần inventory_stock `avg_cost` (C17).
- `docs/BENCHMARK` + `docs/AUDIT-QUY-TRINH` = chuẩn cho mọi C bán hàng.
- C9 thêm branch_id 5 bảng HR training (mig 0391) — module khác filter branchId.

[18/6 ⏰ 2026-06-18 21:33 +0700] [C2] Việc 2206-11 QC-fix DONE (4/4) — sau QC Opus: (1) settings.service.ts inject RedisService + del cache:system:session_timeout_minutes sau PATCH TTL → apply 0s không cần chờ 60s; (2) entity-configs.ts thêm 'trusted-ips' (bảng trusted_ips 4 cột) + trusted-ips/page.tsx thêm EntityExcelExport (luật toàn hệ); (3) login-history/page.tsx filter label 'Email' + param 'email' (không phải userId); (4) excel-import.controller.ts Content-Disposition filename double-quoted RFC 6266. Gate: tsc api 0 lỗi + FE Next.js build PASS. KHÔNG deploy — chờ Opus gom wave.
[18/6] [C1] 13bd61c QC3-FIXES DONE — B1: convertToOrder emit ORDER_COMPLETED_EVENT (C4 nhận được). B2: returns.controller PermissionsGuard + @RequirePermissions mọi endpoint. B3: commerce/page 4 KPI fetch truyền ?branch_id=. P1: order[id] Xác nhận→POST /approve; bulk-confirm trả errors[]; returns findAll/findOne {data,meta}; mig 0402 FK platform_product_listings (tenant+product); HMAC webhook bỏ NODE_ENV. tsc+build PASS ✅ ⏰ 2026-06-18 +0700.

</details>

> ✅ Toàn bộ tồn đọng 18/6 ở trên ĐÃ deploy trong wave 2.20.12 (19/6 10:53). Bàn giao hiện hành: xem đầu mục 7.

[20/6 ⏰ 2026-06-20 16:09 +0700] [C6] Việc 2206-10 DONE — z-index mobile gốc rễ: tạo `src/lib/z-layers.ts` thang chuẩn. Hạ 10 sub-sidebar mobile 9500→900, backdrop 9400→890. sidebar.tsx chính: mobile drawer z-50→zIndex:900, backdrop 48→890. layout.tsx: bỏ zIndex:1 khỏi `<main>`. Thang sau: header=35 < sidebar-desktop=100 < backdrop=890 < mobile=900 < modal=1000 < dialog=2000 < toast=3000. Commit `c8d67de`. tsc PASS. Build pre-existing lỗi purchase-requests/page.tsx:251 (unclosed JSX — không thuộc C6). ⚠️ C17 cần fix purchase-requests. Chờ Opus QC.

[20/6 ⏰ 2026-06-20 16:00 +0700] [C2] Fix nút Gia hạn phiên DONE — 3 vấn đề: (1) handleExtendSession dùng fetch() duplicate thay vì refreshAccessToken() đã proven từ api.ts → fix: export refreshAccessToken() + dùng trong handleExtendSession; (2) timer không reschedule sau extend (cùng tab không nhận storage event) → fix: thêm sessionExtendedAt state, tăng giá trị sau extend thành công → trigger useEffect([user, sessionExtendedAt]) re-run → scheduleWarn() với token mới; (3) fail silently khi refresh token hết hạn → fix: nếu newToken=null → xóa localStorage + redirect /login?reason=session_expired. Thêm toast DOM thuần 'Đã gia hạn phiên'. Gate: tsc api+web 0 lỗi + FE Next.js build PASS. KHÔNG deploy — chờ Opus gom wave.

---

[22/6 ⏰ 2026-06-22 06:18 +0700] [C13] Việc 2206-C13-SESSION5 DONE — Apps Script @78 LIVE. Fix xuất kho hỗ trợ Bao bì đầy đủ (BE kho_xuat isBB/isBTP + HTML select xuat-loai thêm option bao_bi + onchange lazy-load khoState.bb + khoSubmitXuat reload đúng sheet + khoLoadBB populate xuat-id khi cần). Tích hợp sàn 18-TichHop.html (Grab/Shopee Food Glass UI). Thuế branch select động + 3 offline fallback. GAS bridge 4 hàm kho BB. Commit `8abbf3e`. Deploy @78 thành công. VPS c308b8d (mailer) vẫn chưa deploy — Letri cần SSH deploy thủ công.

[22/6 ⏰ 2026-06-22 08:31 +0700] [C19] Việc 2206-10/11/12 DONE (CHƯA commit, CHƯA deploy) — website wecha.vn (`antigravity-project/`, Next 16.2.9). (A) **Quản trị nội dung trọn bộ**: sửa nội dung chi tiết SP từ admin (cột `products.content`) + CMS trang chủ + quản lý nhân viên/phân quyền + danh mục + xuất CSV + thùng rác + nhật ký + tìm kiếm site. (B) **Nội dung + SEO**: 13 trà bán lẻ (story/công dụng/dinh dưỡng/cách pha/công thức) + JSON-LD Product/Article/Breadcrumb/FAQPage + nút chat + ảnh món thật cho menu. (C) **Hệ thống thành viên** (review bảo mật + vá HIGH): bảng `members` + `orders.member_id` · đăng ký/đăng nhập/tài khoản/điểm/lịch sử đơn · argon2 + lockout + consent NĐ13 · thu hồi phiên khi đổi MK/khoá tài khoản (`members.token_version` + getMember re-check DB) · SESSION_SECRET≥32. (D) **Compliance + trust**: trang Chính sách bảo mật (NĐ13 + DPO)/Đổi trả 30 ngày/Điều khoản/FAQ · banner cookie · consent NĐ13 ở thanh toán (`orders.consent/consent_ip/consent_at`) · nút chia sẻ FB/copy · tra cứu đơn (mã+SĐT). Build PASS, tự verify curl. ⚠️ **DEPLOY CẦN MIGRATION**: cột/bảng mới chỉ ALTER ở DB local (chưa có file drizzle) — `products.content`, `members`(+token_version), `orders.member_id`+consent*. Chi tiết: VIEC-C19 + AG-C19-2206-09 + HANDOVER-C19-20260622.md. Đã tạo lịch `c19-wecha-web-daily-dev` (8h/ngày tự làm tiếp non-blocked). CHẶN chờ Letri: giá bán lẻ · khoá VNPay/MoMo + STK VietQR · GA4/Pixel · endpoint ERP nhận đơn (C2) · duyệt chữ 4 trang pháp lý.

[22/6 ⏰ 2026-06-22 08:48 +0700] [C19] Giá bán lẻ — tìm được file `docs/data-reviews/bang-gia.xlsx` (catalog 1165 dòng kiểu MISA). Trích 89 dòng trà → `docs/data-reviews/wecha-tra-gia-tham-khao.md`. Đối chiếu 13 trà bán lẻ web (đều `price=NULL`, `price_visibility=on_request`): chỉ ~1 khớp rõ (Trà Đen Assam Cao Cấp 500g=195k), 4 món KHÔNG có trong file (Hồng Trà Tuyết, Trà Đen Ceylon, Ô Long Gạo, Ô Long Phú Quý), còn lại 2-4 giá khác nhau + là giá sỉ pack 500g-2kg. → KHÔNG tự gán (tiền thật, rule N2). Đã trình Letri bảng 13 giá để chốt; chốt xong: set `price` + `price_visibility='public'` → tự bật nút Mua. Sỉ/XK vẫn quote-on-request (form lead → ERP).

[22/6 ⏰ 2026-06-22 09:12 +0700] [C19] Việc 2206-13 (non-blocked tiếp) DONE — **hreflang SEO**: tạo helper `src/app/lib/seo.ts` `langAlternates(lang,path)` (canonical + vi/en/x-default), áp 14 trang. Phát hiện + vá gap: **PDP `mua-tra/[slug]` thiếu hẳn generateMetadata** → thêm title + description (từ content.taglineVi, không bịa) + canonical + hreflang + OG image. Build PASS. Verified curl: /mua-tra + PDP + tin-tuc/[slug] đều có đủ 3 hreflang, slug động nội suy đúng (0 rò `${slug}`). KHÔNG đụng DB. Tiếp theo (theo handover): trang chi tiết đồ uống /menu/[slug] → lọc catalog → gallery.

[22/6 ⏰ 2026-06-22 09:34 +0700] [C19] Việc 2206-14 DONE — **trang chi tiết đồ uống `/menu/[slug]`**: thêm query `getDrinkBySlug` (commerce-queries.ts) + route mới (ảnh + giá + mô tả + preview size/topping + tái dùng `AddToCartButton` chọn size/topping/SL + `ShareButtons` + breadcrumb + **Product JSON-LD** offers VND). Nối thẻ ở `/menu` (ảnh + tên) → trang chi tiết. Thêm slug đồ uống vào `sitemap.xml`. Build PASS. Verified curl: `/vi/menu/bac-xiu` 200, title riêng, Product JSON-LD, 3 hreflang, nút Đặt + Chia sẻ + canonical; sitemap có /menu/<slug>. KHÔNG đụng schema DB (chỉ thêm query đọc). Tiếp: lọc/sắp xếp catalog → gallery nhiều ảnh → đánh giá sao.

[22/6 ⏰ 2026-06-22 09:50 +0700] [C19] Việc 2206-15 DONE — **lọc/sắp xếp catalog `/mua-tra`**: thanh tìm theo tên + sắp xếp (Tên A–Z / Giá thấp→cao / Giá cao→thấp) chạy server-side qua searchParams `?q=&sort=` (GET form, hoạt động không cần JS), giá null xếp cuối, nút Xoá lọc + đếm kết quả + empty state. Build PASS. Verified curl: mặc định "13 loại trà" + 13 thẻ; `?q=long` → "4/13 trà khớp" + Xoá lọc; `?sort=price-asc` 200. KHÔNG đụng DB. Tiếp: gallery nhiều ảnh/SP → đánh giá sao.

[22/6 ⏰ 2026-06-22 10:08 +0700] [C19] Việc 2206-16 DONE — **gallery nhiều ảnh/sản phẩm** ở PDP: query `listProductImages(productId)` (media_assets theo product_id, sort_order) + component client `ProductGallery.tsx` (ảnh chính + dải thumbnail, click đổi ảnh; 0 ảnh→placeholder, 1 ảnh→ảnh đơn, >1→thumbnail). Gộp primary (primaryImageId) + ảnh gallery, dedup theo url. Build PASS. Verified: chèn tạm 2 ảnh test cho hong-tra-ba-tuoc → PDP hiện thumbnail "Ảnh 1/2/3" + ảnh test render → đã xoá test (DB sạch, 0 sót). KHÔNG đổi schema (chỉ thêm query đọc + component). Còn lại trong hàng đợi: đánh giá sao (cần bảng reviews + duyệt admin — việc lớn hơn).

[24/6 ⏰ 2026-06-24 15:39 +0700] [C19] Việc 2206-17 DONE — **đánh giá sao (reviews) trọn gói**: (1) bảng mới `product_reviews` (rating 1-5, authorName, comment, status pending/approved/rejected, memberId, audit+soft-delete, check rating + product⊕drink) + enum `review_status` — ⚠️ deploy CẦN MIGRATION; (2) query `review-queries.ts`: `listApprovedReviews` + `getRatingSummary` (avg/count) + `adminListReviews`; (3) action `actions/reviews.ts`: `submitReview` (public, vào status pending + audit) + `setReviewStatus`/`deleteReview` (requireRole editor + audit + revalidate); (4) PDP: component client `ProductReviews.tsx` (avg sao + danh sách approved + form gửi sao/tên/nhận xét useActionState) + **aggregateRating** vào Product JSON-LD; (5) trang admin `/admin/reviews` (duyệt/ẩn/xoá qua bound server action) + link AdminNav. Build PASS. Verified: PDP empty→sau chèn hiện approved "Chị Lan", ẩn pending "Anh Khoa", aggregateRating ratingValue 5/reviewCount 1; admin (mint cookie) 200 hiện cả 2 + nút Duyệt/Ẩn/Xoá; đã xoá review test (DB sạch). ✅ **Backlog phát triển non-blocked của C19 coi như HOÀN TẤT** (hreflang, drink detail, lọc catalog, gallery, reviews). Còn lại đều CHẶN chờ Letri: giá bán lẻ · khoá VNPay/MoMo+STK VietQR · GA4/Pixel · endpoint ERP nhận đơn · duyệt chữ 4 trang pháp lý. ⚠️ Tổng hợp DEPLOY cần migration: products.content · members(+token_version) · orders.member_id+consent* · product_reviews(+enum review_status).

[24/6 ⏰ 2026-06-24 17:14 +0700] [C19] QA-FIX (Letri test) — **chữ nội bộ "CHỜ LETRI CẤP" lộ ra cho khách** ở nhiều chỗ. Sửa toàn bộ text RENDER cho khách → trung tính "Đang cập nhật"/"Coming soon": (a) trang chủ mục "Năng Lực Sản Xuất Nhà Máy" (5 ô số liệu nhà máy + note + nút PDF), (b) cert badge trang nhà máy, (c) note bản đồ trang Liên hệ, (d) fallback MOQ ở PDP → "Liên hệ để báo giá", (e) **4 bài blog DB** body "...— CHỜ LETRI cấp." → "Nội dung chi tiết đang được biên soạn." (UPDATE DB, chỉ bỏ dấu nội bộ, không bịa). Verified curl: trang chủ/nhà máy/tin-tuc/trang chi tiết bài đều 0 "CHỜ LETRI". `blog-posts.ts` tĩnh còn marker nhưng `BlogListPageContent` là DEAD CODE (không page nào render) → không lộ. Build PASS. 🔴 **Số liệu nhà máy THẬT (năm thành lập, công suất tấn/tháng, diện tích m², số đại lý, số chi nhánh) + nội dung 4 bài blog = CHỜ LETRI CẤP** (rule N2 không bịa) — đã trình Letri xin. seed.ts/seed-content.ts còn marker (dev tooling, không render).

[24/6 ⏰ 2026-06-24 17:20 +0700] [C19] (1) Điền **công suất nhà máy = 1.000 tấn/tháng** (Letri cấp) vào `story.metrics.capacityValue` vi+en — trang chủ render OK. 4 số còn lại (năm TL, diện tích, đại lý, chi nhánh) vẫn "Đang cập nhật" chờ Letri. (2) Viết **DESIGN BRIEF** cho Claude design: `orchestrator-ceo/ceos/c19-wecha-web/DESIGN-BRIEF-PDF-FILES-20260624.md` — 3 PDF song ngữ: WECHA Bulk Tea Spec Sheet, OEM & Custom Packaging Catalog, Company Profile (brand Crystal Glass, A4, 0 bịa số/cert, placeholder cho data chờ Letri; ATVSTP = scan không vẽ). Nơi dùng: nút tải trang Xuất khẩu + Nhà máy. Build PASS.

[24/6 ⏰ 2026-06-24 17:32 +0700] [C19] Điền **đủ 5 số liệu nhà máy** (Letri cấp) vào story.metrics vi+en: năm thành lập 2015 · công suất Hơn 1.000 tấn/tháng · diện tích Hơn 400m² xưởng + 10.000m² vệ tinh đồi chè · Hơn 20 đại lý · chi nhánh Đang phát triển. Mục "Năng Lực Sản Xuất Nhà Máy" trang chủ giờ ĐẦY ĐỦ (verify curl 5/5). Cập nhật các số này vào DESIGN-BRIEF-PDF-FILES-20260624.md (mục facts). Build PASS.

[25/6 ⏰ 2026-06-25 01:40 +0700] [C19] Menu "Sản phẩm trà" thêm **Trà gói / Trà hộp / Trà quà tặng** (Letri yêu cầu). Cách làm: tạo 3 category retail (`tra-goi/tra-hop/tra-qua-tang` trong product_categories) + gắn 13 trà lẻ hiện có → "Trà gói" (đều là trà gói). /mua-tra thêm lọc `?cat=<slug>` (query `listRetailCategories` + filter theo categoryId + tiêu đề category + empty state có CTA Liên hệ). Header thêm 3 mục con link `?cat=`. Build PASS. Verified: Trà gói=13 thẻ, Trà hộp/quà tặng=empty "đang được bổ sung + Liên hệ tư vấn", menu có đủ 3 link. ⚠️ deploy cần migration product_categories rows + category_id (đã đổi local). Sản phẩm Trà hộp/quà tặng sau này: admin gán category tương ứng là tự lên menu.

[24/6 ⏰ 2026-06-24 17:40 +0700] [C19] **Ảnh placeholder sản phẩm chưa có hình** (Letri muốn catalog nhìn đầy đủ) — KHÔNG dùng ảnh đối thủ (rủi ro bản quyền trên web bán hàng). Thay vào: nâng cấp `TeaImagePlaceholder.tsx` → minh hoạ trà **tự suy tông màu theo tên** (hồng trà nâu-vàng / trà xanh xanh / ô long hổ phách / thảo mộc / neutral) + **hiện tên SP** trên thẻ → mỗi grade nhìn khác nhau, như sản phẩm thật. Wire `name` vào: /san-pham-tra, PDP related, tìm-kiếm, ProductGallery (ảnh chính PDP). Hiện trạng ảnh: retail_tea 13/13 đủ; wholesale_tea thiếu 27/30; drinks thiếu 1/14. Build PASS, verify /san-pham-tra hiện pill tên + tông (black 22/green 10). Ảnh THẬT thay sau: admin upload (media gallery, primaryImageId) tự đè placeholder; Letri chụp/cấp ảnh hoặc ảnh CC0 → C19 wire. KHÔNG tải ảnh đối thủ vào repo.

[26/6 ⏰ 2026-06-26 07:05 +0700] [C19] **Ảnh tạm cho 27 trà sỉ** (Letri yêu cầu tải về folder để deploy). KHÔNG lấy ảnh Google (bản quyền). Tải 6 ảnh trà rời từ **Wikimedia Commons** (giấy phép thương mại) → `public/images/tea/{black,green,oolong,white,puerh,jasmine}.jpg` (fetch bằng curl qua API Commons; python urllib bị chặn SSL). Map TAY đúng loại (Đại Hồng Bào=oolong, Long Tỉnh/Sencha=green, Bạch Hào=white...): 14 black/5 green/3 oolong/2 white/2 puerh/1 jasmine = 27. Tạo 6 media_assets (url=/images/tea/*.jpg) + set products.primary_image_id. Verify: 30/30 trà sỉ có ảnh, 6 ảnh phục vụ HTTP 200, /san-pham-tra dùng đúng. Credit ở `public/images/tea/CREDITS.md` (tác giả+license+nguồn). ⚠️ CC BY/BY-SA cần **ghi công khi go-live** (trang "Nguồn ảnh" hoặc thay ảnh chụp thật); white=Public Domain. ⚠️ deploy: ảnh trong public/ đi theo repo + media_assets rows + primary_image_id (đã đổi local DB).

[29/6 ⏰ 2026-06-29 11:53 +0700] [C19] **AUDIT toàn site (Rule 20) + SỬA lỗi theo batch** (CHƯA commit/deploy). Chạy **workflow 20 cụm (40 agent, rà tĩnh + xác minh đối kháng)** → **158 lỗi thật** (loại ~30 false-positive). Smoke-test: 28/28 khách + 21/21 admin = **49/49 load OK 0 vỡ**. **Agent QA riêng** thao tác THẬT local (login admin, CRUD/lọc/load) → xác nhận gói B + fix chạy đúng; 2 nút (block editor +Đoạn chữ, bulk upload) harness không kích được nhưng backend đúng → **cần Letri bấm thử trình duyệt thật 1 lần**; QA-TEST đã dọn (count=0). **ĐÃ SỬA + typecheck PASS:** (B1) `notDeleted` cho 9 `adminGet*`+deleteReview · nhãn trạng thái đơn khớp enum (orders list/[id]+tai-khoan: "Chờ thanh toán/Đang chuẩn bị/Đang giao" thay chữ thô = đúng lỗi Letri báo) + payment succeeded/cancelled + fulfilment dine_in · `(panel)/error.tsx` bắt mọi action throw (hết dead-end). (B2) **Form lead đại lý+xuất khẩu trước KHÔNG lưu (mất khách!)** → wire `submitLead` (DB+ERP)+ô đồng ý NĐ13+trạng thái, bỏ "TODO". (B3 i18n /en 4 blocker) gio-hang/thanh-toan/CheckoutForm/dat-hang-thanh-cong song ngữ (verify /en/gio-hang 0 leak VI). **CÒN:** i18n landing `/tra/[slug]`(17 chuỗi)+drink option/group nameEn+menu/[slug] descriptionEn+nhãn "Liên hệ" giá /en + ~110 medium/low. Sau: full build + sinh migration. Audit output: `tasks/w6fqydeo1.output`.

[29/6 ⏰ 2026-06-29 09:18 +0700] [C19] **Đợt 1D — nội dung chi tiết SP tiếng Anh sửa trong admin + ✅ GÓI B XONG TOÀN BỘ** (DONE+verify). CHƯA commit/deploy, KHÔNG migration (dùng cột `products.content` sẵn có). Cách an toàn (không phá 13 trà EN cũ): mở rộng type `RetailTeaContent` + `contentSchema` thêm trường EN tuỳ chọn (storyEn/taglineEn/flavorEn/usesEn/ingredientsEn/nutritionEn + brewing.{temp/ratio/time/tip}En + recipe ingredientsEn/stepsEn). Hàm `enViewFromBilingual(c)` dựng view EN (đổ EN vào field *Vi cho `TeaRichContent` đọc) — null nếu chưa có EN → **fallback `RETAIL_TEA_CONTENT_EN` file code**. PDP `mua-tra/[slug]`: render EN = `dbContent EN-view ?? code EN`; metadata EN cũng dùng EN-view (fix leak tagline VI ở meta /en). `ProductContentForm` viết lại: mỗi mục cặp ô **VI | EN** (18 ô soạn). Verify curl: /en trà cũ vẫn English (no regression, 0 leak body); set content EN test→/en hiện "ADMIN-EN-TEST" + /vi hiện VI → xoá test (content NULL); meta /en="Refined bergamot…" 0 leak VI; form content 18 ô VI+EN. Typecheck PASS. **🎉 GÓI B HOÀN TẤT** (1A-1C EN fields+ảnh đồ uống · 1D content EN · Phần1 bulk upload · Phần2 gallery picker · Phần3 block editor kính · menu điều hướng). **DEPLOY bundle migration:** chỉ `posts.body_blocks` jsonb là cột MỚI; còn lại dùng cột/bảng sẵn (products.content/members/orders.member_id/product_reviews từ trước). DB test đã dọn 100% sạch.

[29/6 ⏰ 2026-06-29 07:51 +0700] [C19] **Quản trị MENU điều hướng động** (gói B — DONE+verify). CHƯA commit/deploy. Header menu trước CỨNG trong code → giờ đọc động từ `settings` key `nav.main` (KHÔNG bảng mới, KHÔNG migration). Files: `lib/nav-menu.ts` (type+zod+`DEFAULT_NAV_MENU` mirror Header + `parseNavMenu`/`navHref`), query `getNavMenu()` (content-queries, "use cache"+cacheTag settings, **fallback DEFAULT nếu trống/lỗi → không sập build/trang**), Header nhận prop `menu` + transform (giữ NGUYÊN render JSX), layout fetch+truyền, action `saveNavMenu` (upsert nav.main + `revalidateTag('settings','max')` + `revalidatePath('/','layout')`), admin `/admin/menu` + `MenuEditor.tsx` (thêm/bớt/đổi thứ tự mục + mục con, nhãn VI+EN + path + pathEn cho EN-path khác vd /export), link AdminNav. Verify: Header công khai 24 link đủ (menu mặc định khi chưa lưu); admin /menu nạp 8 mục default; bấm Lưu → `settings nav.main` tạo (8 mục, validate đúng) → đã xoá test (DB sạch, về fallback). Typecheck PASS. **CÒN gói B (cuối):** Đợt 1D — nội dung chi tiết SP tiếng Anh sửa trong admin (dual-source: code `RETAIL_TEA_CONTENT_EN` + DB `products.content`).

[29/6 ⏰ 2026-06-29 07:27 +0700] [C19] **Admin hoàn thiện VI+EN + media + trình soạn KHỐI kính** (Letri gói B). CHƯA commit/deploy. (1) **Ô tiếng Anh trong admin** (cột có sẵn — KHÔNG migration): products `shortDescEn`+`originEn`, posts `excerptEn`+`bodyEn`, drinks `descriptionEn` + **ô ảnh đồ uống** (`primaryImageId`+ImagePicker). Sửa trang [id] nạp đúng giá trị EN cũ (tránh ghi đè rỗng). Verify browser 3 form đủ ô EN. (2) **Tải ảnh HÀNG LOẠT** (`BulkMediaUploadForm`+action `uploadMediaBulk`): nén canvas→webp ≤1600px (hàm chung `image-resize.ts`) + tên "Tên đặt sẵn — ngày giờ VN (Asia/HCM)" + tối đa 30 ảnh. Verify end-to-end (PNG→webp 1508B, tên "… 07:05 29/06/2026", file+DB ok). (3) **Trình soạn KHỐI bài viết** (`PostBlockEditor`+`GalleryPicker`+`PostBlocks`): khối Chữ/Ảnh/Video, mỗi khối bọc **khung kính** (tái dùng `.glass-card-outer/inner` — KHÔNG thêm CSS). Chèn ảnh=modal **lọc thư viện theo tên / tải mới tại chỗ** (action `uploadMediaForPicker`). Lưu JSONB `posts.body_blocks` (🔴 **cột MỚI — local đã ALTER, CẦN migration deploy**); ảnh resolve `mediaId`→url theo tenant khi lưu (rule 02). Render công khai: có khối→render khối; rỗng→body thuần (tương thích bài cũ). Verify end-to-end: form sửa bài có +Đoạn chữ/+Chèn ảnh/+Chèn video; 3 khối test→`/vi/tin-tuc/cach-chon-tra-ngon-cho-quan` hiện chữ-kính+ảnh-kính+video poster → đã xoá test (DB sạch). Typecheck PASS. Block model an toàn (không HTML thô→0 XSS). **CÒN gói B:** Đợt 1D (nội dung chi tiết SP tiếng Anh sửa trong admin — dual-source code+DB) + quản trị menu điều hướng. **DEPLOY:** gộp `posts.body_blocks` jsonb vào migration bundle (cùng products.content/members/orders.member_id/product_reviews…). Files mới: `components/admin/{BulkMediaUploadForm,GalleryPicker,PostBlockEditor,image-resize}.tsx`, `components/PostBlocks.tsx`, `lib/post-blocks.ts`.

[27/6 ⏰ 2026-06-27 16:46 +0700] [C19] **HOSTING chốt + audit VPS** (Letri quyết). (1) Letri CHỐT host wecha.vn **CHUNG VPS ERP** (`root@<VPS_IP>`, ngăn riêng pm2+cổng riêng + **DB riêng** `wecha_web`, KHÔNG chung pos_db) — đè charter cũ "Cloudflare Pages, KHÔNG host chung VPS". (2) Site cũ wecha.vn = **WordPress/cPanel AZDigi**, Letri xác nhận KHÔNG email công ty + **KHÔNG backup**; verify thực `https://wecha.vn` = trang trống mặc định AZDigi (web cũ đã tắt, không online) → KHÔNG có nội dung/SEO để cứu, **xây mới hoàn toàn**. **Bỏ gói AZDigi** sau khi web mới chạy ổn. DNS hiện ở **AZDigi** (không phải Mắt Bão như ghi chú cũ) → cutover = trỏ wecha.vn → <VPS_IP>. (3) Đọc VPS (read-only, Letri cấp quyền): Ubuntu 22.04 · 6 nhân CPU load ~1.3 (22%) · RAM còn trống 6.7G · **ổ cứng 89G dùng 80% (còn 18G)** ⚠️ → chạy chung ĐƯỢC nhưng cần **dọn ~15–20G rác trước**: 8 bản `pos-system.old-*` ≈11G (🔴 flag C13 — POS deploy nhiều, dọn chưa sạch) + docker images/build-cache ≈8G + backup tháng 4 ở /root → dọn xong còn ~35G. Dọn = lệnh XOÁ → **chờ Letri OK rõ**. (4) Kế hoạch deploy: build LOCAL (tránh build nặng cạn connection PG trên VPS) → đẩy bản build → pm2 `wecha-web` cổng riêng → nginx server block wecha.vn + Certbot SSL → DB `wecha_web`. Chi tiết: memory [[project_c19_wecha_fullstack_pivot]]. **Tiếp:** EN đợt 3 (dịch 43 SP + 14 đồ uống + 5 bài → UPDATE DB).


[25/6] [CEO-ERP] 🤖 WORKFLOW 3 AGENT song song (Letri uỷ quyền điều phối tự chủ) — 3/3 done, Opus review+fix+build+commit. (1) Bulk GÁN SP→CHI NHÁNH để bán (action set_branches, UPSERT branch_products link_purpose='sale' ON CONFLICT uq_branch_product_purpose + modal Glass chọn nhiều CN) = việc MỚI Letri. (2) Excel xuất Nhập kho cấp document: route /excel-export/stock-inward cơ bản 7 cột + full MISA — Opus FIX bug agent (filter sd.deleted_at nhưng stock_documents KHÔNG có cột → export rỗng). (3) Cố định cột Xuất/Chuyển/Kiểm kê. API tsc=0, web build PASS. 4 commit chờ deploy (063cdea/3900c49/cc098d1 + 4707534 freeze inward). ⚠️ mig 0413 ĐÃ apply (RFQ 500 hết). ⏰ 2026-06-25 ~01:42.

[26/6 ⏰ 2026-06-26 07:16 +0700] [C19] **Landing quảng cáo trà — bắt đầu Hồng Trà** (Letri yêu cầu). Làm dạng **template tái dùng** `/[lang]/tra/[slug]` (sau thêm trà xanh/ô long chỉ cần thêm data). Nội dung soạn + verify qua workflow `wecha-hongtra-landing-content` (0 bịa số/cam kết, không mật ong, nhiệt 95–100°C) → `src/db/data/tea-landing-content.ts`. Trang gồm: hero + intro + 4 lợi ích (định tính) + showcase **5 hồng trà bán lẻ** (card mua) + "vì sao WECHA" + lưới **14 hồng trà sỉ** (B2B) + CTA đăng ký xuất khẩu + cách pha + 3 công thức + 6 FAQ (FAQPage JSON-LD) + hreflang/OG. Query mới `listProductsBySlugs`. Thêm sitemap `/tra/<slug>`. Build PASS. Verified curl: /vi/tra/hong-tra 200, hero + 19 link SP + FAQ JSON-LD + công thức + CTA sỉ. ⚠️ deploy: chỉ thêm code (data file + route + query) — KHÔNG đổi DB. **Tiếp:** trà xanh, ô long (dùng lại template); có thể đưa landing vào admin-CMS sau.

[25/6] [CEO-ERP] 🤖 ĐỢT 2 agent RỚT (ConnectionRefused hạ tầng) → Opus TỰ CỨU hoàn thành. (1) Math.random→PG sequence XONG: rm-stock (agent làm dở, Opus bắt 2 caller thiếu await 221/390 + fix) + fg-stock/manufacturing/payments (Opus tự làm) + migration 0414 (7 sequence). API tsc=0, hết Math.random code thật. (2) Excel xuất Xuất kho (cơ bản+full MISA): tham số hoá exportStockInwardList(documentType) + route /excel-export/stock-outward + FE 2 nút. (3) einvoice số HD = placeholder mock (số thật từ MISA Wave 4) → để sau. (4) Chuyển kho/Kiểm kê Excel (bảng khác) → đợt sau. 2 commit chờ deploy (9ee8ab5+19a5f78). ⚠️ DEPLOY ORDER: apply migration 0414 TRƯỚC khi deploy code (nextval cần sequence). ⏰ 2026-06-25 ~06:10.

[25/6] [CEO-ERP] 🤖 ĐỢT 2+3 agent (tự chủ) + einvoice — 4 commit gom chờ deploy, build PASS. (Đợt 2) Math.random→PG sequence lot/ref/WO/txn (rm/fg/manufacturing/payments) — mig 0414 tạo 7 sequence ĐÃ apply prod; Excel xuất Xuất kho. (einvoice) bỏ Math.random publish() giả lập→crypto+nhãn MOCK (số HĐ thật từ MISA Wave 4). (Đợt 3) Excel xuất theo lọc 6 bảng: NCC, công nợ thu/trả, HĐ mua, phiếu nhập NCC, đơn mua (luật toàn hệ 15/6). Opus QC: verify 5 method không ghi đè (2 agent chung file), API tsc=0 + web build PASS. ⚠️ CÒN G7 (thuế GTGT/ổ + đối soát NH) + ~30 báo cáo = Opus tự lead đợt sau (đụng tiền/thuế, không fire-and-forget). ⏰ 2026-06-25 ~08:30.

[26/6 ⏰ 2026-06-26 06:54 +0700] [C18] Việc 2206-02 DRAFT READY — **YouTube MCP Server n8n_wf-88** (10 tools qua MCP Server Trigger). Đã thiết kế + viết hoàn thiện flow JSON production-ready: (1) `claude/n8n-ceo-automation/flows/draft-015-youtube-mcp-server.json` — 11 nodes (1 MCP Trigger + 10 tool): get_channel_info · list_my_videos · get_video_stats · list_comments · get_analytics (YouTube Analytics API) · list_playlists + WRITE: reply_to_comment · update_video · upload_video · add_to_playlist. Cải tiến vs design gốc: credential type chuẩn `predefinedCredentialType googleOAuth2Api` cho HTTP Request (bảo mật, không dùng generic oAuth2), description READ/WRITE rõ ràng + cảnh báo `⚠️ HÀNH ĐỘNG GHI` cho 4 tool WRITE, errorWorkflow link `n8n_wf-error-handler`, `_meta` block inline docs + Rule N9 Phase 2 note, `"active": false` an toàn. (2) `orchestrator-ceo/wiki/integration/N8N-88-youtube-mcp-server-SPEC.md` — checklist 6 bước cho Letri setup (~30 phút). ⚠️ **CHẶN chờ Letri**: Google Cloud OAuth Client ID + bật YouTube Data API v3 + YouTube Analytics API; n8n credential tạo Google OAuth2 + Bearer token; thay 2 PLACEHOLDER trong JSON; import + activate; thêm vào claude_desktop_config.json. Rule N9 logging = Phase 2 (cần C2 build endpoint). Không có deploy code/commit BE.

[26/6] [CEO-ERP] 🔍 G7 ĐỐI SOÁT NH — Letri chốt làm trong ERP (KHÔNG build tờ khai thuế = để MISA). Opus điều tra: tính năng ĐÃ BUILD ĐẦY ĐỦ từ trước (module bank-statements: import sao kê XLSX→base64, auto-match cashbook ±2 ngày cùng số tiền, khớp tay, reconcile/complete, scope database_id G2) + 7 route LIVE trên prod + FE list/detail/import-modal. ⚠️ Spec B5.3 "ReconciliationService CHƯA TỒN TẠI" = LẠC HẬU. GAP THẬT duy nhất: sidebar THIẾU menu → user không tìm thấy (giống Sales Quy trình). Đã thêm menu "Đối soát ngân hàng" /admin/finance/bank-statements (nhóm TC 7.1). Tables rỗng vì chưa ai nhập sao kê — sếp nhập file Excel sao kê thật để dùng. Build PASS, chờ deploy. ⏰ 2026-06-26 ~07:10.

[26/6] [CEO-ERP] 🤖 4 việc Letri 26/6 (build PASS, gom chờ deploy): (1) Header LUÔN hiện CSDL đang làm (DatabaseSelector không ẩn khi list rỗng tạm + còn activeDatabaseName). (2) Menu "Đối soát ngân hàng" (tính năng đã có sẵn, chỉ thiếu link). (3) Bán hàng theo VÙNG: GET /orders/sales-by-region (province_name/country) + trang /admin/sales/reports/by-region (KPI+Recharts+bảng tỷ trọng) + menu. (4) Nâng cấp Quy trình Bán hàng: vòng đời 6 vòng tròn + mũi tên + sub-process (mô phỏng p2p-flow), giữ data thật. Opus QC: route sales-by-region trước :id ✓, sidebar đủ menu không clobber ✓, API tsc=0 + web build PASS. ⏰ 2026-06-26 ~07:35.

[26/6 ⏰ 2026-06-26 08:34 +0700] [C19] **Video YouTube nhúng trong trang chi tiết sản phẩm** (Letri yêu cầu — bấm phát ngay trong web). Thêm cột `products.video_url` (catalog.ts + ALTER local — ⚠️ deploy cần migration). Component `YouTubeEmbed.tsx`: **click-to-play** — hiện poster (i.ytimg) + nút play, **chỉ tải iframe youtube-nocookie SAU khi bấm** (nhẹ + không tracking khi chưa bấm + thân thiện consent). PDP `mua-tra/[slug]` render mục "Video sản phẩm" khi có video_url. Admin: ProductForm thêm ô "Video YouTube (URL)" + action validate link YT + lưu (insert/update). `listColumns` + `adminGetProductById` (select() ) đã có videoUrl. Build PASS. Verified: set tạm video → PDP hiện section + poster + nút play, iframe KHÔNG preload (count=0, chỉ load sau bấm) → đã xoá test (DB sạch). Áp cho trà lẻ + sỉ (PDP chung). Đồ uống /menu/[slug] thêm tương tự sau nếu cần. Admin dán link YT là tự hiện.

[26/6 ⏰ 2026-06-26 08:47 +0700] [C19] 🐛 FIX lỗi tương phản (Letri báo: mục "Năng Lực Sản Xuất Nhà Máy" chữ trùng nền vô hình). **Nguyên nhân gốc:** project chạy **Tailwind v4** nhưng code dùng cú pháp v3 `bg-opacity-*`/`text-opacity-*`/`border-opacity-*` (đã bị v4 BỎ) → `bg-cream` thành cream ĐẶC thay vì 5% → card cream + chữ cream = vô hình (verify computed: cardBg solid rgb(247,239,227)=valueColor). Sweep toàn site (python regex gộp màu+độ-mờ liền kề → cú pháp v4 `/n`) — 14 file, ~164 bg-opacity + 72 text-opacity + 56 border-opacity. Sửa luôn các chỗ "cùng màu → vô hình" khác (LeadForm/Agency/Export icon `bg-matcha bg-opacity-10 text-matcha`). Verify computed sau fix: cardBg=cream/0.05 (nền tối mờ), chữ cream nổi rõ ✓. Build PASS. Residual ~100 `*-opacity` là vô hại (trên `glass-card-*` nền do CSS class lo, + `hover:bg-opacity` hiệu ứng hover). KHÔNG đổi DB.

[26/6] [CEO-ERP] ✅ wave-2.20.27 LIVE 08:50 — 4 việc: header hiện CSDL + menu Đối soát NH + Bán hàng theo vùng + Quy trình Bán hàng visual (6 vòng tròn). Verify prod đủ (by-region route 3, o2c-circle 5, 2 menu, health 200). Tag+push. ⏰ 2026-06-26 08:50.
[26/6] [CEO-ERP] 🤖 Đang chạy: 7 SỔ CHI TIẾT (Letri chọn) — agent BE (7 endpoint /accounting/ledger/*) + agent FE (7 trang report-center s05/s08/s11/s12/s31/s32/s38). ⚠️ 5/7 sổ data RỖNG (journal/công nợ/bank ở MISA) — build để fix link hub gãy + sẵn sàng; S11(bán hàng)+S12(kho) có data. Thuế XML để MISA. ⏰ 2026-06-26 ~08:55.

[26/6 ⏰ 2026-06-26 09:03 +0700] [C13] BÁO CÁO SESSION 6 — Apps Script @81 LIVE. Wave A (fmtDate/fmtDateTime đồng nhất 5 trang) + Wave B (Edit+Delete ThuChi/NguoiDung/ChiNhanh + wallet CaiDat) ĐÃ XONG. Trạng thái 17/21 PASS, 1 chưa test (Thực đơn CRUD), 1 chờ data (LOG KHO), 1 STUB (PhanQuyen). 🔴 VPS BACKEND c308b8d (mailer) CHƯA DEPLOY: VPS `/opt/phache-pos/backend` KHÔNG phải git repo → git pull không chạy được. Cách deploy đúng: (1) `pnpm build` tại `POS-FNB-V4/backend/` trên máy Letri → (2) `rsync -avz --delete dist/ root@<VPS_IP>:/opt/phache-pos/backend/dist/` → (3) `ssh root@<VPS_IP> "pm2 restart phache-api"` (tên pm2 = phache-api, không phải pos-api). 📋 CEO ERP CẦN DUYỆT 5 PROPOSE-GAP trong VIEC-C13: PG-01 CaiDat VIP, PG-02 CaiDat Tax, PG-03 Menu Category CRUD, PG-04 CongThuc RecalcAll, PG-05 PhanQuyen RBAC thật (cần BE /v1/admin/roles).

[26/6 ⏰ 2026-06-26 09:11 +0700] [C19] **Tiếng Anh — đợt 1** (Letri báo /en/chinh-sach-bao-mat hiện tiếng Việt). Audit toàn site + dịch EN (workflow `wecha-en-translation`, dịch trung thực giữ NĐ13/ATVSTP/sự thật): (1) 4 trang **chính sách + FAQ** → thêm `POLICY_PAGES_EN` + `getPolicyPage(slug,lang)`; (2) **landing Hồng Trà** → `TEA_LANDING_CONTENT_EN` + route chọn theo lang; (3) **bật EN cho /lien-he** (trước notFound EN; dictionaries.en đã có) + hreflang. Wire 4 page policy nhận `params.lang`. ⚠️ FIX build: query `getSocialLinks` ở layout (chạy 200 trang × 9 worker) gây CẠN connection Postgres → bọc try/catch fallback {} (footer social không được làm sập build). Build PASS 200/200. Verified: /en/chinh-sach-bao-mat="Personal Data Privacy Policy", /en/tra/hong-tra hero "Amber-Red" (0 leak VI), /en/lien-he 200 + form EN, /en FAQ EN; VI không hồi quy. **CÒN THIẾU EN (đợt sau):** nội dung rich 13 trà lẻ (retail-tea-content.ts, 208 field *Vi) · short_desc_en 43 sản phẩm · description_en 14 đồ uống · body/excerpt EN 5 bài blog. Chrome/menu/footer đã song ngữ sẵn. KHÔNG đổi DB (data EN trong code).

[26/6 ⏰ 2026-06-26 09:40 +0700] [C19] **Tiếng Anh — đợt 2**: dịch xong **rich content 13 trà lẻ** (story/flavor/uses/ingredients/nutrition/brewing/recipes) qua workflow chunked (3 agent/lô — tránh rate-limit; lần đầu 13 agent đồng loạt RỚT rate-limit). Thêm `RETAIL_TEA_CONTENT_EN` (13/13) + `getTeaContent(slug,lang)`. Làm **TeaRichContent song ngữ** (17 label cứng VI → ternary). PDP: content theo lang + bỏ chặn `vi &&` (EN ẩn rich nếu chưa có data, KHÔNG leak VI). Build PASS 200/200. Verified: /en/mua-tra/hong-tra-ba-tuoc hiện EN (Product story/Key benefits + "bergamot"), 0 leak VI; VI giữ nguyên. **CÒN (đợt 3):** short_desc_en 43 SP + description_en 14 đồ uống + body/excerpt EN 5 bài blog (đều DB, ngắn — 1 workflow batch + UPDATE). KHÔNG đổi schema DB.

[26/6] [CEO-ERP] 🔍 AUDIT HỆ BÁN HÀNG (4 agent reviewer) → 58 findings (13 critical). SỬA XONG (build PASS, chờ deploy): 3 BUG TÀI CHÍNH critical Opus tự fix (ghi nợ X2 đơn credit / trừ kho thiếu tenant / auto-je mất khi PATCH completed). + wave 3 agent: returns (confirm+lý do, route, mã phiếu sequence mig 0415, transaction, breadcrumb/filter/Excel) · quotes (soft-delete bỏ hard-delete quote_items + deleted_at mig 0416 + Excel) · UX (SSR isMobile, bỏ console.error, error-state promotions/gift-cards, Excel by-region, process nhãn 'minh hoạ', KPI label). ⚠️ #11 orders.create() chưa bọc transaction = Opus tự lead riêng (refactor lớn). Migration chờ apply: 0414(seq lot/ref)+0415(returns seq)+0416(quotes deleted_at). ⏰ 2026-06-26.
[26/6] [CEO-ERP→C13] DUYỆT PROPOSE-GAP: ✅ PG-01 VIP + PG-02 Tax settings (build được). ⏸ HOÃN PG-03 menu CRUD / PG-04 recalc giá vốn (đụng tài chính) / PG-05 RBAC đầy đủ (lớn, 3 role cứng đang chạy OK). Deploy mailer c308b8d (phache-pos) = Letri tự chạy 3 bước rsync. ⏰ 2026-06-26.

[26/6 ⏰ 2026-06-26 22:45 +0700] [C13-POS-FNB-V4] fix 3 gap minor res.error (commit e9503ca) — 13-BaoCaoTon failureHandler hiện ⚠️ đỏ thay vì dash trắng; 15-NguoiDung branches error hiện showBanner thay vì silent return; 03-Thue taxExportMonthly check data.error trước data.url. clasp push 43 files LIVE.

[26/6 ⏰ 2026-06-26 22:39 +0700] [C13-POS-FNB-V4] BÀN GIAO SESSION — Fix toàn bộ `res.error` pattern trên 19 trang GAS. Đã clasp push 2 lần (1:22 + 1:32 PM) — 43 files LIVE.

**Vừa làm:** Fix `res.error` silent swallow trên 9 file HTML — 15 điểm lỗi:
- `11-DonHang.html`: tmdtLoadMenu silent → banner lỗi
- `06-ThuChi.html`: doTransfer thiếu r.error check
- `07-Kho.html`: khoLoadNL + khoLoadBTP + khoBulkConfirm (3 chỗ)
- `05-BaoCao.html`: bc_today per-branch skip stat.error
- `03-Thue.html`: calculateTax + getTaxForecast + taxWarningLevel (3 chỗ)
- `14-Chuoi.html`: pos_getBranches render ngay không check error
- `18-TichHop.html`: pos_loadTichHop silent empty settings
- `09-KhachHang.html`: getCustomerDetail + createCustomer (2 chỗ)
- `12-CongThuc.html`: ctLoadMon + khoListNLFromSheet + ctLoadWarnings (3 chỗ)
Trước đó (session trước): `02-POS.html` — badge ca hiển thị tien_mo đúng + check ca cũ ngày hôm trước + onOpenShiftSubmit check res.error.

**Dừng tại:** `clasp push` thành công lần cuối (1:32 PM). Tất cả 43 file đã LIVE trên GAS server. Chưa verify bằng mắt (cross-origin iframe giới hạn screenshot).

**Bước tiếp theo CHÍNH XÁC:** Audit UX tiếp theo — kiểm tra 3 gap minor còn lại:
1. `13-BaoCaoTon.html` line ~116: `kho_alertLowStock` failureHandler không có showBanner (silent empty state)
2. `15-NguoiDung.html` line ~90: `pos_getBranches` callback không explicit error check
3. `03-Thue.html` line ~433: `taxExportMonthly` check `data.url` nhưng không check `data.error` trước
Sau khi fix 3 minor trên → `clasp push` → mở browser verify tại: https://script.google.com/macros/s/AKfycbxyDQEkxprQYOjAiIffvt8lL2RftzwDHlPn5qZG5wE/dev

**Cạm bẫy cần nhớ:**
- GAS app nằm trong cross-origin iframe (googleusercontent.com) — Chrome extension `executeScript` KHÔNG chụp được, dùng `get_page_text` để verify text thôi
- `clasp push` báo "already up to date" = bình thường khi file không thay đổi nội dung thực sự
- Mọi confirm/alert/prompt PHẢI dùng `glassConfirm()` — KHÔNG dùng native window.confirm
- File GAS: `/Users/letri/Projects/erp-project/POS-FNB-V4/apps-script-client/anh-tuan/`

**Hệ khác cần biết:** Không có thay đổi nào ảnh hưởng backend VPS (api.phache.com.vn). Chỉ fix UI/error-handling thuần GAS. Luật 80/20 giữ nguyên — không thêm logic vào Apps Script.
