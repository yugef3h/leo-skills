## Frontend Interaction Flow (MVP 前端交互)


The ChatBI frontend is a single-page conversation interface. The user sees a chat-like UI where they type questions and receive data results as interactive charts.

### Page Layout (MVP minimum)

```
+------------------------------------------------------------------+
|  [Agent Selector]  E-commerce Sales Agent                    [?]  |
+------------------------------------------------------------------+
|                                                                    |
|  ┌──────────────────────────────────────────────────────────────┐ |
|  │  Bot: Hello! I can answer questions about sales data.       │ |
|  │  Try: "上个月销售额", "各区域订单量排名", "线上渠道趋势"     │ |
|  └──────────────────────────────────────────────────────────────┘ |
|                                                                    |
|  ┌──────────────────────────────────────────────────────────────┐ |
|  │                                       User: 上个月各区域销售额 │ |
|  └──────────────────────────────────────────────────────────────┘ |
|                                                                    |
|  ┌─ Bot ────────────────────────────────────────────────────────┐ |
|  │  ✅ Intent Parsed (0.3s)                                     │ |
|  │  Dataset: 销售订单   Mode: 聚合查询                          │ |
|  │  Metrics: [营收]   Dimensions: [区域]                        │ |
|  │  Filter: 上月 (2025-05-01 ~ 2025-05-31)                      │ |
|  │                                                               │ |
|  │  ✅ Data Queried (0.8s)                                      │ |
|  │  ┌─────────────────────────────────────────────────────────┐ │ |
|  │  │  [Bar Chart: 各区域销售额]                     [📊 📋]  │ │ |
|  │  │  ████████████████████ 华东  ¥5,200,000                   │ │ |
|  │  │  ██████████████ 华南      ¥3,800,000                     │ │ |
|  │  │  ██████████ 华北          ¥2,900,000                     │ │ |
|  │  │  ...                                                     │ │ |
|  │  └─────────────────────────────────────────────────────────┘ │ |
|  │                                                               │ |
|  │  📝 2025年5月总营收1,250万元，华东区域贡献最大（占42%）。    │ |
|  │    线上渠道增速最快（+18.5%），线下门店基本持平。            │ |
|  │                                                               │ |
|  │  🔍 Drill-down: [渠道] [品类] [时间]                         │ |
|  │  💡 Related metrics: [订单数] [客单价] [毛利率]              │ |
|  │                                                               │ |
|  │  📎 Similar questions:                                       │ |
|  │    • 各区域销售额环比变化                                     │ |
|  │    • 上季度各渠道订单量对比                                   │ |
|  └───────────────────────────────────────────────────────────────┘ |
|                                                                    |
+--------------------------------------------------------------------+
|  [💬 Type your question...                              ]  [Send]  |
|  [+ New Chat] [📜 History]                                          |
+--------------------------------------------------------------------+
```

### Core Interaction State Machine

```
User sends question
      │
      ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   PARSING    │───▶│  EXECUTING   │───▶│   DISPLAY    │
│  "意图解析中" │    │ "数据查询中"  │    │  chart/table │
│   spinner     │    │   spinner    │    │  + summary   │
└──────────────┘    └──────────────┘    └──────────────┘
      │                                       │
      ✗ (fail)                                │
      ▼                                       ▼
  "意图解析失败"                        User interacts:
  + retry button                        • drill-down click
                                        • chart type toggle
                                        • date range change
                                        • metric switch
                                            │
                                            ▼
                                     re-query with new params
                                     (skip PARSING, go to EXECUTING)
```

### API Calls from Frontend

| Step | Endpoint | What happens |
|------|----------|--------------|
| User types | `POST /api/chat/query/search` | Debounced auto-complete suggestions |
| User sends | `POST /api/chat/query/parse` | Returns `SemanticParseInfo` (metrics, dims, filters, date) |
| Auto-execute | `POST /api/chat/query/execute` | Returns `QueryResult` (columns + rows) |
| Poll summary | `POST /api/chat/query/getExecuteSummary` | LLM-generated text interpretation (polled) |
| Re-query | `POST /api/chat/query/queryData` | Drill-down / metric switch / date change |
| Similar Qs | `GET /api/chat/manage/getChatQuery/{id}` | Related questions for follow-up |

### Key Frontend Interaction Patterns

#### 1. Parse Result Display (ParseTip)

After parsing, show what the system understood BEFORE executing:
- **Dataset name** — which data source was matched
- **Query mode** — "聚合查询" (aggregate) or "明细查询" (detail)
- **Metrics** — which metrics were identified (e.g., "营收")
- **Dimensions** — which dimensions were identified (e.g., "区域")
- **Filters** — interactive controls showing current filters:
  - Date range: a date picker with presets (近7天/本月/上月/...)
  - Dimension filters: dropdowns to select specific values (e.g., region = "华东")
  - Changes to filters trigger a re-query

#### 2. Interactive Result Display (ExecuteItem → ChatMsg)

Three-part result area:
- **Top**: Chart/table toggle button (user can switch between chart and table view)
- **Middle**: Auto-selected chart (trend/bar/pie/card) with chart type toggle where applicable
- **Below chart**: 
  - LLM-generated text summary (markdown, streamed/polled)
  - Drill-down dimension chips (clickable → re-queries with added dimension)
  - Related metric suggestions (clickable → switches metric)
  - Expandable "similar questions" panel

#### 3. Drill-down Flow

```
Initial query: "各区域销售额"
  → Bar chart: [华东: 5.2M, 华南: 3.8M, 华北: 2.9M]
  → Drill-down chips shown: [渠道] [品类] [时间]

User clicks [渠道]
  → Re-query: "各区域各渠道销售额"
  → Table/Grouped bar: region × channel matrix
  → New drill-down chips: [品类] [时间] (渠道 removed, already applied)
  → "Undo drill-down" link to go back
```

#### 4. Chart Type Toggle

- Trend charts: user can switch between `line` and `bar`
- Pie charts: user can switch between `pie` and `bar`
- All chart types: user can switch to `table` view
- Switching is client-side (same data, different render) — no API call

#### 5. Conversation Management

- **New Chat**: clears context, starts fresh session
- **History sidebar**: lists past conversations, click to resume (loads message history + context)
- **State persistence**: `chatId` in URL/localStorage, survives page refresh
- **Agent switching**: different agents = different datasets + configurations

### Minimum Frontend Components

```typescript
// Component tree for MVP
<ChatPage>                          // page entry, reads agentId from URL
  <Chat>                            // main orchestrator, manages messageList state
    <ChatHeader />                  // agent name + description
    <MessageContainer>              // scrollable message list
      <AgentTip />                  // empty state: example questions
      <ChatItem>                    // one per Q&A pair
        <ParseTip />                // shows what was understood (metrics, dims, filters)
        <ExecuteItem>               // loading → result
          <ChatMsg>                 // auto-chart selector
            <MetricCard />          // single number
            <MetricTrend />         // time series (line/bar toggle)
            <BarChart />            // categorical comparison
            <PieChart />            // proportion
            <Table />               // fallback
          />
        />
        <DrillDownDimensions />     // clickable dimension chips
        <MetricOptions />           // clickable metric chips
        <SimilarQuestions />        // expandable follow-up suggestions
        <Tools />                   // export, retry, feedback
      </ChatItem>
    </MessageContainer>
    <ChatFooter>                    // input box + auto-complete + toolbar
      <AutoComplete />              // schema element suggestions while typing
    </ChatFooter>
    <ConversationSidebar />         // history list (slide-out)
    <AgentList />                   // agent switcher (if multiple agents)
  </Chat>
</ChatPage>
```

### Streaming / Polling for Text Summary

The LLM-generated interpretation (`textSummary`) is fetched separately after the query result:

```typescript
// After execute returns, start polling for summary
const pollSummary = async (queryId: string) => {
    const interval = setInterval(async () => {
        const res = await getExecuteSummary(queryId);
        if (res.queryMode !== null) {  // LLM finished
            setTextSummary(res.textSummary);
            clearInterval(interval);
        }
    }, 500);  // poll every 500ms
};
```

For a Python FastAPI backend, you can replace this polling with proper SSE (Server-Sent Events):

```python
# FastAPI SSE endpoint
async def execute_stream(query_id: str):
    async def generate():
        yield f"data: {json.dumps({'type': 'result', 'data': query_result})}\n\n"
        summary = ""
        async for chunk in llm.stream(prompt):
            summary += chunk
            yield f"data: {json.dumps({'type': 'summary_chunk', 'text': chunk})}\n\n"
        yield f"data: {json.dumps({'type': 'done'})}\n\n"
    return StreamingResponse(generate(), media_type="text/event-stream")
```

---

