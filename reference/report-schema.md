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


## 2. PROTECTION ASSESSMENT

| Access Method | Direct | Datacenter | Residential |
|--------------|--------|-----------|-------------|
| Main page    | [OK/Blocked/Challenge] | [OK/Blocked/Challenge] | [OK/Blocked/Challenge] |
| [Endpoint 1] | [OK/Blocked/429] | [OK/Blocked/429] | [OK/Blocked/429] |
| [Endpoint 2] | [OK/Blocked/429] | [OK/Blocked/429] | [OK/Blocked/429] |

Protection system: [None / Cloudflare / DataDome / Akamai / Imperva / PerimeterX / Custom]
Minimum required proxy level: [Direct / Datacenter / Residential]
Rate limits: [observed limits per endpoint]
  - [Endpoint]: [N] req/min → recommended safe rate: [M] req/min
Stealth requirements: [cloakbrowser default sufficient / TLS spoofing needed for HTTP clients / advanced measures needed]
Protection cookies: [list of protection cookies observed]


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
     - Required proxy level: [Direct / Datacenter / Residential]
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

| Data Point | Best Method | Endpoint/Source | Proxy Level | Confidence |
|-----------|-------------|-----------------|-------------|------------|
| [name]   | [API/JSON/Cheerio/Browser] | [endpoint or source] | [Direct/DC/Residential] | [H/M/L] |
| [name]   | [API/JSON/Cheerio/Browser] | [endpoint or source] | [Direct/DC/Residential] | [H/M/L] |

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
Screenshots: [list of screenshot file paths taken during reconnaissance]
Traffic summary: [number of exchanges captured, total bytes written to session]
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

### Confidence Levels

- **High**: Data point clearly present, tested and verified, consistent across multiple checks
- **Medium**: Data point found but structure may vary across pages, or only tested on limited samples
- **Low**: Data point sometimes present, or extraction method is uncertain (e.g., relies on specific page state)

### Proxy Levels

- **Direct**: No upstream proxy needed — the method works from any IP
- **Datacenter**: Requires at minimum a datacenter proxy
- **Residential**: Requires a residential proxy for reliable access

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
- **Proxy escalation**: See `../strategies/proxy-escalation.md`
