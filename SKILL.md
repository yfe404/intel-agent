---
name: intel-agent
description: Reconnaissance tool that takes a target URL and desired data points, tests extraction methods (API, JSON-in-HTML, Cheerio, Browser) via proxy-mcp traffic interception, observes protection signal under the active proxy/IP config, and outputs a structured intelligence report with per-data-point strategies.
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

**This skill does NOT implement scrapers.** It discovers extraction methods and outputs a structured intelligence report. For implementation, hand off to the **poc-agent** skill (builds + validates a minimal scraper from the report) or directly to **web-scraper** (productionization).

**Terminology**: **Step N** = strictly sequential — each step builds on the previous and cannot be skipped.

**References** (loaded on demand):
- `reference/tool-reference.md` — Tool signatures, known limitations, important rules
- `reference/data-point-types.md` — Type classification and search strategies per type
- `reference/report-schema.md` — Report output format
- `strategies/cheerio-vs-browser-test.md` — Cheerio vs Browser extraction test procedure

---

## Input Parsing

Extract from the user's request: **Target URL** and **Data points** to extract.

Normalize each data point into a type (text / numeric / boolean / list / nested) with search terms. See `reference/data-point-types.md` for type classification and search strategies.

If data points are NOT specified, ask the user before proceeding.

---

## The Workflow

Six steps, executed sequentially. Each step builds on the previous.

### Step 1: Initialize & Capture Baseline Traffic

Start the MITM proxy with **full-body persistence** (required — the default ring buffer caps bodies at 4 KB, which truncates `__NEXT_DATA__`, JSON-LD, and most API responses). Then launch the stealth browser and capture the baseline.

```
proxy_start(persistence_enabled: true, capture_profile: "full", session_name: "recon-[domain]-[YYYYMMDD]", max_disk_mb: 2048)
interceptor_browser_launch(url: "[target URL]")
interceptor_browser_screenshot(target_id)
```

Note: `proxy_start` with `persistence_enabled: true` auto-starts a session. A subsequent `proxy_session_start()` will return the existing session (not error), but the cleaner pattern is to pass `session_name` directly to `proxy_start`. Capture the returned `session_id` — every body-search and HAR-export call needs it.

**Navigation tip**: `interceptor_browser_navigate(target_id, url, wait_for_proxy_capture: true)` returns `matchedHostExchangeIds` — the exchange IDs for every target-host request fired during load. Pass those straight to `proxy_get_session_exchange(session_id, exchange_id, include_body: true)` in Step 2 instead of re-querying via `proxy_list_traffic`.

**Console signal** for site profiling: `interceptor_browser_list_console(target_id)` — React/hydration warnings → CSR SPA, network errors on `/api/*` → API-driven, clean console → static SSR.

**Record**: Session name/ID, page load status, loading behavior (SSR vs SPA), interstitials, framework indicators.

**Dismiss interstitials**: prefer locator-based clicks over CSS selectors — `humanizer_click` auto-waits for visible + enabled + stable + in-view. Try in this order:
```
humanizer_click(target_id, role: "button", name: "Accept all")
humanizer_click(target_id, text: "I agree")
humanizer_click(target_id, selector: ".cookie-accept")   # fallback
```
Multiple popups may appear sequentially (e.g., Cloudflare challenge then cookie consent) — repeat until the page is clean.

### Step 2: Scan Response Bodies for Data Points

For each data point, search three locations to determine extraction methods.

#### 2a. Raw HTML body (Cheerio method)

Use the **session-backed full-body path** — the ring buffer preview (`proxy_get_exchange`) truncates at 4 KB and will miss embedded JSON blobs.

```
proxy_search_session_bodies(session_id, query: "[data point search term]", hostname_contains: "[target domain]")
proxy_list_traffic(url_filter: "[target domain]", method_filter: "GET")   # locate main HTML exchange id
proxy_get_session_exchange(session_id, exchange_id: "[main HTML exchange id]", include_body: true)
```

If the search term is found in the raw HTML body → **Cheerio works**.

#### 2b. JSON blobs in HTML (JSON-in-HTML method)

Search the raw HTML body (or use `proxy_search_session_bodies` directly) for: `application/ld+json`, `__NEXT_DATA__`, `__INITIAL_STATE__`, `__NUXT__`, `__APOLLO_STATE__`, `__RELAY_STORE__`. Parse and search for data points. If found → **JSON-in-HTML works** (preferred over Cheerio).

#### 2c. Rendered DOM (Browser method)

```
interceptor_browser_snapshot(target_id)
# scoped (token-saver):
interceptor_browser_snapshot(target_id, selector: "main, article, [itemtype*='Product']")
# with refs (lets later humanizer_click reuse the locator):
interceptor_browser_snapshot(target_id, mode: "ai")
```

Output is YAML with `role`, `name`, `text` fields — match data-point search terms against those. If found in rendered DOM but NOT in raw HTML / JSON blobs / web storage → **Browser extraction required**.

#### 2d. Web storage scan (often overlooked — SPAs hydrate from here)

```
interceptor_browser_list_storage_keys(target_id, storage_type: "local")
interceptor_browser_list_storage_keys(target_id, storage_type: "session")
interceptor_browser_get_storage_value(target_id, storage_type: "local", item_id: "[item_id]")
```

If a search term is found in a storage value → **Browser extraction required, but the data is stable + parseable from storage keys** (record the key name).

#### 2e. Decision matrix

| Raw HTML? | JSON Blob? | Web Storage? | Rendered DOM? | Method |
|---|---|---|---|---|
| Yes | — | — | — | Cheerio |
| — | Yes | — | — | JSON-in-HTML (preferred) |
| Yes | Yes | — | — | JSON-in-HTML (preferred) or Cheerio |
| No | No | Yes | — | Browser required; parseable from storage |
| No | No | No | Yes | Browser required (DOM scrape) |
| No | No | No | No | Requires interaction → Step 2f |

See `strategies/cheerio-vs-browser-test.md` for the detailed procedure.

#### 2f. Trigger interactions for missing data points

For data points not found: `proxy_clear_traffic()`, interact (locator-based `humanizer_click`, `humanizer_scroll`), then re-check DOM + traffic. `humanizer_click` auto-waits for stability — no explicit idle needed before reading traffic.

#### 2g. Full-page scroll capture (MANDATORY)

Scroll the entire page before API sniffing to capture lazy-loaded API calls:

1. `proxy_clear_traffic()`
2. Scroll 2-3x: `humanizer_scroll(target_id, delta_y: 800)`
3. **Mid-scroll visual check**: `interceptor_browser_screenshot(target_id)` + `interceptor_browser_snapshot(target_id)` — inspect for interstitials (CAPTCHA, cookie consent, modals). Dismiss via `humanizer_click(role + name / text)`. If a hard block (CAPTCHA) is detected, stop scrolling and proceed to Step 3 with traffic captured so far.
4. Scroll 2-3x more: `humanizer_scroll(target_id, delta_y: 800)`
5. `interceptor_browser_screenshot(target_id)` + `interceptor_browser_snapshot(target_id)` — final state
6. `proxy_list_traffic()` — review newly triggered API calls

**MANDATORY** even if all data points already found — lazy APIs often provide better-structured data.

### Step 3: Sniff APIs

#### 3a. Filter existing traffic

The MITM proxy is the single source of truth — it sees strictly more than any in-browser network view (3rd-party domains, CONNECT tunnels, every TLS handshake):

```
proxy_list_traffic(url_filter: "/api/")
proxy_list_traffic(url_filter: "/graphql")
proxy_list_traffic(url_filter: "/_next/data/")
proxy_list_traffic(url_filter: "/wp-json/")
proxy_list_traffic(url_filter: ".json")
proxy_search_traffic(query: "application/json")
proxy_search_session_bodies(session_id, query: "__typename", content_type_contains: "application/json")
```

#### 3b. Trigger more traffic via interactions

`proxy_clear_traffic()` → interact (pagination, search, filters, detail click, load more, sort) via locator-based `humanizer_click` → `proxy_list_traffic()`. No explicit idle — `humanizer_click` auto-waits for stability.

#### 3c. Inspect each discovered endpoint

Pull full request + response body from the session (not the 4 KB preview): `proxy_get_session_exchange(session_id, exchange_id, include_body: true)`. Record: URL, method, headers, auth, body, response structure, pagination, data points covered, rate limits. See `reference/report-schema.md` Section 4 for the full endpoint template.

### Step 4: Assess Protection Under Active Config

Assume the active proxy/IP is the right one for this target — don't escalate
through datacenter/residential tiers. Single-pass: characterize what the site
returns under the current config, and surface a proxy/IP **hypothesis** only
when blocking is observed.

**Observe under the active config**:
```
interceptor_browser_list_cookies(target_id)             # bot-management cookies (_abck, bm_sz, datadome, cf_clearance, ...)
proxy_search_traffic(query: "403")                      # blocked responses
proxy_search_traffic(query: "challenge")                # interstitials / JS challenges
proxy_search_traffic(query: "captcha")
```

For each access method tested in Steps 1–3 (main page, each API endpoint),
record: status, blocked? (status ≥ 400 or interstitial served), cookies set,
challenge type if any.

**TLS verification**: `proxy_list_tls_fingerprints(hostname_filter: "[target domain]")`.
- JA3 varies + JA4 stable → browser ClientHello passthrough (cloakbrowser presents authentic Chrome fingerprint).
- JA3 identical across all connections → an upstream proxy re-terminates TLS (your real fingerprint is the proxy's, not Chrome's). For HTTP-only clients downstream, recommend `proxy_set_fingerprint_spoof(chrome_*)`.

**Cookie JA4-binding probe** — record whether the WAF ties its session
cookie to the originating TLS fingerprint. This decides whether the
deployed actor's Tier 1 (impit) call needs to use the same impit
`browser` profile as the cloak warmup, or whether any profile works.

Procedure: after the browser run mints session cookies (datadome /
incap_ses / cf_clearance / etc.), have the consumer replay one
representative GET against the same target with two distinct impit
profiles (`browser=firefox` and `browser=chrome`) **on the same
warmed jar**. Compare the responses:

- Both profiles return real-page bytes with the required data point
  paths resolving → **`tier1_cookie_replay: cross-engine`**. JA4 is
  not part of the WAF's session-binding hash. The consumer can
  use any impit profile.
- Only the matched profile (the engine that minted the cookie)
  returns real-page bytes; the mismatched profile returns a
  challenge / fresh `cid` / 200 with an interstitial body →
  **`tier1_cookie_replay: matched-only`**. The consumer's Tier 1
  client must use the same impit profile as the cloak warmup, or
  the cascade can never ride Tier 1. DataDome is the canonical
  example.
- Could not test (Tier 1 unreachable for unrelated reasons; only
  one profile ever attempted) → **`tier1_cookie_replay: untested`**.

Record the observation in §2 of the report and propagate the
recommendation through to Phase 3 (`PathPolicy.browser` for the
WAF-fronted host must match the warmup engine when
`matched-only`). See the toolkit reference
`skills/_shared/references/treat-as-success.md` for the underlying
mechanism.

**Proxy/IP hypothesis (record only if blocking was observed)**:

When the page loads but a specific endpoint 403s, or the page itself returns
a challenge, do not re-test through a different proxy tier — flag the
suspected cause for the human operator instead. Common patterns:

| Symptom | Likely cause | Operator action |
|---|---|---|
| 403 on every endpoint, page itself blocked | Datacenter IP on a site that requires residential, OR wrong-country IP on a geo-locked site | Re-run intel-agent with a residential / matching-country exit |
| Page OK, specific API 403 | API requires session cookies set by browser load (not raw HTTP) OR API geo-locked stricter than page | Browser warmup → cookie replay. If the WAF JA4-binds its session cookie (run the cookie JA4-binding probe above), the consumer's Tier 1 (impit) `browser` profile must match the warmup engine. Test API directly with same-country IP if not session-gated. |
| Cookies present, periodic 429 / rate-limit | IP reputation is fine, but rate budget low | Lower request rate or rotate session per N requests |
| TLS JA3 identical across calls | Upstream proxy re-terminating | Either disable proxy mid-stream or use `proxy_set_fingerprint_spoof` |

**Out of scope**: actually swapping proxies and re-running the recon.
intel-agent reports under one configuration. Tier swapping is an operator
decision based on the hypothesis above.

### Step 5: Rank Extraction Methods

For each data point, rank available methods. **Priority**: API > JSON-in-HTML > Cheerio > Browser.

Per method include: source details (endpoint/selector/JSON path), required proxy level, confidence (High/Medium/Low).

### Step 6: Generate Intelligence Report

Compile findings into the report format. See `reference/report-schema.md` for the complete structure.

**Output contract — critical**: the report is **written to a file**, not pasted into the chat. The report body is long (hundreds of lines); dumping it into the conversation wastes the caller's context window and makes the handoff harder (downstream skills read the file by path, not by scrolling back through chat).

- Write the full report to `./intel-<domain>-<YYYYMMDD>.md` via the Write tool.
- The chat message at the end of the run is a **short status summary only**: report path, HAR path, session_id, data-point coverage count (e.g., "5 of 6 AVAILABLE"), any deviations or limitations, and the handoff line.
- Do NOT paste the report body into chat. If the user explicitly says "show me the report in chat", that's a separate follow-up request after the file exists.

**6a. Export session evidence**:
```
# Read analysis resources (free aggregation — no manual counting of proxy_list_traffic):
ReadMcpResourceTool("proxy://sessions/{session_id}/summary")    # §6 totals + §2 status breakdown
ReadMcpResourceTool("proxy://sessions/{session_id}/findings")   # §2 rate-limit endpoints, §4 per-endpoint errors
ReadMcpResourceTool("proxy://sessions/{session_id}/timeline")   # §2 block-onset minute (60s buckets)

proxy_get_session_handshakes(session_id)   # JA3/JA4 coverage for §2
proxy_session_stop()
proxy_export_har(session_id: "[session_id]", file_path: "recon-[domain]-[YYYYMMDD].har", include_bodies: true)
```

Include in Section 6 (all REQUIRED):
- Session name, ID, HAR path, handshake coverage
- From `summary`: total exchanges, avg duration, top hostnames, status code breakdown
- From `findings`: high-error endpoints, slowest exchanges, host error rates (filter to target domain before quoting — excludes 3rd-party analytics/CDN noise)
- From `timeline`: bucket where `errorCount` spikes → "protection engaged after ~N requests" signal for §2

---

## What This Skill Does NOT Do

This skill is **pure discovery**. It does NOT write scraping code, create Actors, implement pagination, or deploy anything.

**Handoff**: After generating the report, pick one:

```
# Recommended — small validated POC first
Intelligence report complete. To build + validate a minimal POC scraper from
these findings, use the poc-agent skill:
  "build POC from intel report at /path/to/report.md"

# Or skip straight to production
To implement a production scraper, use the web-scraper skill:
  "scrape [data points] from [URL] using the intel report above"
```
