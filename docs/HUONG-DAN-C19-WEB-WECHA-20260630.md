# 🌐 HƯỚNG DẪN C19 — WEBSITE TRÀ "web-wecha" (Letri duyệt 30/6/2026)

> **Mô hình chốt:** Cách A — **Postgres RIÊNG**, cô lập tuyệt đối khỏi ERP. C19 chỉ làm website, **CẤM đụng ERP / hệ khác / CẤM deploy**.

---

## 0. 🔒 RÀNG BUỘC TỐI THƯỢNG (khoá cứng — đọc trước)
1. **C19 CHỈ được:** code website web-wecha + dùng database riêng của web-wecha.
2. **TUYỆT ĐỐI CẤM:** chạm `pos_db` (ERP cổng 5433), `phache_pos_db` (POS V4), Postgres cổng 5432 (n8n/Docker), bất kỳ folder/process hệ khác trên VPS.
3. **CẤM C19 (và mọi C) tự deploy.** Code xong → nộp **CEO ERP review** → **Letri duyệt** → CEO ERP deploy bằng **script riêng của web-wecha**.
4. KHÔNG dùng chung 1 bảng nào với ERP. Web hỏng ≠ ERP hỏng.

---

## 1. Hạ tầng (CEO ERP set 1 lần, C19 KHÔNG tự làm trên VPS)
- **Database riêng:** Postgres RIÊNG cho web-wecha (container Docker mới, cổng riêng vd **5434** — KHÔNG phải 5433/5432). DB tên `web_wecha`, user/mật khẩu RIÊNG (C19 không có credential `pos_db`).
- **Code riêng:** folder `/opt/web-wecha` (tách hẳn `/opt/pos-system`).
- **Domain:** wecha.vn (hoặc subdomain) → reverse proxy Caddy trỏ riêng port web-wecha (CEO ERP cấu hình, không sửa block ERP).
- C19 phát triển **LOCAL trên máy** với Postgres local riêng → push code → CEO ERP deploy lên hạ tầng trên.

## 2. Phạm vi C19 code (website trà)
- Schema RIÊNG cho web (sản phẩm trà hiển thị, bài viết, đơn hàng web, liên hệ...) trong DB `web_wecha`.
- Theo charter C19 đã chốt (full-stack: admin CMS + e-commerce + ...). Mọi bảng nằm trong `web_wecha`, KHÔNG tạo bảng trong `pos_db`.

## 3. Liên kết với ERP (nếu website cần data sản phẩm/giá/tồn từ ERP)
> ⏳ **CHỜ Letri chốt:** website đứng độc lập hoàn toàn, hay cần lấy data từ ERP?
- **NẾU cần:** chỉ qua **API 1 CHIỀU, READ-ONLY** — website GỌI API ERP (hoặc ERP đẩy sang web định kỳ). CEO ERP thiết kế endpoint riêng (read-only, có token). **CẤM** web nối thẳng `pos_db`.
- **NẾU không:** web-wecha tự quản data sản phẩm riêng, không động ERP.

## 4. Quy trình deploy web-wecha (khoá cứng 2 cổng)
1. C19 code + test LOCAL kỹ (build PASS) → push lên repo/branch web-wecha.
2. **CEO ERP review** (bảo mật + không rò rỉ + đúng phạm vi web).
3. **Letri duyệt** (cổng cuối).
4. CEO ERP chạy **`safe-deploy-web-wecha.sh`** RIÊNG (target `/opt/web-wecha`, cổng web-wecha) — **KHÔNG đụng** `safe-deploy.sh` của ERP.
5. Verify web-wecha health riêng. ERP không bị ảnh hưởng (khác process, khác DB, khác script).

## 5. Việc CEO ERP làm trước khi C19 bắt đầu
- [ ] Dựng Postgres container web-wecha (cổng riêng) + DB `web_wecha` + user riêng.
- [ ] Tạo folder `/opt/web-wecha` + Caddy block riêng.
- [ ] Viết `safe-deploy-web-wecha.sh` (2 cổng CEO ERP + Letri, target riêng).
- [ ] Cấp C19 thông tin kết nối DB `web_wecha` (KHÔNG cấp `pos_db`).

---
*Trình Letri duyệt. Sau khi anh chốt mục 3 (có/không liên kết ERP), CEO ERP set hạ tầng → C19 bắt đầu code (chỉ phạm vi web). Mọi deploy qua CEO ERP + Letri.*
