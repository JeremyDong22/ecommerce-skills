# Taobao Product Scraper Command
# v1.1 - Dynamic scraping via Chrome DevTools MCP with per-plugin browser isolation
---
description: Scrape Taobao/Tmall product data using Chrome DevTools MCP. Handles login, navigation, and intelligent data extraction with A/B test detection.
---

# Taobao Product Scraping Workflow

**FIRST ACTION**: Load the skill for domain knowledge:

```
Use Skill tool: "taobao-skill:taobao-product-scraper"
```

Read the loaded skill carefully — it contains all Taobao page structure knowledge, selectors, and extraction strategies.

---

## Prerequisites: Chrome DevTools MCP

This plugin bundles its own Chrome DevTools MCP configuration via `.mcp.json` with an isolated browser profile at `/tmp/taobao-skill-chrome-profile`. This means:

- Each plugin gets its own Chrome instance — no conflicts with other Claude Code sessions
- Taobao login sessions persist across command invocations (same profile directory)
- No manual Chrome setup required — the MCP server manages the browser lifecycle

**If Chrome DevTools MCP tools are NOT available** (e.g., `mcp__chrome-devtools__list_pages` fails), tell the user:

> Chrome DevTools MCP is not connected. This plugin includes a `.mcp.json` that configures it automatically.
> Please ensure the plugin is installed via marketplace (`/plugin install taobao-skill@taobao-skill`).
> If you already have a global `chrome-devtools` MCP configured, you may need to remove it first
> to avoid conflicts: run `claude mcp remove chrome-devtools` then restart Claude Code.

---

## Phase 1: Browser Setup

### 1.1 Check Chrome DevTools connection

```
Use mcp__chrome-devtools__list_pages
```

If connection works → go to Phase 2.

If connection fails with "browser already running" error → the profile is locked by another session. Tell the user:

> Another Chrome instance is using the same profile. Please close other Claude Code sessions using this plugin, or run:
> `rm /tmp/taobao-skill-chrome-profile/SingletonLock`

### 1.2 Confirm page is available

Use `list_pages` to find an available page. If no pages exist, use `new_page` to open one.

---

## Phase 2: Login Handling

### 2.1 Navigate to Taobao

```
Use mcp__chrome-devtools__navigate_page to https://www.taobao.com
```

### 2.2 Check login status

```
Use mcp__chrome-devtools__evaluate_script with this function:
```

```js
() => {
  const getCookie = (name) => {
    const value = `; ${document.cookie}`;
    const parts = value.split(`; ${name}=`);
    if (parts.length === 2) return parts.pop().split(';').shift();
    return null;
  };
  const nickEl = document.querySelector('.site-nav-login-info-nick');
  const dnk = getCookie('dnk');
  const tbToken = getCookie('_tb_token_');
  return {
    isLoggedIn: !!nickEl && !!dnk && !!tbToken,
    username: nickEl?.textContent?.trim() || null,
    dnk: dnk ? decodeURIComponent(dnk) : null
  };
}
```

### 2.3 If not logged in

1. Navigate to `https://login.taobao.com`
2. Tell user: "Please scan the QR code in the Chrome browser window to login."
3. Poll login status every 5 seconds using the script above
4. Once `isLoggedIn === true`, proceed

### 2.4 If redirected to login page with "快速进入" button

Take a snapshot, look for a button containing "快速进入" text, click it, wait 3 seconds.

---

## Phase 3: Ask User for Product

If the user hasn't already provided a product link, ask:

"What Taobao product do you want to scrape? Provide a product URL, short link, share text, or product ID."

Supported formats:
- Product ID: `692014283130`
- Direct URL: `https://detail.tmall.com/item.htm?id=692014283130`
- Short link: `https://e.tb.cn/h.xxx`
- Share text: `【淘宝】商品名 https://e.tb.cn/h.xxx`

---

## Phase 4: Navigate to Product Page

### 4.1 Resolve product input

**If short link** (`e.tb.cn` or `s.click.taobao.com`):
- Use `navigate_page` to the short link URL
- After navigation, the browser will redirect to the real product page
- Extract the product ID from the final URL's `id=` parameter

**If share text**:
- Extract the short link from the text using pattern: `https?://e\.tb\.cn/[A-Za-z0-9.]+`
- Follow the short link resolution above

**If direct URL or product ID**:
- Extract the numeric product ID (12-13 digits)

### 4.2 Navigate to clean product URL

Always navigate to a clean URL to avoid share link issues:
```
https://detail.tmall.com/item.htm?id={product_id}
```

Use `navigate_page` and then `wait_for` the product title to appear.

---

## Phase 5: Data Extraction (JS Priority)

### 5.1 Try JS context extraction first

Use `evaluate_script` with the comprehensive extraction function from the skill's reference document. This extracts data directly from `window.__ICE_APP_CONTEXT__` — bypassing DOM selectors entirely.

### 5.2 Detect A/B test version

Check the JS extraction result:
- **Complete version**: Has 5+ images, parameters, reviews, detail images URL
- **Simplified version**: Has 0-3 images, no parameters, no reviews

If simplified version detected:
1. Tell user: "Got the simplified A/B test version. Retrying..."
2. Reload page with `navigate_page` (type: reload)
3. Try extraction again (up to 2 retries)

### 5.3 Fallback to DOM snapshot

If JS context is unavailable or incomplete, use `take_snapshot` and extract data from the accessibility tree. Click tabs to load their content, then extract.

### 5.4 Tab navigation for additional data

Some data requires clicking tabs to load:

1. **Parameters tab** (Tab index 1): Click → extract parameters
2. **Detail images tab** (Tab index 2): Click → extract detail image URLs
3. **Reviews tab** (Tab index 0): Click → extract reviews

Use `take_snapshot` to find tab elements, `click` to switch tabs, then `evaluate_script` or `take_snapshot` to extract content.

---

## Phase 6: Results

Present all extracted data to the user in a structured format:

1. Basic info (title, price, store, product ID)
2. Product images (gallery, detail, SKU, review — as URLs)
3. Parameters table
4. SKU specifications
5. Shipping info
6. Shop details
7. Guarantees
8. Reviews
9. Q&A
10. Detail images

---

## DevTools MCP Tools Reference

| Tool | Purpose |
|------|---------|
| `list_pages` | Check connection, find available pages |
| `new_page` | Open a new browser tab |
| `navigate_page` | Go to URL, reload, back/forward |
| `wait_for` | Wait for text to appear |
| `take_snapshot` | Get page accessibility tree (find elements) |
| `take_screenshot` | Visual screenshot for debugging |
| `evaluate_script` | Run JS in page context (primary extraction method) |
| `click` | Click elements by uid |
| `fill` | Type into inputs |
| `press_key` | Keyboard shortcuts |

---

**START NOW**: Load the skill, then begin Phase 1.
