# leo-skills

A collection of production-grade agent skills for Claude Code and other AI agent terminals.

<p align="center">
  <img src="https://img.shields.io/badge/Skills-3-blue" alt="3 Skills" />
  <img src="https://img.shields.io/badge/License-MIT-yellow" alt="MIT License" />
</p>

## Skills

| Skill | Description | Install |
|-------|-------------|---------|
| [**Oral Sentence Model**](./skills/oral-sentence-model/) | 围绕任意话题生成英语口语"句模"（6问12句），标注词伙/句型/观点，支持拆解复用、跨话题联动、轻量练习 | `npx skills add yugef3h/leo-skills --path skills/oral-sentence-model` |
| [**ChatBI MVP**](./skills/chatbi-mvp/) | ChatBI 系统架构指南：NL2SQL、多轮对话、RAG 知识库、可视化图表、智能归因分析，参考 SuperSonic 架构 | `npx skills add yugef3h/leo-skills --path skills/chatbi-mvp` |
| [**SOP Design**](./skills/sop-design/) | Spec + Harness 双支柱 AI 辅助开发流程：文档先行、TDD、代码审查、AGENTS.md 三次原则维护，集成 BDD/Conventional Commits/Testing Trophy | `npx skills add yugef3h/leo-skills --path skills/sop-design` |

## Quick Start

```bash
npx skills add yugef3h/leo-skills --path skills/<skill-name>
```

Then invoke in your agent terminal:

```bash
# 生成话题句模
"帮我生成一个关于旅行的句模"
"旅行句模"

# 拆解复用
"帮我把这段句模拆解一下，提取可复用的词伙和句型"

# 跨话题联动
"能把食物的句型用到科技话题吗"

# 轻量练习
"帮我练练这个句模"
```

## License

MIT
