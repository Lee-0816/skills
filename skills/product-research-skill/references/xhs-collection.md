# Xiaohongshu Data Collection Reference

## Complete collection script (copy-paste ready)

### 1. Initialize session + interceptor

```javascript
// Run with: npx playwriter@latest -s <SESSION_ID> -e "..."

// Initialize
state.collectedNotes = [];
state.page.removeAllListeners("response");

// Set up API interceptor
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
console.log("Interceptor ready");
```

### 2. Search a keyword

```javascript
// Replace ENCODED_KEYWORD with encodeURIComponent(keyword)
await state.page.keyboard.press("Escape"); // dismiss login popup
await state.page.waitForTimeout(500);
await state.page.goto(
  "https://www.xiaohongshu.com/search_result?keyword=ENCODED_KEYWORD&source=web_search_result_notes",
  { waitUntil: "domcontentloaded" }
);
await state.page.waitForTimeout(3000);
console.log("Collected:", state.collectedNotes.length);
```

### 3. Batch search (multiple keywords in one call)

```javascript
const keywords = ["关键词1", "关键词2", "关键词3"];
for (const kw of keywords) {
  await state.page.keyboard.press("Escape");
  await state.page.waitForTimeout(500);
  await state.page.goto(
    `https://www.xiaohongshu.com/search_result?keyword=${encodeURIComponent(kw)}&source=web_search_result_notes`,
    { waitUntil: "domcontentloaded" }
  );
  await state.page.waitForTimeout(3000);
  console.log(`"${kw}" done, total: ${state.collectedNotes.length}`);
}
```

### 4. Deduplicate, rank, and output

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
console.log("Unique notes:", ranked.length);
console.log(JSON.stringify(ranked.slice(0, 30).map(n => ({
  title: n.title,
  url: n.url,
  nickname: n.nickname,
  liked: n.liked,
  collected: n.collected,
  tags: n.tag_list
})), null, 2));
```

### 5. Brand/product frequency analysis

```javascript
// Customize this array for each topic
const productKeywords = ["品牌A", "品牌B", "品牌C"];
const mentions = {};
for (const kw of productKeywords) {
  let count = 0;
  for (const n of state.collectedNotes) {
    if ((n.title && n.title.includes(kw)) || (n.desc && n.desc.includes(kw))) count++;
  }
  if (count > 0) mentions[kw] = count;
}
console.log("Brand mentions:", JSON.stringify(mentions, null, 2));
```

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Login popup blocks page | `await state.page.keyboard.press("Escape")` |
| No API data captured | Check if interceptor is set up before navigation |
| Session disconnected | `npx playwriter@latest session reset <id>` |
| Extension not connected | Click Playwriter extension icon in Chrome toolbar |
| Rate limited by XHS | Wait 5+ seconds between navigations |
| Note detail requires login | Use search API data only; don't try to open individual notes |

## XHS API response structure

Note URL format: `https://www.xiaohongshu.com/explore/{note_id}`
User profile URL format: `https://www.xiaohongshu.com/user/profile/{user_id}`

```json
{
  "data": {
    "items": [
      {
        "id": "note_id",
        "note_card": {
          "display_title": "笔记标题",
          "desc": "笔记描述",
          "type": "normal|video",
          "user": {
            "nickname": "作者昵称",
            "nick_name": "作者昵称(备用字段)"
          },
          "interact_info": {
            "liked_count": "12345",
            "collected_count": "678",
            "comment_count": "90",
            "shared_count": "12"
          },
          "tag_list": [
            { "name": "标签名" }
          ]
        }
      }
    ]
  }
}
```
