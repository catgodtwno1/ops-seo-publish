---
name: ops-seo-publish
description: Multi-platform SEO article publishing automation. Automates article creation and publishing to Zhihu, Baijiahao, Sohu, Jianshu via browser automation. Covers editor quirks, rate limits, cover image handling, and anti-detection strategies for each platform. Trigger on "发文", "SEO发稿", "多平台发布", "publish articles", "SEO publishing".
---

# ops-seo-publish

Automate article publishing across Chinese content platforms for SEO purposes.

## Supported Platforms & Daily Limits

| Platform | Daily Limit | Editor Type | Cover Required | Notes |
|----------|------------|-------------|----------------|-------|
| 百家号 | 5 (no 原创认证) / unlimited (with) | UEditor iframe | Yes (mandatory) | Baidu SEO weight highest |
| 知乎 | ~5 (new) / 40+ (high level) | Draft.js | Optional | Triggers rate limit silently |
| 搜狐号 | 5-10 | Quill | Optional | Good Baidu indexing |
| 简书 | 2-5 (member: 5) | Draft.js | Optional | Draft.js字数显示可能不更新 |

## Chrome Multi-Profile Setup

Each platform uses a dedicated Chrome profile to isolate cookies/sessions:

```
# Launch script: ~/.chrome-profiles/launch-all.sh
# Profile → Port mapping configured in openclaw.json browser.profiles
```

Use `profile="chrome-N"` in browser tool calls to target specific platforms.

## Platform-Specific Workflows

### 百家号 (Baijiahao)

**URL**: `https://baijiahao.baidu.com/builder/rc/edit?type=news`

**Title**: Set via React-controlled textarea:
```javascript
const ta = document.querySelector('textarea.simulator'); // or textarea with className containing 'simulator'
const setter = Object.getOwnPropertyDescriptor(HTMLTextAreaElement.prototype, 'value').set;
setter.call(ta, 'YOUR TITLE');
ta.dispatchEvent(new Event('input', {bubbles: true}));
ta.dispatchEvent(new Event('change', {bubbles: true}));
```

**Content**: UEditor API (wait for editor to be ready):
```javascript
const editor = window.UE_V2.instants['ueditorInstant0'];
editor.setContent('<p>Your HTML content here</p>');
```

⚠️ **Editor init bug**: After clicking "再写一篇", the editor iframe doesn't reinitialize (`mediaFilter` error). **Must navigate to fresh URL** `https://baijiahao.baidu.com/builder/rc/edit?type=news` each time. Wait 10-12s for full load.

**Cover image** (mandatory — the critical challenge):
1. Generate cover via canvas in browser:
```javascript
const canvas = document.createElement('canvas');
canvas.width = 800; canvas.height = 450;
const ctx = canvas.getContext('2d');
ctx.fillStyle = '#2563eb'; ctx.fillRect(0, 0, 800, 450);
ctx.fillStyle = 'white'; ctx.font = 'bold 34px sans-serif';
ctx.fillText('Title Line 1', 60, 170);
```
2. Upload to Baidu's image proxy:
```javascript
canvas.toBlob(blob => {
  const fd = new FormData();
  fd.append('media', blob, 'cover.jpg');
  fetch('https://baijiahao.baidu.com/materialui/picture/uploadProxy', {
    method: 'POST', body: fd, credentials: 'include'
  }).then(r => r.json()).then(data => {
    // data.ret.bos_url = uploaded image URL
  });
}, 'image/jpeg', 0.9);
```
3. Insert `<img src="bos_url">` into article content (prepend to body)
4. Open cover dialog → "选择封面" → image auto-appears from body → click "确定"
5. Then click "发布"

**Publish flow**: Set title → Set content (with image) → Wait 3s → Open cover dialog → Confirm → Publish → Check for "提交成功"

### 知乎 (Zhihu)

**URL**: `https://zhuanlan.zhihu.com/write`

**Title**: Standard textarea with `placeholder*="标题"`:
```javascript
const ta = document.querySelector('textarea[placeholder*="标题"]');
const setter = Object.getOwnPropertyDescriptor(HTMLTextAreaElement.prototype, 'value').set;
setter.call(ta, 'YOUR TITLE');
ta.dispatchEvent(new Event('input', {bubbles: true}));
```

**Content**: Draft.js editor — **must use ClipboardEvent paste** (NOT innerHTML, NOT execCommand):
```javascript
const editor = document.querySelector('.public-DraftEditor-content');
editor.focus();
const clipboardData = new DataTransfer();
clipboardData.setData('text/html', '<p>Your HTML</p>');
clipboardData.setData('text/plain', 'fallback text');
editor.dispatchEvent(new ClipboardEvent('paste', {
  clipboardData, bubbles: true, cancelable: true
}));
```

⚠️ **Rate limiting**: After ~4 consecutive publishes, the 发布 button clicks but nothing happens (silent rate limit). Wait 24 hours before retrying. No error message displayed.

⚠️ **Login**: If session expires, Zhihu uses NetEase YiDun slider CAPTCHA — very hard to automate via CDP. Keep sessions alive by not clearing cookies.

**Publish flow**: Navigate to /write → Set title → Wait 1s → Focus editor → Paste content → Wait 2s → Click 发布 → Verify URL changes from /edit to /p/

### 搜狐号 (Sohu)

**URL**: `https://mp.sohu.com/mpfe/v3/main/new-batch-text`

**Content**: Quill editor — **innerHTML injection works**:
```javascript
document.querySelector('.ql-editor').innerHTML = '<p>Your HTML</p>';
```

**Slider CAPTCHA**: Sohu uses a slider that can be solved with Playwright drag:
```
act kind=drag startRef=slider endRef=target
```

**Daily limit**: Confirmed 5 articles per day for new accounts.

### 简书 (Jianshu)

**URL**: `https://www.jianshu.com/writer`

**Content**: Draft.js editor similar to Zhihu. Use clipboard paste or `document.execCommand('insertText')`.

⚠️ **字数显示 bug**: Word count may show 0 even after content is successfully inserted. Check actual content via `editor.innerText.length` instead.

**Publish flow**: Click article in sidebar → Write title → Write content → Click "发布" button (or "直接发布")

## Content Guidelines

- **Never mention specific company names** (e.g., "中国移动", "南瑞集团") — use "某大型央企" instead
- Target 800-1200 words per article
- Use H2 headings for structure (知乎/百家号 render well)
- Include company name in title and first paragraph for SEO
- Vary article angles: industry trends, case studies, how-to guides, comparisons

## Subagent Dispatch Pattern

For parallel publishing, spawn one subagent per platform:
```
sessions_spawn(
  task: "Platform-specific publishing task...",
  label: "平台名-发文N篇 (chrome-N)",
  model: "anthropic/claude-sonnet-4-6"  // sonnet for browser automation
)
```

⚠️ Complex editors (百家号 UEditor) work better with main agent direct control. Simpler editors (搜狐号 Quill) can be reliably delegated to subagents.

## Scheduling

For daily publishing, use OpenClaw cron:
```
openclaw cron add --label "SEO-publish-daily" --schedule "0 9 * * *" --task "..."
```

Or track in HEARTBEAT.md for heartbeat-driven execution.
