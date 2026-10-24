# requests proxy python: The Setup Guide That Also Tells You When to Stop Fighting Proxies Yourself

Setting a proxy in Python `requests` takes three lines: build a dict with `http` and `https` keys, pass it to the `proxies` argument, and make sure the proxy URL includes its own scheme. That covers the basic case. The rest of this guide handles what actually breaks in practice — authentication formats, the SOCKS5 dependency trap, DNS leaks, rotation — and then the question most proxy tutorials skip entirely: at what point does rolling your own proxy logic stop making sense, and when does a managed scraping API like ScraperAPI ($49/month entry, free 7-day trial) do the job for less than your time is worth?

> **Quick take:** if you're routing a few hundred requests through one proxy, plain `requests` is all you need — the code below. If you're scraping at any real volume and tired of maintaining proxy pools at 2am, ScraperAPI's Hobby plan gives you 100,000 credits with rotation, CAPTCHA handling and JS rendering built in. There's a free 7-day trial with 5,000 credits and no credit card required: 👉 [Start the free 7-day trial and test it on your own targets](https://www.scraperapi.com/?fp_ref=coupons)

## The core setup: the proxies dict

The `proxies` argument takes a dictionary. Keys are the scheme of the URL you're requesting, values are the proxy URLs:

python
import requests

proxies = {
    "http": "http://123.45.67.89:8080",
    "https": "http://123.45.67.89:8080",
}

r = requests.get("https://httpbin.org/ip", proxies=proxies, timeout=(5, 30))
print(r.json())  # {'origin': '<proxy exit IP>'}


One thing trips up almost everyone: the key describes the **target** URL's scheme, not the proxy's protocol. `httpbin.org` is HTTPS, so the `https` entry gets used even though the proxy itself is an HTTP proxy. And when you tunnel HTTPS through an HTTP proxy, the proxy value still starts with `http://` — writing `https://` there is a common cause of connection errors that look like proxy failures but aren't.

Always set a timeout. The tuple `(5, 30)` means 5 seconds to connect and 30 to read. Without it, a hanging proxy can freeze your script indefinitely, and you'll only notice when your cron job has been stuck for six hours.

## Authentication: where credentials go

Two ways to authenticate against a proxy that requires a username and password.

The portable way — embed credentials in the proxy URL:

python
proxies = {
    "http": "http://USER:PASS@gate.proxyprovider.com:7000",
    "https": "http://USER:PASS@gate.proxyprovider.com:7000",
}


If your password contains `@`, `:` or `/`, URL-encode those characters first (`urllib.parse.quote`). A raw `@` inside a password breaks URL parsing, and the resulting authentication failure looks exactly like a bad proxy.

The alternative is the `auth` tuple, which `requests` converts into proxy authentication automatically:

python
r = requests.get("https://httpbin.org/ip", proxies=proxies, auth=("myuser", "mypass"))


Both work. The URL form is easier to reuse across libraries; the tuple form avoids encoding headaches.

## SOCKS5: the dependency trap

`requests` supports SOCKS5, but not out of the box. Pass a `socks5://` URL without the extra package and you get:


InvalidSchema: Missing dependencies for SOCKS support.


Install the extra dependency and you're set:

bash
pip install requests[socks]


That pulls in PySocks. Then use a `socks5://` or `socks5h://` URL:

python
proxies = {
    "http": "socks5h://user:pass@proxy-host:1080",
    "https": "socks5h://user:pass@proxy-host:1080",
}


Prefer `socks5h://` when you can. The `h` makes the **proxy** resolve DNS instead of your local machine, which stops hostname lookups from leaking to your own resolver — a real problem if the whole point of the proxy is hiding where you're browsing.

## Rotation, retries, and where the code starts piling up

A single proxy gets you one IP. Scraping any target with rate limits needs rotation, and `requests` gives you nothing built-in for that. The typical pattern looks like this:

python
import random

PROXIES = [
    "http://user:pass@proxy1.example.com:8000",
    "http://user:pass@proxy2.example.com:8000",
    "http://user:pass@proxy3.example.com:8000",
]

def get_with_retry(url, max_retries=3):
    for attempt in range(max_retries):
        proxy = random.choice(PROXIES)
        try:
            r = requests.get(
                url,
                proxies={"http": proxy, "https": proxy},
                timeout=(5, 30),
            )
            if r.status_code == 200:
                return r
        except requests.RequestException:
            pass
    return None


This works until it doesn't. Real-world scraping adds sticky sessions (some sites need the same IP across a multi-step flow), CAPTCHA handling, retry backoff, and dead-proxy removal. Each is manageable alone; together they turn a 20-line script into a permanent side project. If you find yourself writing session-tracking code at midnight, that's usually the moment to look at a managed endpoint — more on that below, with numbers.

## ScraperAPI: what you're actually buying

ScraperAPI is a hosted scraping endpoint. Instead of maintaining proxies, you send the target URL to their API and get back HTML (or pre-parsed JSON for Amazon, Google, Walmart and similar). Proxy rotation, CAPTCHA handling, JS rendering and retries happen server-side.

The company has been around since 2018, and the founders' pitch explains the position it occupies: after building scrapers themselves, they got tired of "finding proxies, setting up headless browsers, and handling CAPTCHAs" and built a service that does all of it through one API call. Around 31 people, owned by SaaS.group — a mid-size specialist, and Capterra reviewers give it ★ 4.6 across 62 reviews.

Every plan includes the same core feature set: JS rendering, premium proxies, JSON auto-parsing, rotating proxy pools, custom headers, CAPTCHA and anti-bot detection, custom sessions, desktop and mobile user agents, automatic retries, unlimited bandwidth, and a 99.9% uptime guarantee. Plans differ in credits, concurrency, and geotargeting scope.

Here's the full current lineup from the official pricing page, with annual billing at 10% off:

| Plan | Price (monthly / billed annually) | API credits/mo | Concurrency | Geotargeting | Buy |
| --- | --- | --- | --- | --- | --- |
| Hobby | $49 / $44.10 | 100,000 | 20 | US & EU only | [Get Hobby — $49/mo](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | $149 / $134.10 | 1,000,000 | 50 | US & EU only | [Get Startup — $149/mo](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | $299 / $269.10 | 3,000,000 | 100 | Country-level | [Get Business — $299/mo](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | $475 / $427.50 | 5,000,000 | 200 | Country-level | [Get Scaling — $475/mo](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | $975 / $877.50 | 10,500,000 | 300 | Country-level | [Get Professional — $975/mo](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | $1,975 / $1,777.50 | 21,500,000 | 500 | Country-level | [Get Advanced — $1,975/mo](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | Custom | 22M+ | 500+ | Country-level | Contact sales via the [pricing page](https://www.scraperapi.com/pricing/?fp_ref=coupons) |

No credit card for the trial, and if you just want to poke around after signing up, a free account comes with 1,000 credits and 5 concurrent connections. 👉 [Claim the 1,000 free credits and 5,000-credit trial](https://www.scraperapi.com/?fp_ref=coupons)

## Credits are not requests: the math nobody puts in the headline

This is the detail most reviews gloss over, and it changes which plan you should buy.

One credit is the cost of a *standard* page. Beyond that, costs scale with the target and the features you enable:

| Request type | Credits per request |
| --- | --- |
| Standard HTML page | 1 |
| E-commerce page (Amazon) | 5 |
| Google / Bing search | 25 |
| LinkedIn | 30 |
| JS rendering (`render=true`) | 10 |
| Premium residential proxies | 10 |
| Ultra-premium proxies | 25 |
| Bypassing Cloudflare / Datadome / PerimeterX | +10 per request |

So the Hobby plan's "100,000 credits" holds only for plain static pages. Turn on JS rendering — which most modern sites require — and each request costs 10 credits, meaning you actually get about 10,000 rendered pages. Stack residential proxies on top and a single request can burn 20+ credits. The big numbers on the pricing page describe a workload you probably won't run.

Three things keep this manageable. First, the dashboard has a **Domain Cost Estimator** that tells you the exact credit cost for any URL before you commit. Second, you can set a `max_cost` per request to cap spending on expensive domains. Third, failed requests don't cost credits — you pay for successful responses only. One caveat from reviewers: some report CAPTCHA pages or error pages occasionally being billed as successes, so it's worth spot-checking responses and flagging anomalies to support (who, per reviews, do refund confirmed mis-billing).

Concurrency is enforced but forgiving: exceeding your thread limit returns a 429, the request is turned away rather than queued, and you're not charged. If you regularly blow through credits, Hobby/Startup/Business accounts can opt into auto-upgrade to the next plan, while Scaling and above support pay-as-you-go overage at a fixed per-credit rate with a monthly cap you set.

## How well does it actually work? The benchmark numbers

Marketing pages don't tell you how a scraping API performs against the sites you care about, so here are the independent numbers.

The most recent benchmark (September 2026, 11 target sites, 7 scraping APIs tested) put ScraperAPI at a **52.3% overall success rate** — 4th of 7, just below the tested industry average of 54.2% — with an average response time of 7.5 seconds (faster than the 9.3s industry average) and a cost of $4.05 per 1,000 requests. Other benchmarks from earlier in the year measured it between roughly 62% and 72%, so the exact figure varies meaningfully by methodology. What's consistent across all of them is the pattern by target difficulty:

| Target | Success rate | Speed | Cost/1k |
| --- | --- | --- | --- |
| etsy.com | 96% | 4.5s | $4.90 |
| zillow.com | 94% | 3.4s | $0.49 |
| amazon.com | 90% | 5.3s | $2.45 |
| walmart.com | 84% | 4.9s | $2.45 |
| linkedin.com | 58% | 13.3s | $14.70 |
| indeed.com | 28% | 21.9s | $9.80 |
| booking.com | 1% | 28.0s | $4.90 |
| twitter.com / instagram.com | 0% | — | — |

That table is the whole decision. On e-commerce and standard sites, ScraperAPI performs well at a reasonable price. On aggressively defended platforms, it struggles, and on a few (booking.com, X, Instagram) it effectively can't get through at all. One reviewer's framing captures it well: "ScraperAPI is a good value tool for the easy 80% of scraping, sold to people who sometimes need the hard 20%. Knowing which job you have is the whole decision."

For comparison, in the same September benchmark: Scrapfly scored 98% success at $3.14/1k (slower, 17.8s average), Firecrawl 72% at $2.7, and ScrapingBee 33% at $1.51. On the hardest Cloudflare/DataDome-class targets, a dedicated unlocker like Bright Data's Web Unlocker has been independently measured around 95–98% — at a higher price and with more setup friction. If your target list is mostly protected sites, one of those is the honest answer; if it's mostly standard pages, ScraperAPI's simple endpoint and cheaper entry tier fit better.

## Which plan actually fits your workload

Based on the verified prices, limits and the credit math above, here's the decision rule that holds up:

- **Just testing / side project:** the free trial first, then Hobby at $49/month if it works. Hobby's 20 concurrent threads and US/EU-only geotargeting cover most individual use cases.
- **Small team, moderate volume:** Startup at $149/month buys 10x the credits for 3x the price. Still US/EU geotargeting only.
- **Production workload needing non-US/EU locations:** Business at $299/month is the minimum viable tier — country-level geotargeting and unlimited analytics history start here. Hobby and Startup only keep 30 days of stats, which matters more than it sounds once you're debugging a weekly crawl.
- **Growing team with variable volume:** Scaling at $475/month. This is the plan ScraperAPI itself marks "Most popular," and the main practical reason is pay-as-you-go overage, so a spike doesn't stop your pipeline mid-month.
- **High-volume infrastructure:** Professional ($975) or Advanced ($1,975) by scale; above that, talk to sales.

A note on annual billing: the 10% discount is uniform across all plans, so it's a straightforward saving on whatever tier you've settled on — $44.10/month on Hobby instead of $49, $1,777.50/month on Advanced instead of $1,975.

## FAQ

**Does the proxies dict key refer to the proxy's protocol?**
No. It refers to the scheme of the target URL being requested. Requesting an `https://` URL uses the `"https"` entry, even when the proxy itself is an HTTP proxy.

**Why do I get `InvalidSchema` with SOCKS proxies?**
PySocks isn't installed. Run `pip install requests[socks]`, then use `socks5://` or `socks5h://` URLs. Prefer `socks5h://` so DNS resolves through the proxy.

**How many real requests does ScraperAPI's Hobby plan give me?**
Depends on the target: about 100,000 plain static pages, or roughly 10,000 pages if every request needs JS rendering (10 credits each). Check any specific URL with the Domain Cost Estimator in the dashboard.

**Is there a free option?**
Yes — a 7-day trial with 5,000 credits (no credit card), plus a permanent free account with 1,000 credits and 5 concurrent connections after signup.

**What happens when I run out of credits mid-job?**
Scraping pauses until renewal by default. You can opt into auto-upgrade (Hobby through Professional) or pay-as-you-go with a spending cap (Scaling and above).

## Picking one before you go

The honest decision rule: if you're scraping a handful of cooperative sites, the three-line `proxies` setup and a cheap proxy list will carry you a long way, and there's no reason to pay for anything more. The moment your list includes e-commerce at volume, JS-heavy pages, or targets that rotate CAPTCHAs at you, the maintenance cost of DIY rotation starts exceeding $49/month pretty quickly — and that's where the Hobby plan earns its price.

Two things to check before you commit to any plan: run your actual target URLs through the Domain Cost Estimator so you're buying credits based on real per-request costs, and spend the free trial on your hardest target, not your easiest — that's where the success rate table above will tell you whether ScraperAPI can handle your specific job. If it can, 👉 [compare the full plan lineup and lock in the annual 10% discount](https://www.scraperapi.com/pricing/?fp_ref=coupons).
