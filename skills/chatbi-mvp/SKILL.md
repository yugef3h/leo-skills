---
name: chatbi-mvp
description: Use when building a ChatBI (conversational BI) MVP from scratch, need to understand the 5 core capabilities (NL2SQL, multi-turn dialogue, RAG knowledge base, data visualization, intelligent attribution), or want to reference SuperSonic's architecture to guide implementation
---

# ChatBI MVP Architecture

## Overview

Build a ChatBI system: user asks questions in natural language → system generates SQL → executes → displays interactive charts with AI insights.

**Architecture:** NL → S2SQL (semantic SQL / MQL) → Physical SQL. LLM generates S2SQL using business terms (bizName), a deterministic Translator converts to physical SQL. LLM never touches physical table/column names.

**Tech stack assumed:** React (frontend) + Python/FastAPI (backend). Architecture is language-agnostic.

## Five Core Layers

```
User: "最近7天各分区播放量怎么样"
         │
         ▼
┌─ Layer 3: RAG ────────────────┐  Trie + Embedding recall
│  "播放量" → views (metric)     │  Identify schema elements
│  "分区"   → category (dim)     │  in user query
│  "最近7天" → DateConf{-7d}     │
└───────────────┬───────────────┘
                │
                ▼
┌─ Layer 1: NL → S2SQL → SQL ──┐  5-stage pipeline
│  MAPPING → PARSING →          │  LLM generates S2SQL (bizName)
│  CORRECTING → TRANSLATING     │  Translator → physical SQL
│  → EXECUTE                    │
└───────────────┬───────────────┘
                │
        ┌───────┴───────┐
        ▼               ▼
┌─ Layer 2 ───┐  ┌─ Layer 5 ──────────┐
│ Multi-turn   │  │ Attribution         │
│ Context save │  │ LLM summary + YoY   │
│ + LLM rewrite│  │ + drill-down recs   │
└──────────────┘  └─────────────────────┘
        │               │
        └───────┬───────┘
                ▼
┌─ Layer 4: Visualization ──────┐
│  Auto chart type → ECharts    │
│  User can toggle chart/table  │
│  Drill-down → re-query        │
└───────────────────────────────┘
```

## File Index

This skill is split into focused files. Read in order, or jump to what you need.

### Core Architecture (read first)

| File | Contents | When to read |
|------|----------|-------------|
| `data-models.md` | All shared POJOs: SemanticSchema, SchemaMapInfo, SemanticParseInfo, QueryResult | Always start here — these connect every layer |
| `nl2sql-pipeline.md` | Layer 1 in detail: 5-stage pipeline, prompt template, self-consistency, corrector chain, S2SQL→physical translator | Core of the system |
| `wiring.md` | How all layers connect: full request flow, plugin registration, DB tables | When you need to see the big picture |

### Supporting Layers

| File | Contents | When to read |
|------|----------|-------------|
| `layers-2-3-5.md` | Multi-turn dialogue (Layer 2), RAG knowledge base (Layer 3), Attribution analysis (Layer 5) | After understanding Layer 1 |
| `visualization.md` | Chart auto-classification, ECharts configs, chart type toggle, interaction patterns (Layer 4) | Frontend implementation |

### Implementation Guides

| File | Contents | When to read |
|------|----------|-------------|
| `data-generation.md` | How to generate demo data with Faker + Pandas + SQLite, semantic model YAML, few-shot exemplars | Before you start coding |
| `frontend-interaction.md` | Complete frontend state machine, component tree, API calls, drill-down/metric-switch/date-filter flows | Frontend implementation |
| `bilibili-example.md` | Full end-to-end: B站 creator analytics, from data generation to chart display, trace one query | When you want a concrete example |
| `ui-design-system.md` | CSS variables, color palette, typography, component recipes, shadows, ECharts theme | When building UI |

### Planning

| File | Contents | When to read |
|------|----------|-------------|
| `mvp-plan.md` | 4-week MVP scope, SuperSonic source code reference, common pitfalls, Python+React tech stack recommendations | Project planning |

## Getting Started

**First time:** Read files in this order:
1. `data-models.md` — understand the data structures
2. `nl2sql-pipeline.md` — understand the core engine
3. `bilibili-example.md` — see a concrete example end-to-end
4. `wiring.md` — see how everything connects

**Starting to code:**
5. `data-generation.md` — generate your demo data
6. `mvp-plan.md` — follow the 4-week plan
7. `ui-design-system.md` — copy CSS variables into your project

**Implementing specific layers:**
- `layers-2-3-5.md` — for multi-turn, RAG, or attribution
- `visualization.md` — for charts
- `frontend-interaction.md` — for frontend state machine and components

## Core Principles

1. **Semantic layer isolation**: LLM generates S2SQL (bizName), NEVER physical SQL. The Translator is deterministic.
2. **Plugin chain architecture**: Each layer is a chain of plugins registered in config, executed sequentially. Add/remove plugins without touching core code.
3. **Dual strategy everywhere**: Rule-based (fast, deterministic) first → LLM-based (flexible) fallback. Applies to parsing, correction, and mapping.
4. **Full context persistence**: Save the entire `SemanticParseInfo` (not just query text) after each turn. Include history SQL in multi-turn rewrite prompt.
5. **User can always override**: Chart type auto-selected but user can toggle. Filters auto-detected but user can adjust.

## Reference Implementation

[SuperSonic](https://github.com/tencentmusic/supersonic) is the reference architecture. See `mvp-plan.md` for a source code file map to key classes.
