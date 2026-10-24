# booking.com review scraper: How to Pull Guest Reviews at Scale Without Getting Blocked (and What It Actually Costs)

Scraping reviews from Booking.com comes down to three routes: a free open-source scraper you run and maintain yourself, a dedicated marketplace actor that charges per review, or a general scraping API like ScraperAPI that handles proxies, CAPTCHAs, and JavaScript rendering for you. Which one fits depends on your volume and how much infrastructure you want to own.

This article walks through how Booking.com actually serves its reviews (it's not plain HTML, which trips up a lot of naive scrapers), what data you can realistically extract, what each route costs, and where the hidden limits are — including the credit math most pricing pages don't spell out.

> **Quick pick:** if you just need Booking.com reviews for a research or monitoring project and don't want to babysit proxies, start with a general scraping API. ScraperAPI's 7-day trial gives you 5,000 credits with no credit card, which is enough to see whether the data you need comes through cleanly. 👉 [start the free 7-day trial on ScraperAPI](https://www.scraperapi.com/?fp_ref=coupons)

## How Booking.com review scraping actually works

Booking.com loads reviews through a GraphQL API, not the static page HTML. When a visitor clicks "Read all reviews," the browser fires a `ReviewList` request to an endpoint at `/dml/graphql`, with pagination variables called `skip` and `limit` — the page size is 10 reviews per step. A scraper has to either replay those GraphQL requests directly (reusing captured cookies and query bodies) or render the full page in a headless browser and click through.

This is why simple `requests`-based scrapers often return empty review sections. The reviews simply aren't in the initial HTML.

A few structural details worth knowing before you write any code:

- Each review is split into a `positive` and a `negative` half, and either can be empty. Many guests only leave a score with no text at all, so blank text fields are normal, not a bug in your scraper.
- Pagination uses `limit` and `offset` against a `total_count` value, and you can filter by review language or guest type, with five sort options.
- A hotel's Booking.com URL slug (the part like `/hotel/es/sansipedralbes.html`) is enough to identify the property reliably — you don't need the internal hotel ID, though that ID comes back in the data anyway.
- Beyond the overall 0–10 score, each property carries sub-scores for cleanliness, staff, location, and similar categories.

On top of the technical layer sits Booking.com's bot protection. Pulling reviews at any real volume means rotating residential IPs and, in most cases, JavaScript rendering — which is exactly the part that general scraping APIs charge for.

## Your three options, compared

**Open-source scrapers.** Several free Python projects on GitHub handle Booking.com reviews well. The cleverest approach intercepts the GraphQL request in Playwright, captures the URL, cookies, and query body, then replays it directly with `httpx` for each page — incrementing `skip` by 10 each time. The browser is only used for the initial load, which makes these scrapers fast even on large hotels with thousands of reviews. One project also normalizes scores across platforms, scaling Booking.com's 0–10 and Airbnb's 0–5 onto the same 0–10 scale before mining review text for topics like noise or bed comfort.

The catch is ownership: you maintain the code when Booking.com changes its GraphQL schema, you supply your own proxies when your IP gets flagged, and you solve CAPTCHAs yourself. For a one-off research pull, that's fine. For anything recurring, it becomes a part-time job.

**Dedicated marketplace actors.** On Apify, a "Booking.com Reviews Scraper - Guest Reviews API" actor does exactly one thing: it pulls guest reviews for any hotel via pure HTTP, with no login required. It's billed per review scraped with no monthly rental, outputs JSON/CSV/Excel, and returns a rich field set — reviewer name and country, 0–10 score, title, positive and negative text, review language, ISO date, helpful votes, reviewer's lifetime review count, guest type (Couple, Family, Solo traveller), room type, check-in/check-out dates, nights stayed, the hotel's reply (`partnerReply`), and review photo URLs. `maxReviews` defaults to 100 per hotel; setting it to 0 grabs everything available. You can sort by most relevant, newest, oldest, highest or lowest score.

That's the fastest path if you want structured review data for a handful of hotels today. For recurring, high-volume collection across many properties, per-review pricing needs a closer look against your budget — Apify also strongly recommends its residential proxies as the default, which factors into the effective cost.

**General scraping APIs.** This is where ScraperAPI sits. You send a Booking.com URL to a single API endpoint with parameters like `render=true` (JavaScript rendering) and a premium proxy flag; the service handles proxy rotation, CAPTCHA solving, headers, retries, and sessions, and returns the raw HTML or parsed JSON. You write no scraping infrastructure at all. The trade-off is the credit-based pricing model, which deserves its own section — because it's the number one thing people get wrong about the cost.

## What you can pull from a Booking.com review

Whichever route you take, the realistic field set for each review looks like this:

- Overall score (0–10), title, and the split positive/negative text
- Reviewer display name, country, guest type (couple, family, solo, group), and how many reviews they've written in total
- Review language and post date
- Room type, check-in and check-out dates, and length of stay in nights
- Helpful vote count
- The hotel's official reply, if one exists
- URLs of photos attached to the review

For sentiment analysis or market research, the useful combination is usually score + guest type + stay dates + the two text halves, since it lets you segment complaints by traveler profile — a family's take on a hotel often says nothing about whether it suits a solo business traveler.

One caveat from real output data: `partnerReply` and `photos` are frequently null or empty. Hotels reply to a minority of reviews, and most guests don't attach photos. If your pipeline assumes those fields are always populated, it will break.

## ScraperAPI's full plan and pricing breakdown

ScraperAPI prices by API credits, bundled monthly. All paid plans include the same feature set — JavaScript rendering, premium residential and mobile IPs, CAPTCHA and anti-bot handling, JSON auto-parsing, custom headers and sessions, automatic retries, unlimited bandwidth, and a 99.9% uptime guarantee. Plans differ in credits, concurrency, geotargeting, and overage options.

| Plan | Monthly | Annual (per mo) | Credits | Threads | Get it |
| --- | --- | --- | --- | --- | --- |
| Hobby | $49 | $44.10 | 100,000 | 20 | [ Get Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | $149 | $134.10 | 1,000,000 | 50 | [ Get Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | $299 | $269.10 | 3,000,000 | 100 | [ Get Business](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | $475 | $427.50 | 5,000,000 | 200 | [ Get Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons&plan=scaling) |
| Professional | $975 | $877.50 | 10,500,000 | 300 | [ Get Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | $1,975 | $1,777.50 | 21,500,000 | 500 | [ Get Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | Custom | Custom | 22,000,000+ | 500+ | [ Talk to sales](https://www.scraperapi.com/pricing?fp_ref=coupons) |

Annual billing takes 10% off every plan. Two entry ramps sit outside the grid: a **7-day trial with 5,000 credits and no credit card**, plus a permanent free plan with 1,000 credits capped at 5 concurrent connections.

The rows that matter most but get least attention:

- **Geotargeting.** Hobby and Startup are limited to US and EU locations. Business and above unlock country-level targeting across 50+ countries. If your Booking.com research depends on how listings appear from, say, Japan or Brazil, the two cheapest plans won't get you there.
- **Overage behavior.** On Hobby, Startup, and Business, credits run out and scraping pauses until renewal (or you opt into auto-upgrade to the next tier). Scaling and above can enable pay-as-you-go, continuing at a fixed per-credit rate up to a monthly cap you set. For workloads where you can't predict volume, that difference alone can decide the plan.
- **Analytics history.** The two cheap plans only keep 30 days of analytics; Business and up keep everything.

## The credit math nobody explains properly

Here's the part most write-ups skip: a "100,000 credit" plan does not mean 100,000 pages. Credits are a difficulty meter, not a request counter.

A standard page costs 1 credit. Turn on JavaScript rendering and it's 10. Route it through premium residential proxies and that's another 10. Premium proxies plus rendering together run 25 credits per request, and the ultra-premium tier with rendering hits 75. Some domains carry fixed costs regardless — Amazon-type e-commerce pages cost 5, Google and Bing SERPs 25, LinkedIn 30 — and when ScraperAPI successfully bypasses protection like Cloudflare, Datadome, or PerimeterX, that adds 10 credits per request.

Apply that to Booking.com: a realistic reviews workflow needs rendering and residential IPs, so you're likely in the 10–25 credit range per request. A Hobby plan's 100,000 credits therefore covers roughly 4,000 to 10,000 review-page fetches, not 100,000. One independent analysis put the same point more starkly: the same 100,000-credit plan can yield anywhere from about 1,333 to 100,000 actual scrapes depending on parameters.

Three things soften the blow:

1. **Failed requests cost nothing.** You only pay for successful responses, so blocks and timeouts don't burn your budget.
2. **The Domain Cost Estimator** in the dashboard shows the exact credit cost of any URL before you commit, and you can set a `max_cost` per request so a single unlucky scrape can't blow through your allowance.
3. **The 5,000-credit trial** is genuinely enough to measure your real per-request cost on Booking.com before paying anything. 👉 [test your Booking.com credit cost with the free trial](https://www.scraperapi.com/signup?fp_ref=coupons)

Do that math first and the plan choice becomes mechanical rather than a guess.

## Which plan fits which Booking.com project

- **One-off research, a few hotels:** don't buy anything yet. The 7-day trial's 5,000 credits will cover a small review pull, and the free plan's 1,000 credits handle tiny follow-ups.
- **Ongoing monitoring of a handful of properties:** Hobby at $49/month works if 20 concurrent threads is enough throughput and you don't need non-US/EU geotargeting. If either constraint bites, jump straight to Business — $299/month is the minimum tier with global country-level targeting.
- **Competitive intelligence across markets:** Business. The threefold credit increase over Startup plus global targeting plus unlimited analytics history makes it the practical floor for production work with an international scope.
- **Variable, hard-to-predict volume:** Scaling at $475/month. Its pay-as-you-go overage means a spike month doesn't stall your pipeline mid-run.
- **High-volume infrastructure:** Professional ($975) or Advanced ($1,975), which add priority support and priority routing respectively.

👉 [compare all seven plans on the pricing page](https://www.scraperapi.com/pricing?fp_ref=coupons)

A note on discounts: third-party deal trackers have reported a `START10` promo code giving 10% off your first month on any paid plan, with the code reported still active as of recently. Since that comes from third-party sources rather than ScraperAPI's own site, verify it applies at checkout before counting on it. Annual billing's 10% off is confirmed directly on the pricing page and stacks as the more reliable saving anyway.

## Frequently asked questions

**Is scraping Booking.com reviews legal?** Reviews are publicly viewable data, and scraping publicly available pages is a normal practice for market research — but the responsibility for how you use the data sits with you. That means respecting Booking.com's terms of service, GDPR constraints on personal data in reviews, and copyright where applicable. Dedicated marketplace actors in this space state this explicitly: they collect only public review data without logging in, and usage compliance is the user's job. Apply the same standard to any tool you pick.

**Do I need a Booking.com account or API key?** No. The review data these methods collect is public and requires no login, cookies from a registered account, or official API credentials.

**Why are some review text fields empty?** Because many guests submit a score without writing anything. The positive and negative halves are separate fields, and either can be blank.

**What happens when I hit my credit or thread limit?** Threads return a 429 error telling you to retry within your plan's concurrency — there's no extra charge for going over, the request is simply rejected. Credits pause scraping by default until renewal, unless you've enabled auto-upgrade (lower plans) or pay-as-you-go (Scaling and above).

**Can I get more trial credits?** Yes — ScraperAPI says to contact support if you need additional credits for testing.

## Bottom line before you commit

If you scrape Booking.com reviews a few times a year, an open-source GraphQL-replay scraper or a per-review marketplace actor will cost you less than any subscription — accept that you're the one fixing it when the schema changes. If review collection is a recurring part of your workflow, ScraperAPI's trial-to-Hobby-to-Business ladder lets you scale spend with actual usage, and the free trial means you can measure your true credit cost per Booking.com request before spending a cent.

Two things to check before checkout: confirm whether Booking.com requests need `render=true` and premium proxies for your target hotels (that determines whether your credits buy 4,000 pages or 10,000), and decide now whether you need geotargeting beyond the US and EU, since that rules out the two cheapest plans. 👉 [open ScraperAPI's pricing and start with the free trial](https://www.scraperapi.com/?fp_ref=coupons)
