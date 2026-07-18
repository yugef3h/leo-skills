---
name: pdf-editor
description: Use when the user needs to modify a PDF template — replacing variables like names, dates, amounts, or branding elements while preserving the original layout, fonts, colors, underlines, and spacing. Covers the full pipeline: analyze PDF structure (text layer vs image layer), OCR text blocks, classify each block (static label vs fill-in variable), detect underlines and mixed styles within lines, extract font sizes / colors / weights / indentation levels, build a structured doc_definition.json, and render a new PDF via a reusable engine. Applies to any image-layer-dominant PDF (offer letters, contracts, certificates, invoices, forms).
---

# PDF 模板改写技能

## 核心流程

```
PDF 原件 → 判断材质 → OCR 文本块 → 像素分类 → 关系发现 → doc_definition.json → engine.py → 新 PDF
```

## 第 0 步:判断 PDF 材质

```python
import fitz
doc = fitz.open("未知.pdf")
page = doc[0]
blocks = page.get_text("dict")["blocks"]
text_spans = sum(1 for b in blocks for l in b.get("lines",[]) for _ in l["spans"])
images = page.get_images(full=True)
print(f"文字段:{text_spans}  嵌入图:{len(images)}")
```

| 情况 | 处理策略 |
|---|---|
| 文字段多(>50)、图片少 | 文字层主导 → 直接改 PDF 文字流 |
| 文字段少(<10)、图片是大图 | 图片层主导 → 拍平→像素编辑→重建 |

## 第 1 步:渲染 + OCR

```python
DPI = 200  # A4@200DPI ≈ 1653x2338,匹配常见扫描件分辨率
mat = fitz.Matrix(DPI/72, DPI/72)
pix = page.get_pixmap(matrix=mat)
img = Image.frombytes("RGB", (pix.width, pix.height), pix.samples)
# → 喂给 OCR(macOS Vision / Tesseract / PaddleOCR)
```

OCR 产出: text, bbox(x,y,w,h), confidence

## 第 2 步:文本块分类

### 2.1 像素特征提取

对每个 OCR 块采样:
- **颜色**: `median([p for p in pixels if sum(p) < 600])` — 先过滤白背景再取中位数
- **粗细**: 水平扫描笔画宽度均值 > 4.8px @200DPI → 粗体
- **字高**: bbox 高度(近似)
- **对齐**: x 坐标相对页面中线判断 left/center/right

### 2.2 分类规则(按优先级)

```
规则 1: 字号最大 + 页面顶部 → "主标题"
规则 2: 有颜色 + 字号偏大 → "章节标题"
规则 3: x < 100 + 字号偏大 → "章节标题"
规则 4: x 在段落基准线附近 + 常规体 → "正文"
规则 5: 正文区内 + 粗体 + 有颜色 → "填入变量"
规则 6: 正文区内 + 粗体 + 无颜色 → "强调文字"
规则 7: 右对齐 + 页面底部 → "签名区"
规则 8: 有颜色文字 → 检查是否填入变量
```

## 第 3 步:关系发现

### 3.1 下划线检测

下划线 = "纵向孤立 + 横向连续"的暗像素行:

```python
for y in range(y_min, y_max):
    # 上下 2px 少暗像素 → 纵向孤立(不是文字行)
    if above_dark > 200 or below_dark > 200: continue
    # 水平连续暗段 > 80px → 下划线
    runs = find_dark_runs(gray[y], threshold=100, min_length=80)
```

### 3.2 标签-变量配对(segments 切分)

沿 x 轴逐像素扫描,颜色或粗细变化处切开:

```python
for x in range(block.x, block.x + block.w, 2):
    color = sample_vertical_stripe(img, x, y, y+h)
    bold = measure_stroke_at(img, x, y, y+h)
    if color_diff(color, prev) > 30 or bold != prev_bold:
        segments.append({ "x0": seg_start, "x1": x, ... })
```

结果:同一行拆为 [标签]/[变量]/[标签]/[变量]/[标签],每段独立样式。

### 3.3 缩进层级

所有块 x 坐标 KMeans 聚类(n=5) → section < body < list1 < list2 < indent

## 第 4 步:行距测量

水平投影 + 平滑 + 寻峰:
```python
proj = np.sum(gray < 128, axis=1)
smoothed = uniform_filter1d(proj, size=5)
peaks = find_peaks(smoothed)
gaps = [peaks[i+1] - peaks[i]]
```

## 第 5 步:产出 doc_definition.json

| 分析结果 | JSON 映射 |
|---|---|
| 纯标签行 | `text_template`、style 统一 |
| 标签+变量混合行 | `segments`,每段独立 color/font_weight/underline |
| 有下划线 | 下划线绑定到对应 segment(动态宽度) |
| 有缩进 | `draw_x` + `draw_x_first` |
| 图形/Logo | `action: "erase"` + `texture_fill` |

**segments 示例** (混合样式行):
```json
"segments": [
  { "text": "{标签前缀}", "font_weight": "regular" },
  { "text": "{变量}", "font_weight": "bold", "color": "{FILL_COLOR}",
    "underline": { "color": "{UNDERLINE_COLOR}", "width": 2 } }
]
```

## 第 6 步:渲染引擎要点

- 页面尺寸**必须**精确匹配图像像素: `pw = img.width * 72.0 / DPI`(否则亚像素缩放导致全页色偏)
- 颜色变量全部用 `{KEY}` 占位符,引擎从 config 取值
- 下划线渲染时动态计算: `x = cur_x ± margin`
- Logo 擦除用 `texture_fill`(采样周边像素填充,避免纯色补丁)
- 字体优先用系统黑体(Hiragino Sans GB),PDF 内嵌字体可能是空子集

## 第 7 步:验证

```
1. python engine.py
2. 渲染输出 PDF → OCR → 比对变量值
3. 像素扫描 → 下划线存在 + 颜色正确
4. 原版 vs 输出像素 diff → 未修改区域色偏 < 5
5. 人工目视
```

## 常见坑

| 问题 | 原因 | 解决 |
|---|---|---|
| 嵌入字体渲染空白 | CID 子集 glyph 轮廓为空 | 用 fontTools 逐字形检查 `numberOfContours`;降级系统字体 |
| 颜色采样偏暗 | 阈值太严,白色背景拉偏中位数 | 先过滤 `sum(p)<600`,再取中位数+众数验证 |
| 全页色偏 | A4 尺寸和图像像素不整除 | 页面 pt = 图像 px * 72 / DPI |
| 下划线错位 | JSON 硬编码坐标 | segments 模式,渲染时动态累加 cur_x |
| 纯色填充有接缝 | 纯色 vs JPEG 纹理不一致 | texture_fill 采样周边像素填充 |
| 同 pt 不同字体高度不同 | 字体 metrics 差异 | 实测渲染字高,遍历 pt 找匹配值 |

## 跨项目可复用模块

| 模块 | 功能 |
|---|---|
| `analyze_block(img,bbox)` | 采样式: 颜色/粗细/字高/对齐 |
| `classify_block(features)` | 规则引擎 → 类型标签 |
| `find_underlines(img,y0,y1)` | 下划线检测 |
| `segment_mixed_line(img,block)` | 沿 x 轴切分混合样式行 |
| `cluster_indent_levels(blocks)` | x 坐标聚类 → 缩进层级 |
| `measure_line_spacing(img,y0,y1)` | 水平投影+寻峰 → 行距 |
| `build_doc_definition(classified)` | 分类结果 → JSON 生成 |
| `engine.py` | 通用渲染引擎 |
| `verify.py` | OCR+像素对比验证 |
