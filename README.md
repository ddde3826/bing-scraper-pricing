# bing scraper python: working code, the August 2026 API shutdown, and what a paid plan actually costs per 1,000 Bing results

Searching for a bing scraper in Python used to have a boring answer: use the official Bing Search API. That answer expired on August 11, 2025, when Microsoft retired the Bing Search APIs and left every script calling `api.bing.microsoft.com` dead. Since then you have three real options: scrape the public results page yourself with `requests` + BeautifulSoup (free, works for small jobs), pay a scraping API to handle proxies and bot checks for you, or move to another search data provider entirely. This article walks through the DIY route with working code, then shows exactly what the paid route costs — including one number most guides skip: how many credits a single Bing search actually burns.

> **TL;DR:** If you scrape under ~200 Bing queries total, the free trial credits will cover it. If you need a few thousand Bing queries per month on a schedule, ScraperAPI's Hobby plan ($49/month) covers about 4,000 Bing requests — roughly $12.25 per 1,000. DIY with Python stays free but breaks at scale. 👉 [Grab ScraperAPI's 5,000 free trial credits](https://www.scraperapi.com/?fp_ref=coupons) to test your actual Bing workload before committing to anything.

## Bing scraping is still possible — the API shutdown just changed who needs it

Bing itself is still scrapable, and it's honestly the easiest major engine to scrape. Compared to Google, its anti-bot setup is lighter, and the results page often still returns clean HTML to a plain HTTP request. Each organic result sits in a `li.b_algo` block, with the title and link in `h2 a` and the snippet in `.b_caption`. That markup has survived multiple redesigns, which keeps your parser short and maintenance low.

A few details worth knowing before you write any code:

- Yahoo has served its web results from Bing's index since the alliance completed in 2010, and DuckDuckGo's traditional links largely come from there too. Parse Bing once and the concepts transfer — though Yahoo wraps links in redirects and DuckDuckGo rate-limits harder.
- Pagination is a single `first=` offset: `first=1` for the first page, `first=11` for the second, and so on. Add `cc=us` and `setlang=en` for country and language control.
- One practical caveat: one detailed 2026 write-up reports that Bing now wraps organic hrefs in `bing.com/ck/a` click-tracking redirects, where the real URL is base64-encoded inside the `u=a1...` parameter. Other recent guides describe organic links as direct destination URLs. Both can be true depending on query and region, so write your parser to check whether the href is a redirect wrapper and decode it if so.

## The free Python route: requests + BeautifulSoup, with known limits

For low-volume scraping, the standard stack works. The working pattern looks like this:

python
import requests, time, random
from bs4 import BeautifulSoup

HEADERS = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0 Safari/537.36",
    "Accept-Language": "en-US,en;q=0.9",
    "Referer": "https://www.bing.com/",
}

def scrape_bing(query, pages=1):
    results, seen = [], set()
    for page in range(pages):
        first = 1 + page * 10
        r = requests.get(
            f"https://www.bing.com/search?q={requests.utils.quote(query)}&first={first}",
            headers=HEADERS, timeout=15)
        if r.status_code != 200:
            return []
        soup = BeautifulSoup(r.text, "html.parser")
        for li in soup.select("li.b_algo"):
            link = li.select_one("h2 a")
            snippet = li.select_one(".b_caption p")
            if not link or link["href"] in seen:
                continue
            seen.add(link["href"])
            results.append({
                "position": first + len([x for x in results if x]),
                "title": link.get_text(strip=True),
                "url": link["href"],
                "snippet": snippet.get_text(strip=True) if snippet else "",
            })
        time.sleep(3 + random.uniform(0, 3))
    return results


Three habits make the difference between a script that runs for weeks and one that dies on day two. First, pace your requests — page-to-page random delays of 3–6 seconds are the commonly cited baseline, because request rhythm is what trips anti-bot systems, not parsing. Second, deduplicate by URL, since adjacent Bing pages occasionally repeat a result at the boundary. Third, treat the `li.b_algo` selectors as possibly drifting: anchor on the container element and read inward, rather than chaining deep class paths that break when one nested class changes.

Where does this approach stop working? Not at the parsing stage — at the scale stage. Production-grade Bing scraping hits three documented walls:

1. **TLS fingerprint inspection.** Bing checks the JA3/JA4 signature of your TLS handshake. Python's stdlib `ssl`, `requests`, and `httpx` produce fingerprints no real browser emits, and the server can return `403` before it even reads your query string. Workarounds exist (libraries like curl-cffi impersonate real browser TLS at the socket level), but now you're maintaining fingerprint rotation.
2. **Redirect decoding.** The `bing.com/ck/a` wrappers need base64url decoding to get canonical URLs.
3. **Session consistency.** Rotating proxies mid-pagination shrinks the organic result blocks, adds ads, or returns empty pages. One query's pages need the same session; retries need exponential backoff.

Each is solvable. Together they turn "a 10-line tutorial" into several days of engineering plus ongoing maintenance whenever Bing adjusts something. That maintenance cost is the actual decision point — past a few hundred queries a day, most teams conclude their time costs more than a per-request API.

## ScraperAPI full pricing table (all current plans)

ScraperAPI works on a credit system: you send a request to their endpoint with your API key, they route it through rotating proxies, handle CAPTCHAs and retries, and return the HTML. Bing is an officially listed SERP domain, charged at 25 credits per request. Here's the full current plan lineup:

| Plan | Credits/month | Price/mo (annual billing) | Concurrent threads | Geo targeting | Link |
| --- | --- | --- | --- | --- | --- |
| Free | 1,000 | $0 | 5 | — | [Try the free plan](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | 100,000 | $49 ($44.10) | 20 | US & EU only | [Subscribe to Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | 1,000,000 | $149 ($134.10) | 50 | US & EU only | [Subscribe to Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | 3,000,000 | $299 ($269.10) | 100 | Country-level | [Subscribe to Business](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | 5,000,000 | $475 ($427.50) | 200 | Country-level | [Subscribe to Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | 10,500,000 | $975 ($877.50) | 300 | Country-level | [Subscribe to Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | 21,500,000 | $1,975 ($1,777.50) | 500 | Country-level | [Subscribe to Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | 22,000,000+ | Custom quote | 500+ | Custom | [Contact sales](https://www.scraperapi.com/pricing?fp_ref=coupons) |

All paid plans include JS rendering, premium proxies, JSON auto-parsing, CAPTCHA and anti-bot handling, automatic IP rotation, custom headers and sessions, automatic retries, unlimited bandwidth, and a 99.9% uptime guarantee. Annual billing gives 10% off, shown in parentheses.

## What Bing actually costs on ScraperAPI: the 25-credit math

This is the part competitor articles almost never spell out, and it changes which plan you should buy. Bing requests cost **25 credits each** — the same as Google, versus 1 credit for a standard page and 5 for Amazon. That means plan credit allowances translate to Bing queries like this:

- Free plan: 1,000 credits = **40 Bing searches**
- 5,000-credit free trial (7 days, no credit card) = **200 Bing searches**
- Hobby: 100,000 credits = **4,000 Bing searches/month**
- Startup: 1,000,000 credits = 40,000 Bing searches/month
- Business: 3,000,000 credits = 120,000 Bing searches/month

Per thousand Bing requests, that works out to roughly **$12.25 on Hobby** ($49 ÷ 4), about **$2.49 on Business**, and about **$2.30 on Advanced**. Annual billing shaves each figure by 10%. If your monthly Bing volume is a few thousand queries, Hobby is the only plan that makes sense; the moment you need tens of thousands, the per-query cost drops fourfold at Business.

A few billing rules from the official docs that matter for budgeting:

- You're only charged for successful requests — HTTP `200` and `404` — plus requests you cancel before the 70-second timeout. Blocked or failed attempts don't consume credits.
- Every response includes a `sa-credit-cost` header showing exactly what that request consumed, so there's no guessing.
- Before running a big job, use the Domain Multiplier in the dashboard or the `urlcost` API endpoint to check the exact credit cost of your target URL, and set a `max_cost` cap per request. There's also an API Playground for testing target URLs at small scale first.
- Parameters change the bill: `render=true` adds 10 credits, `premium=true` adds 10, and `country_code`, `session_number`, `device_type`, and `autoparse` are free. For plain Bing SERP HTML you typically need none of the paid add-ons, so budget 25 credits per query.
- Exceeding your concurrency limit returns a `429` and the request is refused — not queued, not billed. When credits run out, scraping pauses by default; Hobby through Professional can opt into auto-upgrade, and Scaling and above support pay-as-you-go up to a monthly cap you set.

That last point is worth pausing on. If your script queries Bing at 20 concurrent threads or fewer, Hobby's 20-thread cap fits; run 50 threads and you need Startup regardless of volume.

## How that compares to the alternatives

The retired official API used to cost around $3–7 per 1,000 queries with a stable JSON schema and built-in geo targeting — that option no longer exists for arbitrary SERP pulls. Current alternatives, by verified pricing:

| Option | Verified pricing | Trade-off |
| --- | --- | --- |
| DIY Python (requests + bs4) | $0 | Breaks at scale; TLS fingerprints, redirects, session management on you |
| SerpApi / DataForSEO | $75–225/month | Structured SERP output, higher floor price |
| Apify Bing scraper actor | ~$0.001/row (~$1.05 per 1,000 rows) | Pay per result row, not per query; platform-specific |
| ScraperAPI | From ~$12.25/1k Bing queries (Hobby) to ~$2.30/1k (Advanced) | Generic scraping API, Bing billed at 25 credits |

The honest reading: for raw price per thousand results, the Apify actor is cheaper than any general scraping API. Where ScraperAPI fits better is when Bing is one of several domains you scrape in the same pipeline — the same API key and credit pool cover your normal 1-credit pages, your Amazon product checks at 5 credits, and your Bing SERP pulls at 25, without juggling separate tools. If you only ever scrape Bing, a dedicated SERP tool deserves a look; if you scrape Bing plus everything else, one unified API is simpler to operate.

## FAQ

**Is scraping Bing legal?** Web scraping sits in a gray area — it's against most engines' terms of service, though courts have generally treated publicly available data collection as lawful in the US. Scraping a public results page is a very different posture from hitting a paid API endpoint, and you should make your own call for anything compliance-sensitive.

**How many Bing searches can I do for free?** Two separate freebies exist: a permanent free plan with 1,000 credits (40 Bing searches, max 5 concurrent connections), and a 7-day trial with 5,000 credits (200 Bing searches), no credit card required. If you need extra credits just to evaluate, support can grant more for testing.

**Does my Python script need to change much to use ScraperAPI?** The parsing code stays identical — you still get back the same HTML with `li.b_algo` blocks. What changes is the request layer: your `requests.get("https://www.bing.com/search?...")` call becomes a call to ScraperAPI's endpoint with your API key and the target URL as a parameter, and proxies, CAPTCHAs, retries, and TLS handling disappear from your codebase.

**What do people actually use Bing scrapers for?** Three documented use cases: tracking which sources ChatGPT and Microsoft Copilot cite (Bing is their retrieval layer, so Bing rank is the cleanest proxy), tracking B2B rankings where the Bing audience skews older and more enterprise than Google's, and studying the ranking delta between Bing and Google as a diagnostic signal. If you only need your own 5–10 branded keywords tracked, a rank tracker SaaS will be cheaper than any scraper — don't build infrastructure for that.

**What happens if I go over my credits mid-project?** Scraping pauses until credits renew, so your script gets throttled rather than surprise-billed. Auto-upgrade (Hobby and up) or pay-as-you-go (Scaling and up) removes the pause if you enable them.

## Final choice, by situation

- **One-off or occasional Bing queries, under ~200 total:** use the DIY code above, then burn the trial credits when you hit your first wall. 👉 [Start with ScraperAPI's 5,000 free credits](https://www.scraperapi.com/?fp_ref=coupons) and see where your own workload breaks.
- **A few hundred to a few thousand Bing queries per month:** Hobby at $49/month (or $44.10 billed annually) covers 4,000 Bing requests with 20 concurrent threads. Don't buy up a tier for features you won't touch — the geo-targeting upgrade to country-level only starts at Business.
- **Tens of thousands of monthly Bing queries, or other domains in the same pipeline:** Business at $299/month brings the effective rate to about $2.49 per 1,000 Bing requests and unlocks country-level targeting. Annual billing takes another 10% off, and year-paid plans can renew early for a fresh batch of credits.
- **Only ever scraping Bing, at high volume, per-row pricing preferred:** a dedicated SERP actor on Apify runs about $1.05 per 1,000 result rows — genuinely cheaper for that narrow case. Your call.

Before you subscribe to anything, one last check: confirm your monthly Bing query count and thread needs against the table, because the per-query price swings fivefold between Hobby and Business, and picking the right tier matters more than any coupon. 👉 [View ScraperAPI's full plan lineup and current annual pricing](https://www.scraperapi.com/pricing?fp_ref=coupons)
