## Data Generation Strategy (造数)


When building a ChatBI MVP, you don't have real business data. You need a script that generates:
1. A realistic database (tables + rows)
2. A semantic model definition (metrics, dimensions, terms)
3. A few-shot exemplar set (question → SQL pairs)

### Recommended Stack: Python with Faker + Pandas + SQLite

| Tool | Purpose |
|------|---------|
| `Faker` | Generate realistic names, dates, categories, numbers |
| `pandas` | Data manipulation, CSV/SQL export |
| `numpy` | Statistical distributions for numeric columns |
| `sqlite3` | Zero-config local database (MVP target DB) |
| `PyYAML` / `JSON` | Semantic model definition files |

**Why SQLite for MVP:** Zero setup, file-based, Python stdlib, sufficient for demo data. Swap to PG/MySQL later.

### What to generate

```python
# Example: E-commerce ChatBI (replace domain as needed)
# Tables:
#   - orders (order_id, date, customer_id, product_id, channel, region, amount, quantity)
#   - products (product_id, name, category, brand, unit_price)
#   - customers (customer_id, name, city, level, registration_date)

# Semantic model definition (dataset.yaml):
#   metrics:
#     - biz_name: "revenue"       name: "amount"        default_agg: "SUM"
#     - biz_name: "order_count"   name: "order_id"      default_agg: "COUNT"
#     - biz_name: "avg_price"     name: "unit_price"    default_agg: "AVG"
#   dimensions:
#     - biz_name: "region"        name: "region"        data_type: "CATEGORY"
#     - biz_name: "channel"       name: "channel"       data_type: "CATEGORY"
#     - biz_name: "date"          name: "date"          data_type: "DATE"
#     - biz_name: "category"      name: "category"      data_type: "CATEGORY"
#   terms:
#     - name: "GMV"     description: "Gross Merchandise Volume = SUM(amount)"
```

### Generation script template

```python
# scripts/generate_data.py
import random, sqlite3
from faker import Faker
import pandas as pd
import yaml

fake = Faker(["zh_CN"])  # Chinese locale for realistic names

# 1. Generate dimension tables
regions = ["华东", "华南", "华北", "华中", "西南"]
channels = ["线上旗舰店", "线下门店", "直播带货", "分销渠道"]
categories = ["手机", "电脑", "配件", "穿戴设备", "智能家居"]

products = [{"id": i, "name": fake.word().capitalize(), 
             "category": random.choice(categories),
             "unit_price": round(random.uniform(50, 5000), 2)} 
            for i in range(1, 51)]

# 2. Generate fact table (orders) — at least 10k rows for meaningful queries
orders = []
for i in range(1, 10001):
    product = random.choice(products)
    qty = random.randint(1, 5)
    orders.append({
        "order_id": i,
        "date": fake.date_between(start_date="-1y", end_date="today").isoformat(),
        "product_id": product["id"],
        "channel": random.choice(channels),
        "region": random.choice(regions),
        "amount": round(product["unit_price"] * qty * random.uniform(0.8, 1.0), 2),
        "quantity": qty,
    })

# 3. Write to SQLite
conn = sqlite3.connect("demo.db")
pd.DataFrame(orders).to_sql("orders", conn, index=False)
pd.DataFrame(products).to_sql("products", conn, index=False)

# 4. Generate semantic model definition
dataset_config = {
    "datasets": [{
        "id": 1, "name": "销售订单", "table_name": "orders",
        "database": "demo.db",
        "metrics": [
            {"id": 1, "biz_name": "revenue", "name": "amount", "default_agg": "SUM",
             "alias": "营收,销售额,收入", "description": "订单总金额"},
            {"id": 2, "biz_name": "order_cnt", "name": "order_id", "default_agg": "COUNT",
             "alias": "订单数,订单量", "description": "订单总数"},
            {"id": 3, "biz_name": "avg_amount", "name": "amount", "default_agg": "AVG",
             "alias": "客单价,均价", "description": "平均每单金额"},
        ],
        "dimensions": [
            {"id": 10, "biz_name": "region", "name": "region", "data_type": "CATEGORY",
             "alias": "区域,地区,大区"},
            {"id": 11, "biz_name": "channel", "name": "channel", "data_type": "CATEGORY",
             "alias": "渠道,销售渠道"},
            {"id": 12, "biz_name": "order_date", "name": "date", "data_type": "DATE",
             "alias": "日期,下单日期,时间"},
        ],
        "terms": [
            {"id": 20, "name": "GMV", "alias": "总交易额,成交额",
             "description": "Gross Merchandise Volume = 所有订单的amount之和"},
        ],
    }]
}
yaml.dump(dataset_config, open("dataset.yaml", "w"), allow_unicode=True)

# 5. Generate few-shot exemplars (question → SQL pairs)
exemplars = [
    {"question": "上个月总销售额是多少",  "sql": "SELECT SUM(amount) FROM orders WHERE date BETWEEN '{{last_month_start}}' AND '{{last_month_end}}'"},
    {"question": "各区域营收排名",        "sql": "SELECT region, SUM(amount) AS revenue FROM orders GROUP BY region ORDER BY revenue DESC"},
    {"question": "线上渠道最近30天日均订单量", "sql": "SELECT AVG(daily.cnt) FROM (SELECT date, COUNT(order_id) AS cnt FROM orders WHERE channel='线上旗舰店' AND date >= '{{-30d}}' GROUP BY date) AS daily"},
    # ... ~20 examples covering common query patterns
]
json.dump(exemplars, open("exemplars.json", "w"), ensure_ascii=False, indent=2)
```

### Key principles
- **10k+ rows** in the fact table for meaningful aggregation results
- **Realistic distributions**: use `random.gauss()` or `numpy.random.lognormal()` for amounts — not uniform
- **Seasonal patterns**: inject weekly/monthly trends, holidays for richer trend queries
- **Semantic model YAML is declarative**: add a new metric by editing the YAML, no code change
- **Exemplars cover patterns, not data**: use template variables (`{{last_month}}`) for date values

---

