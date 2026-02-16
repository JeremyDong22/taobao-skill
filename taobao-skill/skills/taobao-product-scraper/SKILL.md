# Taobao Product Scraper Skill
# v2.1 - Added mi_id solution for A/B testing, updated navigation workflow
---
name: taobao-product-scraper
description: This skill should be used when the user asks to "scrape taobao", "fetch product info from taobao/tmall", "analyze taobao product", "extract taobao data", or any task involving Taobao/Tmall product page data extraction. Provides complete workflow and domain knowledge for dynamically scraping Taobao product pages using Chrome DevTools MCP tools.
version: 2.0.0
---

---

## Layer 1: Prerequisites

### Check Chrome DevTools MCP

Try calling `list_pages` (tool name may be prefixed as `mcp__chrome-devtools__list_pages`).

**If tools are NOT available**, tell the user:

> Chrome DevTools MCP is not configured. This plugin needs it to control Chrome for Taobao scraping.
>
> **Setup: Create a `.mcp.json` file in your project root directory:**
>
> **Option A — Per-session isolation with shared login (recommended):**
>
> Each Claude Code session gets its own Chrome instance, but all copy cookies from a shared base profile. Multiple sessions can run in parallel without conflicts.
>
> ```json
> {
>   "mcpServers": {
>     "chrome-devtools": {
>       "type": "stdio",
>       "command": "sh",
>       "args": [
>         "-c",
>         "BASE=\"$HOME/.chrome-profiles/taobao\" && mkdir -p \"$BASE\" && PROFILE=$(mktemp -d /tmp/taobao-chrome-XXXXXX) && cp -a \"$BASE/.\" \"$PROFILE/\" 2>/dev/null; exec npx chrome-devtools-mcp@latest --user-data-dir=\"$PROFILE\""
>       ]
>     }
>   }
> }
> ```
>
> First-time setup — log in to Taobao once to create the base profile:
> ```bash
> open -na "Google Chrome" --args --user-data-dir="$HOME/.chrome-profiles/taobao" --no-first-run
> ```
> Scan QR code to log in, then close Chrome. All future sessions will copy this login state.
>
> **Option B — Simple isolated mode (no login persistence):**
>
> ```json
> {
>   "mcpServers": {
>     "chrome-devtools": {
>       "type": "stdio",
>       "command": "npx",
>       "args": ["chrome-devtools-mcp@latest", "--isolated"]
>     }
>   }
> }
> ```
> Each session gets a fresh browser. You will need to scan QR code every time.
>
> **Option C — Single persistent session (no parallel support):**
>
> ```json
> {
>   "mcpServers": {
>     "chrome-devtools": {
>       "type": "stdio",
>       "command": "npx",
>       "args": ["chrome-devtools-mcp@latest", "--userDataDir", "/tmp/taobao-chrome-profile"]
>     }
>   }
> }
> ```
> Login persists, but only ONE Claude Code session can use Chrome at a time.
>
> After creating `.mcp.json`, restart Claude Code.

**If connection fails with "browser already running"** — the Chrome profile is locked by another session. Tell the user:

> Another Chrome instance is using the same profile directory. Options:
> 1. Close the other Claude Code session using Chrome
> 2. Switch to Option A or B for per-session isolation
> 3. Manually remove the lock file: `rm /path/to/profile/SingletonLock`

---

## Layer 2: Workflow

### Phase 1: Browser Setup

1. Call `list_pages` to find an available page
2. If no pages exist, use `new_page` to open one

### Phase 2: Login Handling

1. Navigate to `https://www.taobao.com`
2. Check login status with this script via `evaluate_script`:

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

3. **If logged in** → proceed to Phase 3
4. **If redirected to login with "快速进入" button** → `take_snapshot`, find and click it, wait 3 seconds
5. **If not logged in** → navigate to `https://login.taobao.com`, tell user to scan QR code, poll every 5 seconds until `isLoggedIn === true`

See `references/taobao-login-flow.md` for detailed login mechanics and troubleshooting.

### Phase 3: Get Product Input

If the user hasn't already provided a product link, ask them. Supported formats:

| Format | Example |
|--------|---------|
| Product ID | `692014283130` |
| Direct URL | `https://detail.tmall.com/item.htm?id=692014283130` |
| Short link | `https://e.tb.cn/h.xxx` |
| Share text | `【淘宝】商品名 https://e.tb.cn/h.xxx` |

### Phase 4: Navigate to Product Page

#### Step A: Acquire `mi_id` Token (Critical for Complete Data)

Without `mi_id`, Taobao returns a simplified page (fewer images, no parameters, no detail URL). **Always acquire a `mi_id` before navigating to a product page.**

1. Search on Taobao to get a valid `mi_id`:
   ```
   navigate_page → https://s.taobao.com/search?q={any_keyword}
   ```
   Any keyword works (e.g., "手机"). The `mi_id` is not product-specific.

2. Extract `mi_id` from any search result link via `evaluate_script`:
   ```js
   () => {
     const links = document.querySelectorAll('a[href*="item.htm"]');
     for (const a of links) {
       const match = a.href.match(/[?&]mi_id=([^&]+)/);
       if (match) return match[1];
     }
     return null;
   }
   ```

3. Cache this `mi_id` for the session — one token works across all products.

#### Step B: Resolve Product Input

**Short link** (`e.tb.cn` or `s.click.taobao.com`):
- `navigate_page` to the short link → browser follows redirects
- Extract product ID from the final URL's `id=` parameter

**Share text**:
- Extract short link using pattern: `https?://e\.tb\.cn/[A-Za-z0-9.]+`
- Follow the short link resolution above

**Share link detection** — if URL contains any of: `shareurl`, `tbSocialPopKey`, `app`, `cpp`, `short_name`, `sp_tk`, `tk`, `suid`, `bxsign`, `wxsign`, `un`, `ut_sk`, `share_crt_v`, `sourceType`, `shareUniqueId` → it's a share link, must rebuild clean URL.

#### Step C: Navigate with `mi_id`

**Always navigate to the product URL with `mi_id` appended**:
```
https://detail.tmall.com/item.htm?id={product_id}&mi_id={cached_mi_id}
```

Use `wait_for` to confirm the product title appears.

### Phase 5: Data Extraction

Follow the JS-first → DOM-fallback strategy in **Layer 3** below.

### Phase 6: Present Results

Format output using the template in **Layer 4** below. Include all image URLs so the user can view them.

---

## Layer 3: Core Extraction Knowledge

### Principle: JS-First Extraction

Taobao pre-loads ALL product data into `window.__ICE_APP_CONTEXT__` during server-side rendering. Extract most data via a single JavaScript call — **no DOM selectors needed**.

### Primary Extraction Script

Use `evaluate_script` with this function:

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

### A/B Test Detection & Retry

Check the `_version` field in the extraction result:
- **`complete`**: Has 5+ images, parameters, reviews, detail URL → proceed
- **`simplified`**: Sparse data → likely missing `mi_id` in URL

**If simplified detected**:
1. Check if `mi_id` was included in the URL. If not, acquire one (see Phase 4 Step A) and re-navigate
2. If `mi_id` was present, reload the page and retry (max 2 retries)
3. If still simplified after retries, report to user with whatever data was obtained

See `references/taobao-known-issues.md` for full A/B testing details and experiment results.

### Fallback: DOM Snapshot Extraction

If `window.__ICE_APP_CONTEXT__` is unavailable (returns error), fall back to `take_snapshot` and extract from the accessibility tree by visible text and element roles.

### Tab Navigation for Lazy Content

Taobao product pages use tabs — content only loads when a tab is clicked:

| Tab Index | Name | Content |
|-----------|------|---------|
| 0 | 用户评价 (Reviews) | Customer reviews and photos |
| 1 | 参数信息 (Parameters) | Product specifications table |
| 2 | 图文详情 (Detail Images) | Product description images |

To extract tab content:
1. `take_snapshot` to find tab elements (look for text like "评价", "参数", "详情")
2. `click` on the tab uid
3. Wait 2 seconds: `evaluate_script` with `() => new Promise(r => setTimeout(r, 2000))`
4. `take_snapshot` or `evaluate_script` to extract the loaded content

### Data Categories

| # | Category | JS Path | Fallback Hint |
|---|----------|---------|---------------|
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

See `references/taobao-page-structure.md` for full CSS selectors, JS data paths, and image URL cleaning rules.

---

## Layer 4: Output Format

Present results as structured data:

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
