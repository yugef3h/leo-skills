## Layer 4: Data Visualization


### Chart type auto-classification (frontend, pure logic, ~50 lines)

Analyze query columns and data shape to pick the best chart:

```typescript
type ChartType = 'METRIC_CARD' | 'METRIC_TREND' | 'METRIC_BAR' | 'METRIC_PIE' | 'TABLE';

function getChartType(columns: ColumnInfo[], rows: Record<string, any>[]): ChartType {
    const dateCols = columns.filter(c => c.showType === 'DATE');
    const numCols = columns.filter(c => c.showType === 'NUMERIC');
    const catCols = columns.filter(c => c.showType === 'CATEGORY');
    const n = rows.length;

    // Single number → big number card
    if (n === 1 && numCols.length === 1 && catCols.length === 0) return 'METRIC_CARD';

    // Has date column + metric + multiple dates → time series
    if (dateCols.length > 0 && numCols.length >= 1 && hasMultipleDates(rows, dateCols[0].name))
        return 'METRIC_TREND';

    // Category + 1 metric, small dataset, all positive → pie
    if (catCols.length >= 1 && numCols.length === 1 && n <= 10 && allNonNegative(rows, numCols[0].name))
        return 'METRIC_PIE';

    // Category + 1 metric → bar
    if (catCols.length >= 1 && numCols.length === 1 && n <= 50) return 'METRIC_BAR';

    // Everything else → table
    return 'TABLE';
}
```

### Chart Rendering (use ECharts or any chart library)

| Chart Type | Library Config | Notes |
|---|---|---|
| METRIC_TREND | Line/Bar chart | xAxis=date, yAxis=metric, one series per category, user can toggle line↔bar |
| METRIC_BAR | Bar chart | xAxis=category, yAxis=metric, gradient colors |
| METRIC_PIE | Donut chart | `innerRadius: '40%', outerRadius: '70%'`, user can switch to bar |
| METRIC_CARD | Plain HTML/CSS | Large number + period-change badges |
| TABLE | HTML table | Number formatting, DATE sorting, column authorization checks |

### Interactive features (minimum)
- User can toggle chart type (line↔bar for trends, pie↔bar for pies)
- Date range picker → re-query backend
- Drill-down dimension chips → re-query with added dimension
- Metric selector → re-query with different metric
- Chart export as PNG

### Minimum Visualization Checklist

- [ ] `getChartType()` classifier function (~50 lines)
- [ ] 3 chart renderers: LineChart, BarChart, PieChart (ECharts / Recharts / Chart.js)
- [ ] MetricCard component (single value display)
- [ ] Table component (fallback)
- [ ] Chart type toggle (user can override auto-choice)

> **Reference:** SuperSonic `ChatMsg/index.tsx` (getMsgContentType), `Bar/`, `MetricTrend/`, `Pie/`, `MetricCard/` components

---

