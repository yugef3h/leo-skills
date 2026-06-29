## End-to-End Example: B站创作者数据中心 ChatBI


This is a complete walkthrough from data generation to frontend display, using Bilibili (B站) creator analytics as the business scenario.

### Business Context

A ChatBI for B站 content creators (UP主) to analyze their video performance through natural language. Target users: UP主, MCN机构.

### Step 1: Data Generation (造数)

```python
# scripts/generate_bilibili_data.py
import random, sqlite3
from faker import Faker
from datetime import datetime, timedelta
import pandas as pd
import yaml, json

fake = Faker(["zh_CN"])

# ---- Dimension tables ----
categories = ["知识", "生活", "游戏", "音乐", "动画", "科技", "美食", "时尚"]
video_titles = [
    "深入理解React Hooks原理", "Vue3源码分析系列", "我的极简桌面setup",
    "某游戏新手入门攻略", "某游戏高分局解说", "翻唱某流行歌曲",
    "一周穿搭分享", "程序员日常vlog", "某动画深度解析",
    "30天学会Python", "深圳美食探店", "机械键盘评测",
]

videos = []
for i in range(1, 51):
    cat = random.choice(categories)
    videos.append({
        "video_id": i,
        "title": random.choice(video_titles) + f"#{i}",
        "category": cat,
        "duration": random.randint(60, 3600),  # 1min ~ 60min
        "publish_time": fake.date_between(start_date="-1y", end_date="-7d").isoformat(),
    })

# ---- Fact table: daily stats (core table, 10k+ rows) ----
random.seed(42)
video_stats = []
start = datetime.now() - timedelta(days=90)
for video in videos:
    base_views = random.lognormvariate(8, 1.5)  # lognormal = realistic view distribution
    for day_offset in range(90):
        date = (start + timedelta(days=day_offset)).date()
        if date.isoformat() < video["publish_time"]:
            continue
        # Daily views with random fluctuation + weekly pattern
        day_of_week = date.weekday()
        weekend_boost = 1.3 if day_of_week >= 5 else 1.0
        daily_views = max(0, int(random.gauss(0, 1) * base_views * 0.1 + base_views * 0.03 * weekend_boost))
        daily_likes = int(daily_views * random.uniform(0.03, 0.08))
        daily_coins = int(daily_views * random.uniform(0.01, 0.04))
        daily_favorites = int(daily_views * random.uniform(0.02, 0.06))
        daily_shares = int(daily_views * random.uniform(0.005, 0.02))
        daily_danmaku = int(daily_views * random.uniform(0.01, 0.05))
        daily_comments = int(daily_views * random.uniform(0.005, 0.03))

        video_stats.append({
            "video_id": video["video_id"],
            "stat_date": date.isoformat(),
            "views": daily_views,
            "likes": daily_likes,
            "coins": daily_coins,
            "favorites": daily_favorites,
            "shares": daily_shares,
            "danmaku": daily_danmaku,
            "comments": daily_comments,
        })

# ---- Fans table ----
genders = ["男", "女"]
age_groups = ["18岁以下", "18-24", "25-30", "31-40", "40岁以上"]
cities = ["北京", "上海", "广州", "深圳", "杭州", "成都", "武汉"]

fans = []
for i in range(1, 2001):
    fans.append({
        "fan_id": i,
        "gender": random.choice(genders),
        "age_group": random.choice(age_groups),
        "city": random.choice(cities),
        "follow_time": fake.date_between(start_date="-2y", end_date="today").isoformat(),
    })

# ---- Write to SQLite ----
conn = sqlite3.connect("bilibili_demo.db")
pd.DataFrame(videos).to_sql("videos", conn, index=False)
pd.DataFrame(video_stats).to_sql("video_stats", conn, index=False)
pd.DataFrame(fans).to_sql("fans", conn, index=False)
print(f"Generated: {len(videos)} videos, {len(video_stats)} stat rows, {len(fans)} fans")

# ---- Semantic model definition ----
dataset_config = {
    "datasets": [{
        "id": 1,
        "name": "B站视频数据",
        "table_name": "video_stats",
        "join_config": {
            "videos": "video_stats.video_id = videos.video_id",
            "fans": None,  # independent
        },
        "metrics": [
            {"id": 1, "biz_name": "views", "name": "views", "default_agg": "SUM",
             "alias": "播放量,播放数,观看量", "description": "视频播放总次数"},
            {"id": 2, "biz_name": "likes", "name": "likes", "default_agg": "SUM",
             "alias": "点赞,点赞数,赞", "description": "点赞总数"},
            {"id": 3, "biz_name": "coins", "name": "coins", "default_agg": "SUM",
             "alias": "投币,硬币,投币数,币", "description": "投币总数"},
            {"id": 4, "biz_name": "favorites", "name": "favorites", "default_agg": "SUM",
             "alias": "收藏,收藏数", "description": "收藏总数"},
            {"id": 5, "biz_name": "danmaku", "name": "danmaku", "default_agg": "SUM",
             "alias": "弹幕,弹幕数", "description": "弹幕总数"},
            {"id": 6, "biz_name": "comments", "name": "comments", "default_agg": "SUM",
             "alias": "评论,评论数,留言", "description": "评论总数"},
            {"id": 7, "biz_name": "interaction_rate", "name": None, "default_agg": "AVG",
             "expr": "(likes + coins + favorites) / NULLIF(views, 0)",
             "alias": "互动率,交互率", "description": "互动率 = (点赞+投币+收藏)/播放量"},
            {"id": 8, "biz_name": "coin_rate", "name": None, "default_agg": "AVG",
             "expr": "coins * 1.0 / NULLIF(views, 0)",
             "alias": "投币率,硬币率", "description": "投币率 = 投币数/播放量，B站核心质量指标"},
            {"id": 9, "biz_name": "fan_count", "name": "fan_id", "default_agg": "COUNT",
             "alias": "粉丝数,粉丝量,关注数", "description": "粉丝总数"},
        ],
        "dimensions": [
            {"id": 10, "biz_name": "category", "name": "video_stats.video_id",
             "join_table": "videos", "join_column": "category", "data_type": "CATEGORY",
             "alias": "分区,板块,类别,类型"},
            {"id": 11, "biz_name": "stat_date", "name": "stat_date", "data_type": "DATE",
             "alias": "日期,时间,天,日"},
            {"id": 12, "biz_name": "video_title", "name": "video_stats.video_id",
             "join_table": "videos", "join_column": "title", "data_type": "CATEGORY",
             "alias": "视频,稿件,内容"},
            {"id": 13, "biz_name": "duration", "name": "video_stats.video_id",
             "join_table": "videos", "join_column": "duration", "data_type": "NUMERIC",
             "alias": "时长,视频时长,长度"},
            {"id": 14, "biz_name": "gender", "name": "gender", "data_type": "CATEGORY",
             "alias": "性别,男女", "table": "fans"},
            {"id": 15, "biz_name": "age_group", "name": "age_group", "data_type": "CATEGORY",
             "alias": "年龄段,年龄,年龄分布", "table": "fans"},
            {"id": 16, "biz_name": "city", "name": "city", "data_type": "CATEGORY",
             "alias": "城市,地区,地域", "table": "fans"},
        ],
        "terms": [
            {"id": 20, "name": "完播率", "alias": "播放完成率",
             "description": "平均播放时长/视频时长，B站衡量内容质量的核心指标之一"},
            {"id": 21, "name": "爆款", "alias": "热门,火",
             "description": "通常指播放量超过10万的视频，或点赞/投币显著高于平均水平的视频"},
            {"id": 22, "name": "三连", "alias": "一键三连",
             "description": "B站特色互动行为：同时点赞+投币+收藏"},
        ],
    }]
}
yaml.dump(dataset_config, open("dataset.yaml", "w", encoding="utf-8"), allow_unicode=True)

# ---- Few-shot exemplars ----
exemplars = [
    {
        "question": "最近7天播放量趋势",
        "sql": "SELECT stat_date, SUM(views) FROM video_stats WHERE stat_date BETWEEN '{{-7d}}' AND '{{today}}' GROUP BY stat_date ORDER BY stat_date"
    },
    {
        "question": "各分区播放量排名",
        "sql": "SELECT category, SUM(views) FROM video_stats JOIN videos ON video_stats.video_id=videos.video_id GROUP BY category ORDER BY SUM(views) DESC"
    },
    {
        "question": "上个月点赞最多的5个视频",
        "sql": "SELECT title, SUM(likes) FROM video_stats JOIN videos ON video_stats.video_id=videos.video_id WHERE stat_date BETWEEN '{{last_month_start}}' AND '{{last_month_end}}' GROUP BY title ORDER BY SUM(likes) DESC LIMIT 5"
    },
    {
        "question": "我的互动率怎么样",
        "sql": "SELECT AVG((likes+coins+favorites)/NULLIF(views,0)) FROM video_stats WHERE stat_date BETWEEN '{{-7d}}' AND '{{today}}'"
    },
    {
        "question": "对比知识区和生活区的投币率",
        "sql": "SELECT category, AVG(coins*1.0/NULLIF(views,0)) FROM video_stats JOIN videos ON video_stats.video_id=videos.video_id WHERE category IN ('知识','生活') AND stat_date BETWEEN '{{-30d}}' AND '{{today}}' GROUP BY category"
    },
    {
        "question": "最近30天新增粉丝城市分布",
        "sql": "SELECT city, COUNT(*) FROM fans WHERE follow_time BETWEEN '{{-30d}}' AND '{{today}}' GROUP BY city ORDER BY COUNT(*) DESC"
    },
]
json.dump(exemplars, open("exemplars.json", "w", encoding="utf-8"), ensure_ascii=False, indent=2)

print("Done: bilibili_demo.db, dataset.yaml, exemplars.json")
```

### Step 2: What Users Can Ask (典型问答)

```
入门（规则匹配，不需要LLM）:
  "最近7天播放量趋势"           → METRIC_TREND 折线图
  "各分区播放量排名"             → METRIC_BAR 柱状图
  "上个月点赞最多的5个视频"      → TABLE 表格
  "我的互动率"                   → METRIC_CARD 大数字卡片

进阶（需要LLM）:
  "对比下知识区和生活区的投币率" → METRIC_BAR 分组柱状图
  "弹幕最多的视频，看看和收藏数有关系吗" → TABLE 明细
  "最近30天新增粉丝的城市分布"   → METRIC_PIE 饼图

追问（多轮对话）:
  Q1: "上周播放量趋势"           → 折线图，Context存{metric:views, date:上周}
  Q2: "按分区分开"               → LLM改写为"上周各分区播放量趋势"，分组折线图
  Q3: "只看知识区"               → 加filter WHERE category='知识'
  Q4: "看看互动率呢"             → metric切换为interaction_rate
```

### Step 3: Trace One Complete Query

用户输入：**"最近7天我各个分区的播放量和互动率怎么样"**

```
┌─ MAPPING (RAG取指标) ──────────────────────────────────────┐
│ jieba分词 → ["最近", "7", "天", "各个", "分区", "播放量",  │
│               "互动率", "怎么样"]                           │
│                                                             │
│ Trie匹配:                                                   │
│   "最近...天" + "7" → DateConf{start:-7d, end:today}       │
│   "分区"    → SchemaElement{category, DIMENSION, id:10}     │
│   "播放量"  → SchemaElement{views, METRIC, id:1}            │
│   "互动率"  → SchemaElement{interaction_rate, METRIC, id:7} │
│                                                             │
│ → SchemaMapInfo{dataset_1 → [3个匹配]}                     │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─ PARSING (规则匹配) ───────────────────────────────────────┐
│ 检测: 2个指标 + 1个维度 + 1个时间                          │
│ 匹配模式: METRIC_GROUPBY                                    │
│                                                             │
│ 生成 S2SQL (语义SQL，用bizName):                           │
│   SELECT SUM(views), AVG(interaction_rate), category       │
│   FROM video_stats JOIN videos ON ...                      │
│   WHERE stat_date BETWEEN '2025-06-22' AND '2025-06-29'    │
│   GROUP BY category                                         │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─ CORRECTING ───────────────────────────────────────────────┐
│ SchemaCorrector: interaction_rate是派生指标,不是物理列 ✓   │
│ AggCorrector: views→SUM(views) ✓, interaction_rate→AVG ✓  │
│ GroupByCorrector: GROUP BY category ✓                      │
│ TimeCorrector: WHERE有日期过滤 ✓                           │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─ TRANSLATING (S2SQL → 物理SQL, 不经过LLM) ────────────────┐
│ views → SUM(v.views)                                       │
│ interaction_rate → AVG(                                    │
│   (v.likes+v.coins+v.favorites)/NULLIF(v.views,0))         │
│ category → vid.category                                    │
│ video_stats → bilibili_demo.video_stats v                  │
│ videos → bilibili_demo.videos vid                          │
│                                                             │
│ → 物理SQL:                                                 │
│   SELECT vid.category,                                     │
│          SUM(v.views) AS views,                            │
│          AVG((v.likes+v.coins+v.favorites)                 │
│              /NULLIF(v.views,0)) AS interaction_rate       │
│   FROM bilibili_demo.video_stats v                         │
│   JOIN bilibili_demo.videos vid ON v.video_id=vid.video_id │
│   WHERE v.stat_date>='2025-06-22'                          │
│     AND v.stat_date<='2025-06-29'                          │
│   GROUP BY vid.category                                    │
│   LIMIT 1000                                               │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─ EXECUTE ──────────────────────────────────────────────────┐
│   category | views   | interaction_rate                    │
│   知识      | 520000  | 0.082                              │
│   生活      | 380000  | 0.065                              │
│   游戏      | 290000  | 0.071                              │
│   音乐      | 150000  | 0.093                              │
│   科技      | 120000  | 0.078                              │
│   ...                                                      │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─ POST-PROCESS ─────────────────────────────────────────────┐
│ DataInterpretProcessor (LLM):                              │
│   "最近7天总播放量146万。知识区最高(52万,占36%)，          │
│    音乐区虽然播放量最低(15万)但互动率最高(9.3%)，          │
│    说明小众分区粉丝粘性更强。环比上周整体增长12%。"        │
│                                                             │
│ MetricRatioCalcProcessor:                                  │
│   播放量环比 +12%, 互动率环比 +0.5pp                       │
│                                                             │
│ DimensionRecommendProcessor:                                │
│   推荐下钻: [视频] [粉丝性别] [年龄段]                     │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─ FRONTEND ─────────────────────────────────────────────────┐
│ getChartType() → 2个数值列 + 1个分类列 → TABLE (双指标)   │
│                                                             │
│ 显示:                                                      │
│   📊 表格: category | views | interaction_rate             │
│   📝 "最近7天总播放量146万..." (LLM解读)                   │
│   📈 环比标签: 播放量▲12%  互动率▲0.5pp                   │
│   🔍 下钻: [视频] [粉丝性别] [年龄段]                      │
│                                                             │
│ 用户操作1: 点击下钻 [视频]                                 │
│   → re-query: GROUP BY category, video_title               │
│   → 展示: 各分区×各视频的播放量和互动率明细                │
│                                                             │
│ 用户操作2: 点击指标切换 [投币率]                           │
│   → re-query: metric从interaction_rate改为coin_rate        │
│   → getChartType() → 1个数值列 → METRIC_BAR 柱状图         │
│   → 展示: 各分区投币率柱状图                               │
└─────────────────────────────────────────────────────────────┘
```

### Step 4: How to Adapt to Your Own Domain

Replace these pieces and everything else stays the same:

| 替换项 | B站例子 | 你的业务 |
|--------|---------|---------|
| 数据表 | videos, video_stats, fans | 你的业务表 |
| 指标 | views, likes, coins, interaction_rate | 你的业务指标 |
| 维度 | category, stat_date, video_title, gender, city | 你的分析维度 |
| 术语 | 完播率, 投币率, 三连, 爆款 | 你的业务术语 |
| 示例问答 | "最近7天播放量趋势" | 你的典型问题 |
| 造数逻辑 | B站视频数据分布 | 你的业务数据分布 |

**不变的：** 数据模型结构、5阶段流水线、Prompt模板、前端交互模式、插件链架构。

---

