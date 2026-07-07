---
name: sop-design
description: Use when starting a new feature or project, need to establish a spec→plan→task→code workflow, want AI to follow documentation-before-code discipline, or need to set up automated quality gates (Harness) for AI-generated code. Also use when AGENTS.md is growing too large, AI is drifting from spec, or you need to integrate BDD acceptance criteria into your development process. Applies to React frontend + Python/Go backend stacks.
---

# SOP Design — Spec + Harness 双支柱开发流程

## Overview

将 AI 辅助开发工程化为两个支柱：**Spec（文档先行，定义"做什么"）** 和 **Harness（质量保障，验证"做对了没"）**。核心理念：让 AI 自己写 spec、自己拆 task、自己写代码，Harness 自动化验收——人只做 review 和方向引导。

**技术栈：** React 前端 + Python/Go 后端

---

## 🚨 五大风险警告

> **以下风险必须在每个生成的 spec 文档中引用并评估等级。忽视任何一项都将导致项目失控。**

| # | 风险 | 根因 | 应对 |
|---|------|------|------|
| 🔴 **1** | **AI 写测试 → 问题累积** | 测试用例用 AI 写，问题越来越多，目前技术无法根治 | 测试开关默认 **关闭**，仅关键路径手动开启。见 `references/testing-trophy.md` |
| 🔴 **2** | **AGENTS.md 过大 = 无规则** | 规则越多，AI 越容易跑偏，等同于没有规则 | 三次原则写入 + 大小监控（>500 行必须压缩）+ 按需加载。见 `references/harness-pillar.md` |
| 🔴 **3** | **先代码后文档 = 浪费时间** | 直接让 AI 写代码，功能可能对但架构全错，无法维护 | Spec→Plan→Task→Code 绝对顺序，写入 AGENTS 强制执行。违反时 Harness 拒绝通过 |
| 🟡 **4** | **巨型 Task → 难以 review** | Task 太粗，一次生成太多代码，review 困难，改错成本高 | 每个 Task 应在 50-150 行代码量，一个 Task 一次 Conventional Commit |
| 🟡 **5** | **功能都对/架构全错** | AI 生成的代码能跑，但不符合架构设计，维护成本飙升 | Harness 审查必须核对 spec 一致性（不只是功能正确）；定期触发架构重构 |

---

## 核心铁律（写入 AGENTS.md）

```markdown
1. 文档先行：Spec → Plan → Task → Code，不可逆。任何代码生成前必须先有对应 spec。
2. 三次原则：同一问题/模式出现 ≥3 次，才写入 AGENTS.md。杜绝一次教训就上规则。
3. 测试开关：默认 test: false。仅在关键路径/回归保护时手动开启。
4. 即改即提：一个 Task 完成 → 一个 Conventional Commit。不要把对的改错了。
5. 不写自然语言：生成代码时只输出代码，不在代码中夹杂解释性自然语言。
```

---

## 模块路由

先判断用户意图，再加载对应 reference：

| 用户说什么 | 激活模块 | 加载文件 |
|-----------|----------|----------|
| "新项目""新功能""写个 spec""需求文档""技术方案" | **Spec 支柱** | `references/spec-pillar.md` + `references/bdd-gherkin.md` |
| "审查""测试""重构""AGENTS 太大了""加个规则" | **Harness 支柱** | `references/harness-pillar.md` + `references/testing-trophy.md` |
| "提交""commit""task 拆完了" | **交付节奏** | `references/conventional-commits.md` |
| 不确定 | 先问：要写 spec 定义"做什么"，还是用 Harness 验证"做对了没"？ | — |

---

## 双支柱总览

```
用户需求
    │
    ├── Spec 支柱（文档先行）──────────────────────┐
    │   Requirements Spec ──▶ Design Spec           │
    │   (BDD/Gherkin)         (C4/架构图)           │
    │        │                     │                │
    │        └──────┬──────────────┘                │
    │               ▼                               │
    │       Implementation Spec                     │
    │       (Task 拆解 + 依赖 + Checklist)           │
    │               │                               │
    │               ▼                               │
    │         AI 生成代码                            │
    │               │                               │
    └───────────────┼───────────────────────────────┘
                    ▼
    ┌── Harness 支柱（质量保障）────────────────────┐
    │                                               │
    │   TDD ──▶ Code Review ──▶ Refactor            │
    │   (开关)   (Spec一致性)    (人工引导)           │
    │                                               │
    │   AGENTS.md 维护（三次原则 + 压缩）             │
    │                                               │
    │   反馈 ──▶ 更新 Spec（形成闭环）               │
    └───────────────────────────────────────────────┘
```

---

## 快速参考

### Spec 支柱核心

| 层级 | 内容 | 格式 | 谁写 |
|------|------|------|------|
| Requirements Spec | 用户故事、验收标准、业务逻辑、约束 | BDD/Gherkin (Given-When-Then) | AI 生成 + 人 review |
| Design Spec | 技术架构、组件边界、数据流、集成模式 | C4 模型 + 文字描述 | AI 生成 + 人 review |
| Implementation Spec | Task 拆解、依赖关系、实施计划、测试方案 | Task 清单 + Checklist | AI 生成 + 人 review |

### Harness 支柱核心

| 模块 | 做什么 | 频率 | 谁主导 |
|------|--------|------|--------|
| TDD | Red（失败测试）→ Green（最小实现）→ Refactor | 每次写代码（测试开关开时） | AI 执行 |
| Code Review | 检查代码与 spec 一致性、架构合理性 | 每个 Task 完成后 | AI 执行 + 人抽查 |
| Refactor | 不改功能，优化代码质量和架构 | 定期 / 发现坏味道时 | 人引导方向，AI 执行 |
| AGENTS.md | 提炼规范、控制大小、按需加载 | 同一问题 ≥3 次时 | AI 提议 + 人确认 |

---

## 文件索引

| 文件 | 内容 | 何时读 |
|------|------|--------|
| `references/spec-pillar.md` | Spec 三层规范详细写法、Task 拆解标准 | 写 spec 前必读 |
| `references/harness-pillar.md` | TDD 流程、审查清单、重构流程、AGENTS.md 维护 | 做质量保障前必读 |
| `references/bdd-gherkin.md` | Given-When-Then 验收标准写法、示例 | 写 Requirements Spec 前读 |
| `references/conventional-commits.md` | 提交格式规范、Task 粒度标准 | 开始交付前读 |
| `references/testing-trophy.md` | 前端测试分层（Trophy）+ 后端测试分层（Pyramid）+ 测试开关 | 开启测试前必读 |

---

## 工作流：新项目启动

```
1. 讨论需求（先不写代码，把细节说清楚）
      │
      ▼
2. AI 生成 Requirements Spec → docs/specs/xxx-requirements.md
   人 review，对齐目标（这步错了后面全错）
      │
      ▼
3. AI 生成 Design Spec → docs/specs/xxx-design.md
   人 review 架构方向
      │
      ▼
4. AI 拆解 Task → docs/specs/xxx-tasks.md
   每个 Task：状态、依赖、Checklist、对应 spec 段落
   人 review Task 粒度和依赖关系
      │
      ▼
5. 逐个 Task 实现（一个 Task = 一个 Conventional Commit）
   Harness 在每个 Task 后自动审查 spec 一致性
      │
      ▼
6. 有问题 → 回到步骤 2 更新 spec，循环
```

---

## 常见错误

| 错误 | 正确做法 |
|------|----------|
| 跟 AI 说"帮我做个 XX"直接生成代码 | 先讨论需求，输出 spec，review 后再写代码 |
| spec 讨论不对就进入开发 | spec 写完后自己审查一遍，和审查需求文档一样 |
| 越改越多，把对的改错了 | 改对了就提交，一个 Task 一次提交 |
| AGENTS.md 什么都往里塞 | 三次原则，≥3 次才写入。>500 行必须压缩 |
| 测试一直开着 | 默认关闭。AI 写测试问题累积，只在关键路径开 |
| Task 太大 | 50-150 行代码量，拆到无法再拆为止 |
