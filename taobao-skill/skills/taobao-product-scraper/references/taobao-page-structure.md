# Taobao Page Structure Reference
# v1.0 - CSS selectors, tab navigation, JS context data paths, image URL cleaning

## JS Context Data Structure

All product data is available at `window.__ICE_APP_CONTEXT__.loaderData.home.data.res`:

```
res
├── item
│   ├── title (string) - Product title
│   ├── itemId (string) - Product ID
│   ├── images (string[]) - Gallery image URLs
│   ├── pcADescUrl (string) - Detail description page URL (null in simplified version)
│   └── descUrl (string) - Mobile description URL
├── price
│   └── price
│       ├── priceText (string) - Current price
│       ├── originPrice (string) - Original price
│       └── unit (string) - Currency unit
├── skuBase
│   ├── props (array) - SKU property groups
│   │   ├── [].name (string) - Property name (e.g., "颜色")
│   │   └── [].values (array)
│   │       ├── [].name (string) - Value name (e.g., "黑色")
│   │       └── [].image (string) - SKU variant image URL
│   └── skus (array) - Individual SKU combos with prices
├── seller
│   ├── shopName (string)
│   ├── sellerNick (string)
│   ├── shopId (string)
│   └── goodRatePercentage (string)
├── deliveryVO
│   ├── sendTime (string) - e.g., "48小时内发货"
│   ├── freight (string) - e.g., "快递: 免运费"
│   ├── from (string) - Ship from location
│   └── to (string) - Ship to location
├── guaranteeVO
│   └── guaranteeItems (array)
│       └── [].title (string) - e.g., "假一赔四", "7天无理由退换"
├── plusViewVO
│   └── industryParamVO
│       ├── enhanceParamList (array) - Key parameters
│       │   ├── [].key (string) - Param name
│       │   └── [].value (string) - Param value
│       └── basicParamList (array) - Basic parameters
│           ├── [].key (string)
│           └── [].value (string)
├── componentsVO
│   ├── tabVO
│   │   └── tabList (array) - Available tabs
│   ├── rateVO
│   │   └── group
│   │       └── items (array) - Reviews
│   │           ├── [].user.nick (string)
│   │           ├── [].content (string)
│   │           ├── [].date (string)
│   │           ├── [].sku (string)
│   │           └── [].pics (array) - Review photos
│   └── askVO
│       └── askList (array) - Q&A items
│           ├── [].question (string)
│           └── [].answer (string)
```

---

## CSS Selector Hints

These selectors are from November 2025. Taobao may change class names at any time. If a selector doesn't match, use `take_snapshot` to find the element by visible text or structure.

### Basic Info
| Element | Selector | Notes |
|---------|----------|-------|
| Title | `.mainTitle--R75fTcZL` | Product title heading |
| Store Name | `#J_SiteNavOpenShop` | Top nav store link |
| Price (current) | `.text--LP7Wf49z` | Price number |
| Price (unit) | `.unit--zM7V7E0w` | Currency symbol |
| Price (alt) | `.text--Do8Zgb3q` | Alternative price selector |

### Images
| Element | Selector | Notes |
|---------|----------|-------|
| Gallery container | `#picGalleryEle` | Left-side thumbnail gallery |
| Gallery (alt) | `.picGallery--qY53_w0u` | Alternative gallery selector |
| Thumbnail images | `.thumbnailPic--QasTmWDm` | Individual gallery thumbnails |
| Main image | `.mainPic--zxTtQs0P` | Large displayed image |
| SKU variant images | `.valueItemImgWrap--ZvA2Cmim img` | Color/style option images |
| Detail images | `.desc-root img` | Product detail description images |

### Shop
| Element | Selector |
|---------|----------|
| Shop name | `.shopName--cSjM9uKk` |
| Shop link | `.detailWrap--svoEjPUO` |
| Shop rating | `.StoreComprehensiveRating--If5wS20L` |
| Rating items | `.storeLabelItem--IcqpWWIy` |

### Shipping
| Element | Selector |
|---------|----------|
| Shipping time | `.shipping--Obxoxza7` |
| Freight fee | `.freight--oatKHK1s` |
| Delivery location | `.deliveryAddrWrap--KgrR00my span` |

### Guarantees
| Element | Selector |
|---------|----------|
| Container | `.GuaranteeInfo--OYtWvOEt` |
| Text items | `.guaranteeText--hqmmjLTB` |

### SKU Specs
| Element | Selector |
|---------|----------|
| SKU group | `.skuItem--Z2AJB9Ew` |
| SKU label | `.ItemLabel--psS1SOyC` |
| Value items | `.valueItem--smR4pNt4` |
| Stock status | `.quantityTip--zL6BCu6j` |

### Tabs
| Element | Selector |
|---------|----------|
| Tab items | `.tabTitleItem--z4AoobEz` |
| Tab by index | `.tabTitleItem--z4AoobEz:nth-child(N+1)` |

### Reviews
| Element | Selector |
|---------|----------|
| Container | `.comments--ChxC7GEN` |
| Review item | `.Comment--H5QmJwe9` |
| Username | `.userName--KpyzGX2s` |
| Content | `.content--uonoOhaz` |
| Meta (date/variant) | `.meta--PLijz6qf` |
| Photo | `.photo--ZUITAPZq` |

### Parameters
| Element | Selector |
|---------|----------|
| Emphasis param item | `.emphasisParamsInfoItem--H5Qt3iog` |
| Emphasis title | `.emphasisParamsInfoItemTitle--IGClES8z` |
| Emphasis value | `.emphasisParamsInfoItemSubTitle--Lzwb8yjJ` |
| General param item | `.generalParamsInfoItem--qLqLDVWp` |
| General title | `.generalParamsInfoItemTitle--Fo9kKj5Z` |
| General value | `.generalParamsInfoItemSubTitle--S4pgp6b9` |

### Q&A
| Element | Selector |
|---------|----------|
| Container | `.askAnswerWrap--SOQkB8id` |
| Item | `.askAnswerItem--RJKHFPmt` |
| Question | `.questionText--cClStSfJ` |
| Answer | `.answer--GB6EGprf` |

---

## Tab Navigation

| Index | Tab Name | Content | When to Click |
|-------|----------|---------|---------------|
| 0 | 用户评价 | Customer reviews | Need reviews |
| 1 | 参数信息 | Product parameters | Need specs |
| 2 | 图文详情 | Detail description images | Need detail images |
| 3 | 本店推荐 | Shop recommendations | Skip |
| 4 | 看了又看 | Similar products | Skip |

Tab selector pattern: `.tabTitleItem--z4AoobEz:nth-child({index + 1})`

After clicking a tab, wait ~2 seconds for content to load before extracting.

---

## Image URL Cleaning Rules

Taobao CDN adds quality/size suffixes that reduce image resolution. Remove them:

| Pattern | Replace With | Example |
|---------|-------------|---------|
| `.jpg_q50.jpg_.webp` | `.jpg` | Full quality JPEG |
| `_q50.jpg_.webp` | `.jpg` | Remove quality suffix |
| `.jpg_.webp` | `.jpg` | Remove webp conversion |
| `.png_.webp` | `.png` | Preserve PNG format |
| `.jpg_100x100q50.jpg_.webp` | `.jpg` | Remove resize+quality |
| `_60x60`, `_50x50`, etc. | (remove) | Remove size markers |
| `.jpgq30` | `.jpg` | Remove quality number |

Also check for `data-src` and `data-ks-lazyload` attributes on images — these hold the real URL for lazy-loaded images. Filter out placeholder images containing `tps-2-2` in the URL.

---

## Link Formats

| Format | Pattern | Example |
|--------|---------|---------|
| Direct URL | `detail.tmall.com/item.htm?id=NNN` | `https://detail.tmall.com/item.htm?id=692014283130` |
| Direct URL | `item.taobao.com/item.htm?id=NNN` | `https://item.taobao.com/item.htm?id=692014283130` |
| Short link | `e.tb.cn/xxx` | `https://e.tb.cn/h.7CC4UBG9MEyeBd1` |
| Product ID | 12-13 digit number | `692014283130` |
| Share text | Contains short link | `【淘宝】商品名 https://e.tb.cn/h.xxx` |

### Share Link Detection Parameters

A URL is a share link if it contains ANY of these query parameters:
`shareurl`, `tbSocialPopKey`, `app`, `cpp`, `short_name`, `sp_tk`, `tk`, `suid`, `bxsign`, `wxsign`, `un`, `ut_sk`, `share_crt_v`, `sourceType`, `shareUniqueId`

Always rebuild to clean URL when share link detected: `https://detail.tmall.com/item.htm?id={product_id}`
