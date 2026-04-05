# Best Practices: Marketing & Niche Research

---

## 1. Principles

- **Demand first, product second.** Find what people are already paying to search for, then build the thing. Never guess a niche.
- **No discretionary decisions.** Every choice must come from data. If a human is picking the category, the seed keyword, or the niche — it's a discretionary decision and will bias the output.
- **The 3 questions that matter before building anything:**
  1. Will they convert? (ad history + landing page intent match)
  2. Will they click my ad? (existing ad copy + messaging gap)
  3. Can I build it? (fulfillment reality check — if not, skip and move on)
- **Speed to signal over perfection.** Launch fast, read real data, iterate. A live page with $200 in ad spend tells you more than weeks of research.
- **People click ≠ people buy.** CTR and CPC are not conversion signals. Only purchases are conversion signals.
- **High volume ≠ good opportunity.** Top searches by volume are almost always navigational brand queries. Filter by CPC + competition + intent.
- **Long-running ads = proof of profitability.** If a competitor has been running the same ad for 3+ months, it's converting. Fresh ads everywhere = unproven market.
- **CPC range $2-$15 is the learnable zone.** Under $2 = low commercial intent. Over $15 = too expensive to test without high conversion confidence.
- **The 7-day wait comes AFTER launch**, not before. Pre-launch data tells you market size. Post-launch data tells you if your specific offer converts.
- **Build for a specific buyer with a specific problem**, not a general audience. "Better AI" is not a differentiator. "Built for X type of business that needs Y specific output" is.

---

## 2. Data Findings

### Top Categories by Commercial Value (from kdp.amazon.com analysis)
Pulled via `categories_for_domain` endpoint. Ranked by `estimated_paid_traffic_cost` (monthly estimate of what it would cost to buy all organic traffic via Google Ads):

| Category ID | Category Name | Est. Paid Traffic Value |
|---|---|---|
| 10108 | News, Media & Publications | $4,119,888 |
| 11867 | Publishing | $3,212,690 |
| 12973 | Self-Publishing | $3,126,532 |
| 13575 | Retailers & General Merchandise | $780,574 |
| 13860 | Shopping Portals & Search Engines | $772,343 |
| 10877 | Portable Media Devices | $665,641 |
| 10167 | Consumer Electronics | $668,972 |

**Note:** These categories came from KDP (Amazon's publishing platform). The pipeline should eventually be run against all root categories to avoid bias from the seed domain.

### Filter Criteria That Work
- CPC > $2 — confirms commercial intent
- Competition > 0.1 — confirms advertisers are actively bidding
- Intent = `transactional` or `commercial` — confirms buying behavior, not browsing

### What Doesn't Work
- Sorting `top_searches` by volume — returns YouTube, Amazon, ChatGPT misspellings and brand navigational queries
- Filtering for navigational intent — useless for finding buildable products
- Using a single seed domain — biases all downstream category and keyword data toward that domain's business

### Keywords That Passed All Filters (from Self-Publishing category)
From `keywords_for_categories` on categories [10108, 11867, 12973, 10167, 10877], 1000 keywords pulled, filtered to CPC>$2, competition>0.1, transactional/commercial intent:

Notable signals:
- `kindle direct publishing` — 201k vol, $6.67 CPC, commercial intent
- `digital publishing solutions` — 201k vol, $8.93 CPC
- `kindle book publishing` — 201k vol, $6.67 CPC

**Observation:** Most results were Amazon brand navigational queries. The self-publishing opportunity exists but needs deeper drilling with the full category pipeline.

### Full pipeline run (April 2026)
- 368 categories, $8.05 total DataForSEO cost
- 43,667 qualifying keywords returned
- average ~73 keywords per category (well below the 1,000 limit — most categories are sparse)
- strongest unprompted signal: Myers Briggs personality test keywords at $1.65 CPC, 1.4M monthly searches
- real cost per keyword: ~$0.0003 (DataForSEO charges $0.015 per 50 results)

---

## 3. Technical Reference

### Authentication
DataForSEO uses Basic Auth. Credentials are encoded as Base64.

Format: `email:api_password` → encode to Base64

Account: your DataForSEO email
API Password: found in DataForSEO dashboard under "API Credentials" (different from account login password)
Base64: encode `your_email:your_api_password` using Base64 — run `echo -n 'email:password' | base64` in terminal

All curl commands use:
```
--header "Authorization: Basic YOUR_BASE64_HERE"
--header "Content-Type: application/json"
```

The full `Basic YOUR_BASE64_HERE` string (including the word "Basic" and the space) must be stored as one value in env vars. Storing just the Base64 part and prepending "Basic" in code is fine, but make sure the server actually does that — omitting "Basic " causes 401 errors.

---

### Endpoints

#### 1. Top Searches
**What it does:** Returns highest-volume keywords in the US with no seed input.
**Problem:** Sorted by volume = returns navigational brand queries. Does not support filters.
**Use for:** Getting a broad unsorted dump to filter locally.

```bash
curl --location --request POST "https://api.dataforseo.com/v3/dataforseo_labs/google/top_searches/live" \
--header "Authorization: Basic YOUR_BASE64_HERE" \
--header "Content-Type: application/json" \
--data-raw '[{
  "location_name": "United States",
  "language_name": "English",
  "limit": 1000
}]'
```

---

#### 2. Categories for Domain
**What it does:** Returns all Google product/service categories a domain ranks for, with traffic and commercial value metrics.
**Use for:** Discovering which categories have the most commercial activity for a given domain.
**Key metric:** `estimated_paid_traffic_cost` — higher = more commercially valuable category.

```bash
curl --location --request POST "https://api.dataforseo.com/v3/dataforseo_labs/google/categories_for_domain/live" \
--header "Authorization: Basic YOUR_BASE64_HERE" \
--header "Content-Type: application/json" \
--data-raw '[{
  "target": "kdp.amazon.com",
  "location_name": "United States",
  "language_name": "English"
}]'
```

---

#### 3. Category List (Free)
**What it does:** Returns full list of all Google product/service categories with their IDs and names.
**Cost:** Free.
**Use for:** Decoding category IDs into human-readable names.

```bash
curl --location --request GET "https://api.dataforseo.com/v3/dataforseo_labs/categories" \
--header "Authorization: Basic YOUR_BASE64_HERE" \
--header "Content-Type: application/json" > categories.json
```

---

#### 4. Keywords for Categories
**What it does:** Returns keywords within specified category IDs with CPC, volume, competition, and intent data.

**CRITICAL LIMITATION: Does NOT support `filters` or `order_by` parameters — confirmed through live testing.**
If you send a filters array, DataForSEO silently ignores it and returns the full unfiltered result set.
All CPC, competition, intent, and volume filtering must happen locally after results come back.
Do not build UI that implies server-side filtering is happening for this endpoint — it isn't.

**Workaround:** Pull raw data and filter locally.

```bash
curl --location --request POST "https://api.dataforseo.com/v3/dataforseo_labs/google/keywords_for_categories/live" \
--header "Authorization: Basic YOUR_BASE64_HERE" \
--header "Content-Type: application/json" \
--data-raw '[{
  "category_codes": [12973, 11867, 10108],
  "location_name": "United States",
  "language_name": "English",
  "limit": 1000
}]' > category_keywords.json
```

Filter locally:
```bash
python3 -c "
import json
with open('category_keywords.json') as f:
    data = json.load(f)
items = data['tasks'][0]['result'][0]['items']
results = []
for item in items:
    kw = item['keyword_data']['keyword']
    info = item['keyword_data']['keyword_info']
    intent = item['keyword_data'].get('search_intent_info') or {}
    cpc = info.get('cpc') or 0
    competition = info.get('competition') or 0
    volume = info.get('search_volume') or 0
    main_intent = intent.get('main_intent', '')
    if cpc > 2 and competition > 0.1 and main_intent in ['transactional', 'commercial']:
        results.append({'keyword': kw, 'cpc': cpc, 'competition': competition, 'volume': volume, 'intent': main_intent})
results.sort(key=lambda x: x['cpc'], reverse=True)
for r in results[:30]:
    print(f\"{r['keyword']:<45} CPC: \${r['cpc']:<8} Vol: {r['volume']:<10} Competition: {r['competition']}\")
print(f'Total: {len(results)}')
"
```

---

#### 5. Keyword Ideas
**What it does:** Returns keyword ideas related to seed keywords, with CPC, volume, competition, and intent data.
**Supports filters:** Yes — can filter by CPC, competition, intent at API level.
**Use for:** Drilling into a specific niche once identified. Unlike keywords_for_categories, this endpoint genuinely supports server-side filtering.

```bash
curl --location --request POST "https://api.dataforseo.com/v3/dataforseo_labs/google/keyword_ideas/live" \
--header "Authorization: Basic YOUR_BASE64_HERE" \
--header "Content-Type: application/json" \
--data-raw '[{
  "keywords": ["your seed keyword here"],
  "location_name": "United States",
  "language_name": "English",
  "limit": 50,
  "filters": [
    ["keyword_info.cpc", ">", 2],
    "and",
    ["keyword_info.cpc", "<", 15],
    "and",
    ["keyword_info.search_volume", ">", 1000],
    "and",
    ["search_intent_info.main_intent", "=", "transactional"]
  ]
}]'
```

---

#### 6. SERP + Ads (Planned)
**What it does:** Returns live Google search results including paid ads for a keyword.
**Use for:** Seeing what competitors are running, how long ads have been active, what messaging is being used.

```
POST https://api.dataforseo.com/v3/serp/google/organic/live/advanced
```

---

#### 7. Ads Advertisers + Ads Search (Planned)
**What it does:** Returns advertiser data and ad creatives from Google Ads Transparency Center.
**Use for:** Confirming competitors are profitable (long-running ads = converts), finding messaging gaps.

```
POST https://api.dataforseo.com/v3/serp/google/ads_advertisers/task_post
```

---

### Cost Reference
- `top_searches`: $0.015 per 50 results
- `categories_for_domain`: $0.02
- `keywords_for_categories`: $0.015 per 50 results
- `keyword_ideas`: $0.01 + $0.0001 per keyword returned
- `categories` list: Free
- SERP calls: ~$0.002 each
- Ads Advertisers: ~$0.002 each
- Full pipeline run estimated cost: under $0.50

---

### What's Next in the Pipeline
1. Pull ALL root category IDs from DataForSEO categories list
2. Run `keywords_for_categories` across all root categories
3. Filter locally by CPC>$2, competition>0.1, transactional/commercial intent
4. Surface top keywords by CPC × volume score
5. Run `ads_advertisers` on winning keywords to check competitor ad history
6. Run SERP call to pull actual ad copy and find messaging gap
7. Define offer, build landing page, write ads, launch
