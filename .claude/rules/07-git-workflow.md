# Rule 07 — Git Workflow & Release

> Mức độ: 🟠 HIGH | Áp dụng: Branch, commit, PR, release

---

## 1. Branch strategy (Trunk-based + protected)

```
main          ← Production, protected, deploy auto on tag
  ↑
develop       ← Staging, deploy auto on merge
  ↑
feature/<id>-<n>   ← Feature, PR vào develop
fix/<id>-<n>       ← Bug fix
hotfix/<n>         ← Khẩn cấp, PR vào main + develop
chore/<n>          ← Dependencies, config, docs
release/v1.5       ← Release branch (optional)
```

---

## 2. Commit convention (Conventional Commits 1.0)

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types:**

| Type | Khi nào |
|---|---|
| `feat` | Tính năng mới |
| `fix` | Sửa bug |
| `perf` | Cải thiện performance |
| `refactor` | Refactor (không đổi behavior) |
| `style` | Format code |
| `test` | Thêm/sửa test |
| `docs` | Cập nhật docs |
| `build` | Build system, dependencies |
| `ci` | CI config |
| `chore` | Việc lặt vặt |
| `security` | Sửa lỗi bảo mật |
| `revert` | Revert commit |

**Subject rules:**
- Tiếng Việt OK
- Imperative: "thêm" không phải "đã thêm"
- ≤ 72 ký tự
- Không dấu chấm cuối
- Lowercase đầu (trừ proper noun)

**Ví dụ:**
```
feat(orders): thêm chức năng huỷ đơn có lý do

- Endpoint POST /api/v1/orders/:id/cancel
- Yêu cầu reason >= 10 ký tự
- Audit log mọi lần huỷ
- UI: modal xác nhận + textarea lý do

Lý do: Kế toán cần biết lý do huỷ để báo cáo cuối tháng.

Closes #ERP-234
```

---

## 3. PR description template (BẮT BUỘC)

```markdown
## 🎯 Vấn đề
<Mô tả problem statement>

## ✨ Giải pháp
<Approach + key changes>

## 📸 Screenshots / Demo
<Trước/Sau, hoặc video demo>

## 🧪 Cách test thủ công
1. ...
2. ...

## ✅ Checklist
- [ ] Code theo CLAUDE.md rules
- [ ] Test unit + integration coverage > 80%
- [ ] Test security (cross-tenant, IDOR, unauthorized)
- [ ] i18n cho text mới (vi + en)
- [ ] Update API docs (Swagger)
- [ ] Migration reversible (có .down.sql)
- [ ] Mobile-compatible (test với Postman header thường)
- [ ] Audit log cho action nhạy cảm
- [ ] Đã review checklist trong .claude/skills/code-review/SKILL.md
- [ ] Không có console.log debug
- [ ] Không có TODO/FIXME thiếu owner+date
- [ ] Update CHANGELOG.md
- [ ] ADR cho quyết định kiến trúc lớn

## 🔗 Related
- Closes #ERP-123
- Spec: <link>
- Threat model: docs/threat-models/...md (nếu có)

## ⚠️ Risk & Rollback
- Rủi ro: ...
- Rollback plan: ...
```

**PR size:**
- ✅ Ideal: < 400 dòng diff
- ⚠️ Cần justify: 400-800 dòng
- 🔴 Require split: > 800 dòng

**Review process:**
1. Self-review trước khi request
2. CI pass: lint + type + test + build + security scan
3. ≥ 1 approver (≥ 2 cho code security/payment/AI agent)
4. Resolve all conversations
5. Squash merge vào develop
6. Rebase merge develop → main khi release

---

## 4. Hotfix flow

```bash
git checkout main && git pull
git checkout -b hotfix/payment-double-charge
# Fix + test
# PR vào main (priority review)
# Sau khi merge main → trigger deploy production ngay

git checkout develop
git merge main
git push
```

---

## 5. Release flow (Semantic Versioning)

```bash
pnpm version minor    # 1.4.0 → 1.5.0
pnpm version patch    # 1.4.0 → 1.4.1
pnpm version major    # 1.4.0 → 2.0.0

# Update CHANGELOG.md (auto qua conventional-changelog)
git push --follow-tags

# CI tự deploy production khi tag
gh release create v1.5.0 --notes-file CHANGELOG.md
```

---

## 6. CHANGELOG.md format

```markdown
# Changelog

## [1.5.0] - 2026-04-26

### Added
- Module quản lý nhượng quyền WECHA (#123)
- 2FA bắt buộc cho admin (#456)

### Changed
- Nâng pgvector 0.5 → 0.7 (#234)

### Deprecated
- Endpoint `/api/v1/legacy-orders` sẽ remove ở v2.0 (sunset 2027-01-01)

### Fixed
- JWT không refresh khi sắp hết hạn (#234)
- Race condition khi nhiều POS cùng giảm tồn kho (#567)

### Security
- Sửa CORS cho phép origin không whitelist (CVE-2026-XXXX) (#999)

### Performance
- Index `(tenant_id, status, created_at)` cho orders, giảm 80% query time (#777)
```

---

## 7. CẤM tuyệt đối 🔴

- Force push lên `main`, `develop`, shared branch
- Commit `node_modules`, `.env`, file build
- Commit secret/key (chạy `gitleaks` trong pre-commit)
- Commit code chưa pass test
- Merge PR chưa có review
- Skip CI bằng `--no-verify` (trừ trường hợp đặc biệt + báo team)
- Rebase commit của người khác trên shared branch
- Amend commit đã push lên shared branch

---

## 8. Tools setup

```bash
pnpm add -D husky lint-staged commitlint @commitlint/cli @commitlint/config-conventional gitleaks
npx husky init
```

```bash
# .husky/pre-commit
#!/bin/sh
pnpm lint-staged
gitleaks detect --staged --no-banner

# .husky/commit-msg
#!/bin/sh
npx commitlint --edit $1
```

```json
// package.json
{
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md,yml}": ["prettier --write"]
  }
}
```

```js
// commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'subject-case': [2, 'never', ['upper-case']],
    'header-max-length': [2, 'always', 100],
  },
};
```

---

**END Rule 07**
