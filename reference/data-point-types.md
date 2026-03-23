# Data Point Types & Search Strategies

Reference for classifying data points and choosing search strategies during reconnaissance.

---

## Type Classification

- **text** — Single string value (name, title, description)
- **numeric** — Number or currency (price, rating, count)
- **boolean** — True/false state (in stock, verified, active)
- **list** — Repeating items (reviews, variants, images)
- **nested** — Grouped sub-fields (address with street/city/zip, product with name/price/sku)

---

## Search Strategies by Type

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
