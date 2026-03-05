---
name: 'product-research'
description: 'Search Xiaohongshu, Zhihu, and other Chinese platforms for product reviews, comparisons, and tutorials, then compile structured buying guides.'
metadata:
  version: '1.1.0'
---

# Product Research Skill

Search **Xiaohongshu (小红书)**, **Zhihu (知乎)**, **SMZDM (什么值得买)** and other platforms for product reviews, comparisons, and tutorials. Automatically collect, analyze, and compile the results into a structured buying guide or knowledge report.

## Dependencies

This skill depends on the **Playwriter** skill for Xiaohongshu data collection (JS-heavy site requiring real browser). Ensure the Playwriter Chrome extension is installed and connected before using this skill.

## Workflow Overview

```
User Request (e.g., "调研新生儿奶粉推荐")
    │
    ├── Phase 1: Keyword Planning
    │     └── Generate 4-6 search keywords from different angles
    │
    ├── Phase 2: Xiaohongshu Data Collection (via Playwriter)
    │     ├── Create Playwriter session
    │     ├── Set up API response interceptor
    │     ├── Navigate to XHS search for each keyword
    │     └── Extract note metadata (title, author, likes, tags, **note URL**)
    │
    ├── Phase 3: Supplementary Web Research
    │     └── Web search for specs/data from Zhihu, SMZDM, official sites
    │     └── **Preserve source URLs** for every piece of data collected
    │
    ├── Phase 4: Analysis & Compilation
    │     ├── Deduplicate and rank by engagement
    │     ├── Extract product mentions, brand frequency
    │     ├── Cross-reference specs from multiple sources
    │     ├── **Embed reference links inline** throughout the report
    │     └── Generate structured Markdown report with references section
    │
    └── Output: Markdown buying guide saved to file (with source links)
```

## Phase 1: Keyword Planning

Given the user's topic, generate **4-6 search keywords** that cover different aspects:

| Angle | Example (for "新生儿奶粉") |
|-------|--------------------------|
| Core recommendation | `新生儿奶粉推荐` |
| How-to / tutorial | `奶粉攻略 一段` |
| Brand comparison | `婴儿奶粉怎么选 品牌排名` |
| Specs / ingredients | `一段奶粉成分对比 DHA OPO 乳铁蛋白` |
| Practical tips | `新生儿奶粉冲泡方法 注意事项` |
| User experience | `奶粉转奶 换奶粉 新生儿` |

Encode each keyword for URL: `encodeURIComponent(keyword)`.

## Phase 2: Xiaohongshu Data Collection

### Step 1: Create session and set up interceptor

```bash
npx playwriter@latest session new
```

Then set up the API response interceptor to capture search results:

```javascript
// Initialize collection array
state.collectedNotes = [];

// Set up response interceptor for XHS search API
state.page.on("response", async (res) => {
  if (res.url().includes("/api/sns/web/v1/search/notes")) {
    try {
      const body = await res.json();
      if (body.data && body.data.items) {
        for (const item of body.data.items) {
          const card = item.note_card;
          if (card) {
            state.collectedNotes.push({
              id: item.id,
              url: `https://www.xiaohongshu.com/explore/${item.id}`,
              title: card.display_title || "",
              desc: card.desc || "",
              nickname: card.user ? card.user.nickname || card.user.nick_name : "",
              type: card.type,
              liked: card.interact_info ? card.interact_info.liked_count : "0",
              collected: card.interact_info ? card.interact_info.collected_count : "0",
              comment: card.interact_info ? card.interact_info.comment_count : "0",
              tag_list: card.tag_list ? card.tag_list.map(t => t.name) : []
            });
          }
        }
      }
    } catch {}
  }
});
```

### Step 2: Navigate to search pages

For each keyword, navigate to the XHS search URL:

```javascript
const keyword = encodeURIComponent("搜索关键词");
await state.page.goto(
  `https://www.xiaohongshu.com/search_result?keyword=${keyword}&source=web_search_result_notes`,
  { waitUntil: "domcontentloaded" }
);
await state.page.waitForTimeout(3000);
```

**Important notes:**
- XHS may show a login popup. Press `Escape` to dismiss it: `await state.page.keyboard.press("Escape");`
- The API interceptor captures data even behind the login wall
- Wait 3 seconds between navigations to allow API responses to be captured
- Typically each search yields ~20-40 notes

### Step 3: Deduplicate and rank

After collecting from all keywords:

```javascript
const unique = [];
const seen = new Set();
for (const n of state.collectedNotes) {
  if (!seen.has(n.id) && n.title && n.title.length > 3) {
    seen.add(n.id);
    unique.push(n);
  }
}
const ranked = unique.sort((a, b) => parseInt(b.liked || "0") - parseInt(a.liked || "0"));
```

### Step 4: Extract brand/product mentions

Define relevant product keywords and count mentions:

```javascript
const productKeywords = ["品牌A", "品牌B", "品牌C"]; // Customize per topic
const mentions = {};
for (const kw of productKeywords) {
  mentions[kw] = ranked.filter(n => n.title.includes(kw)).length;
}
```

## Phase 3: Supplementary Web Research

Use `web_search` and `web_fetch` to supplement with structured data from:
- **知乎 (Zhihu)**: In-depth reviews and comparison articles
- **什么值得买 (SMZDM)**: Product specs and price tracking
- **Official product pages**: Verified specifications and pricing

Search strategy:
```
site:zhihu.com OR site:smzdm.com OR site:sohu.com [product category] [specs keywords]
```

**Note:** Zhihu and SMZDM often have anti-scraping measures. Use `web_search` snippets as primary data source when `web_fetch` is blocked.

### Source URL Tracking

Maintain a `sources` list throughout Phase 3. For every fact, spec, or data point collected, record:

```javascript
// Example structure to maintain during research
state.webSources = [];
// After each web_search / web_fetch, append:
state.webSources.push({
  title: "23款国行版1段奶粉横评",      // Article title
  url: "https://zhuanlan.zhihu.com/p/...", // Source URL
  platform: "知乎",                       // Platform name
  data_used: "乳铁蛋白含量对比"           // What data was extracted
});
```

## Phase 4: Report Compilation

### Reference Link Rules

**Every recommendation, data point, and tutorial in the report must include a reference link when a source is available.** Follow these rules:

1. **XHS note references**: Use `[笔记标题](note_url)` or `[作者名](note_url)` inline.
2. **Web article references**: Use `[来源标题](url)` inline near the data point.
3. **Brand/product recommendation tables**: Add a "来源" (Source) column or embed links in the product name.
4. **Tutorial/how-to sections**: Link to the original note or article that the tutorial is based on.
5. **Specs/ingredient data**: Footnote or inline-link the data source.
6. **Blogger recommendations**: Link to their XHS profile (`https://www.xiaohongshu.com/user/profile/{user_id}`) or representative note.

### Inline Citation Format

Use one of these formats depending on context:

```markdown
<!-- Option A: Inline link on the data point -->
DHA 含量达到 6.16mg/100kJ（[来源](https://...)）

<!-- Option B: Link on the author/note title -->
根据 [老勋的横评笔记](https://www.xiaohongshu.com/explore/xxx)，这款奶粉排名第一

<!-- Option C: Table with embedded links -->
| 品牌 | DHA | 参考来源 |
|------|-----|---------|
| [爱他美卓萃](https://...) | 5.1mg/100kJ | [测评笔记](https://...) |

<!-- Option D: Section-level source attribution -->
> 以下内容参考自 [23款1段奶粉横评](https://...) 及 [老勋的品牌天花板合集](https://...)
```

### Output Structure

The final Markdown report should include:

```markdown
# [Topic] 攻略

> 基于小红书 N 篇高赞笔记 + 知乎/什么值得买专业评测整理

## 一、基础知识
(Key concepts a beginner needs to know)

## 二、品牌/产品推荐（按热度排名）
### 第一梯队
(Table: brand with link, highlights, price, source link)
### 第二梯队
(Table: brand with link, highlights, price, source link)

## 三、核心参数/成分对比表
> 数据来源：[来源A](url), [来源B](url)
(Detailed spec comparison table, each data point traceable)

## 四、按需求速选指南
(Table: need → recommended product → reason → reference)

## 五、实用教程/操作指南
> 参考自 [笔记标题](xhs_url)（N赞）
(Step-by-step how-to from high-engagement notes, with source links)

## 六、常见问题与避坑
(FAQ compiled from note comments and tags, with source links)

## 七、购买渠道与省钱建议

## 八、推荐关注的账号
(Table: [account name](profile_or_note_url), specialty, engagement)

## 九、参考来源
### 小红书笔记
| # | 笔记标题 | 作者 | 点赞 | 链接 |
|---|---------|------|------|------|
| 1 | 笔记标题 | 作者 | N万赞 | [链接](https://www.xiaohongshu.com/explore/xxx) |

### 其他平台
| # | 标题 | 平台 | 链接 |
|---|------|------|------|
| 1 | 文章标题 | 知乎/什么值得买 | [链接](https://...) |
```

### Quality Standards

- **Data-driven**: Every recommendation must be backed by engagement data (likes, collects)
- **Multi-source**: Cross-reference XHS user opinions with professional reviews
- **Actionable**: Include specific product names, prices, and where to buy
- **Beginner-friendly**: Explain jargon, include glossary tables
- **Honest**: Note data collection date and that recommendations are based on user popularity, not ads
- **Traceable**: Every key data point, recommendation, or tutorial must link back to its source

## Example Usage

User: "帮我调研小红书上关于露营装备的推荐，生成一个新手露营攻略"

Keywords to generate:
1. `露营装备推荐 新手入门`
2. `露营帐篷怎么选 品牌`
3. `露营必备清单 装备`
4. `露营炊具 睡袋 推荐`
5. `露营注意事项 避坑`

Then follow Phase 2 → 3 → 4 to produce a complete guide.

## Tips

- **Rate limiting**: Keep 3+ seconds between XHS page navigations
- **Session recovery**: If XHS blocks, `playwriter session reset <id>` and retry
- **Large topics**: Split into sub-topics and research each independently
- **Freshness**: Note data timestamps; XHS content trends change fast
- **Privacy**: Never collect or store user personal information, only public note metadata
