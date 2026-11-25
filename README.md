# Walmart Scraper API: Which Endpoint Returns What, How Credits Turn Into Requests, and Four Checks Before You Buy (ScraperAPI Plans Compared)

A single GET request to `https://api.scraperapi.com/structured/walmart/search` with three parameters (`api_key`, `query`, `country`) returns Walmart search results as JSON: product name, price, product ID, availability, rating, seller, and a `sponsored` boolean. That covers the job behind most searches for this term, which is seeing which products occupy the top of a keyword's results and what they cost right now.

Four things decide whether an API beats maintaining your own scraper: which Walmart page types it covers, whether the structured fields come back complete on the SKUs you actually track, whether localized requests cost extra, and how you submit a few thousand queries without building a queue yourself. Here's how ScraperAPI handles each, and where Oxylabs and ScrapingBee differ on the details that change the decision.

> **Quick pick:** for keyword rank and price snapshots, the Walmart Search endpoint is the whole job, and it returns JSON without a parsing step. A free ScraperAPI account comes with 5,000 API credits for testing; the free tier itself runs 1,000 credits a month at 5 concurrent threads. Walmart requests bill at 5 credits each, so 1,000 credits buys roughly 200 Walmart searches a month.
> 👉 [See the Walmart Search endpoint and its JSON output](https://www.scraperapi.com/solutions/structured-data/walmart-search-scraper?fp_ref=coupons)

## ScraperAPI plans: what each tier actually changes

A credit is the cost of one standard page request. Walmart sits in the e-commerce category alongside Amazon and eBay at 5 credits per request, five times a plain page, so divide any allowance by five to compare it against a Walmart workload. Add-on parameters stack on top: `render=true` adds 10 credits, `premium=true` adds 10, `ultra_premium=true` adds 30, and `render` plus `premium` together cost 25.

Every plan carries the same core stack (JS rendering, rotating proxy pools, CAPTCHA and anti-bot handling, automatic retries, JSON auto parsing, unlimited bandwidth, country-level geotargeting). What separates the tiers is allowance, concurrency, and how the account behaves when credits run out.

| Plan | What's verified about it | Billing | Get started |
| --- | --- | --- | --- |
| Free | 1,000 credits/month, 5 concurrent threads | No card | [Open a free account](https://www.scraperapi.com/signup?fp_ref=coupons) |
| Hobby | Entry paid tier; opt-in auto-upgrade to the next tier when credits run out | Monthly or annual | [Check Hobby pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | Same mechanics, larger allowance | Monthly or annual | [Check Startup pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | Same mechanics, larger allowance again | Monthly or annual | [Check Business pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | Auto-upgrade plus pay-as-you-go at a fixed per-credit rate, capped at a monthly limit you set | Monthly or annual | [Check Scaling pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | Auto-upgrade plus pay-as-you-go | Monthly or annual | [Check Professional pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | Pay-as-you-go, no auto-upgrade listed | Monthly or annual | [Check Advanced pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | Custom credits, custom concurrency, dedicated support, custom pricing | Negotiated | [See enterprise options](https://www.scraperapi.com/pricing/?fp_ref=coupons) |

The published upper end of the grid runs 5,000,000 credits at 200 threads, 10,500,000 at 300 threads, 21,500,000 at 500 threads, and custom deals from 22,000,000 credits with 500+ threads. Those numbers tell you the ceiling; they say nothing useful about which entry tier you need.

Two mechanics matter more than the tier names. First, running out of credits pauses scraping until the monthly renewal unless you've turned on auto-upgrade (Hobby through Professional) or pay-as-you-go (Scaling and above). Second, hitting your concurrency limit returns a 429 rather than an extra charge; the request is turned away, not queued. Annual billing is cheaper per credit than monthly, and there's a 7-day no-questions-asked refund if the success rate on your keyword list doesn't hold up.

Dollar figures per tier live on the plan cards themselves and move with billing cycle, so check them there. 👉 [Current ScraperAPI plan prices](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## What a Walmart search response looks like

The endpoint returns this shape for each listing in the `items` array:

json
{
  "items": [
    {
      "availability": "In stock",
      "id": "38A9AKGKP84X",
      "sponsored": true,
      "name": "Impressa Products Squishy Stress Relief Balls (3-pack) - Tear - Resistant Stress Relief Ball",
      "price": 13.99,
      "price_currency": "$",
      "rating": { "average_rating": 4.7, "number_of_reviews": 31 },
      "seller": "Impresa Products, LLC",
      "url": "https://www.walmart.com/ip/..."
    }
  ]
}


`price` arrives as a number rather than a string with a dollar sign, which saves you a normalization step before it hits a database. `rating` is nested, so `rating.number_of_reviews` is the field to sort on when you're ranking by social proof.

The `sponsored` flag is the field most tutorials skip. In the example above it's `true` on the first item, which means Walmart's paid placements are mixed straight into the results array. If you build a "top 10 for this keyword" report from `items[0:10]` without filtering on `sponsored`, you're reporting ad inventory as organic rank. Filter it out and you get the actual organic position, or keep it and you get an ad-share metric for the keyword, which is useful for brands checking whether competitors are buying their way into their own category terms.

Output comes as JSON or CSV, and the Python call is three parameters plus your key:

python
payload = {'api_key': 'YOUR_API_KEY', 'query': 'stress ball', 'country': 'us'}
requests.get('https://api.scraperapi.com/structured/walmart/search', params=payload).json()


## Search, product, category: which Walmart pages are covered

ScraperAPI splits Walmart into three structured data endpoints: search, product, and category. Search gives you ranked listings for a keyword; product gives you the deeper record for a single item (specifications, variants, seller details); category lets you walk a browse node rather than a search term.

That third one is the differentiator. Oxylabs exposes three Walmart source values (`walmart_search`, `walmart_product`, and `walmart` for arbitrary URLs), but its documentation limits the arbitrary-URL source to search and product pages only. If your plan is to sweep entire categories for assortment or price-band analysis, ScraperAPI's category endpoint is the shorter path, and DataPipeline's Walmart Search template is the no-code route for keyword lists.

The calling conventions also differ enough to affect migration effort. Oxylabs requires a POST with `source`, `url`, `geo_location`, and `parse: True` to get structured output. ScrapingBee activates its Walmart mode with `search=walmart` plus `q` or `url`, and returns an `organic_results` array carrying `position`, `title`, `price`, `rating`, `reviews_count`, and `product_url`. ScraperAPI returns parsed JSON from a GET with no parse flag to remember.

## Why a hand-written Walmart scraper stalls

Walmart's frontend leans on JavaScript and internal API calls, so product pages are assembled at request time rather than served as static documents. Search pages mix organic and sponsored listings and load more products as the page scrolls. A scraper that works on a static HTML site fails here in five recognizable ways: incomplete HTML responses, blocked IPs, CAPTCHA challenges, missing price or availability data, and selectors that break after a layout change.

The error codes tell you which failure you're looking at. 401 means your credentials are wrong, 403 means access is restricted, 429 means you're over a rate limit, 500 means the target server had a problem. Production scrapers pair that with exponential backoff and request logging, because retrying a 403 on the same IP just burns time.

What you're buying from any of these providers is rendering, proxy rotation, retry logic, and a parser that gets updated when Walmart's markup shifts. ScraperAPI's own marketing claims 40M IPs across 50+ geolocations and a 99.9% uptime guarantee; its custom-trial page goes further with "99.99% success rate on Walmart at scale." Those are vendor statements, not independent benchmarks, and success rate and uptime are different measurements. Treat them as claims to test against your own keyword list during the free trial, and let the refund policy be your hedge.

## Running volume: DataPipeline, async, and where the ceiling sits

For batch work, DataPipeline takes a Walmart Search template, a list of queries, and a delivery method: results go to a webhook or you download them from the project dashboard. Each project monitors up to 10,000 queries, so a 40,000-keyword library becomes four projects rather than one oversized job.

For request-heavy workloads, the Async Scraper accepts large volumes of requests and pushes results back to your application via webhook. That removes the polling loop you'd otherwise write, and it's the right shape when you're refreshing thousands of product pages nightly.

Two habits keep the credit bill predictable. Deduplicate on `product_id` before you fetch, since the same item shows up across many keyword result pages. Cache product pages that haven't changed rather than re-pulling them daily, and normalize price formats at ingestion so a later schema change doesn't break your dashboards.

## Four checks before you commit a single credit

**1. Run one product with variants first.** Structured output is not automatically complete output. Oxylabs publishes a Walmart example (Adidas Moves Body Spray, SKU 710726462) where the record still carried warnings that shipping cost and seller info couldn't be parsed because no product variant was selected, and where `rating` came back as 0/0. Field completeness is a per-SKU question, and a variant-heavy category is where you'll find the gaps.

**2. Sanity-check review counts on a SKU you know.** If a product with thousands of reviews returns `number_of_reviews: 0`, the parser is reading the wrong node on that page layout.

**3. Confirm geotargeting isn't a paid add-on.** Walmart personalizes results by visitor location, so country targeting changes what you see. ScraperAPI includes geotargeting on all plans, and `country_code` carries no extra credit cost per its cost documentation. On other providers this is sometimes a premium-tier feature, which quietly changes the real monthly cost.

**4. Verify the actual per-request cost before a big run.** Every ScraperAPI response includes an `sa-credit-cost` header showing what that request consumed, and the `/account/urlcost` endpoint returns the cost of a URL with your parameters applied. The `max_cost` setting caps any single request so a misconfigured premium flag can't drain a month of credits in an hour.

## Which setup fits your job

Only need to know who ranks for a set of keywords and at what price? The search endpoint with the `sponsored` field filtered out is enough, and the free tier plus a small paid plan covers a few hundred keywords.

Need variants, specifications, and seller data for a catalog? Use the product endpoint, and run the variant check above before you scale.

Sweeping whole categories or tens of thousands of keywords? Category endpoint plus DataPipeline, split into projects of 10,000 queries or fewer.

Two things to settle before you pay: whether your volume is steady (monthly plans) or spiky (pay-as-you-go, available from Scaling upward, or opt-in auto-upgrade on the lower tiers), and whether annual billing plus the 7-day refund gives you enough room to validate success rate on your own keyword list.

👉 [Start with the 5,000 free credits and test the Walmart Search endpoint](https://www.scraperapi.com/signup?fp_ref=coupons)
