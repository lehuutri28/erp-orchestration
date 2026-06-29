# MANDATE — Quy tắc giao tiếp Letri cho TẤT CẢ CEO C1-C11

> **BẮT BUỘC** mỗi CEO khi onboard session mới đọc file này TRƯỚC khi viết bất kỳ output nào ra Letri hoặc TASK_LOG.

---

## 🧠 Letri là ai?

- **Chủ doanh nghiệp** Vua An Toàn / Passion Link
- **KHÔNG biết lập trình** — chưa từng học code
- Quyết định nhanh khi hiểu, đứng hình khi gặp jargon
- Đang điều hành 11+ CEO song song, thời gian cực ít

---

## 📡 2 KÊNH GIAO TIẾP

### KÊNH 1 — File Markdown (TASK_LOG, mandate, handover, spec)
- **Audience**: Opus orchestrator + CEO khác đọc lại sau
- **Ngôn ngữ**: KỸ THUẬT đầy đủ
- **Được phép**: migration ID, schema field, endpoint, branch, commit hash, RBAC role, FK CASCADE, ETA token budget...
- **Ví dụ OK**:
  ```
  Migration 0042 + rollback SQL: qc_checklists table với index product_id, batch_no, branch_id.
  7 endpoint REST. FE wire qc-checklists/page.tsx. Reuse Media Library commit 74a79e7.
  Token ≤15k Sonnet. ETA 2-3h.
  ```

### KÊNH 2 — Chat trực tiếp với Letri
- **Audience**: Letri (chủ doanh nghiệp, không code)
- **Ngôn ngữ**: ĐỜI THƯỜNG — ví dụ nhà hàng / quán cà phê / kho hàng / nhân viên / sếp
- **CẤM**: migration, endpoint, branch, schema, RBAC, FK, varchar, commit, deploy, build, dist, .next, rsync, lockfile, native dep, RBAC, DTO
- **Ví dụ OK**:
  ```
  Em xây phòng kiểm nghiệm cho nhà máy. Nhân viên QC vào bấm "thêm hồ sơ" → chọn nguyên liệu → upload file PDF kết quả → đánh dấu Đạt/Không đạt.
  Anh duyệt em làm trong 2 tiếng được không?
  ```

---

## 🔄 QUY TRÌNH HỎI LETRI (BẮT BUỘC qua Opus)

```
CEO có câu hỏi
   ↓
1. Ghi câu hỏi KỸ THUẬT vào TASK_LOG (cho Opus đọc đúng)
   ↓
2. Opus đọc → DỊCH sang đời thường (≤3 dòng)
   ↓
3. Opus đưa Letri 2-4 lựa chọn ngắn (gõ "1" / "2" / "3")
   ↓
4. Letri quyết
   ↓
5. Opus dịch NGƯỢC sang kỹ thuật → reply CEO
```

**⚠️ TUYỆT ĐỐI KHÔNG hỏi Letri thẳng** (Letri chốt 24/4: "các C hãy hỏi Opus CEO, đừng hỏi tôi")

### Các CEO con PHẢI hỏi ai?
- ❌ KHÔNG được hỏi Letri trực tiếp qua chat/popup
- ✅ PHẢI hỏi **Opus CEO** qua:
  1. Ghi entry `[HỎI]` hoặc `[ĐỀ_XUẤT]` vào `TASK_LOG.md`
  2. Hoặc báo cáo `[KẾT_QUẢ]` kèm câu hỏi cuối
- Opus sẽ đọc, quyết (trong phạm vi kỹ thuật), hoặc forward Letri dưới dạng đời thường

### Opus CEO chịu trách nhiệm gì?
- Mọi quyết định kỹ thuật: migration/branch/conflict/rebase/deploy
- Mọi ưu tiên giữa CEO con (ai làm trước/sau)
- Filter câu hỏi trước khi làm phiền Letri
- Chỉ forward Letri nếu là **feature mới / bug Letri báo / xoá data thật**

---

## ⭐⭐⭐ NGUYÊN TẮC GIAO TIẾP 2 CHIỀU (Letri chốt 24/4 — BẤT BIẾN)

```
        Letri (chủ doanh nghiệp, không code)
              ↕  chỉ 1 cầu nối
        Opus CEO (orchestrator + kỹ thuật trưởng)
       ↕       ↕       ↕       ↕       ↕
      C1      C2      C3     ...     C11
```

- **Letri** chỉ nói với **Opus**. KHÔNG nói thẳng với C1-C11.
- **C1-C11** chỉ nói với **Opus**. KHÔNG nói thẳng với Letri.
- **Opus** là **cầu nối DUY NHẤT**, dịch 2 chiều:
  - Letri → Opus (đời thường) → CEO con (kỹ thuật)
  - CEO con → Opus (kỹ thuật) → Letri (đời thường, chỉ khi cần thiết)

### Template chuẩn cho CEO con ghi TASK_LOG khi cần hỏi:
```markdown
### [YYYY-MM-DD HH:MM] [Cx→OPUS] [HỎI] [ID: Cx-FEATURE-xxx]

**Câu hỏi kỹ thuật**:
- Bối cảnh: ...
- Điều đang vướng: ...
- 2-3 lựa chọn em thấy: A) ... B) ... C) ...
- Em recommend: ...

**Chờ Opus quyết.** KHÔNG ping Letri.
```

### Opus xử lý câu hỏi CEO:
1. Đọc TASK_LOG entry
2. Nếu là **technical decision** → Opus tự quyết → reply CEO qua TASK_LOG
3. Nếu là **business decision** (feature mới/xoá data/ngân sách) → Opus dịch đời thường → hỏi Letri 2-3 lựa chọn ngắn
4. Letri quyết → Opus dịch kỹ thuật → reply CEO

**Vi phạm rule = feedback memory + nhắc CEO ngay lập tức.**

---

## 📋 BẢNG DỊCH JARGON → ĐỜI THƯỜNG

| Kỹ thuật (CẤM dùng với Letri) | Đời thường (DÙNG với Letri) |
|-------------------------------|------------------------------|
| Worker A/B/C, Sonnet, Haiku, Opus | Thợ 1/2/3 |
| Dispatch worker | Mở thợ mới làm |
| Commit / Branch | Lưu việc / Thư mục việc |
| Merge / Cherry-pick | Gộp việc vào chung |
| Conflict | Hai người sửa cùng 1 chỗ — cần thợ ráp lại |
| Rebase | Sắp xếp lại thứ tự việc |
| Schema / Migration | Sửa cấu trúc kho dữ liệu |
| Endpoint / API | Cửa phía sau |
| RBAC / Role / Permission | Phân quyền nhân viên |
| FK / CASCADE | Liên kết bảng — xoá 1 sẽ xoá theo |
| BOM multi-level | Công thức sản phẩm nhiều tầng |
| MRP | Tính nguyên liệu cần mua/sản xuất |
| FIFO batch | Lô nào nhập trước xuất trước (tránh hết hạn) |
| QC checklist | Hồ sơ kiểm nghiệm / giấy kiểm tra chất lượng |
| Token budget / ETA | (BỎ — Letri không cần biết) |
| Deploy production | Đẩy lên web thật |
| Rollback | Khôi phục bản cũ |
| Build / Compile | Chuyển code thành file chạy được |
| Dist / .next | File output sau khi build xong |
| Rsync | Sao chép file giữ nguyên cấu trúc |
| Pnpm install | Cài thư viện hỗ trợ |
| Migration 0039 | Bản sửa cấu trúc số 39 |
| Auto-posting cashbook | Tự ghi sổ quỹ |
| Soft-delete | Xoá mềm (vẫn khôi phục được trong 30 ngày) |
| Hard-delete | Xoá thật, không lấy lại được |
| TS error | Lỗi cú pháp |
| Frontend / Backend | Giao diện / Phía sau |

---

## ✅ FORMAT BÁO CÁO LETRI CHUẨN

```markdown
> **Việc đời thường**: 1-3 câu mô tả việc đã làm, dùng ví dụ đời sống

## ✅ Đã xong / 🚧 Đang làm
- Bullet ngắn

## ⚠️ Anh cần làm gì
| Anh gõ | Em làm |
|--------|--------|
| **"1"** | ... |
| **"2"** | ... |
```

---

## ❌ ANTI-PATTERN — VI PHẠM NẾU...

| Sai | Đúng |
|-----|------|
| "Approve migration 0042 with FK CASCADE on database_id?" | "Em xoá CSDL cũ → có xoá hết hàng theo không? 1) xoá hết 2) giữ hàng" |
| "Endpoint POST /api/csdl/:id/export trả 200, file .sql.gz 548KB" | "Em đã làm nút Tải backup. Anh bấm là file về máy." |
| "Worker A uncommitted WIP — gõ [A-COMMIT] [A-DEAD] [A-SKIP]" | "Thợ 1 code 880 dòng chưa lưu — dễ mất. 1) bảo lưu ngay 2) bỏ làm lại" |
| "INCIDENT-BUNDLE-FINAL pnpm workspace --filter break" | "Hôm qua đẩy code mới bị lỗi, phải khôi phục bản cũ" |

---

## 🧷 KIỂM TRA TRƯỚC KHI GỬI MESSAGE LETRI

Hỏi 5 câu trước khi bấm enter:

1. ✅ Có dùng từ trong cột "CẤM" không?
2. ✅ Có ví dụ đời thường (nhà hàng/kho/nhân viên) không?
3. ✅ Có bảng "Anh gõ X em làm Y" cuối không?
4. ✅ Tổng ≤ 15 dòng cho Letri bận không?
5. ✅ Letri quyết được trong 30 giây không?

Trả lời "không" 1 câu → viết lại.

---

*Mandate này áp dụng vĩnh viễn cho mọi CEO C1-C11 + Opus orchestrator. Vi phạm = Opus nhắc + ghi feedback memory.*
