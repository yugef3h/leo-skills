## Wiring It All Together


### Full Request Flow (Python/React example)

```
POST /api/chat/query
Body: { queryText: "上个月各区域销售额", chatId: "abc123" }

  ┌─ Backend: FastAPI endpoint ─────────────────────────────┐
  │                                                          │
  │  # LAYER 2: Multi-turn rewrite                           │
  │  rewritten_q = rewrite_multi_turn(req.query_text,        │
  │                                   req.chat_id, schema)   │
  │                                                          │
  │  # LAYER 3: RAG recall (also integrated in L1 mapping)   │
  │  map_info = recall(rewritten_q, semantic_schema)         │
  │                                                          │
  │  # LAYER 1: NL2SQL pipeline                              │
  │  #   Stage 1: Mapping (already done above)               │
  │  #   Stage 2-3: Parse + Correct                          │
  │  parse_info = rule_parse(map_info) or llm_parse(...)     │
  │  parse_info = correct_s2sql(parse_info, schema)          │
  │  #   Stage 4: Translate                                  │
  │  physical_sql = translate(parse_info, schema, dialect)   │
  │  #   Stage 5: Execute                                    │
  │  result = execute(physical_sql, db_conn)                 │
  │                                                          │
  │  # LAYER 5: Attribution                                  │
  │  for processor in [DataInterpret(), MetricRatioCalc(),   │
  │                     DimensionRecommend()]:                │
  │      result = processor.process(ctx, result)             │
  │                                                          │
  │  # LAYER 2: Save context for next turn                   │
  │  save_context(req.chat_id, rewritten_q, parse_info)      │
  │                                                          │
  │  return result                                           │
  └──────────────────────────────────────────────────────────┘
        │
        ▼
  ┌─ Frontend: React ───────────────────────────────────────┐
  │  const chartType = getChartType(result.columns,          │
  │                                  result.rows)            │
  │  // → METRIC_BAR for this query                          │
  │                                                          │
  │  <BarChart data={result} />                              │
  │  {result.textSummary && <Markdown>{result.textSummary}</>}│
  │  {result.aggregateInfo && <RatioBadges ... />}           │
  │  {result.recommendedDimensions && <DrillDownChips ... />}│
  └──────────────────────────────────────────────────────────┘
```

### Plugin Chain Registration (non-Spring)

```python
# config.yaml or config.py
PIPELINE = {
    "schema_mappers": [
        "chatbi.mappers.KeywordMapper",
        "chatbi.mappers.EmbeddingMapper",
    ],
    "parsers": [
        "chatbi.parsers.RuleSqlParser",
        "chatbi.parsers.LLMSqlParser",
    ],
    "correctors": [
        "chatbi.correctors.SchemaCorrector",
        "chatbi.correctors.GrammarCorrector",
    ],
    "execute_processors": [
        "chatbi.processors.DataInterpretProcessor",
        "chatbi.processors.MetricRatioCalcProcessor",
        "chatbi.processors.DimensionRecommendProcessor",
    ],
}

# Factory that loads and chains them
class ComponentFactory:
    def __init__(self, config):
        self.mappers = [import_and_instantiate(c) for c in config["schema_mappers"]]
        self.parsers = [import_and_instantiate(c) for c in config["parsers"]]
        # ...
    
    def run_pipeline(self, query, schema, chat_id):
        # Chain each component in order, passing context through
        ...
```

### Minimum Database Tables

```sql
-- Semantic model definition
CREATE TABLE dataset (
    id BIGINT PRIMARY KEY, name VARCHAR, table_name VARCHAR,
    database_name VARCHAR, config JSON
);
CREATE TABLE schema_element (
    id BIGINT PRIMARY KEY, dataset_id BIGINT, biz_name VARCHAR,
    name VARCHAR, expression TEXT, agg VARCHAR, alias VARCHAR,
    description TEXT, data_type VARCHAR, element_type VARCHAR  -- METRIC/DIMENSION/TERM
);

-- Chat session
CREATE TABLE chat_context (
    chat_id VARCHAR PRIMARY KEY, query_text TEXT,
    parse_info JSON, updated_at DATETIME
);
CREATE TABLE chat_history (
    id BIGINT PRIMARY KEY, chat_id VARCHAR, query_text TEXT,
    query_result JSON, parse_info JSON, state VARCHAR, created_at DATETIME
);
```

---

