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

Start the MITM proxy with **full-body persistence** (required — default ring buffer caps bodies at 4 KB, which truncates `__NEXT_DATA__`, JSON-LD, and most API responses), launch the stealth browser, and capture the initial page load.

```
proxy_start(
    persistence_enabled: true,
    capture_profile: "full",
    session_name: "intel-[target-domain]-[timestamp]",
    max_disk_mb: 2048
)

interceptor_browser_launch(url: "[target URL]")
interceptor_browser_screenshot(target_id)
```

`interceptor_browser_launch` already waits for `domcontentloaded` internally. For SPAs that defer render to XHR hydration, use `interceptor_browser_navigate(..., wait_until: "networkidle")` explicitly.

Record the returned `session_id` — every body-search and HAR-export call in later steps needs it.

**Record**:
- Page loads successfully? (status code, visual confirmation via screenshot)
- Loading behavior: immediate content (SSR/static) vs spinners/skeletons (SPA/API-driven)
- Any interstitials: cookie banners, newsletter popups, age gates
- Framework indicators in URL patterns or page source
- **Console signal** for rendering model classification:
  ```
  interceptor_browser_list_console(target_id)
  ```
  Hydration/React warnings → CSR-heavy SPA. Network errors on `/api/*` paths → API-driven. Clean console → static SSR.

**Dismiss interstitials** before proceeding — prefer locator-based clicks over CSS selectors (proxy-mcp v2 auto-waits for visible + enabled + stable + in-view):
```
humanizer_click(target_id, role: "button", name: "Accept all")
humanizer_click(target_id, text: "I agree")
humanizer_click(target_id, selector: ".cookie-accept")   # fallback
```

### Step 2: Scan Response Bodies for Data Points

For each data point, search three locations to determine available extraction methods.

#### 2a. Raw HTML body (Cheerio method)

Session persistence is on, so use the **session-backed full-body path** — the ring buffer preview (`proxy_get_exchange`) truncates at 4 KB and will miss embedded JSON blobs.

First, grep every captured body at once:

```
proxy_search_session_bodies(
    session_id: "[from Step 1]",
    query: "[data point search term]",
    hostname_contains: "[target domain]"
)
```

Returns context snippets per match. For each hit, pull the full body:

```
proxy_list_traffic(url_filter: "[target domain]", method_filter: "GET")
# locate the main HTML document exchange id
proxy_get_session_exchange(
    session_id: "[from Step 1]",
    exchange_id: "[main HTML exchange id]",
    include_body: true
)
```

If the data point's search term is found in the raw HTML body → **Cheerio extraction works**.

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

Get the rendered page as a YAML ARIA tree. For large pages, scope the snapshot to a selector to save tokens:

```
# Full page
interceptor_browser_snapshot(target_id)

# Scoped to main content (token-saver)
interceptor_browser_snapshot(target_id, selector: "main, article, [itemtype*='Product']")
```

Output is YAML with `role`, `name`, and `text` fields — match data-point search terms against those. If found in rendered DOM but NOT in raw HTML / JSON blobs → **Browser extraction required** (JavaScript renders this content).

#### 2d. Web storage scan (often overlooked — SPAs hydrate from here)

Many SPAs stash hydration data in localStorage/sessionStorage rather than (or in addition to) script tags. Scan both storages for each data point:

```
interceptor_browser_list_storage_keys(target_id, storage_type: "local")
interceptor_browser_list_storage_keys(target_id, storage_type: "session")

# For each interesting key, pull full value:
interceptor_browser_get_storage_value(
    target_id,
    storage_type: "local",
    item_id: "[item_id from list above]"
)
```

If a data point's search term is found in a storage value → **Browser extraction required, but the data is stable + parseable from storage keys** (record the key name; an implementation can read it via `page.evaluate`).

#### 2e. Decision matrix

For each data point, classify availability:

| Raw HTML? | JSON Blob? | Web Storage? | Rendered DOM? | Method |
|---|---|---|---|---|
| Yes | — | — | — | Cheerio |
| — | Yes | — | — | JSON-in-HTML (preferred) |
| Yes | Yes | — | — | JSON-in-HTML (preferred) or Cheerio |
| No | No | Yes | — | Browser required; parseable from storage |
| No | No | No | Yes | Browser required (DOM scrape) |
| No | No | No | No | Requires interaction — go to Step 2f |

See `strategies/cheerio-vs-browser-test.md` for the detailed procedure.

#### 2f. Trigger interactions for missing data points

If a data point wasn't found in any location, it may require user interaction to appear (lazy loading, click-to-reveal, scroll-triggered content).

Use an `ai`-mode snapshot first to find stable refs for the trigger element:

```
interceptor_browser_snapshot(target_id, mode: "ai")
# Read YAML to find the "Load more" / "Show reviews" button
```

Then clear traffic and interact — prefer role/text locators over CSS:

```
proxy_clear_traffic()
humanizer_click(target_id, text: "Load more")
# or: humanizer_click(target_id, role: "button", name: "Show reviews")
# or: humanizer_scroll(target_id, delta_y: 2000)
```

Then re-check rendered DOM and freshly captured traffic:
```
interceptor_browser_snapshot(target_id, selector: "[the newly loaded region]")
proxy_list_traffic()
proxy_search_session_bodies(session_id, query: "[data point term]")
```

If the data point appears after interaction, note the required trigger sequence. If a matching API call was triggered, prefer the API method over DOM scraping.

### Step 3: Sniff APIs

Search captured traffic for JSON API endpoints that serve the requested data points.

#### 3a. Filter existing traffic

Source everything from the MITM proxy — it's the single source of truth and sees strictly more than any in-browser network view (3rd-party domains, CONNECT tunnels, every TLS handshake):

```
proxy_list_traffic(url_filter: "/api/")
proxy_list_traffic(url_filter: "/graphql")
proxy_list_traffic(url_filter: "/_next/data/")
proxy_list_traffic(url_filter: "/wp-json/")
proxy_list_traffic(url_filter: ".json")
proxy_search_traffic(query: "application/json")
```

For endpoints whose response bodies are interesting (not just the URL), grep full decompressed bodies:
```
proxy_search_session_bodies(
    session_id: "[from Step 1]",
    query: "[API-ish pattern like \"graphql\" or \"__typename\"]",
    content_type_contains: "application/json"
)
```

#### 3b. Trigger more traffic via interactions

Clear traffic, perform user-like interactions, observe new API calls:

```
proxy_clear_traffic()
humanizer_click(target_id, role: "link", name: "Next")   # or text: / selector:
proxy_list_traffic()
```

`humanizer_click` auto-waits for the click target to be stable — no explicit wait needed before reading traffic.

**Interactions to test** (match to your data points):
- **Pagination** — click next page, scroll to bottom → reveals pagination API parameters
- **Search** — type in search box → reveals search endpoint
- **Filters** — click category/filter → reveals filter parameters
- **Detail view** — click an item → reveals detail/item API
- **Load more** — scroll down on infinite-scroll pages → reveals lazy-load endpoint
- **Sort** — change sort order → reveals sort parameters

#### 3c. Inspect each discovered endpoint

For each API endpoint found, pull the full request + response body from the session (not the 4 KB preview):

```
proxy_get_session_exchange(
    session_id: "[from Step 1]",
    exchange_id,
    include_body: true
)
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
interceptor_browser_list_cookies(target_id)
proxy_search_traffic(query: "403")
proxy_search_traffic(query: "challenge")
```

#### Tier 2: Datacenter proxy

```
proxy_set_upstream("[datacenter proxy URL]")
interceptor_browser_navigate(target_id, url: "[target URL]", wait_until: "networkidle")
interceptor_browser_screenshot(target_id)
```

Compare to baseline:
- Same content loads? Or challenge/block page?
- Check status codes in traffic
- Check for CAPTCHA or challenge elements in DOM snapshot

```
proxy_list_traffic(url_filter: "[target domain]")
interceptor_browser_snapshot(target_id)
```

#### Tier 3: Residential proxy

```
proxy_set_upstream("[residential proxy URL]")
interceptor_browser_navigate(target_id, url: "[target URL]", wait_until: "networkidle")
interceptor_browser_screenshot(target_id)
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

Before writing the report, persist the evidence: export the full session (with decompressed bodies) to HAR so the report can reference it:

```
proxy_export_har(
    session_id: "[from Step 1]",
    file_path: "/tmp/intel-[target]-[timestamp].har",
    include_bodies: true
)
```

Also check handshake metadata availability (useful for the TLS note in section 2):
```
proxy_get_session_handshakes(session_id)
```

The report contains:
1. **Site Profile** — Framework, rendering model, protection summary
2. **Protection Assessment** — Proxy tier table, minimum required level, rate limits
3. **Data Point Analysis** — One subsection per data point with ranked methods
4. **Discovered Endpoints** — Full API specs for every endpoint found
5. **Recommended Strategy** — Summary table mapping each data point to its best method
6. **Raw Evidence** — **session_id + HAR file path (both required)** + screenshot references + total exchanges + bytes written

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
| `proxy_start(persistence_enabled: true, capture_profile: "full", session_name, max_disk_mb: 2048)` | Start MITM proxy with full-body on-disk capture (required for intel-agent) |
| `interceptor_browser_launch(url, timezone?, locale?, viewport_width?, viewport_height?)` | Launch cloakbrowser (stealth Chromium, humanize on by default) |

### Traffic Analysis — ring buffer (headers + previews)
| Tool | Purpose |
|------|---------|
| `proxy_list_traffic(url_filter, method_filter, hostname_filter, status_filter)` | List captured exchanges |
| `proxy_search_traffic(query)` | Full-text search across URLs, headers, 4 KB body previews |
| `proxy_get_exchange(exchange_id)` | Header-level peek (body capped at 4 KB) |
| `proxy_clear_traffic()` | Clear ring buffer before an isolated interaction |

### Session & full-body evidence (primary data-point search)
| Tool | Purpose |
|------|---------|
| `proxy_search_session_bodies(session_id, query, hostname_contains?, content_type_contains?)` | Grep across every captured body (decompressed); returns context snippets |
| `proxy_get_session_exchange(session_id, exchange_id, include_body: true)` | Full decompressed request + response body for one exchange |
| `proxy_query_session(session_id, hostname_contains?, method_filter?, status_filter?)` | Indexed metadata query over the session |
| `proxy_export_har(session_id, file_path, include_bodies: true)` | HAR export for the Raw Evidence section |
| `proxy_get_session_handshakes(session_id)` | JA3/JA4/JA3S handshake metadata availability |

### Browser Inspection
| Tool | Purpose |
|------|---------|
| `interceptor_browser_navigate(target_id, url, wait_until?)` | Navigate and optionally wait for proxy capture |
| `interceptor_browser_screenshot(target_id, file_path?, full_page?)` | Screenshot (saves to disk if file_path given) |
| `interceptor_browser_snapshot(target_id, selector?, mode?)` | YAML ARIA tree; `selector` scopes the tree; `mode: "ai"` adds refs |
| `interceptor_browser_list_console(target_id, types?, text_filter?)` | Buffered console messages since launch |
| `interceptor_browser_list_cookies(target_id, domain_filter?)` | Browser context cookies |
| `interceptor_browser_list_storage_keys(target_id, storage_type)` | local/sessionStorage key listing |
| `interceptor_browser_get_storage_value(target_id, storage_type, item_id)` | Full storage value |

### Human Interaction
| Tool | Purpose |
|------|---------|
| `humanizer_click(target_id, selector? \| role+name? \| text? \| label? \| x+y?)` | Click with human-like behavior; prefer `role+name` / `text` / `label` — auto-waits for visible + enabled + stable + in-view |
| `humanizer_type(target_id, text, wpm?, error_rate?)` | Type with realistic timing |
| `humanizer_scroll(target_id, delta_y, delta_x?, duration_ms?)` | Smooth eased scroll |

(cloakbrowser already humanizes dispatch by default; the `humanizer_*` timing profile layers on top. Explicit idle waits are unnecessary — `humanizer_click` auto-waits for stability, and `interceptor_browser_navigate(..., wait_until: "networkidle")` waits for XHR settle.)

### Protection Testing
| Tool | Purpose |
|------|---------|
| `proxy_set_upstream(url)` | Chain to upstream proxy |
| `proxy_set_host_upstream(hostname, url)` | Per-host upstream proxy |
| `proxy_clear_upstream()` | Remove all upstream proxies |

**Important**:
- Tools take `target_id` from `interceptor_browser_launch` directly — no attach/detach step
- Always start the proxy with `persistence_enabled: true, capture_profile: "full"` — without this, bodies are capped at 4 KB and most data-point evidence is truncated
- Prefer locator-based `humanizer_click` (`role + name`, `text`, `label`) over CSS selectors
- Always `proxy_clear_traffic()` before an interaction to isolate the traffic it generates
- For geographic testing, relaunch the browser with matching `timezone`/`locale` when switching upstream proxy countries — IP geo vs browser geo mismatch is a bot signal
