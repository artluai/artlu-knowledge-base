# Best Practices: Marketing & Niche Research

---

## What this document is

This is a permanent rulebook for finding and validating product opportunities using paid search data. It answers the question: **"what do we know about using DataForSEO and Google Ads data to find buildable, monetizable niches?"**

It is not a session log. It is not a record of what we found this one time. Every rule here should be true six months from now for a completely different niche. If a piece of information only applies to a specific run or a specific product, it belongs in a session note — not here.

A new model in a new session should be able to read this file and immediately know:
- what principles to apply before touching any data
- which endpoints to call and why
- what the filters mean and which thresholds work
- what mistakes to avoid
- how to estimate cost before running anything

---

## 1. Principles

- **Demand first, product second.** Find what people are already paying to search for, then build the thing. Never guess a niche.
- **No discretionary decisions.** Every choice must come from data. If a human is picking the category, the seed keyword, or the niche without data — it will bias the output.
- **The 3 questions that matter before building anything:**
  1. Will they convert? (ad history + landing page intent match)
  2. Will they click my ad? (existing ad copy + messaging gap)
  3. Can I build it? (fulfillment reality check — if not, skip)
- **Speed to signal over perfection.** Launch fast, read real data, iterate. A live page with $200 in ad spend tells you more than weeks of research.
- **People click ≠ people buy.** CTR and CPC are not conversion signals. Only purchases are.
- **High volume ≠ good opportunity.** The highest-volume keywords are almost always navigational brand queries. Filter by intent before looking at volume.
- **Long-running ads = proof of profitability.** If a competitor has been running the same ad for 3+ months, it's converting. Fresh ads everywhere = unproven market.
- **The 7-day wait comes AFTER launch**, not before. Pre-launch data tells you market size. Post-launch data tells you if your specific offer converts.
- **Build for a specific buyer with a specific problem**, not a general audience. "Better AI" is not a differentiator. "Built for X type of person who needs Y specific outcome" is.
- **Use $/conv, not CPC, as the primary evaluation metric.** CPC only tells you what a click costs. $/conv (CPC ÷ CVR) tells you what an outcome costs given your industry's expected conversion rate. A $1 CPC keyword with 0.1% CVR is worse than a $5 CPC keyword with 3% CVR.
- **~70% of keywords at any CPC range are navigational.** Brand lookups, misspelled brand names, foreign-language brand queries. Always filter to transactional + commercial intent before evaluating anything else.

---

## 2. Filter Criteria

### Filters that work (server-side, applied at API level)
- `keyword_info.cpc > 1` — confirms someone is paying for this traffic
- `keyword_info.competition > 0.1` — confirms active bidding; below this is ghost territory
- `keyword_info.competition < 0.5` — avoids markets dominated by big brands with unlimited budgets
- `search_intent_info.main_intent = "transactional"` or `"commercial"` — confirms buying behavior, not browsing
- `keyword_info.search_volume > 10000` — minimum for meaningful traffic at reasonable ad spend

### CPC ranges and what they mean
- **$1-5 CPC** — learnable zone for low budgets. Enough commercial signal to test. est. $/conv at 2% CVR = $50-250.
- **$5-15 CPC** — viable if product price is $100+. est. $/conv at 2% CVR = $250-750.
- **$15+ CPC** — only worth testing with high-confidence conversion data or high-LTV products. est. $/conv can exceed $1,000 at 2% CVR.

### Competition range and what it means
- **0.1-0.5** — active market, not dominated. Advertisers are bidding but no one has locked it down.
- **0.5+** — established brands dominating. Hard to compete without strong differentiation.
- **< 0.1** — almost no competition. Either an untapped niche or there is genuinely no money here.

### What doesn't work
- Sorting by volume — returns brand queries and navigational lookups at the top
- Filtering for navigational or informational intent — useless for finding buildable products
- Using a single seed domain — biases all downstream category data toward that domain's business
- Treating high CPC as a direct signal of opportunity — high CPC means high competition AND high value; only useful if your CVR justifies it

---

## 3. Key Metric: Est. Cost Per Conversion

**Formula:** `est. $/conv = CPC ÷ CVR`

This answers: "if I ran this keyword as an ad, what would one conversion actually cost me?"

Use industry CVR benchmarks when you don't have your own data:

| Industry | CTR | CVR | Notes |
|---|---|---|---|
| General | 3% | 2% | Default fallback |
| Finance | 4% | 5% | High intent, high value |
| Health | 3% | 3% | Condition-specific |
| eCommerce | 4.5% | 3% | Product pages |
| Legal | 2.5% | 4% | High urgency |
| Education | 3% | 2.5% | Course/certification |
| Travel | 3.5% | 2% | Booking intent |
| Real Estate | 3.5% | 2% | Lead gen |

**Color thresholds for fast scanning:**
- Green < $20/conv — strong opportunity for almost any product
- Yellow $20-50/conv — viable for products priced $100+
- Orange $50-150/conv — needs high-value product or subscription
- Red $150+/conv — only for high-LTV B2B or big-ticket sales

**Conv/day formula:** `(volume/30) × CTR × CVR`
Shows the daily conversion ceiling if you captured all traffic for that keyword. Useful for sizing the opportunity before committing to a niche.

---

## 4. Handling Keyword Data

### Duplicate variants
Results almost always contain near-duplicate keyword variants with identical CPC and volume — e.g. "contract template", "template contract", "contracts template" are the same search cluster. Deduplicate by CPC + volume before counting unique opportunities. The count of raw keywords is not the count of unique opportunities.

### Navigational contamination
Even with intent filters applied, navigational queries leak through — brand names DataForSEO misclassifies, foreign-language navigational queries, partial brand name misspellings. Always scan results visually before treating volume numbers as real opportunities.

### The category approach vs seed keywords
Running keywords against category codes produces cleaner results than running against seed keywords. Categories filter out irrelevant verticals entirely before you spend anything. Build and maintain a classified list of relevant categories instead of re-discovering them each run.

---

## 5. Category System

DataForSEO has 3,182 product/service category codes. Each has an integer code, a name, and a parent category.

**Best practice for category-based runs:**
1. Fetch the full category list once (free — `GET /dataforseo_labs/categories`)
2. Classify each category as: `d` (digital), `p` (physical), `b` (both)
3. Build a focused include list of relevant categories before any keyword run
4. Store the classification persistently — do not re-classify from scratch each run

**Why this matters for cost:**
Running all 3,182 categories is expensive and noisy. A focused list of 50-200 targeted categories produces cleaner data at a fraction of the cost. Server-side filters reduce cost further by only returning keywords that match your criteria.

**Category list tiers:**
- Broad sweep (600+ categories) — good for discovery runs
- Curated list (150-200 categories) — best balance of coverage and cost
- Niche list (50-100 categories) — surgical, one specific market

---

## 6. Technical Reference

### Authentication
DataForSEO uses Basic Auth. Credentials are Base64-encoded `email:api_password`.

```
Authorization: Basic <base64(email:api_password)>
Content-Type: application/json
```

Never hardcode credentials in source files. Store in environment variables or localStorage and load at runtime.

---

### Endpoints

#### Category List (Free)
Returns all 3,182 category codes with names and parent categories.
```
GET https://api.dataforseo.com/v3/dataforseo_labs/categories
```
Cost: Free. Run once, cache the result.

---

#### Keywords for Categories
Returns keywords within specified category codes with CPC, volume, competition, and intent.
```
POST https://api.dataforseo.com/v3/dataforseo_labs/google/keywords_for_categories/live
```
**Supports server-side filters.** This is the primary cost-control lever.

```json
[{
  "category_codes": [10161, 12117, 10847],
  "location_name": "United States",
  "language_name": "English",
  "limit": 1000,
  "filters": [
    ["keyword_info.cpc", ">", 1],
    "and",
    ["keyword_info.cpc", "<", 10],
    "and",
    ["keyword_info.competition", ">", 0.1],
    "and",
    ["keyword_info.competition", "<", 0.5],
    "and",
    ["keyword_info.search_volume", ">", 10000],
    "and",
    ["search_intent_info.main_intent", "=", "transactional"]
  ]
}]
```

Filter syntax: `["field", "operator", value]` with `"and"` between each pair.

---

#### Keyword Ideas
Returns keyword ideas related to seed keywords. Use for drilling into a niche once identified.
```
POST https://api.dataforseo.com/v3/dataforseo_labs/google/keyword_ideas/live
```
Supports the same server-side filter syntax.

---

#### Categories for Domain
Returns which categories a domain ranks for, with estimated paid traffic value per category.
```
POST https://api.dataforseo.com/v3/dataforseo_labs/google/categories_for_domain/live
```
Key metric: `estimated_paid_traffic_cost` — higher = more commercially active category.
Note: Results are biased toward the seed domain's industry. Use diverse seed domains or the full category list approach to avoid bias.

---

#### Top Searches
Returns highest-volume US keywords with no seed input. Does not support filters.
Known issue: Sorted by volume = navigational brand queries dominate. Only useful for a broad unsorted dump. Not a reliable primary research method.

---

#### SERP + Ads (validation step)
Returns live Google search results including paid ads for a keyword.
```
POST https://api.dataforseo.com/v3/serp/google/organic/live/advanced
```
Use for: confirming competitors are actively advertising, reading ad copy, finding messaging gaps.

---

#### Ads Advertisers (validation step)
Returns advertiser history from Google Ads Transparency Center.
```
POST https://api.dataforseo.com/v3/serp/google/ads_advertisers/task_post
```
Use for: confirming long-running ads (= profitable market), identifying real competitors.

---

### Cost Reference

| Endpoint | Cost |
|---|---|
| Category list | Free |
| keywords_for_categories | ~$0.015 per 50 results returned |
| keyword_ideas | ~$0.01 + $0.0001 per keyword returned |
| categories_for_domain | ~$0.02 per request |
| SERP calls | ~$0.002 each |
| Ads Advertisers | ~$0.002 each |

**Estimating run cost:**
`cost ≈ categories × avg_keywords_returned_per_category × $0.0003`

With tight server-side filters, avg keywords returned per category drops to 20-100 instead of 500-1000. This is the primary cost lever — filters before running, not after.

Rule of thumb with tight filters:
- 50-100 categories → $1-3
- 150-250 categories → $5-10
- 300-400 categories → $8-15
- No server-side filters → multiply by 5-10x

---

## 7. Validation Sequence

Once a keyword cluster passes data filters, validate before building:

1. **Check ad history** — run ads_advertisers on top keywords. Long-running ads = proven market.
2. **Read competitor ad copy** — run SERP on top keywords. Identify dominant messaging.
3. **Find the gap** — what are competitors NOT saying? What pain point is unaddressed?
4. **Define the specific buyer** — not "people who want X" but "person in Y situation who needs X by Z"
5. **Build the minimum offer** — one landing page, one product. Not a suite.
6. **Launch with a test budget** — $200-500 is enough signal to know if the offer converts
7. **Read the data after 7 days** — only then decide whether to scale, pivot, or move on
