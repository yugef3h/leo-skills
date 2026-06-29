## MVP Scope Recommendation (4 weeks)

**Week 1-2: Layer 1 + Layer 3 (NL2SQL + RAG) — the core**
- Data models (SemanticSchema, SchemaMapInfo, SemanticParseInfo, QueryResult)
- jieba + Trie index + FAISS embedding store for schema recall
- LangChain prompt template with schema injection
- Rule-based parser for simple patterns
- LLM-based parser with self-consistency
- SchemaCorrector + GrammarCorrector
- S2SQL → physical SQL translator (sqlglot)
- SQLite executor

**Week 3: Layer 2 + Layer 4 (Multi-turn + Visualization)**
- ChatContext persistence (JSON in SQLite)
- MultiTurnRewriter (LangChain prompt)
- Chart auto-classifier + ECharts renderers + Table fallback

**Week 4: Layer 5 (Attribution)**
- DataInterpretProcessor (LangChain summary)
- MetricRatioCalcProcessor (period-over-period ratios)
- DimensionRecommendProcessor

**Polish:**
- FastAPI SSE streaming (LangChain `astream`)
- Chart export
- User feedback → exemplar memory (FAISS)

---

## Zero-Install Local Stack (一键启动)

```bash
pip install fastapi uvicorn langchain langchain-openai langchain-community \
           sqlglot jieba faiss-cpu sentence-transformers
python main.py
```

| Need | Choice | Why zero-install |
|------|--------|-----------------|
| Database | **SQLite** | `import sqlite3`, Python stdlib |
| LLM (Chat) | **DeepSeek** (`deepseek-chat`) | OpenAI-compatible API, ¥1/M tokens, 中文最强性价比 |
| LLM (复杂推理) | `deepseek-reasoner` (R1) | SQL纠正、归因分析等复杂任务 |
| LLM framework | **LangChain** (Python) | Unified: chat + prompts + output parsers |
| Embedding model | **BAAI/bge-small-zh-v1.5** | 本地运行，24MB，中文 SOTA，免费离线 |
| Embedding framework | **sentence-transformers** | `pip install`，一行代码加载模型 |
| Vector store | **FAISS** (in-memory) | `pip install faiss-cpu`, no server process |
| Chinese tokenization | **jieba** | `pip install jieba` |
| SQL parsing | **sqlglot** | `pip install sqlglot` |
| Trie dictionary | **marisa-trie** | `pip install marisa-trie`, in-memory |
| Web framework | **FastAPI** + uvicorn | `pip install` |
| Frontend | **Vite** + React + ECharts | `npm install && npm run dev` |

**LangChain + DeepSeek + 本地 Embedding:**

```python
from langchain_openai import ChatOpenAI
from langchain_community.embeddings import HuggingFaceEmbeddings
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
import faiss, numpy as np

# === LLM: DeepSeek (OpenAI 兼容 API) ===
llm = ChatOpenAI(
    model="deepseek-chat",              # 通用：S2SQL、改写、总结
    # model="deepseek-reasoner",        # 复杂推理：SQL纠正、归因
    openai_api_key="sk-your-deepseek-key",
    openai_api_base="https://api.deepseek.com",
    temperature=0.3,
)

# === Embedding: 本地中文模型 (免费离线) ===
embeddings = HuggingFaceEmbeddings(
    model_name="BAAI/bge-small-zh-v1.5",  # 24MB, 中文优化
    model_kwargs={"device": "cpu"},
)

# === S2SQL 生成链 ===
prompt = ChatPromptTemplate.from_messages([
    ("system", "#Role: data analyst ... #Schema: {schema}"),
    ("user", "{question}")
])
s2sql_chain = prompt | llm | StrOutputParser()

# === Schema 向量化 → FAISS ===
schema_texts = ["revenue | 营收,销售额,收入 | total order amount", ...]
vecs = embeddings.embed_documents(schema_texts)
index = faiss.IndexFlatIP(len(vecs[0]))
index.add(np.array(vecs))

# === 用户查询 → 向量召回 ===
query_vec = embeddings.embed_query("最近卖了多少")
scores, ids = index.search(np.array([query_vec]), k=5)
# → [(0.89, "revenue"), (0.72, "order_count"), ...]
```

**DeepSeek 模型选择:**

| 模型 | API名 | 适合 | 价格 |
|------|-------|------|------|
| DeepSeek-V3 | `deepseek-chat` | S2SQL生成、多轮改写、数据解读 | ¥1/M tokens |
| DeepSeek-R1 | `deepseek-reasoner` | 复杂SQL纠正、自一致性验证 | ¥4/M tokens |

**Embedding 方案对比:**

| 方案 | 模型 | 成本 | 中文质量 | 离线 |
|------|------|------|---------|------|
| **本地 (推荐)** | `BAAI/bge-small-zh-v1.5` | 免费 | ⭐⭐⭐⭐⭐ | ✅ |
| 本地大号 | `BAAI/bge-large-zh-v1.5` | 免费 | ⭐⭐⭐⭐⭐ | ✅ |
| OpenAI API | `text-embedding-3-small` | ~$0.02/1M | ⭐⭐⭐⭐ | ❌ |
| 硅基流动 API | `BAAI/bge-m3` | 免费额度 | ⭐⭐⭐⭐⭐ | ❌ |

MVP 用 `bge-small-zh-v1.5`（24MB，pip install sentence-transformers 后首次自动下载），几百条 Schema 元素秒级索引。

**向量查询能力：** Trie (jieba + marisa-trie) 做精确/模糊匹配 → FAISS 做语义兜底。双索引并行，一个进程全搞定。

---

## SuperSonic → MVP 技术对照

### 后端

| 功能 | SuperSonic (Java) | MVP (Python) |
|------|------------------|-------------|
| Web框架 | Spring Boot 3.3.9 | FastAPI + uvicorn |
| LLM框架 | LangChain4j 0.36.2 | LangChain (Python) |
| LLM模型 | 可配置多模型 | OpenAI gpt-4o-mini (开发) |
| Embedding | LangChain4j EmbeddingStore | LangChain OpenAIEmbeddings |
| 向量存储 | Milvus/Chroma/PGVector | FAISS (in-memory, zero setup) |
| 数据库 | MySQL/PG/ClickHouse | SQLite (开发) → PG (生产) |
| ORM | MyBatis-Plus | SQLAlchemy (async) |
| SQL解析 | JSqlParser + Calcite | sqlglot |
| 中文分词 | HanLP | jieba |
| Trie | HanLP BinTrie | marisa-trie |
| 流式响应 | 500ms轮询 | FastAPI SSE (LangChain astream) |
| 插件/SPI | SpringFactoriesLoader | YAML config + ComponentFactory |

### 前端

| 功能 | SuperSonic | MVP |
|------|-----------|-----|
| 框架 | React 18 + UmiJS 4 | React 18 + Vite 5 |
| UI组件库 | Ant Design 5.17.4 | Ant Design 5 |
| 图表 | ECharts 5 | ECharts 5 |
| 状态管理 | UmiJS model | zustand |
| 路由 | UmiJS路由 | react-router-dom v6 |
| CSS方案 | Less | Tailwind CSS 或 CSS Modules |
| 图标 | @ant-design/icons | @ant-design/icons |

### 关键差异

| 保留 | 替换 | 原因 |
|------|------|------|
| 5层架构 + 5阶段流水线 | - | 核心不变 |
| S2SQL中间层 | - | LLM生成bizName SQL |
| Trie+Embedding双索引 | HanLP→jieba, Milvus→FAISS | Python生态替代 |
| Prompt模板结构 | LangChain4j→LangChain | 同概念，Python版更成熟 |
| 前端状态机 + 组件树 | UmiJS→Vite | 更轻更快 |

---

## Reference: SuperSonic Source Code Map

All files under `com/tencent/supersonic/` in the [SuperSonic repo](https://github.com/tencentmusic/supersonic). The `...` represents intermediate package directories.

| Component | Key File | What to look at |
|---|---|---|
| **NL2SQL Pipeline Engine** | `headless/server/.../ChatWorkflowEngine.java` | State machine: MAPPING→PARSING→CORRECTING→TRANSLATING |
| **KeywordMapper** | `headless/chat/.../mapper/KeywordMapper.java` | Trie-based matching (→ jieba + marisa-trie) |
| **EmbeddingMapper** | `headless/chat/.../mapper/EmbeddingMapper.java` | Vector matching (→ FAISS) |
| **LLM SQL Parser** | `headless/chat/.../parser/llm/LLMSqlParser.java` | LLM S2SQL generation (→ LangChain) |
| **PromptHelper** | `headless/chat/.../parser/llm/PromptHelper.java` | Schema string formatting + exemplar selection |
| **OnePassSCSqlGenStrategy** | `headless/chat/.../parser/llm/OnePassSCSqlGenStrategy.java` | Self-consistency voting |
| **RuleSqlCorrector** | `headless/chat/.../corrector/RuleSqlCorrector.java` | Corrector chain (→ sqlglot transforms) |
| **DefaultSemanticTranslator** | `headless/core/.../translator/DefaultSemanticTranslator.java` | S2SQL→Physical SQL (→ sqlglot) |
| **KnowledgeBaseService** | `headless/chat/.../knowledge/KnowledgeBaseService.java` | Trie lifecycle |
| **MetaEmbeddingService** | `headless/chat/.../knowledge/MetaEmbeddingService.java` | Embedding recall (→ FAISS + LangChain) |
| **ChatContextService** | `chat/server/.../service/impl/ChatContextServiceImpl.java` | Context persistence (→ SQLite JSON) |
| **NL2SQLParser (rewriteMultiTurn)** | `chat/server/.../parser/NL2SQLParser.java` | Multi-turn rewrite (→ LangChain prompt) |
| **DataInterpretProcessor** | `chat/server/.../processor/execute/DataInterpretProcessor.java` | LLM data summary (→ LangChain) |
| **MetricRatioCalcProcessor** | `chat/server/.../processor/execute/MetricRatioCalcProcessor.java` | YoY/MoM calculation |
| **DimensionRecommendProcessor** | `chat/server/.../processor/execute/DimensionRecommendProcessor.java` | Drill-down suggestions |
| **ChatQueryController** | `chat/server/.../rest/ChatQueryController.java` | REST API endpoints (→ FastAPI routes) |
| **ChatMsg (getMsgContentType)** | `webapp/.../components/ChatMsg/index.tsx` | Chart type classifier |

---

## Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| LLM generates SQL with wrong column names | Inject full schema string into prompt; post-hoc correct with Corrector chain |
| LLM invents table names | Explicitly forbid in prompt rules; strip in SchemaCorrector |
| Multi-turn loses context after 2+ turns | Persist FULL SemanticParseInfo (not just query text); include history SQL in rewrite prompt |
| Embedding recall returns irrelevant results | Filter by dataset_id metadata; use trie results to scope the embedding search |
| Chart auto-classification picks wrong type | Always let user toggle; fall back to TABLE for edge cases |
| Metric ratio query is expensive (2x queries) | Run previous period query async; cache results |
| Trie dictionary stale after schema update | Scheduled reload (every 60s) + manual refresh API |
| FAISS index lost on restart | Rebuild from SemanticSchema on startup (schema data is small, takes <1s) |
