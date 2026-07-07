# SOP Design Skill — 设计总结

## 定位

混合型 skill：给人看（流程指导），也给 AI 用（行为约束）。核心是将 Spec + Harness 双支柱开发流程工程化落地。

## 技术栈

- 前端：React
- 后端：Python / Go

## 双支柱架构

### Spec 支柱（文档先行）
需求规范 → 设计规范 → 实现规范 → Task 拆解 → 代码生成

- 用 BDD/Gherkin（Given-When-Then）写验收标准
- 三层规范层层递进，不可跳过
- 每个 Task 一次 Conventional Commit

### Harness 支柱（质量保障）
TDD → 代码审查 → 重构 → AGENTS.md 维护

- TDD：Red（写失败测试）→ Green（最小实现）→ Refactor（不改功能只优化）
- 代码审查：AI 检查 spec 一致性和架构合理性
- 重构：人工引导方向，AI 执行，定期进行
- AGENTS.md：三次原则写入 + 大小监控 + 按需压缩

## 五大风险警告（必须在 skill 和生成的 spec 中重点标记）

| # | 风险 | 来源 | 应对策略 |
|---|------|------|----------|
| 1 | **AI 写测试 → 问题累积** | 测试用例用 AI 写，问题越来越多 | 测试开关机制，默认关闭，仅关键路径开启 |
| 2 | **AGENTS.md 过大 = 无规则** | 太大等于没有规则，AI 容易跑偏 | 三次原则写入 + 大小监控 + 按需压缩 + 按需加载 |
| 3 | **先代码后文档 = 浪费时间** | 直接写代码基本就是浪费时间 | Spec→Plan→Task→Code 绝对顺序，写入 AGENTS 强制执行 |
| 4 | **巨型 Task → 难以 review** | Task 太粗，不好推进和审查 | Task 拆细，每个 Task 一次提交，改对就提交 |
| 5 | **功能都对/架构全错** | 代码能跑但无法维护 | Harness 审查必须核对 spec 一致性，不只功能正确；定期重构 |

## 集成业界规范

1. **BDD / Gherkin** — 验收标准用 Given-When-Then 表达，直接对应 Harness 自动化测试
2. **Conventional Commits** — `feat:` `fix:` `refactor:` `test:` 标准化提交，一个 Task 一次提交
3. **Testing Trophy / Testing Pyramid** — 前端 React 以集成测试为重（Testing Trophy），后端以单元测试为重（Testing Pyramid），配测试开关

## 关键设计决策

- **AGENTS.md 三次原则**：同一问题至少出现 3 次才写入，避免膨胀
- **AGENTS.md 按需加载**：拆分为多个小文件，按场景引用
- **测试开关**：默认关闭（`test: false`），保护型/关键路径手动开启（`test: true`）
- **Spec 中必须包含风险声明**：每个生成的 spec 文档必须引用五大风险并标记当前项目的风险等级
- **绝对顺序不可逆**：Spec → Plan → Task → Code，AI 不遵循时 Harness 拒绝通过
- **先讨论再生成**：涉及钱的项目、关键功能，必须先头脑风暴对话确定细节直到满意

## 文件结构

```
sop-design/
├── SKILL.md                          # 主路由 + 核心铁律 + 风险警告（必读）
├── references/
│   ├── spec-pillar.md                # Spec 支柱：三层规范写法详解
│   ├── harness-pillar.md             # Harness 支柱：TDD/审查/重构/AGENTS
│   ├── bdd-gherkin.md                # BDD Given-When-Then 验收标准
│   ├── conventional-commits.md       # 提交规范 + Task 粒度标准
│   └── testing-trophy.md            # 测试分层 + 测试开关机制
```

## 风险声明模板（每个生成的 spec 必须包含）

```markdown
## 🚨 风险警告

> 以下风险来源于 AI 辅助开发的最佳实践总结。请在开发过程中持续关注。

| 风险 | 等级 | 本项目应对 |
|------|------|-----------|
| AI 写测试 → 问题累积 | 🔴 高/🟡 中/🟢 低 | [当前项目的测试开关状态和策略] |
| AGENTS.md 膨胀失效 | 🔴 高/🟡 中/🟢 低 | [AGENTS.md 当前大小和压缩计划] |
| 文档与代码脱节 | 🔴 高/🟡 中/🟢 低 | [如何保证 spec 先行] |
| Task 粒度过大 | 🔴 高/🟡 中/🟢 低 | [Task 拆解标准] |
| 架构偏离 spec | 🔴 高/🟡 中/🟢 低 | [Harness 审查频率和方式] |
```
