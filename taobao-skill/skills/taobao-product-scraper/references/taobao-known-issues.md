# Taobao Known Issues Reference
# v1.0 - A/B testing, share links, URL cleaning, selector drift

## Issue 1: A/B Testing (Critical)

### Problem

Taobao non-deterministically serves two versions of product pages, even with the same URL and login state:

| Metric | Complete Version | Simplified Version |
|--------|-----------------|-------------------|
| Gallery images | 5+ images | 1-3 images |
| Detail images | 15-24 images | 0 images |
| Parameters | 10-20 items | 0 items |
| Reviews | 20+ reviews | 0 reviews |
| `pcADescUrl` | Present (URL string) | null/undefined |

### Root Cause

- Version assignment happens **server-side during SSR**
- Data is embedded in `window.__ICE_APP_CONTEXT__` before any client JS runs
- No subsequent API calls for tab content — everything is pre-loaded
- The `havana_lgc_exp` cookie may influence A/B bucket assignment
- Version can change between page loads for the same user

### Detection

After JS extraction, check the `_version` field:

```js
const isComplete = !!(
  data.item?.pcADescUrl &&
  (data.item?.images?.length || 0) >= 5 &&
  allParams.length > 0
);
```

### Handling Strategy

1. **First attempt**: Extract data normally
2. **If simplified detected**: Reload the page (`navigate_page` with type: reload)
3. **Second attempt**: Extract again after reload
4. **If still simplified**: Report to user with whatever data was obtained
5. **Maximum retries**: 2 (to avoid infinite loops)

### Potential Future Solutions (Not Implemented)

- **Cookie manipulation**: Delete/modify `havana_lgc_exp` before loading
- **API discovery**: Find the underlying data API and call it directly
- **Multiple page loads**: Open multiple tabs and pick the complete one
- **Request header manipulation**: Try different User-Agent or Referer values

---

## Issue 2: Share Links

### Problem

Product links shared from WeChat, Taobao app, or other platforms contain tracking parameters that cause different page rendering. Share links may show incomplete product data.

### Detection

A URL is a share link if it contains ANY of these query parameters:

```
shareurl, tbSocialPopKey, app, cpp, short_name,
sp_tk, tk, suid, bxsign, wxsign, un, ut_sk,
share_crt_v, sourceType, shareUniqueId
```

### Solution

After navigating to a share link:
1. Extract the product ID from the URL's `id=` parameter
2. Rebuild a clean URL: `https://detail.tmall.com/item.htm?id={product_id}`
3. Navigate to the clean URL
4. Proceed with extraction

### Short Link Resolution

Short links (`https://e.tb.cn/h.xxx`) redirect to the full product URL. With Chrome DevTools MCP:
1. Use `navigate_page` to the short link
2. Browser follows redirects automatically
3. Use `evaluate_script` to get `window.location.href` for the final URL
4. Extract product ID from the final URL

---

## Issue 3: Image URL Cleaning

### Problem

Taobao CDN adds quality reduction and resize suffixes to image URLs. Without cleaning, you get low-resolution thumbnails instead of full images.

### Cleaning Rules (Applied in Order)

```
1. .jpg_q50.jpg_.webp  →  .jpg     (quality + webp double suffix)
2. _q50.jpg_.webp      →  .jpg     (quality suffix)
3. .jpg_.webp          →  .jpg     (webp conversion)
4. .png_.webp          →  .png     (png webp conversion)
5. .jpg_100x100q50.jpg_.webp → .jpg (resize + quality)
6. _100x100q50.jpg_.webp → .jpg    (resize + quality variant)
7. .jpgq30             →  .jpg     (inline quality number)
8. _90x90q30.jpg       →  .jpg     (resize with quality)
9. _100x100.jpg        →  .jpg     (resize only)
10. Remove: _60x60, _50x50, _80x80, _90x90, _sum
```

### Placeholder Image Detection

Filter out images containing these patterns:
- `tps-2-2` in URL (2x2 pixel transparent placeholder)
- `spaceball.gif`
- Image dimensions 2x2 pixels

### Lazy Loading

Some images use alternative attributes:
- Check `data-src` if `src` is a placeholder
- Check `data-ks-lazyload` for Taobao's lazy load system
- Scrolling the page triggers lazy image loading

---

## Issue 4: Selector Drift

### Problem

Taobao periodically changes CSS class names (they appear to be generated/hashed). For example:
- `.mainTitle--R75fTcZL` could become `.mainTitle--XyZ123`
- `.text--LP7Wf49z` could become `.text--AbC789`

### Pattern

The class name format is: `{semanticName}--{hash}` where:
- `semanticName` remains stable (e.g., `mainTitle`, `shopName`, `text`)
- `hash` changes when code is redeployed

### Mitigation

1. **JS extraction first**: `window.__ICE_APP_CONTEXT__` data paths are more stable than CSS class names
2. **Snapshot fallback**: Use `take_snapshot` and look for elements by visible text content, not CSS classes
3. **Partial matching**: When searching DOM, use `[class*='mainTitle--']` instead of `.mainTitle--R75fTcZL`
4. **Role-based**: Use accessibility roles when available (buttons, links, headings)

### Stable Selectors (Less Likely to Change)

| Selector | Why Stable |
|----------|-----------|
| `#picGalleryEle` | ID-based, not class hash |
| `#J_SiteNavOpenShop` | ID-based |
| `.desc-root` | Simple class without hash |
| `.site-nav-login-info-nick` | Login system, rarely changes |

---

## Issue 5: Rate Limiting

### Symptoms

- Product page loads but shows empty data
- Redirected to CAPTCHA/verification page
- "系统繁忙" (System busy) message

### Prevention

- Don't scrape too frequently (wait 3+ seconds between product pages)
- Use persistent login session (looks like real user)
- Don't open many product pages simultaneously
- If CAPTCHA appears, tell user to solve it manually in the browser

---

## Issue 6: Mobile vs Desktop Page Versions

### Problem

Taobao serves different page structures for mobile vs desktop URLs:
- Desktop: `detail.tmall.com/item.htm?id=xxx`
- Mobile: `detail.m.tmall.com/item.htm?id=xxx`

### Solution

Always use desktop URLs. The JS context data structure documented in this skill is for the desktop version. Mobile pages have a different data structure.
