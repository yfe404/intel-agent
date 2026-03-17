# Intel Agent

Reconnaissance skill that discovers **how** to extract specific data points from a website — without implementing the scraper.

## What It Does

Give it a URL and a list of data points you want. It returns a structured intelligence report telling you:

- **Where** each data point lives (API endpoint, embedded JSON, raw HTML, rendered DOM)
- **How** to extract it (JSON path, CSS selector, accessibility tree location)
- **What protection** the site uses and what proxy level you need
- **Which method is best** for each data point (API > JSON-in-HTML > Cheerio > Browser)

## What It Does NOT Do

- Write scraping code
- Create Apify Actors
- Run extraction at scale

For implementation, use the [web-scraper](../web-scraper/) skill.

## Installation

Add this directory to your Claude Code skills:

```bash
# Clone or copy to your skills location
cp -r intel-agent/ /path/to/your/skills/
```

## Usage

```
recon https://shop.example.com/products

intel report for https://news.example.com
  data points: article title, author, publish date, body text, tags

find how to extract product name, price, reviews, stock status from https://shop.example.com/widget-pro
```

## Example Output (Abbreviated)

```
================================================================
INTEL REPORT: shop.example.com
================================================================
Target: https://shop.example.com/products/widget-pro
Date: 2026-03-17 14:30 UTC
Requested data points: product name, price, reviews, stock status

## 1. SITE PROFILE
Framework: Next.js 14 (SSR + CSR hybrid)
Primary data source: Internal REST API + __NEXT_DATA__ JSON

## 2. PROTECTION ASSESSMENT
| Access Method | Direct | Datacenter | Residential |
|--------------|--------|-----------|-------------|
| Main page    | OK     | OK        | OK          |
| Product API  | OK     | 403       | OK          |
Minimum required: Residential proxy for API access

## 3. DATA POINT ANALYSIS
### 3.1 Product Name
Status: AVAILABLE
  1. JSON-in-HTML ← RECOMMENDED
     Source: __NEXT_DATA__
     JSON path: props.pageProps.product.name
     Confidence: High

### 3.2 Price
Status: AVAILABLE
  1. API ← RECOMMENDED
     Endpoint: GET /api/v2/products/{id}
     JSON path: product.price.current
     Confidence: High

### 3.3 Reviews
Status: AVAILABLE
  1. API ← RECOMMENDED
     Endpoint: GET /api/v2/products/{id}/reviews?page={n}&limit=20
     Confidence: High

### 3.4 Stock Status
Status: AVAILABLE
  1. JSON-in-HTML ← RECOMMENDED
     Source: __NEXT_DATA__
     JSON path: props.pageProps.product.inStock
     Confidence: High

## 5. RECOMMENDED STRATEGY
| Data Point    | Method       | Source           | Proxy    | Confidence |
|--------------|--------------|------------------|----------|------------|
| Product name | JSON-in-HTML | __NEXT_DATA__    | Direct   | High       |
| Price        | API          | /api/v2/products | Resident.| High       |
| Reviews      | API          | /api/v2/.../reviews | Resident.| High    |
| Stock status | JSON-in-HTML | __NEXT_DATA__    | Direct   | High       |

Complexity: Medium
================================================================
```

## How It Works

1. **Initialize** — Start MITM proxy, launch stealth Chrome, capture baseline traffic
2. **Scan for data points** — Search raw HTML, JSON blobs, and rendered DOM for each requested value
3. **Sniff APIs** — Filter captured traffic for JSON endpoints, trigger interactions to discover more
4. **Test protection** — Escalate through Direct → Datacenter → Residential proxy tiers
5. **Rank methods** — For each data point, rank available extraction methods
6. **Generate report** — Output structured intelligence report

## Requires

- **proxy-mcp** — MITM traffic interception, stealth Chrome, DevTools bridge, humanizer

## File Structure

```
intel-agent/
├── SKILL.md                          # Main workflow (start here)
├── README.md                         # This file
├── strategies/
│   ├── cheerio-vs-browser-test.md    # Three-way extraction test procedure
│   └── proxy-escalation.md           # Protection testing across proxy tiers
└── reference/
    └── report-schema.md              # Canonical output report format
```

## Relationship to web-scraper

The **intel-agent** is the "what and where" — it discovers extraction paths.
The **web-scraper** is the "how" — it implements the scraper.

Typical workflow:
1. Run intel-agent to get the intelligence report
2. Hand off to web-scraper: "scrape [data points] from [URL] using the intel report above"
