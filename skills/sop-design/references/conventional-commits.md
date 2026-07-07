# Conventional Commits：提交规范 & Task 粒度

## 是什么

Conventional Commits 是标准化的 Git 提交信息格式。在 SOP Design 中，一个 Task = 一个 Conventional Commit = 一次原子化交付。

**规范来源：** [conventionalcommits.org](https://www.conventionalcommits.org/) v1.0.0

---

## 基本格式

```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

### Type（类型）

| Type | 含义 | 使用场景 |
|------|------|----------|
| `feat` | 新功能 | Task 交付新功能时 |
| `fix` | Bug 修复 | 修复 spec 中定义的验收标准不通过的问题 |
| `refactor` | 重构 | 不改功能，只优化代码结构 |
| `test` | 测试 | 添加或修改测试（测试开关开启时） |
| `docs` | 文档 | Spec 更新、README 更新 |
| `style` | 格式 | 空格、分号、格式化，不影响代码逻辑 |
| `chore` | 杂项 | 依赖更新、构建配置、CI 配置 |
| `perf` | 性能优化 | 提高性能的代码变更 |

### Scope（范围，可选）

React 项目示例：`feat(SearchBar): add debounced input handling`
Python 项目示例：`feat(items): add POST /api/items endpoint`

### Description（描述）

- 用祈使句（imperative mood）：`add` 不是 `added`；`fix` 不是 `fixed`
- 不超过 72 个字符
- 不以句号结尾
- 中文/英文均可，保持全项目一致

---

## 示例

### React 前端

```bash
git commit -m "feat(SearchBar): add debounced input with 300ms delay"
git commit -m "fix(ResultList): fix empty state not showing on zero results"
git commit -m "refactor(useSearch): extract API call logic into separate hook"
git commit -m "test(SearchBar): add integration test for search flow"
git commit -m "style(MainPanel): format with Prettier"
```

### Python/Go 后端

```bash
git commit -m "feat(items): add POST /api/items endpoint"
git commit -m "fix(auth): fix 401 on expired token refresh"
git commit -m "refactor(repository): extract pagination logic to base class"
git commit -m "perf(query): add index on items.created_at for faster listing"
git commit -m "chore(deps): update fastapi to 0.110.0"
```

---

## 与 Task 的对应关系

```
Implementation Spec 中的一个 Task
    │
    ├── 开发完成 → 所有 Checklist 项打勾
    │
    ├── Harness 审查通过（Spec 一致性 + 架构质量）
    │
    └── 一个 Conventional Commit
         │
         └── feat(xxx): [Task 标题的简短描述]
```

### Task 粒度标准

| 维度 | 标准 |
|------|------|
| 代码量 | 50-150 行 |
| 时间 | 30 分钟内可完成 |
| 可验证 | 至少对应一个 Spec 中的验收标准 (AC) |
| 可提交 | 可以独立 commit，不依赖未完成的代码 |
| 可 revert | 出问题可以单独回滚，不影响其他功能 |

### ❌ 巨型 Task（拒绝）

```markdown
### Task 1: 实现用户管理模块
- 包含登录、注册、修改密码、找回密码、个人资料编辑、头像上传
```

### ✅ 合理 Task

```markdown
### Task 1: 用户注册表单 UI → feat(auth): add register form component
### Task 2: 注册 API 对接 → feat(auth): connect register form to POST /auth/register
### Task 3: 注册表单验证 → feat(auth): add client-side validation for register form
### Task 4: 登录表单 UI → feat(auth): add login form component
### Task 5: 登录 API 对接 → feat(auth): connect login form to POST /auth/login
```

---

## Footer（页脚，可选）

```bash
git commit -m "feat(items): add POST /api/items endpoint

Closes #42
Refs: US-01, AC-01"
```

Footer 中可引用：
- `Closes #issue-number` — 关闭 issue
- `Refs: US-01, AC-01` — 关联 spec 中的用户故事和验收标准
- `BREAKING CHANGE:` — 不兼容变更说明

---

## 自动化

### Git Hook（推荐，非强制）

在项目中配置 commit-msg hook 校验格式：

```bash
# .git/hooks/commit-msg
#!/bin/sh
commit_regex='^(feat|fix|refactor|test|docs|style|chore|perf)(\(.+\))?: .{1,72}$'
if ! grep -qE "$commit_regex" "$1"; then
    echo "Commit message must match Conventional Commits format."
    echo "Example: feat(auth): add login form component"
    exit 1
fi
```

### 配合 standard-version 自动发版

```bash
npx standard-version  # 根据 commit 历史自动 bump 版本号 + 生成 CHANGELOG
```

---

## 常见错误

| 错误 | 正确 |
|------|------|
| `feat: added login` (过去式) | `feat(auth): add login form` (祈使句) |
| `fix bug` (没有 type) | `fix(search): handle empty query parameter` |
| 一个 commit 包含 3 个不相关的 Task | 一个 Task 一个 commit，不混在一起 |
| `feat: 修改了一些东西` (描述不具体) | `feat(profile): add avatar upload with crop` |
