## Layer 2: Multi-turn Dialogue


### Core mechanism: Context persistence + LLM rewrite

```
Turn N:   User asks → Parse → Execute → Save SemanticParseInfo as context
Turn N+1: User asks follow-up → Load context → LLM rewrites question → Parse → Execute
```

### Implementation

```python
# 1. Persist SemanticParseInfo after each successful query
# Store as JSON blob: {chat_id} → {query_text, parse_info_json, updated_at}

def save_context(chat_id: str, query_text: str, parse_info: SemanticParseInfo):
    db.upsert("chat_context", {
        "chat_id": chat_id,
        "query_text": query_text,
        "parse_info": json.dumps(parse_info),  # full SemanticParseInfo as JSON
        "updated_at": now()
    })

def load_context(chat_id: str) -> dict | None:
    return db.get("chat_context", chat_id)

# 2. On next query: load history → LLM rewrites the question
def rewrite_multi_turn(current_question: str, chat_id: str, current_schema: str) -> str:
    history = load_context(chat_id)
    if not history:
        return current_question  # first turn, no rewrite needed
    
    prompt = f"""#Role: You are a data product manager experienced in data requirements.
#Task: Given current and history questions with their schema elements, rewrite a standalone question.
#Rules:
1. ALWAYS keep relevant entities, metrics, dimensions, values and date ranges from history
2. ONLY respond with the rewritten question text, nothing else

#Current Question: {current_question}
#Current Mapped Schema: {current_schema}

#History Question: {history['query_text']}
#History Mapped Schema: {extract_schema_summary(history['parse_info'])}
#History SQL: {history['parse_info']['sql_info']['corrected_s2sql']}

#Rewritten Question:"""
    
    return llm.generate(prompt).strip()
```

**Example:**
```
Turn 1: "上个月销售额按区域分布"
  → SQL: SELECT SUM(revenue), region ... WHERE dt IN ('2025-05')
  → Context saved: {metrics: [revenue], dimensions: [region], dateInfo: {2025-05}, SQL: ...}

Turn 2: "再看环比"  
  → LLM fuses with context → "上个月销售额按区域分布及环比增长率"
  → Now a standalone question that can be parsed independently
```

### Minimum Multi-turn Checklist

- [ ] Context persistence: `{chat_id, query_text, parse_info_json, updated_at}`
- [ ] MultiTurnRewriter: load history + current question → LLM rewrites to standalone question
- [ ] Save full `SemanticParseInfo` after each successful execution (not just query text)

> **Reference:** SuperSonic `ChatContextServiceImpl.java`, `NL2SQLParser.java` (rewriteMultiTurn method)

---


## Layer 3: RAG Knowledge Base


### Dual-index architecture: Trie (fast/exact) + Embedding (semantic/fuzzy)

```
[Semantic Schema]
       │
       ▼
┌──────────────────────────────┐
│ Trie Index                    │  ← exact / prefix / suffix match
│ - metric names + aliases      │     response in <10ms, handles ~80%
│ - dimension names + aliases   │
│ - dimension values            │
│ - business terms              │
└──────────────────────────────┘
       │
       ▼
┌──────────────────────────────┐
│ Embedding Vector Store        │  ← semantic similarity
│ - metric name+desc+alias emb  │     handles synonyms, fuzzy expressions
│ - dimension name+desc emb     │
│ - question→SQL exemplar pairs │
└──────────────────────────────┘
```

### Build Pipeline

```python
def build_knowledge_base(schema: SemanticSchema):
    # 1. Extract dictionary words from schema
    words = []
    for dataset in schema.datasets.values():
        for dim in dataset.dimensions:
            words.append(DictWord(dim.name, nature="DIMENSION", element_id=dim.id))
            for alias in (dim.alias or "").split(","):
                if alias.strip():
                    words.append(DictWord(alias.strip(), nature="DIMENSION", element_id=dim.id))
        for metric in dataset.metrics:
            words.append(DictWord(metric.name, nature="METRIC", element_id=metric.id))
        for term in dataset.terms:
            words.append(DictWord(term.name, nature="TERM", element_id=term.id))
    
    # 2. Register into trie (for exact/fuzzy match)
    trie = PrefixTrie()
    for w in words:
        trie.insert(w.word, {"nature": w.nature, "element_id": w.element_id})
        trie.insert_suffix(w.word[::-1], ...)  # suffix trie for suffix match
    
    # 3. Build embedding index (for semantic match)
    embeddings = EmbeddingStore("meta_collection")
    for dataset in schema.datasets.values():
        for element in dataset.metrics + dataset.dimensions:
            text = f"{element.name} | {element.description or ''} | {element.alias or ''}"
            embeddings.add(text, metadata={
                "id": element.id, "biz_name": element.biz_name,
                "type": "METRIC" if element in dataset.metrics else "DIMENSION",
                "dataset_id": dataset.id
            })
```

### Recall Pipeline (on user query)

```python
def recall(query: str, schema: SemanticSchema) -> SchemaMapInfo:
    tokens = tokenize(query)  # jieba for Chinese
    
    # 1. Trie search (exact/prefix/suffix match)
    trie_matches = []
    for token in tokens:
        trie_matches.extend(trie.prefix_search(token, limit=10))
        trie_matches.extend(trie.suffix_search(token, limit=5))
    
    # 2. Embedding search (semantic similarity)
    query_emb = embed(query)
    embedding_matches = embedding_store.search(query_emb, top_k=5,
        metadata_filter={"dataset_id": candidate_dataset_ids})
    
    # 3. Merge & rank: trie results first, embedding as supplement
    return merge_and_deduplicate(trie_matches, embedding_matches)
```

### LLM Prompt Injection (the RAG → NL2SQL bridge)

Recalled schema elements are formatted into the schema string injected into the LLM prompt (see Layer 1, Stage 2):

```
Schema: Metrics=[<revenue COMMENT 'total revenue'>, ...], Dimensions=[<region COMMENT 'sales region'>, ...]
SideInfo: CurrentDate=2025-06-29, DomainTerms=[<GMV COMMENT 'Gross Merchandise Volume'>]
```

### Pure Knowledge Q&A (no SQL execution)

When a user asks "销售额指标包含哪些费用", the system matches the term "销售额" → finds its `SchemaElement.description` and `expression` → returns the definition directly without executing SQL.

### Minimum RAG Checklist

- [ ] Schema extraction: iterate SemanticSchema → word list
- [ ] Trie index: prefix + suffix trie (use marisa-trie / pygtrie for Python)
- [ ] Embedding store: ChromaDB / LanceDB / FAISS for Python MVP
- [ ] Recall pipeline: trie first, embedding as fallback, merge & deduplicate
- [ ] Rebuild task: scheduled re-index when schema changes
- [ ] LLM prompt injection: format matched elements into schema string

> **Reference:** SuperSonic `KnowledgeBaseService.java`, `SearchService.java`, `MetaEmbeddingService.java`

---


## Layer 5: Intelligent Attribution Analysis


### Three post-execution processors (run after query, before returning to frontend)

```python
class ExecuteResultProcessor:
    def process(self, context: ExecuteContext, result: QueryResult) -> QueryResult:
        raise NotImplementedError
```

### 5a. Data Interpretation (AI summary)

```python
class DataInterpretProcessor(ExecuteResultProcessor):
    def process(self, context, result):
        top_rows = result.rows[:20]  # top 20 rows for context window
        
        prompt = f"""#Role: You are a data expert who communicates with business users everyday.
#Task: Interpret the query result data and organize a brief answer.
#Rules: 
1. Respond in the same language as the question
2. Reference key data points in your answer
3. Be concise — 2-4 sentences

#Question: {context.query_text}
#Data: {json.dumps(top_rows, ensure_ascii=False)}

#Summary:"""
        
        result.text_summary = llm.generate(prompt)
        return result
```

**Example output:**
> "2025年5月总营收为1,250万元，环比增长12.3%。华东区域贡献最大（占42%），华南次之（占28%）。其中线上渠道增速最快（+18.5%），线下门店基本持平。"

### 5b. Trend Analysis (YoY/MoM/WoW ratios)

```python
class MetricRatioCalcProcessor(ExecuteResultProcessor):
    def process(self, context, result):
        # For aggregate queries with time dimension:
        # 1. Extract current period data
        # 2. Query previous period (same metrics, date range shifted back)
        # 3. Compute ratio = (current - previous) / previous
        
        if context.parse_info.query_mode != "AGGREGATE":
            return result
        
        # Shift date range back by one period
        current_start = context.parse_info.date_info["start"]
        current_end = context.parse_info.date_info["end"]
        period_days = (current_end - current_start).days
        
        prev_context = copy(context)
        prev_context.parse_info.date_info = {
            "start": current_start - timedelta(days=period_days),
            "end": current_start - timedelta(days=1)
        }
        
        prev_result = execute_query(prev_context)  # can be async
        
        result.aggregate_info = {
            "metricInfos": [
                {
                    "metric": {"name": m.biz_name},
                    "statistics": {
                        "current": current_value,
                        "previous": prev_value,
                        "ratio": (current_value - prev_value) / prev_value,
                        "ratioType": determine_ratio_type(period_days)  # DAY_OVER_DAY, MONTH_OVER_MONTH, etc.
                    }
                }
                for m in context.parse_info.metrics
            ]
        }
        return result
```

**Supported ratio types:** DAY_OVER_DAY, WEEK_OVER_DAY, MONTH_OVER_MONTH, YEAR_OVER_YEAR

### 5c. Drill-down Dimension Recommendation

```python
class DimensionRecommendProcessor(ExecuteResultProcessor):
    def process(self, context, result):
        # Look up each metric's configured drill-down dimensions in the schema
        # Return dimensions NOT already in the current query
        
        used_dims = {d.biz_name for d in context.parse_info.dimensions}
        
        recommendations = []
        for metric in context.parse_info.metrics:
            # SchemaElement should have a "related_dimensions" field
            for dim in metric.related_dimensions or []:
                if dim.biz_name not in used_dims:
                    recommendations.append({
                        "name": dim.biz_name,
                        "label": dim.alias or dim.name,
                        "type": dim.data_type
                    })
        
        result.recommended_dimensions = recommendations[:5]  # top 5
        return result
```

**Example:** Query "营收按区域" → recommends drilling down by "渠道", "品类", "门店类型"

### Minimum Attribution Checklist

- [ ] `DataInterpretProcessor`: LLM prompt with top-20 rows → text summary
- [ ] `MetricRatioCalcProcessor`: query previous period → compute ratio
- [ ] `DimensionRecommendProcessor`: lookup related dimensions from schema

> **Reference:** SuperSonic `DataInterpretProcessor.java`, `MetricRatioCalcProcessor.java`, `DimensionRecommendProcessor.java`

---

