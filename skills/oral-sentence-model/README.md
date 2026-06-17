# oral-sentence-model

> 地道英语口语"句模"生成器 —— 围绕任意话题，生成 6 个高频提问 + 12 句母语级长句，标注词伙、句型、观点。

## 安装

```bash
npx skills add yugef3h/leo-skills --skill oral-sentence-model
```

## 五大能力

| 模块 | 说什么 | 做什么 |
|------|--------|--------|
| **句模生成** | "旅行句模""帮我生成一个关于XX的句模" | 6问12句 + 词伙/句型/观点标注 + 使用建议 |
| **拆解复用** | "拆解一下这句""提取词伙和句型" | 提取可复用单元，给跨话题示例 |
| **跨话题联动** | "把食物的句型借过来回答科技问题" | 将句型/词伙适配到新话题 |
| **轻量练习** | "帮我练练""出个题" | 基于句模出题、点评、复述挑战 |
| **复述法实操** | "帮我复述这句""用自己的话说一遍""retell this" | 四步复述流程，对比分析差距，逐轮内化 |

## Temperature（风格档位）

| 档位 | 适用场景 | 示例话题 |
|------|----------|----------|
| **Medium**（默认） | 日常社交，实用生动 | 食物、旅行、爱好、购物 |
| **Low** | 正式场合，专业务实 | 面试、商务、学术 |
| **High**（需主动说"脑洞大一点"） | 创意表达 | 人生哲学、未来畅想 |

## 示例

### 输入
```
帮我生成一个关于旅行的句模
```

### 输出（节选）

```markdown
# 句模：旅行 / Travel | Temperature: Medium

### 问题 1：What's your take on traveling?
> 你怎么看待旅行？

**句 1**：I wouldn't say traveling is a necessity, but there's something about
**putting yourself in** a completely **unfamiliar environment** that just
**resets your brain** in a way nothing else really can.

> 📎 词伙：`putting yourself in` / `unfamiliar environment` / `resets your brain`
> 🔄 句型：I wouldn't say [X] is a necessity, but there's something about [Y] that just [Z] in a way nothing else really can.
> 💡 观点：旅行的独特价值在于把自己放到陌生环境中，这种精神重启无可替代

### 问题 2：...
```

完整句模包含 6 个问题（中英双语）、12 句长句（词伙句中加粗）、词伙/句型/观点汇总表、使用建议。

## 核心理念

不是让用户死记硬背零散短句，而是围绕话题生成高质量长句，通过**拆解复用**和**跨话题联动**搭建可自由拼贴的口语表达生态，再通过**复述法实操**将素材内化为脱口而出的能力。句模解决"说什么"，复述法解决"怎么说出口"——两者配合，从被动输入走向主动内化。

> 参考：[史上最明晰口语提升方案](https://www.bilibili.com/video/BV1Dz4y1D7qh) | 复述法：见 `references/retelling-method.md`
