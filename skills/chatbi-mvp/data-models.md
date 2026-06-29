## Minimum Data Models (The Glue)


These models connect all 5 layers. Define equivalents in your language (Python dataclasses/dicts, TypeScript interfaces).

### SemanticSchema — metadata registry

```python
# What tables/metrics/dimensions the user can query
@dataclass
class SchemaElement:
    id: int
    biz_name: str       # logical name (e.g. "revenue"), used in S2SQL
    name: str           # physical column name (e.g. "total_revenue_amt")
    description: str    # for fuzzy matching and LLM prompt
    alias: str | None   # alternative names ("营收", "销售额")
    data_type: str      # "DATE" | "NUMERIC" | "CATEGORY"
    default_agg: str    # "SUM" | "COUNT" | "AVG" | None

@dataclass
class DataSetSchema:
    id: int
    name: str
    table_name: str             # physical table
    metrics: list[SchemaElement]     # numeric columns
    dimensions: list[SchemaElement]  # group-by columns
    terms: list[SchemaElement]       # business glossary ("GMV means...")

@dataclass
class SemanticSchema:
    datasets: dict[int, DataSetSchema]  # datasetId -> schema
```

### SchemaMapInfo — matching result (Layer 1 ↔ Layer 3 bridge)

```python
@dataclass
class SchemaElementMatch:
    element_id: int
    word: str           # the matched text span
    type: str           # "METRIC" | "DIMENSION" | "VALUE" | "TERM"
    similarity: float

# datasetId -> list of matched elements
SchemaMapInfo = dict[int, list[SchemaElementMatch]]
```

### SemanticParseInfo — parsed query (Layer 1 → 2 → 4 → 5 bridge)

```python
@dataclass
class SqlInfo:
    parsed_s2sql: str       # raw LLM output
    corrected_s2sql: str    # after schema correction
    query_sql: str          # translated to physical SQL

@dataclass
class QueryFilter:
    biz_name: str
    operator: str   # "=" | ">" | "IN" | "LIKE"
    value: str

@dataclass
class SemanticParseInfo:
    dataset_id: int
    metrics: list[SchemaElement]
    dimensions: list[SchemaElement]
    dimension_filters: list[QueryFilter]
    date_info: dict          # {"start": "2025-01-01", "end": "2025-06-01"}
    sql_info: SqlInfo
    query_mode: str          # "AGGREGATE" | "DETAIL" | "LLM_S2SQL"
    properties: dict         # extensible
```

### QueryResult — execution result (Layer 4 ↔ 5 input)

```python
@dataclass
class ColumnInfo:
    name: str
    show_type: str  # "DATE" | "NUMERIC" | "CATEGORY"

@dataclass
class QueryResult:
    columns: list[ColumnInfo]
    rows: list[dict[str, any]]
    chat_context: SemanticParseInfo | None  # for multi-turn
    text_summary: str | None               # AI interpretation
    aggregate_info: dict | None            # YoY/MoM ratios
    recommended_dimensions: list[dict]     # drill-down suggestions
```

---

