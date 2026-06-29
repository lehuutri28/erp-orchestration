# Rule 15 — Compliance Việt Nam

> ⚠️ Đồng bộ ARCHITECTURE_CONTRACT 29/5/2026: PK=uuid, tenant_id=uuid, TS field camelCase, RBAC dấu `.`. Contract THẮNG nếu mâu thuẫn. Xem docs/ARCHITECTURE_CONTRACT.md.

> Mức độ: 🔴 CRITICAL | Áp dụng: PII, kế toán, hoá đơn, payment

---

## 1. NĐ 13/2023/NĐ-CP — Bảo vệ Dữ liệu Cá nhân (PDPL)

**Hiệu lực:** 01/07/2023. **Phạt:** 50-100M VND/vi phạm + đình chỉ.

### Phân loại dữ liệu

**Dữ liệu cá nhân cơ bản:**
- Họ tên, ngày sinh, giới tính
- Nơi sinh, nơi đăng ký khai sinh
- Quốc tịch, hình ảnh
- Số điện thoại, email
- Số CMND/CCCD/passport
- Tình trạng hôn nhân
- Thông tin về mối quan hệ gia đình

**Dữ liệu cá nhân nhạy cảm:** (cần consent rõ ràng + có DPA)
- Quan điểm chính trị, tôn giáo
- Tình trạng sức khoẻ
- Đời sống tình dục, xu hướng tính dục
- Dữ liệu sinh trắc học (vân tay, khuôn mặt)
- Vị trí cá nhân
- Thông tin tài chính cá nhân
- Số thẻ tín dụng

### Yêu cầu hệ thống

**1. Đăng ký xử lý PII với Bộ CA** (BẮT BUỘC):
- Trước khi process PII của ≥ 10000 khách
- Hồ sơ xử lý dữ liệu (DPA - Data Processing Agreement)
- Đánh giá tác động (DPIA - Data Protection Impact Assessment)

**2. Consent (Đồng ý xử lý):**
```typescript
// schema/consent_records.ts
export const consentRecords = pgTable('consent_records', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull(),
  tenantId: uuid('tenant_id').notNull(),
  consentType: text('consent_type').notNull(),  // 'marketing', 'cookie', 'data_processing'
  purpose: text('purpose').notNull(),             // Mục đích cụ thể
  granted: boolean('granted').notNull(),
  ip: text('ip').notNull(),
  userAgent: text('user_agent').notNull(),
  policyVersion: text('policy_version').notNull(),
  grantedAt: timestamp('granted_at').notNull(),
  revokedAt: timestamp('revoked_at'),
}, (t) => ({
  userIdx: index('consent_user_idx').on(t.userId, t.consentType),
}));
```

**3. Quyền của chủ thể dữ liệu:**

| Quyền | Endpoint | SLA |
|---|---|---|
| Xem dữ liệu (Right to access) | `GET /api/v1/me/data` | 72h |
| Sửa dữ liệu (Right to rectification) | `PATCH /api/v1/me/profile` | Realtime |
| Xoá dữ liệu (Right to be forgotten) | `DELETE /api/v1/me/account` | 30 ngày |
| Hạn chế xử lý | `POST /api/v1/me/restrict-processing` | 72h |
| Phản đối | `POST /api/v1/me/object` | 72h |
| Di chuyển (Data portability) | `POST /api/v1/me/export` | 30 ngày |
| Rút consent | `POST /api/v1/me/revoke-consent` | Realtime |

**4. Mã hoá dữ liệu nhạy cảm:**
- CCCD, CMND, passport — pgcrypto AES-256 + SHA-256 hash để search
- Lương — column-level encryption, chỉ HR admin decrypt
- Thông tin sức khoẻ (nếu có) — encrypt + audit mọi access

**5. Audit log access PII** (BẮT BUỘC):
```typescript
@PiiAccess('customer.cccd')  // Custom decorator
async getCustomerCccd(id: string, user: AuthUser) {
  await this.auditService.log({
    actor_id: user.id,
    action: 'pii.read',
    resource_type: 'customer',
    resource_id: id,
    field: 'cccd',
    reason: req.body.reason,  // Bắt buộc
  });
  
  // ...decrypt and return
}
```

**6. Data retention:**
- Khách hàng inactive 24 tháng → notify + auto-delete sau 6 tháng
- Marketing consent rút → xoá trong 30 ngày
- Audit log PII access — 7 năm

**7. DPO (Data Protection Officer):**
- Bắt buộc khi process PII của ≥ 10000 khách
- Email: dpo@wecha.vn (BẮT BUỘC publish trên website)

---

## 2. TT 99/2025/TT-BTC — Chế độ Kế toán Doanh nghiệp

> 📎 **Cross-ref:** Bảng retention chi tiết theo từng loại chứng từ + implementation xem [skills/accounting-tt99/SKILL.md](../skills/accounting-tt99/SKILL.md#retention)

**Hiệu lực:** 01/01/2026 (thay thế hoàn toàn TT 200/2014/TT-BTC).

### Yêu cầu hệ thống

**1. Hệ thống tài khoản kế toán** theo Phụ lục II của TT 99 — tự thiết kế nhưng phải đảm bảo:
- Phân loại đúng bản chất nghiệp vụ
- Không trùng lặp đối tượng
- Không thay đổi chỉ tiêu trên BCTC

**2. Báo cáo tài chính** (đổi tên từ TT200):
- **Báo cáo tình hình tài chính** (thay Bảng cân đối kế toán)
- **Báo cáo kết quả hoạt động toàn diện** (thay Báo cáo KQHĐ)
- **Báo cáo lưu chuyển tiền tệ** (giữ nguyên)
- **Bản thuyết minh BCTC** (giữ nguyên)

**3. Đơn vị tiền tệ:**
- Mặc định: VND
- Nếu không VND → phải chuyển đổi BCTC sang VND khi nộp

**4. Lưu trữ chứng từ kế toán:**

| Loại | Thời gian lưu |
|---|---|
| Chứng từ tài chính (hoá đơn, phiếu thu/chi) | **10 năm** |
| Sổ kế toán | **10 năm** |
| Báo cáo tài chính năm | **Vĩnh viễn** |
| Báo cáo quản trị | **5 năm** |

**Áp dụng cho ERP:**
```typescript
// schema/accounting_documents.ts — bảng append-only, không UPDATE/DELETE
// → Soft delete cấp A (Letri chốt 29/5): xoá qua duyệt + thùng rác + khôi phục, NHƯNG purge_after=NULL (giữ 10 năm TT99). RLS vẫn chặn hard-delete. Xem CONTRACT mục 1.
export const accountingDocuments = pgTable('accounting_documents', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  documentType: text('document_type').notNull(), // 'invoice', 'receipt', 'voucher'
  documentNumber: text('document_number').notNull(),
  date: date('date').notNull(),
  
  amount: decimal('amount', { precision: 15, scale: 2 }).notNull(),
  currency: text('currency').notNull().default('VND'),
  vatAmount: decimal('vat_amount', { precision: 15, scale: 2 }),
  
  accountDebit: text('account_debit').notNull(),   // TK Nợ (theo TT99 phụ lục II)
  accountCredit: text('account_credit').notNull(), // TK Có
  
  description: text('description').notNull(),
  referenceId: text('reference_id'),  // FK đến bảng nghiệp vụ
  referenceType: text('reference_type'),
  
  createdAt: timestamp('created_at').notNull().defaultNow(),
  createdBy: uuid('created_by').notNull(),
  
  // KHÔNG có updated_at, deleted_at — append only
});

// Constraint: KHÔNG cho UPDATE/DELETE qua RLS
CREATE POLICY no_update_accounting ON accounting_documents
  FOR UPDATE USING (false);

CREATE POLICY no_delete_accounting ON accounting_documents
  FOR DELETE USING (false);
```

**5. Phần mềm kế toán** (Điều 28 TT99):
- Phải đảm bảo yêu cầu tối thiểu về chuyên môn
- Nếu dùng MISA AMIS — đã chuẩn TT99
- Nếu tự build — cần chứng nhận của BTC (phức tạp)
- **Khuyến nghị:** Dùng MISA AMIS, ERP chỉ làm operational + đẩy data sang AMIS

---

## 3. NĐ 70/2025/NĐ-CP + TT 32/2025/TT-BTC — Hoá đơn điện tử

**Hiệu lực:** Cập nhật quy định về HĐĐT — đặc biệt HĐĐT khởi tạo từ máy tính tiền (POS).

### Yêu cầu hệ thống POS

**1. Mọi giao dịch POS PHẢI xuất HĐĐT** trong vòng:
- Ngay tức khắc với người mua có nhu cầu lấy hoá đơn
- Cuối ngày — tổng hợp hoá đơn các giao dịch không yêu cầu

**2. Tích hợp với nhà cung cấp HĐĐT** đã được Tổng cục Thuế cấp phép:
- MISA meInvoice (khuyến nghị)
- Viettel Sinvoice
- VNPT Invoice
- ...

**3. Format hoá đơn từ máy tính tiền:**
```typescript
interface POSInvoice {
  // BẮT BUỘC theo NĐ 70/2025
  invoice_number: string;
  invoice_date: string;
  store_tax_code: string;        // MST của cửa hàng
  store_name: string;
  
  buyer_name?: string;            // Optional cho khách lẻ
  buyer_tax_code?: string;        // Optional, cần nếu xuất hoá đơn cho DN
  
  items: Array<{
    name: string;
    quantity: number;
    unit_price: number;
    total: number;
    vat_rate: number;             // 0, 5, 8, 10
    vat_amount: number;
  }>;
  
  subtotal: number;
  total_vat: number;
  total_amount: number;
  
  payment_method: 'cash' | 'card' | 'transfer' | 'momo' | 'vnpay' | 'qr';
  
  // Mã CQT trả về sau khi đăng ký
  cqt_code?: string;
}
```

**4. Lưu hoá đơn điện tử:**
- Lưu file gốc (XML format) **10 năm**
- Truy xuất theo: số hoá đơn, mã CQT, ngày, MST
- Backup R2 + tier archive

---

## 4. Luật An toàn Thông tin Mạng + Luật An ninh Mạng

**Yêu cầu:**
- Lưu trữ data trong VN (data residency)
  - DB chính ở VPS Việt Nam ✅
  - Backup R2: chọn region Asia-Singapore (gần VN, tốc độ nhanh) hoặc ưu tiên provider VN cho data nhạy cảm
- Báo cáo sự cố an toàn thông tin trong 24h
- Có DPO (Data Protection Officer)
- Có quy chế ATTT bằng văn bản

---

## 5. Luật Quảng cáo (NĐ 38/2021/NĐ-CP)

**Áp dụng cho marketing module:**
- Email marketing — phải có nút unsubscribe rõ ràng
- SMS quảng cáo — phải có "TC <số tin nhắn>" để từ chối
- Zalo OA — không spam
- Pop-up quảng cáo — phải có nút đóng

---

## 6. Luật Bảo vệ Người tiêu dùng

**Áp dụng cho POS + e-commerce:**
- Phải có chính sách đổi trả trong 30 ngày
- Phải hiển thị giá đầy đủ (đã bao gồm VAT)
- Phải có thông tin pháp nhân + địa chỉ + hotline
- Khiếu nại phải xử lý trong 7 ngày

---

## 7. Compliance checklist (BẮT BUỘC review trước go-live)

**🔴 Critical:**
- [ ] Đăng ký xử lý PII với Bộ CA (nếu > 10000 khách)
- [ ] Có DPA + DPIA written
- [ ] Privacy policy trên website (vi + en)
- [ ] Cookie banner consent
- [ ] Endpoint export data (JSON + PDF)
- [ ] Endpoint xoá account (cascade soft-delete)
- [ ] Hạch toán theo TT 99/2025 hoặc tích hợp MISA AMIS
- [ ] Bảng accounting_documents append-only
- [ ] Tích hợp HĐĐT MISA meInvoice (NĐ 70/2025)
- [ ] DB primary tại VN
- [ ] Email DPO publish trên website
- [ ] Audit log PII access (retention 7 năm)
- [ ] Audit log financial (retention 10 năm)
- [ ] Backup tier archive 10 năm cho chứng từ kế toán

**🟠 High:**
- [ ] Encryption CCCD/CMND
- [ ] Marketing consent + unsubscribe
- [ ] Chính sách đổi trả
- [ ] Quy chế ATTT bằng văn bản
- [ ] Đào tạo nhân viên về PII

---

**END Rule 15**
