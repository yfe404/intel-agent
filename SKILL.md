---
name: intel-agent
description: Reconnaissance tool that takes a target URL and desired data points, tests extraction methods (API, JSON-in-HTML, Cheerio, Browser) via proxy-mcp traffic interception, tests protection levels, and outputs a structured intelligence report with per-data-point strategies.
license: MIT
---

# Intel Agent: Data Point Reconnaissance

## When This Skill Activates

Activate when user requests:
- "recon [URL]"
- "intel report for [URL]"
- "find how to extract [data] from [URL]"
- "discover data from [URL]"
- "what's the best way to get [data] from [site]"
- "how would I scrape [data points] from [URL]"

**This skill does NOT implement scrapers.** It discovers extraction methods and outputs a structured intelligence report. For implementation, hand off to the web-scraper skill.

---

## Input Parsing

Extract two inputs from the user's request:

1. **Target URL** — The page or site to investigate
2. **Data points** — What the user wants to extract

### If data points are specified

Normalize each into a structured entry:

```
Data Point: "product price"
  Type: numeric
  Search terms: ["price", "$", "USD", "cost"]

Data Point: "review list"
  Type: list
  Search terms: ["review", "rating", "stars", "comment"]
```

**Type classification**:
- **text** — Single string value (name, title, description)
- **numeric** — Number or currency (price, rating, count)
- **boolean** — True/false state (in stock, verified, active)
- **list** — Repeating items (reviews, variants, images)
- **nested** — Grouped sub-fields (address with street/city/zip, product with name/price/sku)

### If data points are NOT specified

Ask the user:

```
I'll run reconnaissance on [URL]. What data points do you want to extract?

Examples:
- Product name, price, description, images, reviews
- Article title, author, date, body text, tags
- Business name, address, phone, hours, ratings
```

Do not proceed until data points are confirmed.

---

## The Workflow

Six steps, executed sequentially. Each step builds on the previous.

### Step 1: Initialize & Capture Baseline Traffic

Start the MITM proxy, launch a stealth browser, and capture the initial page load.

```
proxy_start()

interceptor_chrome_launch(
    url: "[target URL]",
    stealthMode: true
)

interceptor_chrome_devtools_attach(target_id)

humanizer_idle(target_id, 3000)

interceptor_chrome_devtools_screenshot()
```

**Record**:
- Page loads successfully? (status code, visual confirmation via screenshot)
- Loading behavior: immediate content (SSR/static) vs spinners/skeletons (SPA/API-driven)
- Any interstitials: cookie banners, newsletter popups, age gates
- Framework indicators in URL patterns or page source

**Dismiss interstitials** before proceeding:
```
humanizer_click(target_id, "[cookie accept button selector]")
humanizer_idle(target_id, 1000)
```

### Step 2: Scan Response Bodies for Data Points

For each data point, search three locations to determine available extraction methods.

#### 2a. Raw HTML body (Cheerio method)

Get the main document response from captured traffic:

```
proxy_list_traffic(url_filter: "[target domain]")
```

Find the main HTML document exchange (usually the first GET with `text/html` content type), then:

```
proxy_get_exchange(exchange_id)
```

Search the response body for each data point's search terms. If found in raw HTML → **Cheerio extraction works**.

#### 2b. JSON blobs in HTML (JSON-in-HTML method)

Search the raw HTML response body for embedded JSON structures:

- `<script type="application/ld+json">` — Structured data (schema.org)
- `<script id="__NEXT_DATA__"` — Next.js server-side data
- `window.__INITIAL_STATE__` — Redux/Vuex hydration state
- `window.__NUXT__` — Nuxt.js hydration data
- `window.__APOLLO_STATE__` — Apollo GraphQL cache
- `window.__RELAY_STORE__` — Relay GraphQL cache

For each JSON blob found, parse it and search for data point values. If a data point is found in a JSON blob → **JSON-in-HTML extraction works** (preferred over Cheerio — structured data, no selector fragility).

#### 2c. Rendered DOM (Browser method)

Get the rendered accessibility tree:

```
interceptor_chrome_devtools_snapshot()
```

Search the snapshot text for each data point's search terms. If found in rendered DOM but NOT in raw HTML → **Browser extraction required** (JavaScript renders this content).

#### 2d. Decision matrix

For each data point, classify availability:

| Found in Raw HTML? | Found in JSON Blob? | Found in Rendered DOM? | Method |
|---------------------|---------------------|----------------------|--------|
| Yes | — | — | Cheerio |
| — | Yes | — | JSON-in-HTML (preferred) |
| Yes | Yes | — | JSON-in-HTML (preferred) or Cheerio |
| No | No | Yes | Browser required |
| No | No | No | Requires interaction — go to Step 2e |

See `strategies/cheerio-vs-browser-test.md` for the detailed procedure.

#### 2e. Trigger interactions for missing data points

If a data point wasn't found in any of the three locations, it may require user interaction to appear (lazy loading, click-to-reveal, scroll-triggered content):

```
proxy_clear_traffic()
humanizer_scroll(target_id, "down", 2000)
humanizer_idle(target_id, 2000)
```

Then re-check the rendered DOM:
```
interceptor_chrome_devtools_snapshot()
```

And check for new API calls triggered by the interaction:
```
proxy_list_traffic()
```

If the data point appears after interaction, note the required trigger action.

### Step 3: Sniff APIs

Search captured traffic for JSON API endpoints that serve the requested data points.

#### 3a. Filter existing traffic

```
proxy_list_traffic(url_filter: "/api/")
proxy_list_traffic(url_filter: "/graphql")
proxy_list_traffic(url_filter: "/_next/data/")
proxy_list_traffic(url_filter: "/wp-json/")
proxy_list_traffic(url_filter: ".json")
proxy_search_traffic(query: "application/json")
```

Also check browser-side network view:
```
interceptor_chrome_devtools_list_network(resource_types: ["xhr", "fetch"])
```

#### 3b. Trigger more traffic via interactions

Clear traffic, perform user-like interactions, observe new API calls:

```
proxy_clear_traffic()
humanizer_click(target_id, "[pagination next button]")
humanizer_idle(target_id, 2000)
proxy_list_traffic()
```

**Interactions to test** (match to your data points):
- **Pagination** — click next page, scroll to bottom → reveals pagination API parameters
- **Search** — type in search box → reveals search endpoint
- **Filters** — click category/filter → reveals filter parameters
- **Detail view** — click an item → reveals detail/item API
- **Load more** — scroll down on infinite-scroll pages → reveals lazy-load endpoint
- **Sort** — change sort order → reveals sort parameters

#### 3c. Inspect each discovered endpoint

For each API endpoint found:

```
proxy_get_exchange(exchange_id)
```

**Record per endpoint**:
- **URL**: Full URL with query parameters
- **Method**: GET / POST
- **Request headers**: Authorization, Content-Type, custom headers (X-API-Key, etc.)
- **Request body**: For POST requests — query structure, variables
- **Authentication**: None / cookie-based / token-based / API key
- **Response structure**: JSON schema, field names
- **Pagination**: Parameters (page, offset, cursor, limit), total count, has-more indicator
- **Data points covered**: Which of the user's requested data points appear in this response
- **Rate limit indicators**: `X-RateLimit-*` headers, `Retry-After` headers

### Step 4: Test Protection Levels

Test how the site responds across three proxy tiers to determine the minimum required access level.

See `strategies/proxy-escalation.md` for the detailed procedure.

#### Tier 1: Direct (no upstream proxy)

Already tested in Steps 1-3. Record baseline results:
- Status codes (200 OK, 403 blocked, 429 rate limited)
- Challenge pages (Cloudflare "checking your browser", DataDome, Akamai)
- Protection cookies (`cf_clearance`, `__cf_bm`, `datadome`, `_abck`)

Check for protection indicators:
```
interceptor_chrome_devtools_list_cookies()
proxy_search_traffic(query: "403")
proxy_search_traffic(query: "challenge")
```

#### Tier 2: Datacenter proxy

```
proxy_set_upstream("[datacenter proxy URL]")
interceptor_chrome_devtools_navigate("[target URL]")
humanizer_idle(target_id, 3000)
interceptor_chrome_devtools_screenshot()
```

Compare to baseline:
- Same content loads? Or challenge/block page?
- Check status codes in traffic
- Check for CAPTCHA or challenge elements in DOM snapshot

```
proxy_list_traffic(url_filter: "[target domain]")
interceptor_chrome_devtools_snapshot()
```

#### Tier 3: Residential proxy

```
proxy_set_upstream("[residential proxy URL]")
interceptor_chrome_devtools_navigate("[target URL]")
humanizer_idle(target_id, 3000)
interceptor_chrome_devtools_screenshot()
```

Same comparison as Tier 2.

#### Clean up

```
proxy_clear_upstream()
```

#### Protection decision

| Direct | Datacenter | Residential | Minimum Required |
|--------|-----------|-------------|-----------------|
| OK | OK | OK | Direct (no proxy needed) |
| Blocked | OK | OK | Datacenter proxy |
| Blocked | Blocked | OK | Residential proxy |
| Blocked | Blocked | Blocked | Residential + stealth + possible CAPTCHA solving |

Record: which tier works, any rate limits observed, cookie/header requirements.

### Step 5: Rank Extraction Methods

For each data point, rank all available methods from best to worst:

**Priority order**: API > JSON-in-HTML > Cheerio > Browser

For each available method per data point, include:

- **API method**: Endpoint URL, HTTP method, required headers, auth, pagination params, JSON path to the data point
- **JSON-in-HTML method**: Script tag identifier (`__NEXT_DATA__`, `ld+json`, etc.), JSON path to the data point
- **Cheerio method**: CSS selector or XPath, expected HTML structure
- **Browser method**: Accessibility tree location, required interactions (scroll, click), wait conditions

Also include per method:
- **Required proxy level**: Direct / Datacenter / Residential
- **Confidence**: High (data clearly present and tested) / Medium (data found but structure may vary) / Low (data sometimes present or requires specific conditions)

### Step 6: Generate Intelligence Report

Compile all findings into the canonical report format.

See `reference/report-schema.md` for the complete report structure.

The report contains:
1. **Site Profile** — Framework, rendering model, protection summary
2. **Protection Assessment** — Proxy tier table, minimum required level, rate limits
3. **Data Point Analysis** — One subsection per data point with ranked methods
4. **Discovered Endpoints** — Full API specs for every endpoint found
5. **Recommended Strategy** — Summary table mapping each data point to its best method
6. **Raw Evidence** — Session HAR path, screenshot references

Output the report directly to the user. Use the exact format from `reference/report-schema.md`.

---

## Handling Different Data Point Types

### Text and Numeric (name, price, title, rating)

Direct value search. Look for the exact value (or close match) in:
- Raw HTML body text
- JSON blob values
- Rendered DOM text
- API response fields

### Boolean (in stock, verified, active)

Look for:
- Explicit boolean fields in API responses (`"inStock": true`)
- Text indicators in HTML (`"In Stock"`, `"Out of Stock"`, `"Sold Out"`)
- CSS classes indicating state (`.in-stock`, `.unavailable`)
- Structured data properties (`schema:availability`)

### List (reviews, variants, images, related items)

Look for:
- **API responses**: Array fields, pagination (indicates a list endpoint)
- **HTML**: Repeating DOM structures (`.review-item`, `li.variant`, `.product-card`)
- **JSON blobs**: Array values in `__NEXT_DATA__` or `ld+json`

For lists, also determine:
- Total count (from API pagination metadata or DOM count)
- Whether all items load at once or require pagination/scroll
- Whether list items link to detail pages with more data

### Nested (address, product details, author info)

Look for:
- **API responses**: Nested JSON objects (`"address": {"street": "...", "city": "..."}`)
- **JSON blobs**: Nested structures in embedded JSON
- **HTML**: Grouped DOM elements (`.address > .street`, `.address > .city`)

Treat each sub-field as its own data point for extraction method ranking, but group them in the report.

### Images and Media

Look for:
- `src` attributes on `<img>` tags
- CDN URL patterns (`/images/`, `/media/`, `/cdn/`)
- API response fields with image URLs (`"imageUrl"`, `"thumbnail"`, `"photos"`)
- `srcset` attributes for responsive images
- Background images in CSS (`background-image: url(...)`)
- Structured data image properties (`schema:image`)

Record the full CDN URL pattern — often images follow a template like `https://cdn.site.com/images/{size}/{id}.jpg` where size can be manipulated.

---

## What This Skill Does NOT Do

This skill is **pure discovery**. It does NOT:

- Write scraping code or scripts
- Create Apify Actors
- Implement pagination logic
- Build data pipelines
- Deploy anything
- Run extraction at scale

The intelligence report contains everything needed to implement a scraper, but implementation is a separate step.

**Handoff**: After generating the report, tell the user:

```
Intelligence report complete. To implement a scraper based on these findings,
use the web-scraper skill:
  "scrape [data points] from [URL] using the intel report above"
```

---

## Tool Quick Reference

### Initialization
| Tool | Purpose |
|------|---------|
| `proxy_start()` | Start MITM proxy |
| `interceptor_chrome_launch(url, stealthMode: true)` | Launch stealth Chrome |
| `interceptor_chrome_devtools_attach(target_id)` | Attach DevTools bridge |

### Traffic Analysis
| Tool | Purpose |
|------|---------|
| `proxy_list_traffic(url_filter, method_filter)` | List captured exchanges |
| `proxy_search_traffic(query)` | Full-text search across traffic |
| `proxy_get_exchange(exchange_id)` | Full request/response details |
| `proxy_clear_traffic()` | Clear buffer before action |

### Browser Inspection
| Tool | Purpose |
|------|---------|
| `interceptor_chrome_devtools_navigate(url)` | Navigate (preserves DevTools) |
| `interceptor_chrome_devtools_screenshot()` | Capture screenshot |
| `interceptor_chrome_devtools_snapshot()` | Accessibility tree (rendered DOM) |
| `interceptor_chrome_devtools_list_network(resource_types)` | Browser network view |
| `interceptor_chrome_devtools_list_cookies(domain_filter)` | Get cookies |

### Human Interaction
| Tool | Purpose |
|------|---------|
| `humanizer_click(target_id, selector)` | Click with human-like behavior |
| `humanizer_type(target_id, text)` | Type with realistic timing |
| `humanizer_scroll(target_id, direction, amount)` | Smooth scroll |
| `humanizer_idle(target_id, duration_ms)` | Idle with micro-movements |

### Protection Testing
| Tool | Purpose |
|------|---------|
| `proxy_set_upstream(url)` | Chain to upstream proxy |
| `proxy_set_host_upstream(hostname, url)` | Per-host upstream proxy |
| `proxy_clear_upstream()` | Remove all upstream proxies |

### Session Management
| Tool | Purpose |
|------|---------|
| `proxy_session_start(name)` | Start recording session |
| `proxy_session_stop(session_id)` | Stop recording |
| `proxy_export_har(session_id, path)` | Export session as HAR |

**Important**:
- Always use `interceptor_chrome_devtools_navigate()` for navigation — NOT `interceptor_chrome_navigate()` (loses DevTools session)
- Always use `stealthMode: true` when launching Chrome
- Always `proxy_clear_traffic()` before an interaction to isolate the traffic it generates
