# Intelligence Report Schema

Canonical output format for the intel-agent skill. Every report follows this structure.

---

## Report Template

```
================================================================
INTEL REPORT: [domain.com]
================================================================
Target: [full URL]
Date: [YYYY-MM-DD HH:MM UTC]
Requested data points: [comma-separated list]
================================================================


## 1. SITE PROFILE

Framework: [Next.js / React SPA / WordPress / Static HTML / etc.]
Rendering: [SSR / CSR / SSG / Hybrid SSR+CSR]
Primary data source: [Internal REST API / GraphQL / HTML-embedded JSON / Static HTML]
Page type: [Product page / Listing page / Article / Search results / etc.]
URL routing:
  Pattern: [how entity IDs are embedded in URLs, e.g., "/product-slug-d{commodityId}.htm"]
  Redirect behavior: [e.g., "slug is decorative, numeric ID is authoritative, may redirect to canonical slug"]
  Canonical URL: [final URL after redirects, or from <link rel="canonical">]
  Authoritative identifier: [which part of the URL is the real key, e.g., "numeric ID in d{id}.htm"]
Entity identifiers:
  - [identifier name]: [value or pattern] — used by [list of endpoints]
  - [identifier name]: [value or pattern] — used by [list of endpoints]


## 2. PROTECTION ASSESSMENT

Active proxy/IP: [describe what was used — e.g., "Apify residential, country=US" or "Direct (no upstream)"]

| Access Method | Status | Blocked? | Challenge | Notes |
|--------------|--------|----------|-----------|-------|
| Main page    | [200/403/...] | [No/Yes] | [None / JS / CAPTCHA / Interstitial] | [cookies set, redirects, etc.] |
| [Endpoint 1] | [200/403/429] | [No/Yes] | [None/...] | [...] |
| [Endpoint 2] | [200/403/429] | [No/Yes] | [None/...] | [...] |

Protection system: [None / Cloudflare / DataDome / Akamai / Imperva / PerimeterX / Custom]
Rate limits: [observed limits per endpoint]
  - [Endpoint]: [N] req/min → recommended safe rate: [M] req/min
Stealth requirements: [cloakbrowser default sufficient / TLS spoofing needed for HTTP clients / advanced measures needed]
TLS fingerprint verification:
  - ClientHello passthrough: [Yes — browser ClientHello forwarded to target / No — proxy re-terminates TLS]
  - JA3 behavior: [varies per-connection (Chrome randomization) / identical across requests (proxy fingerprint)]
  - JA4 observed: [value, e.g., "t13d1517h2_8daaf6152771_b6f405a00624"]
  - Implication: [e.g., "HTTP-only clients need proxy_set_fingerprint_spoof(chrome_136)" or "Browser sessions are transparent"]
Protection cookies: [list of protection cookies observed]
Tier 1 cookie replay: [matched-only / cross-engine / untested]
  - Probe: replay one representative GET against the warmed jar with both impit `browser=firefox` and `browser=chrome` (see SKILL.md "Cookie JA4-binding probe").
  - matched-only → consumer's `PathPolicy.browser` MUST match the cloak warmup engine; mismatched profile mints fresh CID and serves challenge.
  - cross-engine  → any impit profile works; `PathPolicy.browser` only needs to match the cloak engine choice (Imperva/Linux-Chrome avoidance), not the impit profile.
  - untested      → could not run the probe (Tier 1 unreachable for other reasons, or only one profile attempted).
  - Evidence: [byte counts / fresh-CID header presence / status code from each profile]

Proxy/IP hypothesis (only if blocking observed under the active config):
  - Suspected cause: [datacenter IP on residential-only site / wrong-country IP on geo-locked site / API requires browser-warmed cookies / IP reputation / etc.]
  - Operator action: [re-run with residential exit / matching-country IP / browser-warmup → cookie replay / lower request rate / etc.]
  - Confidence: [High / Medium / Low — based on observed signal]


## 3. DATA POINT ANALYSIS

### 3.1 [Data Point Name]

Status: [AVAILABLE / PARTIALLY AVAILABLE / NOT FOUND]
Type: [text / numeric / boolean / list / nested]

Methods (ranked best → worst):

  1. [Best method] ← RECOMMENDED
     - Type: [API / JSON-in-HTML / Cheerio / Browser]
     - [For API]: Endpoint: [URL], Method: [GET/POST]
     - [For API]: Headers: [required headers]
     - [For API]: Auth: [none / cookie / token / API key]
     - [For API]: JSON path: [path to data point in response]
     - [For JSON-in-HTML]: Source: [__NEXT_DATA__ / ld+json / __INITIAL_STATE__]
     - [For JSON-in-HTML]: JSON path: [path to data point]
     - [For Cheerio]: Selector: [CSS selector]
     - [For Browser]: Snapshot location: [accessibility tree path]
     - [For Browser]: Required interaction: [none / scroll / click selector]
     - Confidence: [High / Medium / Low]

  2. [Fallback method]
     - [same fields as above]

### 3.2 [Data Point Name]

[same structure as 3.1]

[...repeat for each data point...]


## 4. DISCOVERED ENDPOINTS

### 4.1 [Endpoint Name]

URL: [full URL with parameter placeholders]
Method: [GET / POST]
Content-Type: [application/json / text/html / etc.]

Request:
  Headers:
    - [header]: [value or description]
  Parameters:
    - [param]: [type] — [description]
  Body (POST only):
    [body structure or example]

Response:
  Status: [200 / etc.]
  Structure:
    [JSON structure summary or example]
  Key fields:
    - [field]: [type] — [which data point this serves]

Authentication: [None / Cookie-based / Bearer token / API key]
Pagination:
  - Type: [offset / cursor / page-based / none]
  - Parameters: [page, limit, offset, cursor, etc.]
  - Total items indicator: [field name or "not provided"]
  - Has-more indicator: [field name or "not provided"]
Rate limit:
  - Observed: [N req/min or "not tested"]
  - Headers: [X-RateLimit-* values if present]

Data points covered: [list of data points this endpoint provides]

### 4.2 [Endpoint Name]

[same structure as 4.1]

[...repeat for each endpoint...]


## 5. RECOMMENDED STRATEGY

Summary:

| Data Point | Best Method | Endpoint/Source | Confidence |
|-----------|-------------|-----------------|------------|
| [name]   | [API/JSON/Cheerio/Browser] | [endpoint or source] | [H/M/L] |
| [name]   | [API/JSON/Cheerio/Browser] | [endpoint or source] | [H/M/L] |

Complexity estimate: [Low / Medium / High]
  - [Low]: All data from single API or JSON-in-HTML, no auth, no rate limits
  - [Medium]: Multiple sources, some auth or rate limits, but straightforward
  - [High]: Browser required, complex auth, strict rate limits, or multiple interaction steps

Estimated total items: [count or "unknown"]
Rate limit budget: [total requests needed] at [safe rate] = [estimated time]

Notes:
  - [any caveats, edge cases, or additional observations]


## 6. RAW EVIDENCE

Session ID: [REQUIRED — returned by proxy_start with persistence_enabled: true]
Session name: [intel-<target-domain>-<timestamp>]
HAR export: [REQUIRED — path written by proxy_export_har with include_bodies: true]
Capture profile: [must be "full" for intel-agent runs]
Screenshots: [list of screenshot file paths taken during reconnaissance]

Session summary (from `proxy://sessions/{id}/summary`):
  - Total exchanges: [N]
  - Avg duration: [N ms]
  - Top hostnames: [{hostname, count} × up to 10 — useful for CDN/analytics attribution]
  - Status code breakdown: [{200: N, 403: N, 429: N, ...}]
  - Method breakdown: [{GET: N, POST: N, ...}]

Session findings (from `proxy://sessions/{id}/findings`, filtered to target domain):
  - High-error endpoints: [{endpoint, errors} × up to 10]
  - Slowest exchanges: [{exchangeId, duration_ms, url} × up to 10]
  - Host error rates: [{hostname, total, errors, errorRate} — target domain row primary]

Session timeline (from `proxy://sessions/{id}/timeline`, 60s buckets):
  - Buckets: [{bucketStart, count, errorCount} series]
  - Block-onset observation: [e.g., "errorCount jumped from 0 to 7/12 at bucket 3 → ~N requests in ~3 minutes"]

Handshake metadata: [JA3/JA4 coverage — from proxy_get_session_handshakes]

================================================================
END OF REPORT
================================================================
```

---

## Field Definitions

### Status Values

- **AVAILABLE**: Data point found reliably in at least one source with high confidence
- **PARTIALLY AVAILABLE**: Data point found but with caveats (only on some pages, requires specific conditions, inconsistent presence)
- **NOT FOUND**: Data point not discovered in any source during reconnaissance
- **INCONCLUSIVE**: Data point search was limited (e.g. all sources checked but element only appears after a specific interaction we did not trigger). Cannot confirm presence or absence. The report notes what prevented a definitive check.

### Confidence Levels

- **High**: Data point clearly present, tested and verified, consistent across multiple checks
- **Medium**: Data point found but structure may vary across pages, or only tested on limited samples
- **Low**: Data point sometimes present, or extraction method is uncertain (e.g., relies on specific page state)
- **Inconclusive**: Cannot determine — element only surfaces after an interaction we did not trigger, or a CAPTCHA stopped traversal. Re-run with the missing interaction explicitly performed (or with valid proxy credentials for protection-blocked endpoints) for definitive results.

### Complexity Estimates

- **Low**: Single data source, no authentication, no rate limits, straightforward extraction. Implementation: < 1 hour.
- **Medium**: Multiple data sources or endpoints, some authentication or rate limits, but well-documented extraction paths. Implementation: 1-3 hours.
- **High**: Browser required for some data points, complex authentication flows, strict rate limits, multiple interaction steps, or data spread across many page types. Implementation: 3+ hours.

---

## Guidelines for Report Generation

1. **Be specific** — Include exact URLs, exact JSON paths, exact selectors. The report should contain enough detail to implement without re-running reconnaissance.

2. **Be honest about confidence** — Mark things as "Low" confidence when uncertain rather than overstating findings.

3. **Include all methods found** — Even if an API method exists, also document the Cheerio/Browser fallback if available. Methods fail; fallbacks save time.

4. **Note what wasn't found** — If a requested data point wasn't discovered, say so clearly with notes on what was tried.

5. **Keep endpoint specs complete** — Every endpoint should have enough detail (URL, method, headers, auth, pagination) to make a working request without revisiting the site.

6. **Record rate limits precisely** — Include the actual header values or observed thresholds, not just "rate limited."

7. **Separate facts from recommendations** — Sections 1-4 are facts (what was observed). Section 5 is the recommendation (what to do). Keep them distinct.

---

## Related

- **Main workflow**: See `../SKILL.md` Step 6
- **Cheerio vs Browser test**: See `../strategies/cheerio-vs-browser-test.md`
