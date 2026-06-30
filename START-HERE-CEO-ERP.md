# 🟢 KHỞI ĐỘNG — CEO ERP (đọc ĐẦU TIÊN, mọi session desktop + mobile)

> Bạn tiếp tục phát triển hệ ERP bán hàng Passion Link / Vua An Toàn. Code liền mạch trên hệ HIỆN TẠI — không tạo bản sao, không paste rời.

## ⛔ ĐỌC TRƯỚC THEO THỨ TỰ
1. **`docs/PROGRESS.md` mục 7 "📋 BÀN GIAO CHO SESSION SAU"** — entry mới nhất = SOURCE OF TRUTH (việc đang treo + bước tiếp + cạm bẫy).
2. `docs/ARCHITECTURE_CONTRACT.md` — HIẾN PHÁP 10 luật.
3. `.claude/rules/` (00→20) — luật code. `.claude/skills/` — chuyên môn.

## 🎯 NGUỒN ĐÚNG (đừng nhìn nhầm)
- **Code deployed:** repo `erp-wecha-pos`, **branch `n8n/005-wecha-pos-zalo-keypos`** (trunk, sửa hằng ngày). Local: `claude/pos-system`.
- ⛔ **KHÔNG audit `main`** (cũ/chết). ⛔ **KHÔNG đọc `orchestrator-ceo/` C1–C18** (kế hoạch LỊCH SỬ, KHÔNG phải code deployed).
- Clone đúng: `git clone -b n8n/005-wecha-pos-zalo-keypos git@github.com:lehuutri28/erp-wecha-pos.git`

## 🔴 LUẬT VÀNG (Letri nhấn nhiều lần)
- **KIỂM THẬT KỸ — KHÔNG ĐOÁN BỪA.** Số liệu lấy từ **query DB prod thật** (port 5433), KHÔNG đếm file / đọc doc cũ (RLS đếm bảng `rowsecurity=true`, migration applied check DB — không đếm số cao nhất).
- **VERIFY-EXISTING-FIRST**: trước khi code tính năng (bulk/filter/endpoint) → grep xem ĐÃ CÓ chưa, REUSE đừng tạo trùng (vd customers đã có 9 bulk endpoint).
- **Tiền là nhạy cảm**: bug tính tiền (giảm giá %/VND) phải reviewer 3 vòng + test.

## 🚀 DEPLOY (đúng quy trình)
1. Build LOCAL PASS (api tsc + web build) TRƯỚC.
2. **Apply migration port 5433 TRƯỚC deploy** (`ssh ... psql -p 5433 pos_db < mig.sql`). KHÔNG safe-deploy tự chạy migration.
3. `cd erp-project && bash scripts/safe-deploy.sh n8n/005-wecha-pos-zalo-keypos` → gõ YES DEPLOY.
4. Verify prod (health 200 + marker dist + DB) → tag + push → cập nhật PROGRESS → giao C8.

## 🔒 BẢO MẬT
- KHÔNG để lộ secret lên GitHub. Repo PHẢI private. KHÔNG commit pass DB/IP/token (scrub trước).

## 🔚 CUỐI SESSION
Cập nhật `docs/PROGRESS.md` mục 7 (5 dòng: vừa làm/dừng tại file:dòng/bước tiếp/cạm bẫy/hệ khác) + push.
