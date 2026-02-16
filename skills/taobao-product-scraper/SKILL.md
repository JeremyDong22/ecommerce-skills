# Taobao Product Scraper Skill
# v1.0 - Domain knowledge for dynamic Taobao/Tmall scraping via Chrome DevTools MCP
---
name: taobao-product-scraper
description: This skill should be used when the user asks to "scrape taobao", "fetch product info from taobao/tmall", "analyze taobao product", "extract taobao data", or any task involving Taobao/Tmall product page data extraction. Provides complete domain knowledge for dynamically scraping Taobao product pages using Chrome DevTools MCP tools.
version: 1.0.0
---

# Taobao Product Scraper — Domain Knowledge

This skill contains all the knowledge you need to dynamically scrape Taobao/Tmall product pages using Chrome DevTools MCP. Unlike fixed Playwright scripts with hardcoded selectors, you will use `evaluate_script` and `take_snapshot` to **see the actual page and adapt**.

## Core Principle: JS-First Extraction

Taobao pre-loads ALL product data into `window.__ICE_APP_CONTEXT__` during server-side rendering. This means you can extract most data via a single JavaScript call — **no DOM selectors needed**.

### Primary Extraction Script

Use `mcp__chrome-devtools__evaluate_script` with this function to get all product data at once:

```js
() => {
  const ctx = window.__ICE_APP_CONTEXT__;
  if (!ctx) return { error: 'No ICE_APP_CONTEXT found' };

  const data = ctx?.loaderData?.home?.data?.res;
  if (!data) return { error: 'No product data in context' };

  const item = data.item || {};
  const price = data.price?.price || {};
  const skuBase = data.skuBase || {};
  const components = data.componentsVO || {};

  // Extract parameters
  const paramVO = data.plusViewVO?.industryParamVO || {};
  const enhanceParams = paramVO.enhanceParamList || [];
  const basicParams = paramVO.basicParamList || [];
  const allParams = [...enhanceParams, ...basicParams].map(p => ({
    name: p.key || p.title || '',
    value: p.value || p.subTitle || ''
  }));

  // Extract reviews
  const rateVO = components.rateVO || {};
  const reviewItems = rateVO.group?.items || [];
  const reviews = reviewItems.map(r => ({
    username: r.user?.nick || '',
    content: r.content || '',
    date: r.date || '',
    variant: r.sku || '',
    photos: (r.pics || []).map(p => p.url || p)
  }));

  // Extract Q&A
  const askVO = components.askVO || {};
  const qaItems = askVO.askList || [];
  const qa = qaItems.map(q => ({
    question: q.question || '',
    answer: q.answer || ''
  }));

  // Extract shop info
  const seller = data.seller || {};

  // Extract SKU specs
  const skuProps = skuBase.props || [];
  const specs = skuProps.map(prop => ({
    name: prop.name || '',
    values: (prop.values || []).map(v => ({
      name: v.name || '',
      image: v.image || null
    }))
  }));

  // Detect A/B version
  const detailUrl = item.pcADescUrl || item.descUrl || null;
  const imageCount = (item.images || []).length;
  const paramCount = allParams.length;
  const reviewCount = reviews.length;
  const isCompleteVersion = !!(detailUrl && imageCount >= 5 && paramCount > 0);

  return {
    // Version detection
    _version: isCompleteVersion ? 'complete' : 'simplified',
    _imageCount: imageCount,
    _paramCount: paramCount,
    _reviewCount: reviewCount,

    // Basic info
    title: item.title || '',
    productId: item.itemId || '',
    images: (item.images || []).map(url => {
      // Clean CDN suffixes for full resolution
      return url.replace(/\.jpg_\d+x\d+[^.]*\.jpg_\.webp$/, '.jpg')
                .replace(/_q\d+\.jpg_\.webp$/, '.jpg')
                .replace(/\.jpg_\.webp$/, '.jpg')
                .replace(/\.png_\.webp$/, '.png');
    }),

    // Price
    currentPrice: price.priceText || price.price || null,
    originalPrice: price.originPrice || null,
    priceUnit: price.unit || '¥',

    // Detail images URL (needs separate fetch if present)
    detailDescUrl: detailUrl,

    // Parameters
    parameters: allParams,

    // SKU specifications
    specifications: specs,

    // Reviews
    reviews: reviews,

    // Q&A
    qa: qa,

    // Shop info
    shop: {
      name: seller.shopName || seller.sellerNick || '',
      shopId: seller.shopId || '',
      rating: seller.goodRatePercentage || '',
    },

    // Shipping
    shipping: {
      time: data.deliveryVO?.sendTime || '',
      fee: data.deliveryVO?.freight || '',
      from: data.deliveryVO?.from || '',
      to: data.deliveryVO?.to || ''
    },

    // Guarantees
    guarantees: (data.guaranteeVO?.guaranteeItems || []).map(g => g.title || g.text || '')
  };
}
```

### What to Do With the Result

1. **Check `_version`**: If `'simplified'`, the page is serving the A/B test simplified version. Retry by reloading the page (up to 2 times).

2. **Check `detailDescUrl`**: If present, this is a URL to the detail description HTML. You can navigate to it separately or use snapshot on the detail tab to get detail images.

3. **Clean image URLs**: The extraction script already cleans CDN suffixes. If you find additional URLs from DOM, apply these patterns:
   - `.jpg_q50.jpg_.webp` → `.jpg`
   - `_q50.jpg_.webp` → `.jpg`
   - `.jpg_.webp` → `.jpg`
   - `.png_.webp` → `.png`
   - Remove size markers: `_60x60`, `_50x50`, `_80x80`, `_90x90`

---

## Fallback: DOM-Based Extraction via Snapshot

If `window.__ICE_APP_CONTEXT__` is not available (rare), fall back to `take_snapshot` and extract from the accessibility tree.

### Tab Navigation

Taobao product pages use tabs — content only loads when a tab is clicked:

| Tab Index | Name | Content |
|-----------|------|---------|
| 0 | 用户评价 (Reviews) | Customer reviews and photos |
| 1 | 参数信息 (Parameters) | Product specifications table |
| 2 | 图文详情 (Detail Images) | Product description images |
| 3 | 本店推荐 (Shop Recs) | Not needed |
| 4 | 看了又看 (Also Viewed) | Not needed |

To extract tab content:
1. `take_snapshot` to find tab elements (look for text like "评价", "参数", "详情")
2. `click` on the tab uid
3. Wait 2 seconds (use `evaluate_script` with `() => new Promise(r => setTimeout(r, 2000))`)
4. `take_snapshot` or `evaluate_script` to extract the loaded content

### Key CSS Selectors (as hints, may change)

See `references/taobao-page-structure.md` for the full selector list. These are hints — if they don't work, use `take_snapshot` to find the actual elements by their visible text or role.

---

## 10 Data Categories to Extract

| # | Category | JS Path | Fallback Selector Hint |
|---|----------|---------|----------------------|
| 1 | Title | `item.title` | `.mainTitle--*` |
| 2 | Price | `price.price.priceText` | `.text--*` near price area |
| 3 | Gallery Images | `item.images[]` | `#picGalleryEle img` |
| 4 | Detail Images | Click detail tab → `.desc-root img` | Tab index 2 |
| 5 | SKU Images | `skuBase.props[].values[].image` | `.valueItemImgWrap--* img` |
| 6 | Parameters | `plusViewVO.industryParamVO.*` | Tab index 1 content |
| 7 | Shipping | `deliveryVO.*` | `.shipping--*`, `.freight--*` |
| 8 | Shop Info | `seller.*` | `.shopName--*` |
| 9 | Guarantees | `guaranteeVO.guaranteeItems[]` | `.guaranteeText--*` |
| 10 | Reviews | `componentsVO.rateVO.group.items[]` | Tab index 0 content |
| 11 | Q&A | `componentsVO.askVO.askList[]` | `.askAnswerWrap--*` |

---

## Login Flow

See `references/taobao-login-flow.md` for detailed login handling. Key points:

1. **Detection**: Check BOTH DOM element (`.site-nav-login-info-nick`) AND cookies (`dnk` + `_tb_token_`)
2. **QR Code**: Navigate to `https://login.taobao.com`, tell user to scan QR
3. **Quick Entry**: If already logged in but needs confirmation, find and click "快速进入" button
4. **Session Persistence**: Using `--user-data-dir=/tmp/taobao-chrome-profile` keeps login across Chrome restarts

---

## Known Issues

See `references/taobao-known-issues.md` for details. Critical ones:

1. **A/B Testing**: Taobao randomly serves complete vs simplified pages. Detect via `_version` field and retry.
2. **Share Links**: Short links from WeChat contain tracking params. Always rebuild clean URL: `https://detail.tmall.com/item.htm?id={product_id}`
3. **Lazy Loading**: Some images use `data-src` instead of `src`. Scroll to trigger loading.

---

## Output Format

Present results to the user as structured data. Example:

```
## Product: [Title]
- Price: ¥[price]
- Store: [shop name]
- Product ID: [id]
- Shipping: [time], [fee], from [location]
- Guarantees: [list]

### Images
- Gallery: [N] images
- Detail: [N] images
- SKU variants: [N] images

### Parameters
| Name | Value |
|------|-------|
| ... | ... |

### Reviews ([N] total)
- [username]: [content] ([date])

### Q&A ([N] total)
- Q: [question]
- A: [answer]
```

Include image URLs so the user can view them.
