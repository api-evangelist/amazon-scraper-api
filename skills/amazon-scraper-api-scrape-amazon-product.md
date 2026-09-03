---
name: scrape-amazon-product
description: "Scrape live Amazon product data (price, rating, reviews, brand, stock, variants, images) by ASIN or URL across 20 marketplaces."
metadata: { "chocodata": { "emoji": "🛒", "requires": { "env": ["ASA_API_KEY"] } } }
allowed-tools: ["bash"]
---

# Scrape Amazon Product (by ASIN or URL)

Live Amazon product data, title, price, rating, review count, brand, availability, buybox seller, images, variants, features, description, and category ladder, for any ASIN across 20 Amazon marketplaces. Uses the Amazon Scraper API at `https://amazonscraperapi.com`.

## When to use this skill

Trigger when the user asks anything like:
- "What's the price/rating of Amazon ASIN B09HN3Q81F?"
- "Scrape this Amazon product: <URL>"
- "Is this item in stock on amazon.de?"
- "Pull title, brand, and reviews for this ASIN"
- "Compare this product's price on amazon.com vs amazon.co.uk vs amazon.de"
- "Monitor pricing for this listing"
- "Give me the variants and images for this Amazon product"

If the user pastes an Amazon URL, extract the ASIN (the 10-character code that follows `/dp/` or `/gp/product/`) or pass the full URL directly as the `query` parameter, both work.

## Setup

1. Get a free API key (1,000 requests, no credit card) at https://app.amazonscraperapi.com
2. Store the key as an environment variable. The key format is `asa_live_...`:
   - macOS/Linux: `export ASA_API_KEY="asa_live_xxx"`
   - Windows PowerShell: `$env:ASA_API_KEY = "asa_live_xxx"`
3. Never hard-code the key in source files. Read it from `os.environ` / `process.env`.

## API contract

Base URL: `https://api.amazonscraperapi.com/v1/`
Auth: header `X-API-Key: <key>`

### GET `/amazon/product`

Query params:

| param      | required | default | notes |
|------------|----------|---------|-------|
| `query`    | yes      | -       | 10-char ASIN (e.g. `B09HN3Q81F`) or full Amazon product URL |
| `domain`   | no       | `com`   | Marketplace TLD. Enum: `com, co.uk, de, fr, it, es, nl, pl, se, ca, com.mx, com.br, com.au, co.jp, sg, in, com.tr, ae, sa, eg` |
| `language` | no       | -       | Content language `xx_YY` (e.g. `en_US`, `de_DE`) |

### Response shape (key fields, ~55 total)

```json
{
  "asin": "B09HN3Q81F",
  "title": "...",
  "brand": "...",
  "price": { "current": 29.99, "was": 39.99, "savings_pct": 25, "currency": "USD" },
  "rating": 4.6,
  "review_count": 12483,
  "availability": "In Stock",
  "buybox_seller": "Amazon.com",
  "images": ["https://...", "..."],
  "variants": [{ "asin": "...", "name": "Color: Black", "price": 29.99 }],
  "features": ["bullet 1", "bullet 2"],
  "description": "...",
  "category_ladder": ["Electronics", "Headphones", "..."],
  "scraped_at": "2026-05-15T12:34:56Z"
}
```

Non-2xx status codes:
- `400` invalid ASIN or domain
- `401` missing/invalid API key
- `404` product not found on that marketplace
- `429` rate limit, back off and retry with exponential delay
- `5xx` upstream issue, retry once after 2-5 seconds

## Working code

### Python (uses `requests`)

```python
import os, requests

API_KEY = os.environ["ASA_API_KEY"]
BASE = "https://api.amazonscraperapi.com/v1"

def fetch_product(asin_or_url: str, domain: str = "com", language: str | None = None):
    params = {"query": asin_or_url, "domain": domain}
    if language:
        params["language"] = language
    r = requests.get(
        f"{BASE}/amazon/product",
        params=params,
        headers={"X-API-Key": API_KEY},
        timeout=30,
    )
    r.raise_for_status()
    return r.json()

if __name__ == "__main__":
    p = fetch_product("B09HN3Q81F", domain="com")
    print(p["title"], "|", p["price"]["current"], p["price"]["currency"], "|", p["rating"], "stars")
```

### curl

```bash
curl -G "https://api.amazonscraperapi.com/v1/amazon/product" \
  -H "X-API-Key: $ASA_API_KEY" \
  --data-urlencode "query=B09HN3Q81F" \
  --data-urlencode "domain=com"
```

### Node.js (built-in `fetch`)

```javascript
const key = process.env.ASA_API_KEY;
const url = new URL("https://api.amazonscraperapi.com/v1/amazon/product");
url.searchParams.set("query", "B09HN3Q81F");
url.searchParams.set("domain", "com");

const res = await fetch(url, { headers: { "X-API-Key": key } });
if (!res.ok) throw new Error(`HTTP ${res.status}`);
const product = await res.json();
console.log(product.title, product.price.current, product.rating);
```

## Common workflows

### 1. Quick price check
```python
p = fetch_product("B09HN3Q81F")
print(f"{p['title']}: {p['price']['current']} {p['price']['currency']}")
```

### 2. Stock alert (poll once, return boolean)
```python
p = fetch_product("B09HN3Q81F")
in_stock = "in stock" in p.get("availability", "").lower()
print("AVAILABLE" if in_stock else "OUT OF STOCK")
```

### 3. Compare price across marketplaces
```python
markets = ["com", "co.uk", "de", "fr", "it", "es"]
for d in markets:
    try:
        p = fetch_product("B09HN3Q81F", domain=d)
        print(f"amazon.{d}: {p['price']['current']} {p['price']['currency']}")
    except requests.HTTPError as e:
        print(f"amazon.{d}: {e.response.status_code}")
```

### 4. Rating + review snapshot
```python
p = fetch_product("B09HN3Q81F")
print(f"Rating {p['rating']} / 5  ({p['review_count']:,} reviews)  Brand: {p['brand']}")
```

### 5. Variant + image dump for a comparison table
```python
p = fetch_product("B09HN3Q81F")
for v in p.get("variants", []):
    print(v.get("name"), v.get("price"), v.get("asin"))
for i, img in enumerate(p.get("images", [])[:5]):
    print(f"img{i}: {img}")
```

## Pitfalls

- Do NOT try to fetch `https://www.amazon.com/dp/...` directly with `requests` or `fetch`. Amazon blocks vanilla clients with CAPTCHAs, IP throttles, and JS challenges. Use this API.
- Do NOT commit `ASA_API_KEY` to git. Read from env vars only.
- Always handle non-2xx responses. A missing product returns 404, not an empty 200.
- ASIN must be exactly 10 characters, case-sensitive. If the user pastes a URL, regex `/dp/([A-Z0-9]{10})` to extract.
- The `domain` value is just the TLD (`co.uk`, not `amazon.co.uk` or `https://...`).
- Some fields can be `null` or missing on certain listings (e.g. `price.was`, `variants`, `buybox_seller`). Guard with `.get()`.
- The free tier is 1,000 requests total, not per month. Cache results when iterating across marketplaces.

## Cost guidance

- Free tier: 1,000 requests on signup, no credit card.
- Pay-as-you-go: $0.90 per 1,000 successful requests after the free tier.
- For >100 ASINs in one job, ask the user if they want batch processing; the API also exposes a batch endpoint that is more efficient than looping single calls.
- Failed requests (4xx/5xx) do not count against your quota.
