## Layer 1: NL → S2SQL → SQL Engine


### Key insight: Two-stage, not one-stage

Unlike naive approaches where LLM directly generates physical SQL, this architecture uses a **semantic layer (S2SQL/MQL)** as intermediate representation. LLM generates S2SQL using business terms, and a deterministic Translator converts it to physical SQL.

**Why this matters:**
- LLM never sees physical table/column names → no hallucination on table references
- Metric formulas (e.g., `revenue = price * quantity - discount`) are resolved deterministically at translate time, not guessed by LLM
- Same S2SQL can be translated to MySQL/ClickHouse/PG dialect without changing the LLM prompt

### Pipeline (5 stages)

```
User Query → [Mapping] → [Parsing] → [Correcting] → [Translating] → [Executing]
              match NL    LLM generates  fix S2SQL      S2SQL/MQL →     run physical
              to schema   S2SQL/MQL      errors         physical SQL    SQL on DB
              (RAG)       (bizName)                     (deterministic)
```

### Stage 1: Schema Mapping

**Goal:** Identify which metrics/dimensions/values the user mentioned.

**Dual strategy — trie + embedding:**

```python
# Strategy A: Trie-based exact/fuzzy match (fast, no LLM, handles ~80% of cases)
# - Tokenize query (jieba for Chinese, whitespace for English)
# - Build prefix/suffix trie from all SchemaElement.name + alias
# - Match tokens against trie, return matches with similarity scores

# Strategy B: Embedding-based semantic match (handles synonyms)
# - Pre-compute embeddings for "element.name | element.description | element.alias"
# - Embed user query, cosine-similarity search, return top-K

def map_schema(query: str, schema: SemanticSchema) -> SchemaMapInfo:
    tokens = tokenize(query)
    result = {}
    for dataset_id, dataset in schema.datasets.items():
        matches = []
        # 1. Trie match (primary)
        for token in tokens:
            trie_results = trie_search(token, dataset)  # prefix + suffix
            matches.extend(trie_results)
        # 2. Embedding fallback (handles synonyms like "卖了多少" → revenue)
        query_embedding = embed(query)
        embedding_results = vector_search(query_embedding, dataset, top_k=5)
        matches.extend(embedding_results)
        # 3. Deduplicate by element_id
        matches = deduplicate_and_rank(matches)
        result[dataset_id] = matches
    return result
```

**Minimum mapper chain:**
1. `KeywordMapper`: trie match (primary, handles exact/fuzzy)
2. `EmbeddingMapper`: vector similarity (fallback for synonyms)
3. `TimeRangeParser`: extract date expressions ("上个月" → {"start": "-30d", "end": "today"})

> **Reference:** SuperSonic `KeywordMapper.java`, `EmbeddingMapper.java`, `TimeRangeParser.java`

### Stage 2: Semantic Parsing (S2SQL generation)

**Goal:** Generate S2SQL (semantic SQL using bizName, not physical column names).

**Dual strategy — rule-based + LLM-based:**

```python
# Rule-based (deterministic, instant, handles simple patterns)
def rule_based_parse(map_info: SchemaMapInfo, schema: SemanticSchema) -> SemanticParseInfo | None:
    # Match query pattern:
    # - metric + dimension → METRIC_GROUPBY: "SELECT {metric}, {dim} ... GROUP BY {dim}"
    # - metric + dimension + value → METRIC_FILTER: Above + WHERE {dim} = '{value}'
    # - metric + dimension + "top N" → METRIC_ORDERBY: Above + ORDER BY {metric} DESC LIMIT N
    # - single metric → METRIC_MODEL: "SELECT {agg}({metric}) FROM {table}"
    
    patterns = [
        GroupByPattern(), FilterPattern(), OrderByPattern(), SimpleMetricPattern()
    ]
    for p in patterns:
        result = p.try_match(map_info, schema)
        if result:
            return result
    return None

# LLM-based (handles complex/ambiguous queries)
def llm_parse(query: str, map_info: SchemaMapInfo, schema: SemanticSchema) -> SemanticParseInfo:
    prompt = build_s2sql_prompt(query, map_info, schema)
    candidates = []
    for i in range(3):  # self-consistency: 3 candidates
        sql = llm.generate(prompt, temperature=0.3 + i * 0.1)
        if is_valid_sql(sql):
            candidates.append(sql)
    best_sql = majority_vote(candidates)
    return SemanticParseInfo(sql_info=SqlInfo(parsed_s2sql=best_sql), ...)
```

**The prompt template — this is the single most critical piece for accuracy:**

```
#Role: You are a data analyst experienced in SQL languages.
#Task: Convert natural language to SQL query.
#Rules:
1. Use ONLY metrics and dimensions listed in the Schema section below
2. Reference columns by their exact names shown in Schema (use the name before any ALIAS/COMMENT)
3. ALWAYS apply an aggregate function (SUM/COUNT/AVG) to metric columns unless user asks for raw detail
4. Put date filters in WHERE clause
5. DO NOT invent table names or column names — only use what's in the Schema

#Exemplars (similar question → correct SQL from memory):
{{few_shot_examples}}

#Query:
Question: {{user_question}}
Schema: {{schema_string}}
SideInfo: CurrentDate={{today}}, {{domain_terms}}
```

**Schema string format (token-efficient):**
```
Table=[sales_table]
Metrics=[<revenue ALIAS '营收/销售额' COMMENT 'total revenue = price * quantity' AGGREGATE 'SUM'>,
         <order_cnt ALIAS '订单数' COMMENT 'order count' AGGREGATE 'COUNT'>]
Dimensions=[<region ALIAS '区域' DATATYPE 'CATEGORY'>,
            <dt ALIAS '日期' DATATYPE 'DATE' FORMAT 'yyyy-MM-dd'>]
Values=[<region='华东'>, <region='华南'>]
```

> **Reference:** SuperSonic `LLMSqlParser.java`, `OnePassSCSqlGenStrategy.java`, `PromptHelper.java`

### Stage 3: S2SQL Correction

**Goal:** Fix common LLM errors automatically (no LLM needed for most).

```python
def correct_s2sql(parse_info: SemanticParseInfo, schema: SemanticSchema) -> SemanticParseInfo:
    sql = parse_info.sql_info.parsed_s2sql
    ast = parse_sql(sql)  # use sqlglot / sqlparse / JSqlParser equivalent
    
    # Corrector chain (each fixes one category of error):
    # 1. SchemaCorrector: alias → bizName, fix field names, add missing aggregations
    for field in ast.select_fields:
        if field.is_metric and not field.has_aggregation:
            field.wrap_with_agg(schema.get_default_agg(field.name))
        field.ensure_valid_name(schema)
    
    # 2. TimeCorrector: add/remove/fix date filters
    if parse_info.date_info and not ast.has_date_filter():
        ast.add_where_clause(f"dt BETWEEN '{parse_info.date_info.start}' AND '{parse_info.date_info.end}'")
    
    # 3. GrammarCorrector:
    #    - SelectCorrector: GROUP BY fields MUST be in SELECT
    #    - GroupByCorrector: aggregate queries MUST have GROUP BY for non-metric fields
    #    - AggCorrector: bare metric references get default aggregation
    
    parse_info.sql_info.corrected_s2sql = ast.to_sql()
    return parse_info
```

> **Reference:** SuperSonic `RuleSqlCorrector.java`, `GrammarCorrector.java`

### Stage 4: Semantic Translation (S2SQL → Physical SQL)

**Goal:** Replace bizName with physical column names and expressions.

```
S2SQL input:
  SELECT SUM(revenue), region FROM sales WHERE dt BETWEEN '2025-01-01' AND '2025-06-01' GROUP BY region

After replacing metric expression (revenue = price * quantity):
  SELECT SUM(price * quantity), region FROM sales ...

After replacing dimension expression (region = COALESCE(region, 'Unknown')):
  SELECT SUM(price * quantity), COALESCE(region, 'Unknown') FROM sales ...

After table resolution + dialect + LIMIT:
  SELECT SUM(s.price * s.quantity) AS revenue,
         COALESCE(s.region, 'Unknown') AS region
  FROM sales_db.sales_table s
  WHERE s.dt >= '2025-01-01' AND s.dt <= '2025-06-01'
  GROUP BY COALESCE(s.region, 'Unknown')
  LIMIT 1000
```

```python
def translate(s2sql: str, schema: SemanticSchema, db_dialect: str) -> str:
    ast = parse_sql(s2sql)
    
    # Step 1: Replace bizName with physical column name
    for field in ast.all_fields():
        element = schema.find_element(field.name)
        field.name = element.name  # "revenue" → "total_revenue_amt"
    
    # Step 2: Replace bizName with computed expression
    for field in ast.all_fields():
        element = schema.find_element(field.name)
        if element.expression:  # metric with formula like "price * quantity"
            field.replace_with_expression(element.expression)
    
    # Step 3: Resolve table name
    ast.table = f"{schema.database}.{schema.table_name}"
    
    # Step 4: Apply DB dialect
    if db_dialect == "mysql":
        ast.apply_mysql_syntax()
    elif db_dialect == "clickhouse":
        ast.apply_clickhouse_syntax()
    
    # Step 5: Add LIMIT if missing
    if not ast.has_limit():
        ast.add_limit(1000)
    
    return ast.to_sql()
```

> **Reference:** SuperSonic `DefaultSemanticTranslator.java`, `SqlQueryParser.java`

### Stage 5: Execution

```python
def execute(physical_sql: str, db_connection) -> QueryResult:
    cursor = db_connection.cursor()
    cursor.execute(physical_sql)
    columns = [ColumnInfo(name=c[0], show_type=infer_type(c)) for c in cursor.description]
    rows = [dict(zip([c.name for c in columns], row)) for row in cursor.fetchall()]
    return QueryResult(columns=columns, rows=rows)
```

### Minimum NL2SQL Checklist

- [ ] `SemanticSchema` + `DataSetSchema` + `SchemaElement` data models
- [ ] Schema mapper: trie match + embedding fallback
- [ ] LLM prompt template with: Role, Task, Rules, Exemplars, Schema, SideInfo
- [ ] Rule-based parser for simple patterns (metric+dim, metric+filter, topN)
- [ ] LLM-based parser with self-consistency (3 candidates, majority vote)
- [ ] Corrector chain: at minimum SchemaCorrector + GrammarCorrector
- [ ] Translator: bizName → physical name/expression + DB dialect + LIMIT
- [ ] JDBC/DB-API executor

---

