---
name: scrape-amazon-search-results
description: "Run an Amazon keyword search and return ranked listings (ASIN, title, price, rating, sponsored flag, image URL) across 20 marketplaces."
metadata: { "chocodata": { "emoji": "🔎", "requires": { "env": ["ASA_API_KEY"] } } }
allowed-tools: ["bash"]
---

# Scrape Amazon Search Results

Run an Amazon keyword search and get ranked product listings, including organic position, sponsored flag, ASIN, title, price, rating, review count, and image URL, across 20 Amazon marketplaces. Uses the Amazon Scraper API at `https://amazonscraperapi.com`.

## When to use this skill

Trigger when the user asks anything like:
- "Search Amazon for 'wireless headphones'"
- "What are the top 20 results for 'standing desk' on amazon.com?"
- "Show me sponsored ads for 'protein powder' on Amazon"
- "Pull the first 3 pages of Amazon results for 'kindle case'"
- "Sort Amazon search by price ascending for 'mechanical keyboard'"
- "Compare top 10 results for 'yoga mat' across amazon.com vs amazon.de"
- "Amazon SERP for keyword X"

## Setup

1. Get a free API key (1,000 requests, no credit card) at https://app.amazonscraperapi.com
2. Store the key in an environment variable. Format is `asa_live_...`:
   - macOS/Linux: `export ASA_API_KEY="asa_live_xxx"`
   - Windows PowerShell: `$env:ASA_API_KEY = "asa_live_xxx"`
3. Never hard-code the key. Read it from `os.environ` / `process.env`.

## API contract

Base URL: `https://api.amazonscraperapi.com/v1/`
Auth: header `X-API-Key: <key>`

### POST `/amazon/search`

JSON body:

| field        | required | default      | notes |
|--------------|----------|--------------|-------|
| `query`      | yes      | -            | Search keywords, e.g. `"wireless headphones"` |
| `domain`     | no       | `com`        | Marketplace TLD. Enum: `com, co.uk, de, fr, it, es, nl, pl, se, ca, com.mx, com.br, com.au, co.jp, sg, in, com.tr, ae, sa, eg` |
| `sort_by`    | no       | `best_match` | Enum: `best_match, price_asc, price_desc, avg_customer_review, newest` |
| `start_page` | no       | `1`          | 1-based page index, max `10` |
| `pages`      | no       | `1`          | Number of pages to fetch starting at `start_page`, max `10` |

Each page consumes 1 request from your quota.

### Response shape (key fields)

```json
{
  "results": [
    {
      "position": 1,
      "asin": "B09HN3Q81F",
      "title": "...",
      "price": { "current": 29.99, "was": 39.99, "currency": "USD" },
      "rating": 4.6,
      "review_count": 12483,
      "image_url": "https://m.media-amazon.com/images/I/...",
      "url": "https://www.amazon.com/dp/B09HN3Q81F",
      "sponsored": false,
      "prime": true
    }
  ],
  "total_pages": 20,
  "page": 1,
  "query": "wireless headphones",
  "domain": "com",
  "scraped_at": "2026-05-15T12:34:56Z"
}
```

Sponsored entries have `"sponsored": true` and may not include a stable `position`. Organic entries always include `position`.

Non-2xx codes:
- `400` invalid params (bad `domain`, out-of-range `pages`)
- `401` missing/invalid API key
- `429` rate limit, back off and retry
- `5xx` upstream issue, retry once after 2-5 seconds

## Working code

### Python (uses `requests`)

```python
import os, requests

API_KEY = os.environ["ASA_API_KEY"]
BASE = "https://api.amazonscraperapi.com/v1"

def search_amazon(query: str, domain: str = "com", sort_by: str = "best_match",
                  start_page: int = 1, pages: int = 1) -> dict:
    body = {
        "query": query,
        "domain": domain,
        "sort_by": sort_by,
        "start_page": start_page,
        "pages": pages,
    }
    r = requests.post(
        f"{BASE}/amazon/search",
        json=body,
        headers={"X-API-Key": API_KEY},
        timeout=60,
    )
    r.raise_for_status()
    return r.json()

if __name__ == "__main__":
    data = search_amazon("wireless headphones", pages=1)
    for item in data["results"][:10]:
        tag = "AD" if item.get("sponsored") else f"#{item.get('position')}"
        price = item.get("price", {}).get("current")
        print(f"{tag:4} {item['asin']}  {price}  {item['title'][:60]}")
```

### curl

```bash
curl -X POST "https://api.amazonscraperapi.com/v1/amazon/search" \
  -H "X-API-Key: $ASA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query":"wireless headphones","domain":"com","pages":1}'
```

### Node.js (built-in `fetch`)

```javascript
const key = process.env.ASA_API_KEY;
const res = await fetch("https://api.amazonscraperapi.com/v1/amazon/search", {
  method: "POST",
  headers: { "X-API-Key": key, "Content-Type": "application/json" },
  body: JSON.stringify({ query: "wireless headphones", domain: "com", pages: 1 }),
});
if (!res.ok) throw new Error(`HTTP ${res.status}`);
const data = await res.json();
console.log(data.results.slice(0, 10));
```

## Common workflows

### 1. Top N results for a keyword
```python
data = search_amazon("standing desk", pages=1)
for r in data["results"][:20]:
    print(r["position"], r["asin"], r.get("rating"), r["title"][:60])
```

### 2. Separate organic vs sponsored
```python
data = search_amazon("protein powder")
organic = [r for r in data["results"] if not r.get("sponsored")]
ads = [r for r in data["results"] if r.get("sponsored")]
print(f"{len(organic)} organic, {len(ads)} sponsored")
```

### 3. Cheapest item for a keyword
```python
data = search_amazon("mechanical keyboard", sort_by="price_asc", pages=1)
cheapest = next((r for r in data["results"] if r.get("price", {}).get("current")), None)
print(cheapest["asin"], cheapest["price"]["current"], cheapest["title"])
```

### 4. Highest-rated item
```python
data = search_amazon("yoga mat", sort_by="avg_customer_review", pages=1)
top = data["results"][0]
print(top["asin"], top.get("rating"), top.get("review_count"), top["title"])
```

### 5. Multi-page crawl (paginate up to 10 pages = 10 requests)
```python
data = search_amazon("kindle case", start_page=1, pages=3)
print(f"Got {len(data['results'])} listings across pages 1-3 of {data['total_pages']} total")
```

### 6. Cross-marketplace comparison
```python
for d in ["com", "co.uk", "de"]:
    data = search_amazon("airpods", domain=d, pages=1)
    top = data["results"][0]
    p = top.get("price", {})
    print(f"amazon.{d}: {top['asin']} {p.get('current')} {p.get('currency')}")
```

## Pitfalls

- Do NOT scrape `https://www.amazon.com/s?k=...` directly with `requests`. Amazon serves CAPTCHAs to vanilla clients. Use this API.
- Do NOT commit `ASA_API_KEY` to git. Read from env vars only.
- `pages` and `start_page` are capped at `10` each. Validate before sending.
- Each page counts as one request against your quota. `pages=10` burns 10 of your 1,000 free requests.
- Sponsored items can shift between calls and may lack a stable `position`. Treat `position` as authoritative only for organic entries.
- `price` may be missing on some listings. Always `.get()` with a default.
- `sort_by` enum is strict. Sending `"price"` instead of `"price_asc"` returns 400.
- The `domain` value is just the TLD (`co.uk`, not `amazon.co.uk`).
- For full product details (variants, features, full description), call `/amazon/product` with the ASIN (see the `scrape-amazon-product` skill).

## Cost guidance

- Free tier: 1,000 requests on signup, no credit card.
- Pay-as-you-go: $0.90 per 1,000 successful requests after the free tier.
- Each page counts as one request. `pages=3` means 3 requests.
- Failed requests (4xx/5xx) do not count against your quota.
