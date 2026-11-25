# ScraperAPI Node.js SDK: How to Install It, Send Your First Request, and Pick the Right Plan (render, country_code, Retries, and Credit Costs)

`npm install scraperapi-sdk`, then `require('scraperapi-sdk')('APIKEY')` and call `.get(url)`. That is the entire setup. The package is a thin wrapper around ScraperAPI's HTTP endpoint, it takes your key at require time, and every documented API parameter travels in the second argument.

What follows covers the three request methods the SDK exposes, how `render` and `country_code` change what comes back, what a failed request costs you, how to read concurrency off `account()`, and how the credit system decides which plan a Node.js project actually needs.

> **Short version.** New accounts get 5,000 credits to test with. The free plan then renews at 1,000 credits a month with 5 concurrent connections, enough to finish a small job and not enough to run one every night. Hobby is listed at $49/month; Business, where country targeting outside the US and EU opens up, at $299/month. 👉 [Claim the free credits and point the SDK at your own URLs](https://www.scraperapi.com/signup?fp_ref=coupons)

## The plans side by side

ScraperAPI bills in credits, and the tier you buy sets two things: how many credits you get each cycle, and how many requests you can have in flight at once. Concurrency is the limit you hit first on a real crawl, because it caps throughput no matter how many credits are sitting in the account.

| Plan | Best for | Listed price | When credits run out | Start |
| --- | --- | --- | --- | --- |
| Free | Testing the SDK, one-off jobs | $0 | Scraping pauses until credits renew | [Check current terms](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | First paid step, single project | $49/month | Opt-in auto-upgrade to the next tier | [See Hobby pricing](https://www.scraperapi.com/?fp_ref=coupons) |
| Startup | More credits and concurrency than Hobby | Shown on the pricing page | Opt-in auto-upgrade | [See Startup pricing](https://www.scraperapi.com/?fp_ref=coupons) |
| Business | Country targeting outside the US and EU | $299/month | Opt-in auto-upgrade | [See Business pricing](https://www.scraperapi.com/?fp_ref=coupons) |
| Scaling | Volume work that varies month to month | Shown on the pricing page | Auto-upgrade or pay-as-you-go | [See Scaling pricing](https://www.scraperapi.com/?fp_ref=coupons) |
| Professional | Agencies running several client jobs | Shown on the pricing page | Auto-upgrade or pay-as-you-go | [See Professional pricing](https://www.scraperapi.com/?fp_ref=coupons) |
| Advanced | Sustained high volume | Shown on the pricing page | Pay-as-you-go only | [See Advanced pricing](https://www.scraperapi.com/?fp_ref=coupons) |
| Enterprise | Custom commitments | Custom quote | Pay-as-you-go at contract rates | [Talk to sales](https://www.scraperapi.com/?fp_ref=coupons) |

Two details in that table matter more than the numbers. Overage works differently by tier: below Scaling you can only opt into an automatic upgrade to the next plan, while pay-as-you-go at a fixed per-credit rate, capped at a monthly limit you set, starts at Scaling. And credits do not roll over, so buying a bigger plan than you need is money spent on nothing.

Annual billing is available and renewing early releases a fresh batch of credits. Prices move, and the live pricing page is the authority over anything written here, including this table.

## Install and send one request

bash
npm install scraperapi-sdk --save


js
const scraperapiClient = require('scraperapi-sdk')('APIKEY');

scraperapiClient.get('http://httpbin.org/ip')
  .then(response => {
    console.log(response)
  })
  .catch(error => {
    console.log(error)
  });


Two things to notice. The key is passed to the factory function the moment you require the package, not stored on the client object afterwards, so keep it in an environment variable rather than in source control. And `httpbin.org/ip` in the example is just a test target that echoes your IP back; swap in whatever URL you actually want to fetch.

The published examples are all CommonJS. The package sits at version 2.0.1 and has not shipped a release in a while, so if your project is ESM or TypeScript-first, check how the package resolves before you build around it rather than assuming.

## POST, PUT, and your own headers

The SDK ships shorthand methods for the other two verbs. Body and headers go in an options object:

js
const scraperapiClient = require('scraperapi-sdk')('APIKEY');

const options = {
  body: JSON.stringify({ foo: 'bar' }),
  headers: {
    'Content-Type': 'application/json',
  }
};

scraperapiClient.post('http://httpbin.org/anything', options)
  .then(response => console.log(response))
  .catch(error => console.log(error));

scraperapiClient.put('http://httpbin.org/anything', options)
  .then(response => console.log(response))
  .catch(error => console.log(error));


The `JSON.stringify` is not optional if you are posting JSON. Passing a plain object as `body` sends `[object Object]`, and ScraperAPI forwards it to the target site as-is. Custom headers work the same way on a GET, as a `headers` key in the second argument:

js
scraperapiClient.get('http://httpbin.org/anything', {
  headers: { 'X-MyHeader': '123' }
})


## render and country_code: when they are required, not optional

These are the two parameters people search for most, and both are just keys in the options object:

js
const params = {
  render: true,
  country_code: 'us'
};

scraperapiClient.get('https://www.homedepot.com/b/Appliances-Refrigerators/N-5yc1vZc3pi', params)
  .then(response => console.log(response))
  .catch(error => console.log(error));


`render: true` makes ScraperAPI execute the page's JavaScript before handing you the HTML. `country_code: 'us'` sends the request through an exit node in that country.

The Home Depot walkthrough in ScraperAPI's own docs is the clearest illustration of why both are sometimes mandatory. Home Depot runs a modern frontend framework that injects product data with JavaScript, so a raw HTML fetch returns a shell with no prices in it. The site is also only served in some countries, so a request from the wrong geography gets blocked outright. Turning both on is what made the scrape work. That tutorial calls the REST endpoint through Axios rather than the SDK; with the SDK, the same two parameters go into the options object and nothing else changes.

My rule: start with neither, inspect the response, and add `render: true` only when the fields you need are missing from the HTML. Rendering costs more credits per request than a plain page, so turning it on everywhere is an expensive habit.

## Watching usage and concurrency with account()

Most tutorials stop at fetching. The SDK also exposes `account()`, which hits the `/account` endpoint and returns JSON:

js
scraperapiClient.account()
  .then(response => console.log(response));


That response carries how many concurrent requests you are currently using and how many requests you have made. Wire it into a cron job or a health check and you get two useful signals for free: whether your worker pool is sized above your plan's concurrency ceiling (requests queue or fail, and you waste time on retries), and how fast you are burning credits relative to the billing cycle.

For a scraped pipeline that runs unattended, this is the difference between finding out you hit a limit at 3 a.m. and knowing on Tuesday that Thursday's run will run dry.

## What ScraperAPI charges you per request

A credit is not a page. A standard page costs 1 credit, Amazon costs 5, Google and Bing including their subdomains cost 25, and LinkedIn costs 30. Sites sitting behind bot protection such as Cloudflare, Datadome, or PerimeterX add 10 credits per request when ScraperAPI has to bypass them.

That multiplier is where budgets go wrong. A crawl of 50,000 Google result pages is 1.25 million credits, not 50,000, and the plan that looked generous for HTML pages is suddenly too small. The dashboard includes a Domain Cost Estimator that prices any URL before you commit, and you can pass `max_cost` on a request to cap what a single scrape is allowed to spend. Use `max_cost` on any endpoint where the domain mix is unpredictable, such as user-submitted URLs.

👉 [Compare what a plan costs against what your crawl actually burns](https://www.scraperapi.com/?fp_ref=coupons)

## Two limits that break code written by people who skipped the docs

**The 60-second internal retry window.** When you send a request, ScraperAPI picks a proxy and header configuration and tries. If the response is a ban page, a CAPTCHA, or simply too slow, it retries with a different configuration and keeps cycling for up to 60 seconds before handing you a 500. If your HTTP client is set to a 10-second timeout, you will kill requests that were about to succeed and blame the API. Set client timeouts to at least 60 seconds, or 70 if you want margin.

**The 2 MB request ceiling.** Images, PDFs, and other files are all scrapable the same way as HTML, but each request is capped at 2 MB. Anything larger needs to be split or fetched differently.

**A 500 is not billed.** Failed requests return a 500 status and are not charged. They are also not retried for you at the SDK level, so you own the retry loop. The official guidance has long been at least three retries, and a common pattern is an exponential backoff starting around 5 seconds and doubling. Treat 200, 403, and 404 as terminal so you do not burn attempts on pages that will never resolve.

👉 [See how failed requests and retries are handled on each plan](https://www.scraperapi.com/?fp_ref=coupons)

## Which plan a Node.js project actually needs

Work backwards from concurrency and credit burn, in this order:

- **Nothing running yet.** Stay on the free plan. 1,000 credits a month and 5 concurrent connections will get a scraper through development and a first small run. Use the 5,000 starting credits to price your target domains with the estimator before you pay for anything.
- **One project, a few thousand pages a month, plain HTML.** Hobby at $49/month. You are buying concurrency more than credits here, because 5 threads is the real bottleneck once the code works.
- **Targeting outside the US and EU, or a steady daily crawl.** Business at $299/month. Country-level geotargeting beyond those regions is gated at this tier, so if your data depends on it there is no cheaper route.
- **Volume that fluctuates, or several clients.** Scaling and Professional, where pay-as-you-go lets a spike pass without a permanent plan change. Set the monthly cap before you enable it.
- **Predictable thousands of concurrent requests.** Enterprise, where pricing is negotiated across your usage.

The upgrade that most people get wrong is buying credits when the actual problem was threads. If your jobs are slow and the credit balance is fine, you need concurrency. If your jobs are fast and the balance drains in a week, look at your domain multipliers first, then at credits.

One case where ScraperAPI is the wrong purchase: if you do not want to own the fetching and parsing code at all, and would rather schedule pre-built scrapers on someone else's infrastructure, a platform built around ready-made actors fits that workflow better. ScraperAPI assumes you are writing the Node.js code and just want the unblocking layer handled.

## FAQ

**SDK or the REST endpoint?**
Same service, same parameters, same credits. The SDK saves you from assembling query strings; the endpoint gives you direct control over the HTTP client, which matters if you need custom agents, interceptors, or streaming. Neither is faster or cheaper.

**Do I need `render: true` for every site?**
No, and you should not use it that way. It costs more per request. Fetch plain first, and only switch it on when required fields are missing from the HTML, which is a sign the page builds its content with JavaScript.

**Are failed requests charged?**
A 500 from ScraperAPI is not billed. You still need to catch it and retry, because the SDK does not retry for you.

**How far do the free credits go?**
5,000 to start, then 1,000 a month with 5 concurrent connections. On standard pages that is a comfortable test budget. On Google, Bing, or LinkedIn, at 25 and 30 credits per request, it is a few hundred pages.

**Does `country_code` cost extra credits?**
Geotargeting is a plan feature rather than a per-request surcharge, and outside the US and EU it requires Business or above.

## Before you buy

Match the purchase to the bottleneck you actually measured. Development work and a first test run: free plan, no card needed. A working scraper that is slow because of 5 threads: Hobby. Data that depends on non-US/EU geotargeting: Business. Seasonal spikes: a tier with pay-as-you-go, with the monthly cap set first.

Two things to confirm on the pricing page before you enter card details: the current credit allowance and concurrency for the tier you picked, and whether annual billing changes the effective monthly rate enough to justify the commitment. Both change, and neither is worth guessing at.

👉 [Open a free ScraperAPI account and run your first requests](https://www.scraperapi.com/signup?fp_ref=coupons)
