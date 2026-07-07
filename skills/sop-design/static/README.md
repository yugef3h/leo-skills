# 三张图核心内容整理
## 1. AI 三类规范文档（AI Agent 代码生成输入体系）
### 三大规范分层
1. **Requirements Specifications 需求规范**
    向设计规范提供输入依据，AI Agent 读取解读
    - 用户故事 User stories
    - 验收标准 Acceptance criteria
    - 业务逻辑 Business logic
    - 约束条件 Constraints
2. **Design Specifications 设计规范**
    承接需求、指导实现，AI Agent 读取解读
    - 技术架构 Technical architecture
    - 组件边界 Component boundaries
    - 数据流 Data flow
    - 集成模式 Integration patterns
3. **Implementation Specifications 实现规范**
    由设计规范指引，AI Agent 读取解读
    - 离散任务 Discrete tasks
    - 任务依赖 Task dependencies
    - 实施计划 Implementation plan
    - 测试方案 Testing approach

### 整体流程
需求规范 → 支撑 → 设计规范 → 指引 → 实现规范
三类规范全部输入 AI Agent，Agent 统一读取解析后，输出**架构一致的生成代码 Generated Code**

---

## 2. TDD 测试驱动开发循环（红-绿-重构）
循环可重复执行，三步闭环：
1. **RED（失败）**
    编写一条会执行失败的测试用例 write a test that fails
2. **GREEN（通过）**
    只编写刚好能让测试通过的最简代码 make only enough code for it to pass
3. **REFACTOR（重构）**
    不改动功能，优化代码质量 improve code quality

循环顺序：RED → GREEN → REFACTOR → RED（持续重复）

---

## 3. 编码 Agent 工程管控框架（Harness engineering）
### 角色：Human 人类（人工调控 Steering）
双向循环输出两类管控模块，分别正向/反向作用于 Coding Agent
1. **Guides 正向前馈 feedforward（给 Agent 生成代码提供标准）**
    - Principles 开发原则、CfRs 约束要求、Rules 编码规则
    - Ref Docs 参考文档、How-tos 操作指南
    - Language Servers、CLIs & scripts、Code mods 语言/工具配套
2. **Sensors 反向反馈 feedback（采集代码问题用于自修正）**
    - Static analysis 静态代码分析、Review agents 评审智能体
    - Logs 运行日志、Browser 浏览器运行观测

### Coding Agent 内部双阶段循环
1. initial generation 初始代码生成（接收 Guides 前馈标准）
2. self-correcting 自我修正迭代（接收 Sensors 反馈问题）