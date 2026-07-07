# Spec 支柱：三层规范写法详解

## 核心理念

**Spec 是 Prompt Engineering 的工程化落地。** 不是写完就扔的文档，而是 AI Agent 的输入规范——它决定了 AI 生成代码的边界、格式、质量。好的 spec 让 AI 一次生成到位，差的 spec 让后面的步骤全部跑偏。

---

## 三层规范总览

```
Requirements Spec（需求规范）
    │  用户故事、验收标准、业务逻辑、约束条件
    │  格式：BDD/Gherkin (Given-When-Then)
    │
    ▼
Design Spec（设计规范）
    │  技术架构、组件边界、数据流、集成模式
    │  格式：C4 模型 + 文字描述
    │
    ▼
Implementation Spec（实现规范）
    │  Task 拆解、依赖关系、实施计划、测试方案
    │  格式：Task 清单 + Checklist + 状态机
    │
    ▼
AI Agent 生成代码
```

**关键规则：**
- 每层 spec 独立输出一个文档，不混在一起
- 生成 spec 时**不生成 task**，避免每步都生成 task 浪费 token
- 每层 spec 必须经过人 review 后才进入下一层

---

## Layer 1: Requirements Spec（需求规范）

### 目的
告诉 AI "用户要什么"，不涉及任何技术实现细节。

### 包含内容
1. **用户故事 (User Stories)**：`As a [角色], I want [功能], so that [价值]`
2. **验收标准 (Acceptance Criteria)**：用 BDD/Gherkin 的 Given-When-Then 格式
3. **业务逻辑 (Business Logic)**：核心规则、边界条件、异常情况
4. **约束条件 (Constraints)**：性能要求、兼容性、安全要求、法律法规

### 输出格式

```markdown
# Requirements Spec: [功能名称]

> 版本: v1.0 | 日期: YYYY-MM-DD | 状态: Draft / Reviewed / Approved

## 🚨 风险声明

> 参照 SKILL.md 五大风险，评估本项目当前风险等级：

| 风险 | 等级 | 本项目说明 |
|------|------|-----------|
| AI 写测试 → 问题累积 | 🔴/🟡/🟢 | [当前策略] |
| AGENTS.md 膨胀失效 | 🔴/🟡/🟢 | [当前大小和策略] |
| 文档与代码脱节 | 🔴/🟡/🟢 | [保证措施] |
| Task 粒度过大 | 🔴/🟡/🟢 | [拆解标准] |
| 架构偏离 spec | 🔴/🟡/🟢 | [审查策略] |

---

## 用户故事

### US-01: [故事标题]
> As a [角色]
> I want [功能描述]
> So that [业务价值]

### US-02: ...

---

## 验收标准

### AC-01: [对应 US-01 的场景名称]

**Scenario: [场景描述]**

- **Given** [前置条件]
- **When** [用户操作]
- **Then** [预期结果]

### AC-02: ...

---

## 业务逻辑

### 核心规则
1. [规则描述]
2. ...

### 边界条件
| 场景 | 输入 | 预期行为 |
|------|------|----------|
| [边界1] | [输入] | [行为] |

### 异常处理
| 异常 | 触发条件 | 预期行为 |
|------|----------|----------|
| [异常1] | [条件] | [行为] |

---

## 约束条件

- **性能**：[响应时间、并发量 等]
- **兼容性**：[浏览器、设备、API 版本 等]
- **安全**：[认证、授权、数据加密 等]
- **其他**：[法规、第三方依赖 等]
```

### 人 Review 清单

- [ ] 用户故事是否覆盖所有核心场景？
- [ ] 验收标准是否可测试？（每个 Then 是否可验证？）
- [ ] 业务逻辑是否有遗漏的边界条件？
- [ ] 约束条件是否明确可量化？（不说"要快"，说"<200ms"）
- [ ] **涉及钱的功能？必须头脑风暴对话直到每个细节确认**

---

## Layer 2: Design Spec（设计规范）

### 目的
承接需求，定义"怎么做"的技术方案。AI 读到这个 spec 后应该知道：用什么技术、怎么分层、数据怎么流、组件怎么拆。

### 包含内容
1. **技术架构 (Technical Architecture)**：整体架构图、技术选型、分层设计
2. **组件边界 (Component Boundaries)**：前端组件树 / 后端模块划分，每个单元的职责和接口
3. **数据流 (Data Flow)**：API 路由、数据模型、状态管理、数据库表结构
4. **集成模式 (Integration Patterns)**：第三方服务、微服务通信、认证流程

### C4 模型层级（推荐用于架构描述）

- **Context（系统上下文）**：系统与外部用户/系统的关系
- **Container（容器图）**：前端 SPA / 后端 API / 数据库 / 缓存 等顶层容器
- **Component（组件图）**：每个容器内部的模块划分和数据流

### 输出格式

```markdown
# Design Spec: [功能名称]

> 版本: v1.0 | 日期: YYYY-MM-DD | 状态: Draft / Reviewed / Approved
> 关联: Requirements Spec v1.0

## 技术架构

### 整体架构
[文字描述 + 架构决策]

### 技术选型
| 层级 | 技术 | 选型理由 |
|------|------|----------|
| 前端框架 | React 18+ | [理由] |
| 状态管理 | [Zustand / Context / ...] | [理由] |
| 后端框架 | [FastAPI / Gin / ...] | [理由] |
| 数据库 | [PostgreSQL / SQLite / ...] | [理由] |

---

## 组件边界

### 前端组件树

```
PageComponent
├── Header（纯展示，无副作用）
├── MainPanel（容器组件，管理状态）
│   ├── SearchBar（受控组件，向上抛 onChange）
│   └── ResultList（纯展示，接收 data prop）
└── Footer
```

**组件职责：**
| 组件 | 类型 | 职责 | Props | 依赖 |
|------|------|------|-------|------|
| PageComponent | 页面 | 路由入口 | — | Header, MainPanel, Footer |
| MainPanel | 容器 | 管理搜索状态和结果 | — | SearchBar, ResultList, useSearch hook |
| SearchBar | 展示 | 输入框 + 搜索按钮 | value, onChange, onSubmit | — |
| ResultList | 展示 | 渲染结果列表 | data: Result[] | ResultItem |

### 后端模块划分

```
api/
├── routes/          # 路由层（仅参数校验和响应格式化）
├── services/        # 业务逻辑层
├── repositories/    # 数据访问层
└── models/          # 数据模型
```

---

## 数据流

### API 设计
| Method | Path | Request Body | Response | 说明 |
|--------|------|-------------|----------|------|
| GET | /api/items | ?q=&page= | {items:[], total:} | 列表查询 |
| POST | /api/items | {name, desc} | {id, name, desc} | 创建 |

### 前端数据流
```
User Action → Hook (useXxx) → API call → State Update → Re-render
```

### 数据库变更
```sql
-- 新增表
CREATE TABLE xxx (...);
-- 新增字段
ALTER TABLE yyy ADD COLUMN ...;
```

---

## 集成模式

[认证流程 / 第三方服务调用 / 消息队列 等]
```

### 人 Review 清单

- [ ] 技术选型是否与现有项目一致？（不引入无必要的新依赖）
- [ ] 组件/模块边界是否清晰？（能改内部实现而不影响外部？）
- [ ] 数据流是否完整？（每个 API 都有明确的 request/response 定义）
- [ ] 是否有安全遗漏？（认证、授权、SQL 注入、XSS）
- [ ] 架构决策是否值得写入 ADR？（见下方 ADR 规范）

### ADR（架构决策记录）

当架构决策不显而易见时（有多个合理方案），写 ADR：

```markdown
### ADR-001: [决策标题]

- **状态**：Proposed / Accepted / Deprecated
- **背景**：[为什么需要做这个决策]
- **决策**：[选择了什么方案]
- **后果**：[带来的好处和需要承担的代价]
- **备选方案**：[考虑过但未选择的方案及原因]
```

---

## Layer 3: Implementation Spec（实现规范）

### 目的
把设计方案拆成离散的 Task，每个 Task 可独立实现、独立提交、独立验证。

### Task 粒度标准（硬性约束）

- **代码量**：每个 Task 产生的代码在 **50-150 行**之间
- **时间**：每个 Task 应在 **30 分钟内**可完成
- **独立性**：每个 Task 可以独立编译/运行/测试
- **提交**：一个 Task = 一个 Conventional Commit

### Task 状态机

```
Pending ──▶ In Progress ──▶ Done
   │                          │
   └──────── Blocked ─────────┘
```

### 输出格式

```markdown
# Implementation Spec: [功能名称]

> 版本: v1.0 | 日期: YYYY-MM-DD | 状态: Draft / In Progress / Done

## Task 总览

| # | Task | 状态 | 依赖 | 预估行数 | 对应 Spec |
|---|------|------|------|----------|-----------|
| 1 | [Task 标题] | Pending | — | ~80 | US-01, AC-01 |
| 2 | [Task 标题] | Pending | #1 | ~120 | US-01, AC-02 |
| 3 | [Task 标题] | Pending | #1 | ~60 | Design: API /items |

---

## Task 详情

### Task 1: [标题]

- **状态**：Pending
- **依赖**：—
- **阻塞**：#2, #3
- **预估行数**：~80 行
- **对应 Spec**：Requirements US-01, AC-01；Design: 组件 MainPanel

**Checklist：**
- [ ] 创建 MainPanel 组件文件
- [ ] 实现 useSearch hook 基础逻辑
- [ ] 对接 SearchBar 的 onChange/onSubmit
- [ ] 添加 PropTypes/TypeScript 类型定义
- [ ] 编写基础渲染测试（如测试开关开启）

**产出文件：**
- `src/components/MainPanel.tsx`
- `src/hooks/useSearch.ts`
- `src/components/MainPanel.test.tsx`（如 test: true）

---

### Task 2: [标题]
...
```

### 人 Review 清单

- [ ] Task 粒度是否合理？（50-150 行，30 分钟内可完成）
- [ ] 依赖关系是否正确？（无循环依赖）
- [ ] Checklist 是否可验证？（每一步都有明确的完成标准）
- [ ] 是否标明了每个 Task 对应的 spec 段落？（可追溯）
- [ ] 是否有巨型 Task 需要再拆？

---

## Spec 的自我审查（AI 写完后自己查）

生成任何 spec 后，AI 应自问：

1. **Placeholder 扫描**：有 "TBD" "TODO" 或未完成的段落吗？
2. **内部一致性**：Requirements 的约束是否在 Design 中体现？Design 的模块是否覆盖了所有 Task？
3. **范围检查**：有功能不在当前 spec 范围内吗？（YAGNI——不是现在的需求就不要写）
4. **歧义检查**：有任何一个验收标准可以被两种方式解读吗？

---

## 常见错误

| 错误 | 正确做法 |
|------|----------|
| 三个 spec 混在一个文档里 | 每个 spec 独立输出，按顺序生成 |
| Requirements 里写技术实现 | Requirements 不能说"用 React 的 useState"，只能说"用户可以搜索" |
| 生成 spec 时顺带生成 task | spec 生成和 task 拆解分开两步，避免每步都出 task 浪费 token |
| spec 没 review 就进入开发 | 每层 spec 必须人 review 通过后才能进入下一层 |
| 设计过度（Over-engineering） | Design Spec 只覆盖当前需求，不为"可能的需求"设计 |
