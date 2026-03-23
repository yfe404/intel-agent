# Cheerio vs Browser Test

Determine whether each data point requires JavaScript rendering or can be extracted from raw HTML.

## Overview

The three-way extraction test checks three locations for each data point:
1. **Raw HTML** — The server's initial HTML response (no JavaScript executed)
2. **JSON blobs in HTML** — Structured data embedded in `<script>` tags
3. **Rendered DOM** — The final page after JavaScript execution

The comparison tells you the simplest extraction method available.

---

## Procedure

### 1. Get Raw HTML

Fetch the main document's response body from captured proxy traffic:

```
proxy_list_traffic(url_filter: "[target domain]")
```

Identify the main HTML document exchange — the first GET request returning `text/html`. Then:

```
proxy_get_exchange(exchange_id)
```

Save the response body for searching. This is what Cheerio would see — no JavaScript has run.

**Body truncation check**: `proxy_get_exchange()` returns a `bodyPreview` which may be truncated for large responses. Compare `bodySize` in the exchange metadata to the actual preview length.

- If `bodySize` significantly exceeds the preview length: the preview only covers a portion of the HTML (often just `<head>` and beginning of `<body>`). **Searches of this preview are incomplete.**
- To get the full body: use `proxy_get_session_exchange(session_id, exchange_id: exchange_id, include_body: true)` if a session was started with `capture_profile: "full"`.
- If the full body is unavailable: note that raw HTML search results are **partial** and defer authoritative presence/absence decisions to the rendered DOM snapshot (step 2). A search returning 0 results on a truncated body does NOT mean the data is absent.

### 2. Get Rendered DOM

Capture the accessibility tree after JavaScript has executed:

```
interceptor_chrome_devtools_snapshot()
```

This represents the fully rendered page — what a real user sees. This is what a browser-based extractor would see.

**If DevTools sidecar is unavailable (DEGRADED_MODE)**:

This step cannot be performed — `interceptor_chrome_devtools_snapshot()` requires the sidecar. Consequences:
- The "Browser required" classification cannot be confirmed
- Data points not found in raw HTML or JSON blobs cannot be definitively classified
- Mark such data points as **INCONCLUSIVE** rather than "Not Found"
- The decision matrix below has additional rows for this scenario

### 3. Search for Each Data Point

For every data point, search both the raw HTML body and the rendered DOM snapshot for the value or its search terms.

**Search strategy by data point type**:

- **Text** (e.g., product name "Ultra Widget Pro"): Search for the exact string
- **Numeric** (e.g., price "$29.99"): Search for the number with and without formatting ("29.99", "$29.99", "2999")
- **Boolean** (e.g., in stock): Search for indicator text ("In Stock", "Available", "Sold Out")
- **List** (e.g., reviews): Search for at least one item's content to confirm presence
- **Nested** (e.g., address): Search for each sub-field independently

### 4. Check for JSON Blobs

Search the raw HTML response body for embedded JSON structures. These are script tags that contain structured data the frontend framework hydrates from.

**Common patterns**:

#### Schema.org / JSON-LD
```html
<script type="application/ld+json">
{
  "@type": "Product",
  "name": "Ultra Widget Pro",
  "offers": { "price": "29.99" }
}
</script>
```
Search for: `application/ld+json` in the raw HTML.

#### Next.js
```html
<script id="__NEXT_DATA__" type="application/json">
{
  "props": { "pageProps": { "product": { ... } } }
}
</script>
```
Search for: `__NEXT_DATA__` in the raw HTML.

#### Redux / Vuex / Pinia Hydration
```html
<script>
window.__INITIAL_STATE__ = { "products": { ... } }
</script>
```
Search for: `__INITIAL_STATE__`, `__PRELOADED_STATE__`, or `__PINIA__` in the raw HTML.

#### Nuxt.js
```html
<script>
window.__NUXT__ = { data: [{ product: { ... } }] }
</script>
```
Search for: `__NUXT__` in the raw HTML.

#### Apollo / Relay GraphQL Cache
```html
<script>
window.__APOLLO_STATE__ = { ... }
window.__RELAY_STORE__ = { ... }
</script>
```
Search for: `__APOLLO_STATE__` or `__RELAY_STORE__` in the raw HTML.

When a JSON blob is found, parse it and search for data point values within the JSON structure. Record the JSON path (e.g., `props.pageProps.product.price`).

---

## Decision Matrix

For each data point, classify based on where it was found:

### Found in Raw HTML → Cheerio Works

The data point is present in the server's initial HTML response. Cheerio (or any HTML parser) can extract it without a browser.

**Advantages**: Fast, lightweight, no browser overhead, highly parallelizable.

**Record**: The approximate location in the HTML (tag, class, ID) for selector construction.

### Found in JSON Blob → JSON-in-HTML Works (Preferred)

The data point is present in an embedded JSON structure within the raw HTML.

**Advantages**: Structured data (no fragile selectors), easy to parse, often contains more fields than what's displayed, same performance as Cheerio.

**Why preferred over Cheerio**: JSON paths are more stable than CSS selectors. `props.pageProps.product.price` won't break when the site redesigns, but `.product-detail .price-tag span.amount` will.

**Record**: Which JSON blob (`__NEXT_DATA__`, `ld+json`, etc.) and the JSON path to the value.

### Found in Rendered DOM Only → Browser Required

The data point appears only after JavaScript executes. It's not in the raw HTML or any JSON blob.

**Implications**: Must use a browser-based approach (Playwright, DevTools bridge). Slower and more resource-intensive.

**Record**: The accessibility tree location and any wait conditions needed.

### Found in None → Requires Interaction

The data point isn't visible in any of the three locations. It may require user interaction to appear.

**Next steps**:

1. **Scroll down** — Content may lazy-load:
   ```
   proxy_clear_traffic()
   humanizer_scroll(target_id, "down", 2000)
   humanizer_idle(target_id, 2000)
   interceptor_chrome_devtools_snapshot()
   proxy_list_traffic()
   ```

2. **Click to reveal** — Content behind tabs, accordions, "show more" buttons:
   ```
   proxy_clear_traffic()
   humanizer_click(target_id, "[reveal button selector]")
   humanizer_idle(target_id, 1000)
   interceptor_chrome_devtools_snapshot()
   proxy_list_traffic()
   ```

3. **Navigate to detail page** — Data may only be on individual item pages, not list pages.

4. **Search or filter** — Some data only appears in specific views.

After each interaction, re-run the three-way test on the newly visible content and check proxy traffic for API calls that were triggered.

**Record**: The required interaction sequence and whether an API call was triggered (if so, the API method is preferred).

### Truncated Body + Rendered DOM Available

When the body preview is truncated but the rendered DOM snapshot IS available:
- Use the rendered DOM as the **authoritative** check for "is the data point on the page"
- If found in rendered DOM: it exists on the page (method: Browser, but may also be extractable via Cheerio from the full HTML)
- If NOT in rendered DOM: genuinely absent from the page at load time (may still require interaction — go to step above)
- To determine if Cheerio works for DOM-found data points: retrieve the full body from the session via `proxy_get_session_exchange(session_id, exchange_id, include_body: true)` and re-check

### Truncated Body + No Rendered DOM → INCONCLUSIVE

When the body preview is truncated AND the rendered DOM snapshot is unavailable (DEGRADED_MODE):
- The data point **cannot be confirmed absent** from the page
- It may exist in the truncated portion of the HTML body
- It may exist in the rendered DOM that could not be inspected
- Mark as: **INCONCLUSIVE — body truncated, no snapshot available**
- Recommendation: Re-run with `capture_profile: "full"` session and DevTools sidecar installed for definitive results

---

## Example Walkthrough

**Target**: `https://shop.example.com/products/widget-pro`
**Data points**: product name, price, reviews, stock status

### Raw HTML search results

| Data Point | Found in Raw HTML? | Location |
|-----------|-------------------|----------|
| product name | Yes | `<h1 class="product-title">Ultra Widget Pro</h1>` |
| price | Yes | `<span class="price">$29.99</span>` |
| reviews | No | — |
| stock status | Yes | `<span class="stock-status">In Stock</span>` |

### JSON blob search results

| Data Point | Found in JSON Blob? | Source | JSON Path |
|-----------|---------------------|--------|-----------|
| product name | Yes | `__NEXT_DATA__` | `props.pageProps.product.name` |
| price | Yes | `__NEXT_DATA__` | `props.pageProps.product.price` |
| reviews | No | — | — |
| stock status | Yes | `__NEXT_DATA__` | `props.pageProps.product.inStock` |

### Rendered DOM search results

| Data Point | Found in Rendered DOM? | Location |
|-----------|----------------------|----------|
| product name | Yes | Heading element |
| price | Yes | Text element |
| reviews | Yes | List of review items (loaded via JavaScript) |
| stock status | Yes | Text element |

### Conclusions

| Data Point | Best Method | Reasoning |
|-----------|-------------|-----------|
| product name | JSON-in-HTML | Found in `__NEXT_DATA__`, structured and stable |
| price | JSON-in-HTML | Found in `__NEXT_DATA__`, structured and stable |
| reviews | Browser OR API | Not in raw HTML/JSON — check traffic for review API |
| stock status | JSON-in-HTML | Found in `__NEXT_DATA__`, boolean field |

For reviews: check `proxy_list_traffic()` for a reviews API endpoint. If found, API method is preferred over browser.

---

## Related

- **Main workflow**: See `../SKILL.md` Step 2
- **Proxy tool reference**: See proxy-tool-reference in the web-scraper skill
