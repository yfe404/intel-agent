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
interceptor_chrome_devtools_snapshot()
interceptor_chrome_devtools_screenshot()
```
Look for:
- Cloudflare "Checking your browser..." interstitial
- DataDome CAPTCHA
- Akamai bot manager page
- PerimeterX challenge
- Generic "Access Denied" or "Unusual Activity" text

**Protection cookies**:
```
interceptor_chrome_devtools_list_cookies()
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

Check response headers on API endpoints:
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
interceptor_chrome_devtools_navigate("[target URL]")
humanizer_idle(target_id, 3000)
interceptor_chrome_devtools_screenshot()
interceptor_chrome_devtools_snapshot()
```

Check traffic for blocks:
```
proxy_list_traffic(url_filter: "[target domain]")
```

If API endpoints were discovered in Step 3, test them through the datacenter proxy:
```
proxy_clear_traffic()
interceptor_chrome_devtools_navigate("[API endpoint URL]")
humanizer_idle(target_id, 2000)
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
interceptor_chrome_devtools_navigate("[target URL]")
humanizer_idle(target_id, 3000)
interceptor_chrome_devtools_screenshot()
interceptor_chrome_devtools_snapshot()
proxy_list_traffic(url_filter: "[target domain]")
```

Test API endpoints through residential proxy:
```
proxy_clear_traffic()
interceptor_chrome_devtools_navigate("[API endpoint URL]")
humanizer_idle(target_id, 2000)
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

Use per-host upstream proxy to target specific countries:

```
proxy_set_host_upstream("[target domain]", "[country-specific proxy URL]")
```

Test and record per-country results. Then clean up:

```
proxy_remove_host_upstream("[target domain]")
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
- Is `stealthMode: true` enabled? (should always be)
- Are humanizer interactions being used? (not raw DOM clicks)
- Is there a JavaScript challenge that requires specific execution?

**TLS requirements**: If switching from browser to HTTP-only client for production:
- Test with `proxy_set_fingerprint_spoof(preset: "chrome_136")` (only for HTTP clients, not browser sessions)
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
  - Browser stealth mode: sufficient (no additional patches needed)
  - HTTP client: TLS spoofing recommended if switching to gotScraping
```

---

## Related

- **Main workflow**: See `../SKILL.md` Step 4
- **Anti-blocking layers**: See anti-blocking strategy in the web-scraper skill
- **Proxy tool reference**: See proxy-tool-reference in the web-scraper skill
