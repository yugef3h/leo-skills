# 3-Hour ChatBI MVP Quickstart

## What You'll Build

A working ChatBI demo: type a question → get data + chart + AI summary.

```
你: "最近7天各分区播放量"
Bot: 📊 柱状图 + "知识区52万播放最高，音乐区互动率最好"
```

## What's NOT Included (by design)

No multi-turn, no chart auto-classification, no SQL corrector, no drill-down, no streaming, no agent/history UI. All of those come after the demo works.

---

## Hour 1: Backend (60 min)

### Step 1.1: Setup & Dependencies (5 min)

```bash
mkdir chatbi-mvp && cd chatbi-mvp
mkdir backend frontend
cd backend
python -m venv venv && source venv/bin/activate
pip install fastapi uvicorn langchain langchain-openai langchain-community \
           sqlglot jieba faiss-cpu sentence-transformers faker pandas
```

### Step 1.2: Generate Demo Data (10 min)

Create `backend/generate_data.py`:

```python
"""Generate Bilibili-style demo data. Run once: python generate_data.py"""
import sqlite3, random, json, yaml
from datetime import datetime, timedelta
import pandas as pd
from faker import Faker

fake = Faker(["zh_CN"])
random.seed(42)

# --- Dimension tables ---
categories = ["知识", "生活", "游戏", "音乐", "科技", "美食", "动画", "时尚"]
channels = ["首页推荐", "搜索", "关注", "动态", "外部链接"]

videos = []
for i in range(1, 31):
    cat = random.choice(categories)
    videos.append({
        "video_id": i,
        "title": f"{random.choice(['React','Vue','Python','Go','Rust'])}教程#{i}",
        "category": cat,
        "channel": random.choice(channels),
        "duration": random.randint(60, 3600),
        "publish_date": fake.date_between(start_date="-1y", end_date="-7d").isoformat(),
    })

# --- Fact table: daily stats (~5000 rows) ---
video_stats = []
start = datetime.now() - timedelta(days=60)
for v in videos:
    base_views = int(random.lognormvariate(9, 1.2))  # lognormal = realistic distribution
    for day_offset in range(60):
        date = (start + timedelta(days=day_offset)).date()
        if date.isoformat() < v["publish_date"]:
            continue
        weekend_boost = 1.3 if date.weekday() >= 5 else 1.0
        views = max(0, int(random.gauss(0, 1) * base_views * 0.08 + base_views * 0.05 * weekend_boost))
        video_stats.append({
            "video_id": v["video_id"],
            "stat_date": date.isoformat(),
            "views": views,
            "likes": int(views * random.uniform(0.03, 0.08)),
            "coins": int(views * random.uniform(0.01, 0.04)),
            "favorites": int(views * random.uniform(0.02, 0.06)),
            "danmaku": int(views * random.uniform(0.005, 0.03)),
        })

# --- Write to SQLite ---
conn = sqlite3.connect("demo.db")
pd.DataFrame(videos).to_sql("videos", conn, index=False)
pd.DataFrame(video_stats).to_sql("video_stats", conn, index=False)
print(f"Generated {len(videos)} videos, {len(video_stats)} stat rows → demo.db")

# --- Semantic model ---
model = {
    "dataset": {
        "name": "B站视频数据",
        "table": "video_stats",
        "join": {"videos": "video_stats.video_id = videos.video_id"},
    },
    "metrics": [
        {"biz_name":"views","column":"views","agg":"SUM","alias":["播放量","播放数","观看量"]},
        {"biz_name":"likes","column":"likes","agg":"SUM","alias":["点赞","点赞数"]},
        {"biz_name":"coins","column":"coins","agg":"SUM","alias":["投币","硬币","投币数"]},
        {"biz_name":"favorites","column":"favorites","agg":"SUM","alias":["收藏","收藏数"]},
        {"biz_name":"danmaku","column":"danmaku","agg":"SUM","alias":["弹幕","弹幕数"]},
        {"biz_name":"interaction_rate","column":None,"agg":"AVG",
         "expr":"(likes+coins+favorites)*1.0/NULLIF(views,0)","alias":["互动率","交互率"]},
    ],
    "dimensions": [
        {"biz_name":"category","column":"videos.category","data_type":"CATEGORY",
         "alias":["分区","板块","类别"]},
        {"biz_name":"channel","column":"videos.channel","data_type":"CATEGORY",
         "alias":["渠道","来源"]},
        {"biz_name":"stat_date","column":"video_stats.stat_date","data_type":"DATE",
         "alias":["日期","时间","天"]},
        {"biz_name":"title","column":"videos.title","data_type":"CATEGORY",
         "alias":["视频","稿件","标题"]},
    ],
    "terms": [
        {"name":"三连","description":"B站特色：同时点赞+投币+收藏"},
        {"name":"爆款","description":"播放量超过10万的视频"},
    ],
}
with open("model.yaml", "w", encoding="utf-8") as f:
    yaml.dump(model, f, allow_unicode=True)
print("Done: demo.db + model.yaml")
```

Run:
```bash
python backend/generate_data.py
```

### Step 1.3: Core Backend — Single File (30 min)

Create `backend/main.py`:

```python
"""ChatBI MVP — single-file backend. Run: uvicorn main:app --reload"""
import json, re, sqlite3, yaml, numpy as np
from datetime import datetime, timedelta
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import jieba
import faiss
from langchain_openai import ChatOpenAI
from langchain_community.embeddings import HuggingFaceEmbeddings
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
import sqlglot

# ─── Config ───
DEEPSEEK_KEY = "sk-your-deepseek-api-key"  # ← CHANGE THIS
DEEPSEEK_BASE = "https://api.deepseek.com"

app = FastAPI()
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"], allow_headers=["*"])

# ─── Init: Load model, LLM, embeddings, FAISS ───
with open("model.yaml", encoding="utf-8") as f:
    MODEL = yaml.safe_load(f)

llm = ChatOpenAI(model="deepseek-chat", openai_api_key=DEEPSEEK_KEY,
                 openai_api_base=DEEPSEEK_BASE, temperature=0.3)
embeddings = HuggingFaceEmbeddings(model_name="BAAI/bge-small-zh-v1.5",
                                   model_kwargs={"device": "cpu"})

# Build FAISS index from schema elements
schema_texts = []
schema_meta = []  # parallel list: {type, biz_name, ...}
for m in MODEL["metrics"]:
    text = f"{m['biz_name']} | {' '.join(m.get('alias',[]))} | {m.get('expr','')}"
    schema_texts.append(text)
    schema_meta.append({"type": "METRIC", **m})
for d in MODEL["dimensions"]:
    text = f"{d['biz_name']} | {' '.join(d.get('alias',[]))}"
    schema_texts.append(text)
    schema_meta.append({"type": "DIMENSION", **d})
vecs = embeddings.embed_documents(schema_texts)
faiss_index = faiss.IndexFlatIP(len(vecs[0]))
faiss_index.add(np.array(vecs).astype("float32"))

def get_conn():
    conn = sqlite3.connect("demo.db")
    conn.row_factory = sqlite3.Row
    return conn

# ─── Time parsing (simple regex) ───
def parse_time(query: str) -> tuple[str, str]:
    """"最近7天" → ('2025-06-22', '2025-06-29')"""
    today = datetime.now().date()
    m = re.search(r"最近(\d+)天", query)
    if m:
        days = int(m.group(1))
        return ((today - timedelta(days=days)).isoformat(), today.isoformat())
    m = re.search(r"上个月|上月", query)
    if m:
        end = today.replace(day=1) - timedelta(days=1)
        start = end.replace(day=1)
        return (start.isoformat(), end.isoformat())
    m = re.search(r"(\d{4}-\d{2}-\d{2}).*?(\d{4}-\d{2}-\d{2})", query)
    if m:
        return (m.group(1), m.group(2))
    return ((today - timedelta(days=30)).isoformat(), today.isoformat())  # default: 30 days

# ─── RAG: Schema recall ───
def recall(query: str) -> dict:
    """Identify metrics, dimensions, date range from NL query."""
    result = {"metrics": [], "dimensions": [], "date_range": parse_time(query)}

    # 1. Trie-like: jieba tokens → substring match against aliases
    tokens = set(jieba.lcut(query))
    for i, m in enumerate(schema_meta):
        aliases = " ".join(m.get("alias", []))
        if any(t in aliases or t in m["biz_name"] for t in tokens):
            if m["type"] == "METRIC" and m["biz_name"] not in result["metrics"]:
                result["metrics"].append(m["biz_name"])
            elif m["type"] == "DIMENSION" and m["biz_name"] not in result["dimensions"]:
                result["dimensions"].append(m["biz_name"])

    # 2. FAISS fallback: if nothing found, use semantic similarity
    if not result["metrics"]:
        q_vec = embeddings.embed_query(query)
        scores, ids = faiss_index.search(np.array([q_vec]).astype("float32"), 3)
        for s, idx in zip(scores[0], ids[0]):
            if s > 0.65:
                m = schema_meta[idx]
                if m["type"] == "METRIC" and m["biz_name"] not in result["metrics"]:
                    result["metrics"].append(m["biz_name"])
    if not result["dimensions"]:
        # Default: add category if metrics were found
        if result["metrics"]:
            result["dimensions"].append("category")

    return result

# ─── Build schema string for LLM prompt ───
def build_schema_str(recall_result: dict) -> str:
    """Format matched schema elements into prompt string."""
    parts = [f"Table=[{MODEL['dataset']['table']}], DatabaseType=[SQLite]"]
    metric_parts = []
    for m in MODEL["metrics"]:
        if m["biz_name"] in recall_result["metrics"] or not recall_result["metrics"]:
            alias_str = " ".join(m.get("alias", []))
            agg_str = f"AGGREGATE '{m['agg']}'" if m["agg"] else ""
            expr_str = f"EXPR='{m['expr']}'" if m.get("expr") else ""
            metric_parts.append(f"<{m['biz_name']} ALIAS '{alias_str}' {agg_str} {expr_str}>")
    parts.append("Metrics=[" + ",\n".join(metric_parts) + "]")

    dim_parts = []
    for d in MODEL["dimensions"]:
        if d["biz_name"] in recall_result["dimensions"] or not recall_result["dimensions"]:
            alias_str = " ".join(d.get("alias", []))
            dim_parts.append(f"<{d['biz_name']} ALIAS '{alias_str}' DATATYPE '{d['data_type']}'>")
    parts.append("Dimensions=[" + ",\n".join(dim_parts) + "]")

    return "\n".join(parts)

# ─── LLM: S2SQL generation ───
S2SQL_PROMPT = ChatPromptTemplate.from_messages([
    ("system", """#Role: You are a data analyst experienced in SQL.
#Task: Convert the user's natural language question to a SQL query.
#Rules:
1. Use ONLY metrics and dimensions from the Schema section below
2. ALWAYS wrap metric columns with their specified AGGREGATE function (SUM/COUNT/AVG)
3. Put date filters in WHERE clause using the stat_date column
4. Use JOIN when querying columns from the videos table
5. ONLY output the SQL query, nothing else — no markdown, no explanation

#Schema:
{schema}

#CurrentDate: {today}"""),
    ("user", "{question}")
])
s2sql_chain = S2SQL_PROMPT | llm | StrOutputParser()

# ─── Translate: S2SQL → Physical SQL ───
def translate(s2sql: str) -> str:
    """Replace bizName → physical column, metric expressions, table names."""
    # 1. Replace table references
    sql = s2sql.replace("video_stats", "video_stats v")
    sql = sql.replace("videos", "videos vid")
    # 2. Replace JOIN condition placeholder
    sql = sql.replace("FROM video_stats v videos vid",
                      "FROM video_stats v JOIN videos vid ON v.video_id = vid.video_id")
    # 3. Replace metric bizName → physical column + agg
    for m in MODEL["metrics"]:
        if m.get("expr"):  # derived metric: e.g., interaction_rate
            pattern = rf"\b{re.escape(m['agg'])}\s*\(\s*{re.escape(m['biz_name'])}\s*\)"
            replacement = f"{m['agg']}({m['expr']})"
            sql = re.sub(pattern, replacement, sql, flags=re.IGNORECASE)
        else:
            sql = re.sub(rf"\b{re.escape(m['biz_name'])}\b", f"v.{m['column']}", sql)
    # 4. Replace dimension bizName → physical column
    for d in MODEL["dimensions"]:
        if "videos." in d["column"]:
            col = "vid." + d["column"].split(".")[1]
        elif "video_stats." in d["column"]:
            col = "v." + d["column"].split(".")[1]
        else:
            col = d["column"]
        sql = re.sub(rf"\b{re.escape(d['biz_name'])}\b", col, sql)
    # 5. Add LIMIT if missing
    if "limit" not in sql.lower():
        sql = sql.rstrip(";") + " LIMIT 1000"
    return sql

# ─── LLM: Data summary ───
SUMMARY_PROMPT = ChatPromptTemplate.from_messages([
    ("system", """#Role: data expert communicating with business users.
#Task: Interpret the query results in 2-3 sentences.
#Rules: respond in Chinese, mention key numbers, be concise."""),
    ("user", "Question: {question}\nData (top 10 rows): {data}")
])
summary_chain = SUMMARY_PROMPT | llm | StrOutputParser()

# ─── API ───
class ChatRequest(BaseModel):
    query: str

class ChatResponse(BaseModel):
    columns: list
    rows: list
    summary: str

@app.post("/api/chat/query", response_model=ChatResponse)
async def chat_query(req: ChatRequest):
    q = req.query

    # 1. RAG: recall matched schema elements
    recalled = recall(q)
    print(f"[RAG] metrics={recalled['metrics']}, dims={recalled['dimensions']}, "
          f"date={recalled['date_range']}")

    # 2. Build schema string & LLM → S2SQL
    schema_str = build_schema_str(recalled)
    today = datetime.now().strftime("%Y-%m-%d")
    s2sql = s2sql_chain.invoke({"schema": schema_str, "today": today, "question": q})
    s2sql = s2sql.strip().removeprefix("```sql").removeprefix("```").removesuffix("```").strip()
    print(f"[S2SQL] {s2sql}")

    # 3. Translate S2SQL → Physical SQL
    physical_sql = translate(s2sql)
    # Inject date range if not already in WHERE
    start, end = recalled["date_range"]
    if "stat_date" in physical_sql and start not in physical_sql:
        physical_sql = physical_sql.replace("WHERE", f"WHERE v.stat_date >= '{start}' AND v.stat_date <= '{end}' AND ")
    print(f"[SQL] {physical_sql}")

    # 4. Execute
    conn = get_conn()
    try:
        cursor = conn.execute(physical_sql)
        columns = [{"name": d[0], "showType": "NUMERIC"} for d in cursor.description]
        rows = [dict(zip([c["name"] for c in columns], row)) for row in cursor.fetchall()]
    except Exception as e:
        print(f"[ERROR] {e}")
        return ChatResponse(columns=[], rows=[], summary=f"SQL 执行失败: {e}")
    finally:
        conn.close()

    # 5. LLM summary
    try:
        summary = summary_chain.invoke({
            "question": q,
            "data": json.dumps(rows[:10], ensure_ascii=False)
        })
    except Exception:
        summary = ""
    print(f"[SUMMARY] {summary}")

    return ChatResponse(columns=columns, rows=rows, summary=summary)

@app.get("/health")
def health():
    return {"status": "ok"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Step 1.4: Test Backend (15 min)

```bash
cd backend
# Replace DEEPSEEK_KEY in main.py first!
python main.py
```

Test in another terminal:
```bash
curl -X POST http://localhost:8000/api/chat/query \
  -H "Content-Type: application/json" \
  -d '{"query":"最近7天各分区播放量"}'
```

Expected: JSON with `columns`, `rows`, `summary`.

**Common fixes during this step:**
- If DeepSeek returns SQL with markdown backticks, the `removeprefix("```sql")` handles it
- If faiss-cpu fails on Mac M1/M2, `pip uninstall faiss-cpu && pip install faiss-cpu --no-cache-dir`
- If sentence-transformers downloads model slowly, set `HF_ENDPOINT=https://hf-mirror.com`

---

## Hour 2: Frontend (50 min)

### Step 2.1: Setup (5 min)

```bash
cd ../frontend
npm create vite@latest . -- --template react-ts
npm install
npm install axios echarts antd @ant-design/icons
npm run dev  # verify it starts
```

### Step 2.2: Replace App.tsx (35 min)

Replace `src/App.tsx` with:

```tsx
import { useState, useRef, useEffect } from 'react';
import { Input, Button, Table, Card, Spin, Typography, Space } from 'antd';
import { SendOutlined } from '@ant-design/icons';
import axios from 'axios';
import * as echarts from 'echarts';

const { TextArea } = Input;
const { Text, Title } = Typography;

const API = 'http://localhost:8000/api/chat/query';

interface ColumnInfo {
  name: string;
  showType: string;
}

interface ChatResult {
  columns: ColumnInfo[];
  rows: Record<string, any>[];
  summary: string;
}

function App() {
  const [query, setQuery] = useState('');
  const [loading, setLoading] = useState(false);
  const [result, setResult] = useState<ChatResult | null>(null);
  const [error, setError] = useState('');
  const chartRef = useRef<HTMLDivElement>(null);
  const chartInstance = useRef<echarts.ECharts | null>(null);

  const handleSend = async () => {
    if (!query.trim()) return;
    setLoading(true);
    setError('');
    setResult(null);
    try {
      const { data } = await axios.post<ChatResult>(API, { query: query.trim() });
      setResult(data);
    } catch (e: any) {
      setError(e.message || '请求失败');
    } finally {
      setLoading(false);
    }
  };

  // Render ECharts when result changes
  useEffect(() => {
    if (!result || result.rows.length === 0) return;
    if (!chartRef.current) return;

    if (!chartInstance.current) {
      chartInstance.current = echarts.init(chartRef.current);
    }
    const chart = chartInstance.current;

    // Simple heuristic: if 1 category + 1 numeric → bar chart
    const numericCols = result.columns.filter(c => c.showType === 'NUMERIC' || !['DATE','CATEGORY'].includes(c.showType));
    const catCol = result.columns.find(c => c.showType !== 'NUMERIC' && c.showType !== 'DATE') || result.columns[0];
    const metricCol = numericCols[0] || result.columns[1] || result.columns[0];

    const categories = result.rows.map(r => String(r[catCol.name] || ''));
    const values = result.rows.map(r => Number(r[metricCol.name]) || 0);

    chart.setOption({
      tooltip: { trigger: 'axis' },
      grid: { left: '3%', right: '4%', bottom: '3%', containLabel: true },
      xAxis: { type: 'category', data: categories, axisLabel: { rotate: categories.length > 5 ? 45 : 0 } },
      yAxis: { type: 'value' },
      series: [{
        name: metricCol.name,
        type: 'bar',
        data: values,
        itemStyle: {
          color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
            { offset: 0, color: '#4e86f5' },
            { offset: 1, color: '#1b4aef' },
          ]),
          borderRadius: [4, 4, 0, 0],
        },
      }],
    });

    return () => { chart.dispose(); chartInstance.current = null; };
  }, [result]);

  const columns = result?.columns.map(c => ({
    title: c.name,
    dataIndex: c.name,
    key: c.name,
  })) || [];

  return (
    <div style={{ maxWidth: 900, margin: '0 auto', padding: 24, fontFamily: '-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,"Microsoft YaHei",sans-serif' }}>
      <Title level={3} style={{ marginBottom: 24 }}>ChatBI Demo</Title>

      {/* Input */}
      <Space.Compact style={{ width: '100%', marginBottom: 24 }}>
        <TextArea
          value={query}
          onChange={e => setQuery(e.target.value)}
          onPressEnter={e => { e.preventDefault(); handleSend(); }}
          placeholder="输入问题，例如：最近7天各分区播放量"
          autoSize={{ minRows: 1, maxRows: 3 }}
          style={{ borderRadius: 8 }}
        />
        <Button type="primary" icon={<SendOutlined />} onClick={handleSend}
                loading={loading} style={{ height: 'auto', borderRadius: 8 }}>
          发送
        </Button>
      </Space.Compact>

      {/* Error */}
      {error && <Text type="danger">{error}</Text>}

      {/* Loading */}
      {loading && (
        <Card style={{ marginBottom: 16, textAlign: 'center' }}>
          <Spin tip="查询中..." />
        </Card>
      )}

      {/* Result */}
      {result && result.rows.length > 0 && (
        <>
          {/* Chart */}
          <Card style={{ marginBottom: 16 }}>
            <div ref={chartRef} style={{ width: '100%', height: 350 }} />
          </Card>

          {/* AI Summary */}
          {result.summary && (
            <Card style={{ marginBottom: 16, background: '#f5f8fb' }}>
              <Text style={{ fontSize: 14, lineHeight: 1.8 }}>{result.summary}</Text>
            </Card>
          )}

          {/* Table */}
          <Card title={`查询结果 (${result.rows.length} 行)`}>
            <Table
              columns={columns}
              dataSource={result.rows.map((r, i) => ({ ...r, _key: i }))}
              rowKey="_key"
              size="small"
              pagination={{ pageSize: 20 }}
              scroll={{ x: 'max-content' }}
            />
          </Card>
        </>
      )}

      {/* Empty result */}
      {result && result.rows.length === 0 && result.summary && (
        <Card>
          <Text>{result.summary}</Text>
        </Card>
      )}
    </div>
  );
}

export default App;
```

### Step 2.3: Add CORS & Test (10 min)

The backend already has CORS middleware. Verify:

```bash
# Terminal 1: backend
cd backend && python main.py

# Terminal 2: frontend
cd frontend && npm run dev
```

Open `http://localhost:5173`, type "最近7天各分区播放量", should see chart + table + summary.

---

## Hour 3: Polish & Demo (60 min)

### Step 3.1: Fix Common Issues (20 min)

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| "SQL 执行失败" | S2SQL has markdown backticks | Already handled in `removeprefix`. If still fails, add more `.removeprefix("```").removeprefix("sql")` |
| Chart renders wrong column | Heuristic picked wrong metric | Force `metricCol` to be the first numeric column |
| No schema elements matched | jieba tokens don't match aliases | Add more aliases to `model.yaml`, increase FAISS threshold |
| DeepSeek returns non-SQL text | Prompt not strict enough | Add "ONLY output SQL, no explanation" to system prompt |
| `faiss-cpu` import error on Mac | Architecture mismatch | `pip install faiss-cpu --no-cache-dir` |

### Step 3.2: Add 2 More Questions to Test (15 min)

Verify these all work:
```bash
curl -X POST http://localhost:8000/api/chat/query -H "Content-Type: application/json" \
  -d '{"query":"上个月各渠道点赞数排名"}'

curl -X POST http://localhost:8000/api/chat/query -H "Content-Type: application/json" \
  -d '{"query":"互动率最高的分区是哪个"}'

curl -X POST http://localhost:8000/api/chat/query -H "Content-Type: application/json" \
  -d '{"query":"知识区和游戏区的弹幕数对比"}'
```

### Step 3.3: Add Nice Touches (25 min)

If there's time, add any of these:

1. **Chart type switching** (5 min): Add a `<Select>` above the chart that switches `bar` → `line` → `table` view, all client-side (same data)

2. **Example question chips** (5 min): Add clickable chips above the input: `["各分区播放量", "点赞数排名", "互动率对比"]` → clicking fills the input

3. **Better error UX** (5 min): If `rows` is empty but `summary` is populated (knowledge Q&A), show just the card without chart/table

4. **Loading animation** (5 min): Add a 3-dot typing indicator while waiting (CSS animation)

5. **Number formatting** (5 min): Format numbers >10000 as "1.2万" in the table and chart tooltip

---

## File Structure After 3 Hours

```
chatbi-mvp/
├── backend/
│   ├── main.py              # All backend logic (~180 lines)
│   ├── generate_data.py     # Demo data generator
│   ├── model.yaml           # Semantic model definition
│   └── demo.db              # SQLite database (generated)
└── frontend/
    ├── src/
    │   └── App.tsx          # All frontend logic (~120 lines)
    ├── package.json
    └── vite.config.ts
```

## Upgrade Path: What To Do After the Demo Works

After quickstart runs, say any of these to trigger the next step. Each step is additive and independently demonstrable.

### Step 1: Fix SQL Quality (1h)

**Say:** "SQL有时候不对，帮我加纠正"

**What gets added:**
- `backend/correctors.py`: sqlglot-based S2SQL fix chain
  - `SchemaCorrector`: fix field alias → bizName, add missing aggregations
  - `GrammarCorrector`: ensure GROUP BY fields are in SELECT, add default aggregation
  - `TimeCorrector`: inject date range into WHERE clause

**Result:** LLM 生成的 SQL 错字段名/缺聚合/缺日期过滤 → 自动修。问答准确率从 60% → 85%。

---

### Step 2: Multi-turn Dialogue (1.5h)

**Say:** "加上多轮对话，能记住上下文"

**What gets added:**
- `backend/context.py`: save `SemanticParseInfo` as JSON in SQLite after each query
- MultiTurnRewriter: load history → LLM rewrites follow-up to standalone question
- Frontend: `chatId` in URL, persisted across page refresh

**Result:**
```
Q1: "上周播放量趋势" → 折线图
Q2: "按分区分开"    → LLM 自动改写为"上周各分区播放量趋势" → 分组图
Q3: "只看知识区"    → 加 filter → 只展示知识区
```

---

### Step 3: Better Visualization (1h)

**Say:** "图表不要写死柱状图，根据数据自动选类型"

**What gets added:**
- Frontend: `getChartType()` function — analyzes columns and data shape
- Components: `MetricCard` (single number), `MetricTrend` (time series, line/bar toggle), `PieChart` (donut)
- Chart type toggle: user can override auto-choice (line↔bar, pie↔bar, chart↔table)

**Result:** 趋势数据自动折线图、单数字自动大卡片、占比自动饼图。用户能手动切换。

---

### Step 4: AI Summary Streaming (1h)

**Say:** "AI解读改成流式输出"

**What gets added:**
- Backend: FastAPI `StreamingResponse` with SSE
- LangChain `llm.astream()` → yield each token as SSE event
- Frontend: `EventSource` listener → typewriter effect for summary text

**Result:** AI 解读文案一个字一个字出来，不用等全部生成完才能看。

---

### Step 5: Drill-down & Metric Switch (1.5h)

**Say:** "加上下钻和切换指标"

**What gets added:**
- Backend: `/api/chat/query/queryData` endpoint for re-query with modified params
- Frontend: `DrillDownDimensions` component — dimension chips below chart, click → re-query
- Frontend: `MetricOptions` component — related metric chips, click → switch metric
- Backend: `DimensionRecommendProcessor` — return drill-down suggestions after each query

**Result:** 图表下面有维度标签（渠道/品类/时间），点击拆分明细。指标标签切换对比。

---

### Step 6: UI Polish (1h)

**Say:** "把UI改好看点"

**What gets added:**
- CSS variables from `ui-design-system.md` (colors, shadows, border-radius)
- Message bubble styling (bot white card shadow, user blue gradient)
- Chat background gradient
- Loading dots animation
- Number formatting (10000 → "1.0万")

---

### Step 7: Better Demo Data (30min)

**Say:** "换一套数据" or "生成 XX 行业的 demo 数据"

**What to do:** Rewrite `generate_data.py` for the new domain. Tables and semantic model change, everything else stays.

---

### Quick Reference: What to Say to Get Each Feature

| 说这个 | 得到这个 |
|--------|---------|
| "SQL 有时候不对，帮我加纠正" | Step 1: S2SQL 纠正链 |
| "加上多轮对话" | Step 2: 上下文记忆 + 追问改写 |
| "图表自动选类型" | Step 3: 趋势/柱/饼/卡片自动切换 |
| "AI 解读流式输出" | Step 4: SSE 打字机效果 |
| "加上下钻和切换指标" | Step 5: 维度标签 + 指标切换 |
| "UI 改好看点" | Step 6: SuperSonic 风格 |
| "接下来优化什么" | 列出剩余的步
