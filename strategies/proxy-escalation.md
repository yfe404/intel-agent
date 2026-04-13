# Proxy Escalation: Three-Tier Protection Testing

Systematically test the target site across three proxy tiers to determine the minimum access level required.

## Overview

Sites employ varying levels of protection. Rather than guessing, test all three tiers upfront and record results. This tells you exactly what infrastructure the scraper will need.

**Three tiers**:
1. **Direct** — No upstream proxy (your own IP)
2. **Datacenter** — Datacenter proxy (cheap, fast, easily detected)
3. **Residential** — Residential proxy (expensive, slow, hard to detect)

---

## Prerequisites

Protection testing uses proxy-mcp's upstream proxy chaining. The MITM proxy is already running from Step 1 of the main workflow.

**Proxy credential configuration**: Datacenter and residential proxy URLs must include credentials. Common formats:

```
http://user:pass@dc-proxy.provider.com:8000
http://user:pass@residential-proxy.provider.com:8000
```

Apify proxy URLs:
```
# Datacenter
http://auto:[APIFY_TOKEN]@proxy.apify.com:8000

# Residential
http://groups-RESIDENTIAL,auto:[APIFY_TOKEN]@proxy.apify.com:8000
```

If proxy credentials are not available, skip Tiers 2-3 and note in the report that protection testing was limited to direct access only.

---

## Tier 1: Direct (Baseline)

This tier is already tested during Steps 1-3 of the main workflow. No upstream proxy — traffic goes directly from your machine through the MITM proxy to the target.

### What to record

**Status codes**:
```
proxy_list_traffic(url_filter: "[target domain]")
```
Check for 403, 429, 503, or redirect-to-challenge responses.

**Challenge pages**:
```
interceptor_browser_snapshot(target_id)
interceptor_browser_screenshot(target_id)
```
Look for:
- Cloudflare "Checking your browser..." interstitial
- DataDome CAPTCHA
- Akamai bot manager page
- PerimeterX challenge
- Generic "Access Denied" or "Unusual Activity" text

**Protection cookies**:
```
interceptor_browser_list_cookies(target_id)
```

Known protection cookie markers:

| Cookie | Protection System |
|--------|------------------|
| `cf_clearance` | Cloudflare |
| `__cf_bm` | Cloudflare Bot Management |
| `__cfduid` | Cloudflare (legacy) |
| `datadome` | DataDome |
| `_abck` | Akamai Bot Manager |
| `ak_bmsc` | Akamai |
| `bm_sv` | Akamai |
| `reese84` | Imperva/Incapsula |
| `px_` prefix | PerimeterX |

**Rate limit indicators**:

Check response headers on API endpoints (header-only peek is fine here — ring buffer is sufficient):
```
proxy_get_exchange(exchange_id)
```

Look for headers:
- `X-RateLimit-Limit` / `X-RateLimit-Remaining` / `X-RateLimit-Reset`
- `Retry-After`
- `X-Rate-Limit`

**TLS fingerprint baseline**:
```
proxy_get_tls_fingerprints()
```

### Record baseline result

```
Tier 1 (Direct):
  Main page: [OK / Blocked / Challenge]
  API endpoints: [OK / Blocked / Rate limited]
  Protection system: [None / Cloudflare / DataDome / Akamai / Other]
  Protection cookies: [list]
  Rate limit: [requests per minute if known]
```

---

## Tier 2: Datacenter Proxy

Route traffic through a datacenter proxy to test if the site allows datacenter IPs.

### Setup

```
proxy_set_upstream("[datacenter proxy URL]")
```

### Test

Navigate to the target and observe:

```
interceptor_browser_navigate(target_id, url: "[target URL]", wait_until: "networkidle")
interceptor_browser_screenshot(target_id)
interceptor_browser_snapshot(target_id)
```

Check traffic for blocks:
```
proxy_list_traffic(url_filter: "[target domain]")
```

If API endpoints were discovered in Step 3, test them through the datacenter proxy:
```
proxy_clear_traffic()
interceptor_browser_navigate(target_id, url: "[API endpoint URL]", wait_until: "networkidle")
proxy_list_traffic()
```

### Compare to baseline

- Same content loads as direct? → Datacenter proxy works
- Challenge page or 403? → Datacenter proxy blocked
- Different content (geo-restricted)? → Note geographic restrictions

### Record result

```
Tier 2 (Datacenter):
  Main page: [OK / Blocked / Challenge / Different content]
  API endpoints: [OK / Blocked / Rate limited]
  Compared to direct: [Same / Degraded / Blocked]
```

---

## Tier 3: Residential Proxy

Route traffic through a residential proxy for maximum stealth.

### Setup

```
proxy_set_upstream("[residential proxy URL]")
```

### Test

Same procedure as Tier 2:

```
interceptor_browser_navigate(target_id, url: "[target URL]", wait_until: "networkidle")
interceptor_browser_screenshot(target_id)
interceptor_browser_snapshot(target_id)
proxy_list_traffic(url_filter: "[target domain]")
```

Test API endpoints through residential proxy:
```
proxy_clear_traffic()
interceptor_browser_navigate(target_id, url: "[API endpoint URL]", wait_until: "networkidle")
proxy_list_traffic()
```

### Record result

```
Tier 3 (Residential):
  Main page: [OK / Blocked / Challenge / Different content]
  API endpoints: [OK / Blocked / Rate limited]
  Compared to direct: [Same / Degraded / Blocked]
```

---

## Clean Up

Remove upstream proxy after testing:

```
proxy_clear_upstream()
```

This returns traffic to direct routing for any subsequent investigation.

---

## Geographic Testing (If Needed)

If the site returns different content based on location, or if you suspect geographic restrictions, test with proxies in different regions.

Use per-host upstream proxy to target specific countries — **and relaunch the browser with matching `timezone` + `locale`** so server-side geo checks don't flag an IP/browser mismatch:

```
proxy_set_host_upstream("[target domain]", "[country-specific proxy URL]")

# Close current browser and relaunch with country-matched identity
interceptor_browser_close(target_id)
interceptor_browser_launch(
    url: "[target URL]",
    timezone: "Europe/Paris",   # match exit country
    locale: "fr-FR"              # match exit country
)
```

Test and record per-country results. Then clean up:

```
interceptor_browser_close(target_id)
proxy_remove_host_upstream("[target domain]")
# relaunch with defaults for the next tier
interceptor_browser_launch(url: "[target URL]")
```

**When to do geographic testing**:
- Site redirects to a country-specific version
- Prices or availability differ by region
- Content is geo-blocked or region-restricted
- User specifically needs data from a particular country's version

---

## Decision Logic

### Protection level determination

| Direct | Datacenter | Residential | Minimum Required Level |
|--------|-----------|-------------|----------------------|
| OK | OK | OK | **Direct** — No proxy needed |
| OK | OK | OK | (but rate limited) → **Direct with rate limiting** |
| Blocked | OK | OK | **Datacenter proxy** minimum |
| Blocked | Blocked | OK | **Residential proxy** required |
| OK (browser) | Blocked (API) | OK (API) | **Datacenter for browser, Residential for API** |
| Blocked | Blocked | Blocked | **Residential + advanced stealth** — may need CAPTCHA solving |

### Additional factors

**Stealth requirements**: If even residential proxies get blocked, check:
- Stealth is built in (cloakbrowser source-level C++ patches — no `stealthMode` toggle). Make sure the browser was launched via `interceptor_browser_launch`, not an external Chrome.
- Are humanizer interactions being used? (not raw Playwright instant clicks)
- Is there a JavaScript challenge that requires specific execution? Run `interceptor_browser_list_console(target_id)` and look for challenge-script errors.
- **Geo mismatch?** If using a country-specific upstream proxy, did you relaunch the browser with a matching `timezone` + `locale`? Server-side bot detection compares IP region vs browser timezone/locale — a mismatch alone can flag the session. See [Geographic Testing](#geographic-testing-if-needed).

**TLS requirements**: If switching from browser to HTTP-only client for production:
- Test with `proxy_set_fingerprint_spoof(preset: "chrome_136")` (only for HTTP clients, not browser sessions — browser already matches real Chrome via cloakbrowser)
- Compare results with and without TLS spoofing

**Rate limit budget**: Even if a tier works, note rate limits. The report should include:
- Observed rate limit threshold
- Recommended safe request rate (70-80% of threshold)
- Whether rate limits differ by proxy tier

---

## Example Report Section

```
## PROTECTION ASSESSMENT

| Access Method | Direct | Datacenter | Residential |
|--------------|--------|-----------|-------------|
| Main page    | OK     | OK        | OK          |
| Product API  | OK     | 403       | OK          |
| Search API   | OK     | 429       | OK          |

Protection system: Cloudflare (cf_clearance cookie detected)
Minimum required: Datacenter proxy for pages, Residential for APIs

Rate limits:
  - Product API: 60 req/min (X-RateLimit-Limit header)
  - Search API: 30 req/min (429 after ~30 rapid requests)
  - Recommended safe rate: 40 req/min (product), 20 req/min (search)

TLS/Stealth:
  - Browser: cloakbrowser default stealth sufficient (no additional patches needed)
  - HTTP client: TLS spoofing recommended if switching to gotScraping
```

---

## Related

- **Main workflow**: See `../SKILL.md` Step 4
- **Anti-blocking layers**: See anti-blocking strategy in the web-scraper skill
- **Proxy tool reference**: See proxy-tool-reference in the web-scraper skill
