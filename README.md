# Amazon Product Data API: What a Request Costs, Which Fields Come Back, and How to Catch a Silent Null Price (ScraperAPI Plans, Credits and Per-ASIN Math)

An Amazon product data API can hand you HTTP 200, a 1.2 MB response, a cleanly parsed product title, and no price anywhere in the body. A public engineering write-up from a competing scraper vendor documents exactly that: `#productTitle` came back on 30 of 30 fetches, including every response that contained no price at all. Any validation check built on "did I get the product page?" passes every time while your price column fills with nulls.

On ScraperAPI, an Amazon request costs **5 credits** against 1 credit for a standard page. The Hobby plan carries **100,000 credits for $49/month**, which is 20,000 Amazon product pages, or roughly **$2.45 per 1,000 ASINs**. What follows is the plan ladder, where the billing stops being obvious, what the structured endpoint actually returns, and five checks that separate a working pipeline from one that looks fine and isn't.

> **Most teams should start on Hobby.** $49/month for 100,000 credits covers a price-monitoring list, a competitor set, or a listing audit at 20,000 Amazon pages a month. Annual billing takes exactly 10% off, which puts Hobby at $44.10/month effective.
> 👉 [See the Hobby plan's credit allowance and current price](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## The full plan ladder

ScraperAPI prices Amazon work per request rather than per seat, so plan size is really a question of how many ASINs you need to touch each month. Every 20,000 Amazon product pages consumes 100,000 credits.

| Plan | Credits per month | Price | What it realistically covers | Start |
| --- | --- | --- | --- | --- |
| Free | 1,000 trial credits, 5 concurrent connections | $0 | Testing the endpoint against your own ASINs before paying | [Create a free account](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Hobby | 100,000 | $49/mo, or $44.10/mo billed annually | ~20,000 Amazon pages/month | [Choose Hobby](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | Steps up from Hobby | Listed on the pricing page | Larger monitoring lists | [Compare Startup](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | Steps up from Startup | Listed on the pricing page | Multi-team catalogue work | [Compare Business](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | 5,000,000 | Listed on the pricing page | ~1,000,000 Amazon pages/month, pay-as-you-go available | [Compare Scaling](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | Above Scaling | Listed on the pricing page | High-volume, pay-as-you-go | [Compare Professional](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | Above Professional | Listed on the pricing page | Sustained large-scale collection | [Compare Advanced](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | Custom volume | Custom quote | Committed-volume contracts | [Request an Enterprise quote](https://www.scraperapi.com/pricing/?fp_ref=coupons) |

The credit allowances and prices for Startup, Business, Professional and Advanced are published on the pricing page rather than in the credit documentation, so the table marks them without inventing figures. Upgrading usually lowers the price per credit, which is the reason to move up a tier instead of stacking two small plans.

Two structural details matter more than the numbers. Credits reset at each monthly renewal and unused credits are lost, not carried forward. And the credit multiplier is a property of the domain, not the plan: moving from Hobby to Scaling does not change what an Amazon request costs, only how many you can afford.

## What one Amazon request actually costs

Amazon is priced at **5 credits per request** across amazon.com and every Amazon domain. For context on the multiplier system: a standard page is 1 credit, Google and Bing are 25, LinkedIn is 30.

There is a second charge that rarely makes it into budget estimates. Sites behind Cloudflare, DataDome or PerimeterX add **10 credits** on top of the base cost, applied automatically when ScraperAPI activates bypass logic rather than when you ask for it. An Amazon request that trips that path costs **15 credits, three times the headline rate**. Budget at 5 and you may run out at a third of your expected volume; the Domain Cost Estimator in the dashboard is the authoritative per-URL figure, and the `max_cost` parameter caps what any single request is allowed to spend.

| Volume | Credits used | Hobby, monthly billing | Hobby, annual billing |
| --- | --- | --- | --- |
| 1 ASIN | 5 | $0.0025 | $0.0022 |
| 1,000 ASINs | 5,000 | $2.45 | $2.21 |
| 20,000 ASINs | 100,000 | $49 (full plan) | $44.10 (full plan) |
| 100,000 ASINs | 500,000 | Needs a larger tier | Needs a larger tier |

Competitors meter in the same shape with a different unit price. Chocodata charges 5 credits per request and 15 with JavaScript rendering, so the 3× rendering penalty is an industry convention rather than a ScraperAPI quirk. Apify's Amazon crawler carries a $40 monthly rental fee on top of usage. Sorftime quotes amounts in local minor units, cents on the US site, and sells historical series rather than single snapshots. If you only need the current price, the cheap path is a low-cost pay-as-you-go plan; if you need hundreds of ASINs a day forever, the per-credit rate is the only number that matters.

👉 [Put your own ASIN count against the published credit tiers](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## Making a request: ASINs in, JSON out

The structured Amazon Product endpoint is ASIN-driven. You POST to the async structured endpoint with your `apiKey` plus either a single `asin` or an `asins` array, and you get a job back with an `id`, a `status`, and a `statusUrl` to poll. A `callback` object with a webhook URL lets you skip polling entirely, which is the sane pattern for anything above a few hundred products. The array input means one submission can carry a batch of ASINs instead of a loop of single calls.

Marketplace coverage runs to 20 Amazon TLDs: com, co.uk, ca, de, es, fr, it, jp, in, cn, sg, mx, ae, br, nl, au, tr, sa, se and pl. `country_code` handles geo targeting separately from the TLD, which is how you scrape amazon.com from a Canadian exit to see Canadian shipping information. Setting both is the documented way to get a price and delivery estimate as a shopper in a specific country sees them.

The response is structured rather than a raw page dump. A documented sample returns a `name`, `brand`, `brand_url`, a `full_description`, and a `product_information` object containing brand, model number, capacity, dimensions, weight, wattage, color, material, ASIN, `date_first_available`, and `best_sellers_rank` as readable strings such as `#14,536 in Kitchen & Dining` and `#49 in Rice Cookers`. The endpoint documentation describes price, reviews, variants and metadata by ASIN.

## The field checklist to run before you commit

Every provider's docs look complete. What differs is which fields come back populated on *your* ASINs. Use the field names below as the completeness baseline, since they are the ones publicly documented by providers with the widest field coverage.

| Field | Why it decides the purchase | Documented elsewhere as |
| --- | --- | --- |
| Current price and currency | The field the pipeline exists for | `price`, `currency` |
| Strikethrough or list price | Discount and promo detection | `listPrice`, `price_strikethrough` |
| Stock status text | "Only 10 left" predicts sellout | `inStockText`, `inStock` |
| Rating, review count, star histogram | Listing audits, review velocity | `stars`, `starsBreakdown` |
| BSR and category ladder | Selection work and rank tracking | `bestsellerRanks`, `categories` |
| Variant ASINs and variant prices | Parent/child mapping, size and color pricing | `variantAsins`, `variationASINs` |
| Buy box seller, ships-from | Offer competition | `seller`, `shipsFrom` |
| Reviews beyond the first page of the product page | Voice-of-customer research | `productPageReviews`, `aiReviewsSummary` |
| Other sellers and their offers | Reseller and pricing intelligence | `other_sellers`, `buying_offers` |

The practical test takes ten minutes and costs almost nothing: pull five ASINs from different categories, then check whether the specific fields you need are populated rather than whether the response contains the key. A null price in a well-formed JSON object is the failure you are trying to catch, and no documentation will tell you it happened.

## Five checks that catch a silent null

These come from a public scraping write-up that measured the failure mode directly. They apply to any provider, including this one, and they are the reason a "does it work" trial needs a design rather than a spot check.

**1. Validate on price, not on title.** The title selector returned on 30 of 30 fetches, including every response with no price. A health check built on the easiest field to find reports success on a broken pipeline.

**2. Vary the ASIN across categories.** Three ASINs with one identical header set produced three different outcomes in the same test. A scraper proven on one product is not proven on a catalogue.

**3. Look at byte counts.** The bare fetch of one sample ASIN returned exactly 1,224,519 bytes on 5 of 5 attempts. Identical byte counts across independent requests mean you are measuring one response repeatedly, not five results.

**4. Test where the job will actually run.** Those measurements came from a residential connection; cloud egress ranges are published and widely filtered. A result from a laptop does not carry over to CI unchanged.

**5. Respect template drift.** Amazon ships distinct templates for books, electronics and subscriptions. A selector tuned on one silently returns None on another, which is why "our parser works" ages badly without alerting that distinguishes empty from broken.

The same write-up compares a free script parsing four fields against a paid endpoint returning 39 populated fields on the same ASIN, with one Echo Dot ASIN returning no price to the free script on 5 of 5 fetches while the endpoint read 49.99. That is a vendor's own comparison of its own product, so treat it as an illustration of how wide the gap can get rather than an independent benchmark. The underlying point holds regardless: a parser that checks `response.ok` and then reads a price selector gets `None`, not an exception, and nothing in the status line or the byte count marks the response as partial.

## Where ScraperAPI stops

The overage behaviour is friendlier than most. Hobby, Startup and Business return a 429 when credits run out rather than billing you for overage, so a runaway job cannot produce a surprise invoice; it just stops. Pipelines need retry and backoff logic to handle that gracefully. Pay-as-you-go, with a monthly cap you set, starts at Scaling. Auto-upgrade to the next tier is opt-in on Hobby through Professional.

Three things the product is not built for. There is no full review history: what comes back reflects the reviews visible on the product page, and complete review corpora have sat behind an Amazon login for years. There are no historical sales or rank curves, the kind of daily series Sorftime sells reaching back several years with `-1` marking days it cannot estimate. And geolocation is country-level; providers that expose a delivery ZIP field are solving a different question than `country_code` answers.

Competitor gaps run in the other direction too, which is worth knowing before you assume a rival covers more. Apify's Amazon scraper returns only the reviews shown on the page, reflects the destination country's default address with no ZIP targeting, excludes the other-sellers panel, caps keyword search around 20 pages, and does not support brand store pages. Unwrangle documents a wider field surface including `other_sellers`, `buying_offers` and review sentiment aspects. Field coverage and request pricing get traded against each other, and the checklist above is how you decide which trade you are making.

👉 [Start on the free tier and run your five test ASINs through it](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## FAQ

**Does the endpoint take an ASIN or a URL?**
An ASIN, either a single `asin` or an `asins` array. There is no keyword parameter in the structured Amazon Product endpoint's documented inputs, so a job that starts from a search term needs keywords resolved to ASINs first.

**How many Amazon marketplaces are covered?**
Twenty TLDs, from amazon.com to amazon.com.tr and amazon.se. Geo targeting is separate: `country_code` sets the exit country, and specifying both TLD and country is how you get shipping information for a country other than the marketplace's home one.

**How many credits does one Amazon request use?**
Five. Add ten if anti-bot bypass logic activates, which makes 15 the realistic worst case. The Domain Cost Estimator in the dashboard is the authoritative figure for any specific URL.

**What happens when credits run out?**
A 429 response, and no overage charge, on Hobby, Startup and Business. Auto-upgrade moves you to the next tier if you turn it on; pay-as-you-go at a fixed per-credit rate with a spending cap is available from Scaling upward.

**Is there a refund or a trial?**
There is a free tier with 1,000 trial credits and a 7-day, no-questions-asked refund window. Annual billing is exactly 10% cheaper than monthly, and unused credits do not roll over at renewal.

## Which plan to buy

Two numbers decide it: how many ASINs you touch per month, and whether you need more than one marketplace.

- **Price monitoring or a competitor watchlist under 20,000 ASINs/month.** Hobby at $49, or $44.10/month on annual billing. Anything smaller leaves credits on the table.
- **Full-catalogue selection with variants across categories.** Size the tier by dividing your ASIN count by 20,000 to get the number of 100,000-credit blocks you need, then add headroom for the bypass surcharge.
- **Multi-marketplace collection at high concurrency.** Scaling or above, where pay-as-you-go absorbs a launch-week spike without a plan change.
- **Committed annual volume.** Enterprise is a quote, not a listed price, and it is the only tier where the per-credit rate is negotiable.

Before you pay, confirm three things on the buying page: whether you are on monthly or annual billing, since the 10% difference is applied at checkout rather than shown as a discount; that the ASIN count you sized the plan from was measured after the bypass surcharge, not before; and that your first week of testing used ASINs from at least three different categories, so the plan you buy matches a pipeline you have actually validated.

👉 [Pick the ScraperAPI plan that matches your monthly ASIN count](https://www.scraperapi.com/pricing/?fp_ref=coupons)
