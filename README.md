# Intel Agent

Reconnaissance skill for Claude Code that discovers **how** to extract data from any website — without writing a single line of scraping code.

Give it a URL and the data points you want. It launches a stealth browser through a MITM proxy, captures all traffic, sniffs APIs, tests protection levels, and returns a structured intelligence report with extraction strategies ranked by reliability.

## Quick Start

```
recon https://www.alza.cz/some-product-d7752990.htm
  data points: name, price, description, availability
```

The skill handles everything: Cloudflare challenge solving, lazy-content discovery via full-page scroll, TLS fingerprint verification, and HAR evidence export.

## What You Get

A structured report with:

- **Per-data-point extraction methods** ranked API > JSON-in-HTML > Cheerio > Browser, with exact JSON paths, CSS selectors, and API endpoint specs
- **Protection assessment** — Cloudflare/DataDome/Akamai detection, proxy tier requirements, TLS fingerprint analysis
- **Discovered API endpoints** — full specs (URL, method, headers, auth, pagination, rate limits)
- **Entity identifier mapping** — how the site's internal IDs relate to URLs and API parameters
- **HAR evidence file** — full request/response bodies for every exchange captured during recon

## What It Does NOT Do

Write scraping code, create Apify Actors, or run extraction at scale. For implementation, hand off to the web-scraper skill.

## Setup

### 1. Install proxy-mcp

The skill requires [proxy-mcp](https://www.npmjs.com/package/proxy-mcp) **≥ 2.0.0** (cloakbrowser + Playwright) as an MCP server. The optional camoufox hard-target fallback (Step 1) needs **proxy-mcp ≥ 3.0.0**. One-liner:

```bash
# For Claude Code
claude mcp add proxy-mcp -- npx -y proxy-mcp@latest
# For OpenCode
opencode mcp add proxy-mcp -- npx -y proxy-mcp@latest
```

Requires Node.js ≥ 20. First launch downloads a ~200 MB stealth Chromium binary (cached afterwards). To enable the camoufox path: `pip install "camoufox[geoip]" && python3 -m camoufox fetch && sudo apt install libnss3-tools` (or the macOS / Fedora equivalents).

### 2. Recommended permissions

The skill makes many MCP tool calls during a single recon session (traffic capture, browser control, humanizer interactions, session management). To avoid approving each one individually, add this to your project's `.claude/settings.local.json`:

```json
{
  "permissions": {
    "allow": [
      "mcp__proxy-mcp"
    ]
  }
}
```

or `opencode.json` for OpenCode:

```json
{
  "mcp": {
    "proxy-mcp": {
      "type": "local",
      "command": ["npx", "-y", "proxy-mcp@latest"]
    }
  }
}
```

This auto-approves all proxy-mcp tools. All other tools still require manual approval.

### 3. Install the skill

Clone or copy this directory into your project:

```bash
git clone https://github.com/yfe404/intel-agent.git
```

Alternatively, you can also put the skill into your global config directory to use it in any project (.clade/skills is recognized by OpenCode as well):
```bash
mkdir -p ~/.claude/skills && git clone https://github.com/yfe404/intel-agent.git ~/.claude/skills/intel-agent
```

## Usage

```
# Full recon with specific data points
recon https://www.example.com/product/123
  data points: name, price, description, reviews, availability

# Let the skill ask what you need
intel report for https://news.example.com

# Direct question format
what's the best way to get product name, price, stock status from https://shop.example.com
```

## How It Works

1. **Initialize** — Start MITM proxy with full-body persistence; launch cloakbrowser (stealth Chromium, source-level fingerprint patches, humanize on by default) via Playwright. For hard targets that block Chromium fingerprints (Cloudflare Turnstile, Akamai bot manager), fall back to camoufox (anti-detect Firefox via `firefox.connect(wsUrl)`) — proxy-mcp ≥ 3.0.0
2. **Scan for data** — Search raw HTML (full decompressed bodies), JSON blobs, web storage (local + session), and rendered ARIA snapshot for each data point
3. **Full-page scroll** — Mandatory scroll to trigger lazy-loaded APIs (descriptions, reviews, carousels)
4. **Sniff APIs** — Filter traffic for JSON endpoints, trigger interactions via locator-based clicks (`role+name` / `text` / `label`) to discover pagination/search/filter APIs
5. **Test protection** — Check Cloudflare/DataDome/Akamai indicators, verify TLS fingerprint passthrough, test proxy tiers (paired locale + timezone for geo-specific upstreams)
6. **Export** — Generate structured report + HAR file with full request/response bodies

## File Structure

```
intel-agent/
├── SKILL.md                              # Main workflow (~200 lines)
├── README.md                             # This file
├── reference/
│   ├── report-schema.md                  # Report output format
│   ├── tool-reference.md                 # Tool signatures, caveats, rules
│   └── data-point-types.md              # Type classification & search strategies
└── strategies/
    └── cheerio-vs-browser-test.md        # Three-way extraction test procedure
```

## Requires

- **Claude Code** with MCP support
- **proxy-mcp** ≥ 2.0.0 — MITM traffic interception with full-body on-disk persistence, cloakbrowser stealth browser, Playwright-driven locators, humanizer, session recording
- **Node.js** ≥ 20

(cloakbrowser ships its own stealth Chromium binary — no separate Chrome install needed.)
