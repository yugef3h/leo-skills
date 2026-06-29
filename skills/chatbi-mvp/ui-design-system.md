## UI Design System (from Supersonic)


Replicating Supersonic's look and feel. The design is clean, modern, with soft shadows, rounded corners, and a blue+orange accent palette. Copy the CSS variables below directly into your React project.

### Color Palette

```css
:root {
  /* ===== Primary Accents ===== */
  --chat-blue: #1b4aef;           /* Primary interactive: buttons, links, active states */
  --primary-green: #31c462;       /* Metric indicators underline */
  --primary-orange: #f87653;      /* Secondary accent / branding */

  /* ===== Semantic Colors ===== */
  --green: #00d59c;               /* Success states, step checkmarks */
  --error-color: #ff4d4f;         /* Error states, failure icons */
  --warning-color: #faad14;       /* Warning */
  --link-color: #3a64ff;          /* Links */
  --highlight-color: #ff4d4f;     /* Highlight / emphasis */

  /* ===== Text Colors ===== */
  --text-color: #181a1a;          /* Primary text */
  --text-color-secondary: #3d4242;/* Secondary text */
  --text-color-third: #626a6a;    /* Tertiary / labels */
  --text-color-fourth: #889191;   /* Quaternary / placeholders / dates */

  /* ===== Backgrounds ===== */
  --body-background: #f7fafa;     /* Page background */
  --component-background: #fff;   /* Card / component bg */
  --light-background: #f5f5f5;    /* Input disabled bg */
  --header-color: #edf2f2;        /* Table header bg */

  /* ===== Borders ===== */
  --border-color-base: #e1e6e6;   /* Default border */

  /* ===== Trend indicators ===== */
  --trend-up: rgb(252, 103, 114);   /* red-ish: values going up */
  --trend-down: rgb(45, 202, 147);  /* green: values going down */

  /* ===== Chart Colors (12-color palette) ===== */
  --chart-1: #1b4aef;  --chart-2: #31c462;  --chart-3: #f87653;
  --chart-4: #ffb924;  --chart-5: #00d59c;  --chart-6: #ff4d4f;
  --chart-7: #3a64ff;  --chart-8: #ff7800;  --chart-9: #446dff;
  --chart-10: #ff8193; --chart-11: #00b354; --chart-12: #c20c0c;
}
```

### Typography

```css
body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial,
               'Microsoft YaHei', 'Noto Sans', sans-serif;
  font-size: 14px;
  -webkit-font-smoothing: antialiased;
}

/* Size scale */
/* 40px  — MetricCard value */
/* 30px  — Trend indicator value */
/* 20px  — Aggregate indicator */
/* 16px  — Page/section titles */
/* 14px  — Body text, chat messages (default) */
/* 13px  — Secondary text, tooltips, filter labels */
/* 12px  — Timestamps, date picker text */
```

### Border Radius

| Size | Where |
|------|-------|
| `2px` | Metric fields, drill-down chips |
| `4px` | Code blocks, SQL display, filter selects |
| `6px` | Select controls, auto-complete dropdown |
| `8px` | Input area, agent list items |
| `10px` | Tags, small cards |
| `12px` | **Message bubbles, chat cards, table header** ← most common |
| `16px` | Parse tip option cards |
| `50%` | Avatars, send button, floating action button |

### Shadows

```css
/* Card / bubble shadow — the signature look */
--card-shadow: 0 2px 4px rgba(0, 0, 0, 0.14), 0 0 2px rgba(0, 0, 0, 0.12);

/* Input focus glow */
--input-focus-shadow: rgb(74, 114, 245) 0px 0px 3px;

/* Floating button */
--float-shadow: 8px 8px 20px 0 rgba(55, 99, 170, 0.1);
--float-shadow-hover: 8px 8px 20px rgba(55, 99, 170, 0.3);
```

### Key Component Recipes

**1. Bot Message Bubble**
```css
background: rgba(255, 255, 255, 0.8);
border: 1px solid transparent;
border-radius: 12px;
padding: 8px 16px 10px;
box-shadow: 0 2px 4px rgba(0,0,0,0.14), 0 0 2px rgba(0,0,0,0.12);
```

**2. User Message Bubble**
```css
background: linear-gradient(81.62deg, #2870ea 8.72%, var(--chat-blue) 85.01%);
border-radius: 12px;
padding: 8px 16px;
color: #fff;
box-shadow: 0 2px 4px rgba(0,0,0,0.14), 0 0 2px rgba(0,0,0,0.12);
```

**3. Parse/Execute Card (step container)**
```css
background: #fff;
border: 1px solid transparent;
border-radius: 12px;
padding: 12px 20px 12px 16px;
box-shadow: 0 2px 4px rgba(0,0,0,0.14), 0 0 2px rgba(0,0,0,0.12);

/* Left border indicator for successful step */
.step-success {
  border-left: 1px solid var(--green);  /* #00d59c */
}
.step-error {
  border-left: 1px solid var(--error-color);  /* #ff4d4f */
}
```

**4. Chart Container**
```css
background: #f5f8fb;
border: 1px solid var(--border-color-base);  /* #e1e6e6 */
border-radius: 4px;
padding: 6px 14px 8px;
```

**5. Input Area (Composer)**
```css
.ant-select-selector {
  background: #f9f9f9;
  border: 0;
  border-radius: 8px;
  font-size: 14px;
}
.ant-select-focused .ant-select-selector {
  box-shadow: rgb(74, 114, 245) 0px 0px 3px;
}

/* Send button — round, 30×30px */
background: #b8b8bf;     /* inactive */
background: var(--chat-blue);  /* active #1b4aef */
color: #fff;
border-radius: 50%;
```

**6. Loading Dots (typing indicator)**
```css
width: 4px; height: 4px;
background: var(--text-color);
border-radius: 50%;
animation: dotPulse 1s ease-in-out infinite;
@keyframes dotPulse {
  0%, 100% { opacity: 0; transform: scale(0.5); }
  50%      { opacity: 1; transform: scale(1); }
}
```

**7. Table**
```css
/* Header */
background: #edf2f2;
color: #666;
font-size: 13px;
border-radius: 12px 12px 0 0;
/* Body cells */
padding: 12px 2px;
font-size: 14px;
/* Even row */
background-color: #fbfbfb;
```

**8. Chart Type Toggle / Metric Chips**
```css
/* Inactive chip */
padding: 6px 14px;
border-radius: 16px;
border: 1px solid var(--border-color-base);
/* Active chip */
border-color: var(--chat-blue);
color: var(--chat-blue);
```

### Layout Dimensions

```
┌────────────────────────────────────────────────────┐
│ AgentList (248px) │ ChatArea (flex:1) │ History    │
│ optional sidebar  │                   │ (248px)    │
│ bg: #f9f9f9       │ bg: gradient       │ optional   │
│ border-r: #f1f1f1 │                    │            │
└────────────────────────────────────────────────────┘

Chat Area internals:
  MessageContainer padding: 70px 20px 60px 14px
  ChatFooter margin:      6px 20px 20px
  Input height:           70px
  Message avatar:         40×40px, border-radius:50%
  ChatHeader:             14px 16px, bg: rgba(243,243,247,0.85), backdrop-filter: blur(2px)
```

### Chat Background Gradient

```css
background: 
  linear-gradient(180deg, rgba(23,74,228,0) 29.44%, rgba(23,74,228,0.06) 100%),
  linear-gradient(90deg, #f3f3f7 0%, #f3f3f7 20%, #ebf0f9 60%, #f3f3f7 80%, #f3f3f7 100%);
```

### Icons

Use `@ant-design/icons` (React) or `lucide-react` for the MVP. Key icons needed:
- `CaretRightOutlined` / `Play` — loading/executing indicator
- `CheckCircleFilled` / green check — success state
- `CloseCircleFilled` / red X — error state
- `ReloadOutlined` / refresh — retry button
- `DownloadOutlined` — export
- `SwapOutlined` — chart type toggle
- `InfoCircleOutlined` — info tooltip
- `LoadingOutlined` — spinner

### Chart Theme (ECharts)

```js
// Apply consistently across all charts
const chartColors = ['#1b4aef','#31c462','#f87653','#ffb924','#00d59c',
                     '#ff4d4f','#3a64ff','#ff7800','#446dff','#ff8193'];

// Bar chart: use LinearGradient fill
itemStyle: {
  color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
    { offset: 0, color: '#4e86f5' },
    { offset: 1, color: '#1b4aef' },
  ]),
  borderRadius: [4, 4, 0, 0],
}

// Pie chart: donut style
radius: ['40%', '70%']
```

---

