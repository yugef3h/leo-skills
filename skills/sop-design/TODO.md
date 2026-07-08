# React + Python + Go 全栈工业级自动化最佳实践（分4层：AI工具原生Hook、Git全局钩子、IDE实时校验、CI兜底）
先明确核心结论：你提到的**AI保存自动格式化、提交拦截测试、危险shell拦截**只是第一层AI工具内置Hook；工业落地会叠加三层强制兜底，全程**本地执行、不消耗Token**，覆盖前端React、Python后端、Go服务统一规范。

## 一、第一层：AI编程工具原生Hook（Claude Code / Cursor 内置，零Token）
就是原文讲的专属钩子，AI改完文件瞬间触发，模型不参与、无上下文消耗，分三类强制能力：
### 1. 文件保存后置Hook（AI写完代码立刻执行）
全栈统一格式化，AI生成代码不规范也自动修正：
- React/TS：`eslint --fix + prettier --write` 自动缩进、导入排序、修复语法报错
- Python：`ruff check --fix + black + isort` 统一格式、导入排序、静态语法检查
- Go：`gofmt -w + golint` 官方格式化+代码异味检测
### 2. Shell命令前置安全Hook（拦截高危操作）
AI调用终端前本地拦截，不需要AI判断：
- 黑名单阻断：`rm -rf、rm -r、chmod 777、curl | bash、sudo` 高危命令直接拒绝执行
- 路径白名单：限制AI仅能读写项目目录，禁止访问`/etc、~/.ssh、.env`等敏感文件
- 命令审计：所有AI执行的shell落地日志，团队可追溯
### 3. 任务完成校验Hook（AI完成功能后自动自测）
AI写完接口/组件，自动跑对应单测，不通过则强制回滚修改：
- React：`npm run test --changedSince HEAD` 仅跑改动组件测试
- Python：`pytest -xvs` 增量单元测试
- Go：`go test ./... -short` 快速单元校验

## 二、第二层：跨工具通用 Git Pre-Commit 钩子（企业标准，兼容所有AI工具）
Claude/Cursor原生Hook仅对当前AI工具生效，**Git钩子是仓库级强制约束**，任何人（手动写、AI生成代码）提交代码都会触发，全栈统一一套配置，工业主流两套方案：
### 方案A：pre-commit（跨语言首选，React+Python+Go一站式管理）
`.pre-commit-config.yaml` 一份配置覆盖三端，团队统一提交规范，不依赖Node环境
```yaml
# React/TS 前端
- repo: prettier
  hooks: [id: prettier]
- repo: eslint
  hooks: [id: eslint, args: ["--fix"]]
# Python后端
- repo: ruff
  hooks: [id: ruff-fix]
- repo: black
  hooks: [id: black]
- repo: isort
  hooks: [id: isort]
# Go服务
- repo: https://github.com/golangci/golangci-lint
  hooks: [id: golangci-lint]
```
能力：
1. 只对**暂存变更文件**执行校验，速度快
2. 自动修复格式，无法修复的语法/类型错误直接阻断git commit
3. 拦截提交敏感文件：`.env、*.pem、密钥、数据库配置`

### 方案B：Husky + lint-staged（React项目专用，Node生态）
前端仓库标配，配合monorepo前后端分离项目
```json
// package.json
"lint-staged": {
  "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
  "*.py": ["ruff fix", "black"],
  "*.go": ["gofmt -w"]
}
```
配套`pre-commit`脚本：提交前执行增量格式化+类型校验（`tsc --noEmit`）+ 轻量单测

### Git额外补充工业强制规则（原文未提及）
1. `commit-msg`钩子：commitlint强制提交规范（feat/fix/docs等），杜绝无意义提交日志
2. `pre-push`钩子：完整全量单元测试、安全漏洞扫描，推送远程前拦截烂代码
3. 文件黑名单：禁止提交密钥、本地配置、大体积二进制文件

## 三、第三层：IDE实时本地校验防线（开发时提前拦截，减少返工）
AI写代码的同时编辑器实时标红，不用等到保存/提交才发现问题，无Token消耗：
1. VSCode / JetBrains（WebStorm/PyCharm/GoLand）插件：
   - ESLint / Ruff / golint 实时语法、类型、安全告警
   - 保存自动格式化（editor.formatOnSave），和AI Hook规则对齐
2. SonarLint 本地静态扫描：实时检测AI生成代码的漏洞（XSS、SQL注入、空指针），企业必备安全层

## 四、第四层：CI/CD流水线兜底（最终防线，防止本地Hook被跳过）
本地钩子可手动`--no-verify`跳过，CI流水线强制校验，线上准入唯一标准：
1. 全量lint格式化校验
2. 完整单元测试、集成测试、E2E（React Playwright）
3. SAST代码安全扫描（Semgrep/SonarQube），拦截AI生成的高危漏洞
4. 依赖漏洞扫描（npm audit、safety、go mod vuln）

## 五、原文没提到、工业必须加的补充自动化能力（全栈通用）
### 1. 类型校验强制拦截（AI高频踩坑点）
- React TS：`tsc --noEmit` 类型错误直接阻断提交，杜绝any、类型不匹配
- Python：mypy静态类型检查
- Go：原生强类型，golangci-lint校验类型规范
### 2. 敏感信息自动扫描（安全刚需）
本地Hook自动扫描代码中的硬编码密钥、token、数据库密码，发现直接拦截提交，防止AI随手写死密钥
### 3. 依赖版本锁定校验
AI经常随意升级依赖，钩子校验`package-lock/go.mod/poetry.lock`变更，大版本升级强制人工确认
### 4. 增量测试机制（解决全量测试太慢）
本地只跑改动模块测试；CI跑全量，兼顾开发效率与质量
### 5. AI变更Diff自动安全审查（中大型企业）
pre-commit钩子读取本次AI修改的代码diff，调用本地轻量模型做安全审查，识别危险逻辑、SQL注入、越权接口，**可选消耗少量本地算力，不消耗云端Token**

## 六、四层机制分工对比（清晰区分AI Hook vs 通用Git Hook）
| 机制 | 触发时机 | 是否耗Token | 覆盖范围 | 核心定位 |
|------|----------|-------------|----------|----------|
| Claude/Cursor 原生Hook | AI保存文件、AI执行命令 | ❌ 零消耗 | 仅当前AI工具操作 | 即时兜底，AI写完立刻修正 |
| pre-commit Git钩子 | git commit提交 | ❌ | 所有人、所有修改（手动/AI） | 仓库统一强制规范，跨工具通用 |
| IDE实时校验 | 编码/保存时 | ❌ | 本地开发全程 | 提前预警，减少返工 |
| CI流水线 | 推送远程/PR | ❌ | 所有合并代码 | 最终准入防线，不可绕过 |

## 七、React+Python+Go 落地最简工业组合方案（推荐直接落地）
1. 项目根目录 `CLAUDE.md / AGENTS.md`：软性规范，约束AI行为
2. 开启Claude Code内置Hook：保存自动格式化、shell高危拦截、AI修改后自测
3. 全局配置 `pre-commit`：Git提交统一校验，全栈一套规则
4. IDE开启保存自动格式化+SonarLint实时扫描
5. CI流水线配置全量lint+测试+安全扫描兜底

这套组合解决你原文里所有痛点，同时补齐企业安全、团队协作、跨工具兼容的工业级要求，所有本地自动化流程均不消耗AI上下文Token。