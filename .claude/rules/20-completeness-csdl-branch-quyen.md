# Rule 20 — HOÀN THIỆN TỪNG TRANG: CSDL → CHI NHÁNH → QUYỀN NGƯỜI DÙNG

> Mức độ: 🔴 CRITICAL | Áp dụng: MỌI trang, MỌI nút, MỌI chức năng người dùng
> Letri chốt 28/6/2026 (sau lỗi "Báo giá hết hạn không duyệt được" + "Đơn chờ duyệt có đơn không hiện").
> Áp cho Opus CEO ERP + MỌI CEO con + MỌI worker. Vi phạm = báo cáo DONE sai = Letri phải nhắc lại.

---

## 0. TẠI SAO CÓ LUẬT NÀY

Letri đã nhắc nhiều lần "làm hoàn thiện" nhưng vẫn lọt: trang có nhưng nút không chạy, lọc rỗng, list không hiện dù có dữ liệu, sửa/xoá báo 400 câm, không lọc theo chi nhánh/CSDL, ai cũng xem được dữ liệu nhạy cảm.

**Gốc rễ:** code "cho có trang" rồi báo DONE, KHÔNG đi hết từng nút × từng chức năng × 3 trục (CSDL → Chi nhánh → Quyền).

**Luật này biến việc đó thành CHECKLIST BẮT BUỘC.** Không tick đủ = KHÔNG được báo DONE, KHÔNG được deploy.

---

## 1. ĐỊNH NGHĨA "HOÀN THIỆN 1 TRANG"

Một trang chỉ DONE khi MỌI chức năng người dùng dưới đây **chạy thật** (test reload + verify DB), KHÔNG phải "có nút":

| Chức năng | Phải chạy thật |
|---|---|
| **Load / xem danh sách** | Hiện ĐÚNG dữ liệu (không rỗng giả khi có data); list endpoint trả ĐỦ field FE cần (vd `requiresApproval`, `validUntil`) |
| **Lọc (filter)** | Mọi tab/ô lọc (trạng thái, ngày, tìm kiếm) ra đúng kết quả; param FE↔BE khớp tên (snake↔camel) |
| **Thêm (create)** | Validate đủ, lưu DB, reload thấy ngay |
| **Sửa (update)** | Mở form có data cũ, lưu được, KHÔNG 400 câm; trạng thái cho phép sửa khớp FE↔BE |
| **Xoá (delete)** | Soft-delete + thùng rác; chỉ hiện nút khi được phép xoá |
| **Hành động nghiệp vụ** (duyệt/gửi/từ chối/chuyển/xuất hoá đơn…) | Mỗi nút chỉ hiện khi trạng thái + quyền cho phép; bấm là chạy, KHÔNG dead-end |
| **Xuất Excel theo lọc** | CSV BOM UTF-8, cột đọc ĐÚNG field (xem Rule mọi-bảng-có-Excel) |
| **3 trạng thái UI** | Loading / Error / Empty rõ ràng (Rule 05) |

> 🔴 **CẤM báo DONE khi mới "có giao diện".** Phải đi HẾT bảng trên cho TỪNG nút.

---

## 2. BA TRỤC BẮT BUỘC KIỂM — THỨ TỰ: CSDL → CHI NHÁNH → QUYỀN

Mỗi chức năng (load/lọc/thêm/sửa/xoá/hành động/xuất) PHẢI đúng trên **cả 3 trục**:

### Trục 1 — 🗄️ CSDL (database_id / ổ dữ liệu) — xem SPEC-CSDL-SCOPING
- List/report lọc theo `database_id` đang chọn (header → `databaseFilter()`), fail-open NULL.
- Create resolve `database_id` từ context (KHÔNG tin client).
- Đổi CSDL trên thanh chọn → trang nạp lại đúng tập dữ liệu của ổ đó.
- KHÔNG để dữ liệu ổ A lẫn ổ B.

### Trục 2 — 🏢 CHI NHÁNH (branch_id)
- Có lọc theo chi nhánh đang chọn (`activeBranchId`) — **param FE gửi phải khớp tên BE đọc** (vd FE gửi `branch_id` thì BE DTO phải đọc `branch_id`, KHÔNG phải `branchId` → nếu lệch = lọc bị bỏ qua âm thầm).
- Create auto-fill chi nhánh hiện hành; báo cáo gộp/tách theo chi nhánh đúng.
- NV chi nhánh A KHÔNG thấy dữ liệu chi nhánh B (nếu nghiệp vụ yêu cầu).

### Trục 3 — 🔐 QUYỀN NGƯỜI DÙNG (permission `resource.action`)
- MỌI endpoint mutation + dữ liệu nhạy cảm có `@RequirePermissions('x.action')` (KHÔNG chỉ `JwtAuthGuard`).
- **Thêm guard PHẢI seed quyền + gán policy TRƯỚC** (tránh khoá NV 403) — Rule trong handover.
- FE ẩn/hiện nút theo quyền; bấm nút không có quyền → 403 rõ ràng, KHÔNG vỡ trang.
- Dữ liệu PII/tài chính (lương, sổ quỹ, bút toán, công nợ, giá vốn) — chặn theo vai trò + audit log (Rule 01, 15).
- Đa-tenant: mọi query filter `tenant_id` (Rule 02) — KHÔNG bao giờ bỏ.

---

## 3. QUY TRÌNH BẮT BUỘC (Opus tự làm, KHÔNG đợi Letri nhắc)

Khi nhận 1 mảng/màn hình cần hoàn thiện:

```
1. MỞ AUDIT  → đọc code thật (BE service+controller, FE page) + grep + verify prod (SSH read/DB)
              → đối chiếu mục 1 (8 chức năng) × mục 2 (3 trục) cho TỪNG trang.
2. GAP MATRIX → bảng: Trang | Chức năng | Trục | Trạng thái (✅/⚠️/❌) | Verify (📖🔍🖥) | Việc cần làm.
              → có mockup chưa? Chưa → thiết kế mockup/contract trước khi code.
3. CHIA AGENT → mỗi gap = task rõ spec + chọn model (Haiku/Sonnet/Opus) theo độ khó.
              → BE/migration/tenant/tài chính/bảo mật = Claude (Sonnet/Opus). FE rủi-ro-thấp = được.
4. SOÁT      → reviewer agent (Sonnet) soát đối kháng TRƯỚC commit (IDOR/tenant/tài chính 3 vòng).
5. BUILD     → clean build (rm .next/.turbo) PASS (web webpack + api tsc) TRƯỚC khi báo DONE.
6. DEPLOY    → theo luật: apply migration TRƯỚC + 2 cổng Letri (apply prod + YES DEPLOY).
7. KIỂM SAU DEPLOY → MỞ AGENT ĐIỀU PHỐI kiểm tra TỪNG chức năng vừa deploy trên prod thật
              (click-test CRUD + lọc + 3 trục), đối chiếu gap matrix → đóng từng dòng.
```

> 🔴 Bước 1, 2, 7 KHÔNG được bỏ. "Chia agent code" mà bỏ "audit + gap + kiểm sau deploy" = làm dối.

---

## 4. CHECKLIST TRƯỚC KHI BÁO DONE 1 TRANG 🔴

- [ ] Load list: hiện đúng data thật (test prod/DB), list endpoint trả đủ field FE dùng
- [ ] Mọi tab lọc + ô tìm + lọc ngày: ra đúng kết quả
- [ ] Thêm: lưu được, reload thấy
- [ ] Sửa: mở có data, lưu được, trạng-thái-cho-sửa khớp FE↔BE (không 400 câm)
- [ ] Xoá: soft-delete, chỉ hiện khi được phép
- [ ] Mỗi nút hành động: chỉ hiện đúng trạng thái+quyền, bấm chạy thật, không dead-end
- [ ] Xuất Excel theo lọc: cột đọc đúng field
- [ ] CSDL: đổi ổ → nạp đúng; create resolve database_id
- [ ] Chi nhánh: lọc đúng, param FE↔BE khớp tên, create auto-fill
- [ ] Quyền: endpoint có @RequirePermissions + seed quyền; FE ẩn/hiện theo quyền; tenant filter đủ
- [ ] 3 trạng thái UI (loading/error/empty)
- [ ] Clean build PASS (web + api)
- [ ] Sau deploy: agent điều phối click-test xác nhận từng chức năng

---

## Cross-ref
- `rules/01-security.md` (quyền, PII, audit) · `rules/02-multi-tenant.md` (tenant) · `rules/05-frontend.md` (3 state, FE↔BE shape)
- `docs/SPEC-CSDL-SCOPING-20260621.md` (CSDL scoping) · `docs/AUDIT-PHAN-QUYEN-20260628.md` (244 controller map quyền)
- Memory: `feedback_completeness_per_page_csdl_branch_permission.md` · `feedback_opus_autonomous_dev_audit_loop.md`

---

**END Rule 20**
