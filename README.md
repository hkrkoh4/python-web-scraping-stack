# Web scraping Python: from your first `requests` call to JavaScript pages and 403 blocks (code + tool comparison)

A working Python scraper is three steps: send an HTTP request, parse the HTML that comes back, pull out the fields you want. With `requests` and BeautifulSoup that's about fifteen lines of code, and it handles most article pages, product listings and public tables you'll run into.

Two things break the loop. Pages whose content is injected by JavaScript after load, and sites that answer automated requests with a 403 or a rate-limit page. This guide covers the base loop with two runnable examples, then what to change when either of those happens, including when a hosted scraping API is cheaper than the hours you'd spend building your own browser and proxy rotation.

**Short version**

- Static HTML, a few hundred pages: `requests` + BeautifulSoup. Free, no account, code below.
- Rerun it on a timer: add the `schedule` library, still free.
- Content appears only after JavaScript runs: Selenium or Playwright on your own machine; ScraperAPI's `render=true` if you'd rather not maintain a headless browser.
- 403s, IP bans, or you need real concurrency: hosted API. ScraperAPI's free plan includes 1,000 API credits and a maximum of 5 concurrent connections, which is enough to test on your actual target before paying anything.

👉 [See the current ScraperAPI plans and what each credit allowance covers](https://www.scraperapi.com/pricing?fp_ref=coupons)

## The Python web scraping stack, layer by layer

| Layer | Options | Pick it when |
| --- | --- | --- |
| HTTP client | `requests`, `urllib`, `httpx` | `requests` for readable code; `urllib` when you can't install packages; `httpx` if you want async later |
| Parser | BeautifulSoup, `lxml`, `re` | BeautifulSoup for HTML you can inspect in devtools; `lxml` + XPath for precise paths; regex for cleaning text, not for parsing markup |
| Browser automation | Selenium, Playwright | The data isn't in the initial HTML and you've confirmed it loads via JS |
| Framework | Scrapy | Hundreds or thousands of pages, with retries and throttling built in |
| Scheduling | `schedule` | The script should run hourly, daily or weekly without you |
| Storage | `csv`, pandas | `csv` for raw output; pandas when you want to clean and analyze |

Three HTTP clients do the same job. Pick one and stop reading comparison threads. The tutorial below uses `requests` because it's the shortest path from a URL to a string; an `httpx` example appears later for the same reason it exists as a library.

## Step 1: fetch the page and check what came back

python
import requests

url = "https://quotes.toscrape.com/"
res = requests.get(url, timeout=10)
res.raise_for_status()
print(res.status_code)
print(res.text[:500])


`timeout=10` stops the request from hanging indefinitely when a server accepts the connection and then never answers. Without it, one dead host can stall a loop for minutes per page.

`raise_for_status()` throws an exception on any 4xx or 5xx response. This matters more than it looks: a 404 page is still HTML, BeautifulSoup will happily parse it, and you'll end up with a CSV full of empty rows and no idea why. Raise on the status code first, then parse.

`res.text` gives you the decoded body. If you use `urllib.request.urlopen()` instead, `.read()` returns bytes, so you call `.decode("utf-8")` before parsing.

## Step 2: turn HTML into something you can query

BeautifulSoup converts a string of markup into a searchable tree of elements. Install with `pip install beautifulsoup4`, then:

python
from bs4 import BeautifulSoup

soup = BeautifulSoup(res.text, "html.parser")
print(soup.prettify()[:2000])


`prettify()` indents the tree so you can read the structure. Printing the first 2,000 characters instead of the whole document saves a lot of scrolling.

Four methods cover nearly everything:

- `find("div", class_="product")` returns the first match, or `None`.
- `find_all("p")` returns a list of all matches.
- `select("div.quote")` takes a CSS selector and returns a list.
- `select_one("span.text")` takes a CSS selector and returns the first match.

If you already know CSS, `select` and `select_one` are faster to write because you can copy the selector straight from browser devtools. `lxml` with XPath is the better choice when the element you want is easiest to describe by position, like "the third cell of every table row."

## Step 3: a full script against a site built for practice

Quotes to Scrape exists specifically for practicing web scraping, so nothing breaks when you hammer it. The page shows quote cards, each with text, an author and tags.

python
import requests
from bs4 import BeautifulSoup

url = "https://quotes.toscrape.com/"
res = requests.get(url, timeout=10)
res.raise_for_status()
soup = BeautifulSoup(res.text, "html.parser")

records = []

for card in soup.select("div.quote"):
    text_el = card.select_one("span.text")
    author_el = card.select_one("small.author")

    if not text_el or not author_el:
        continue

    tags = [t.get_text(strip=True) for t in card.select("a.tag")]
    records.append({
        "quote": text_el.get_text(strip=True),
        "author": author_el.get_text(strip=True),
        "tags": ", ".join(tags),
    })

for row in records[:3]:
    print(row)


The output looks like this:

python
{'quote': '“The world as we have created it is a process of our thinking. It cannot be changed without changing our thinking.”',
 'author': 'Albert Einstein',
 'tags': 'change, deep-thoughts, thinking, world'}

{'quote': '“It is our choices, Harry, that show what we truly are, far more than our abilities.”',
 'author': 'J.K. Rowling',
 'tags': 'abilities, choices'}


The `continue` on a missing element is the line people skip and then regret. Real pages have cards with no author, no price, or no rating. Crashing the whole run because entry 340 of 900 is malformed means starting over; skipping it means one gap in the output you can inspect later. Decide per field whether a gap is acceptable or an error worth logging.

`get_text(strip=True)` strips the surrounding whitespace that comes with the HTML indentation. Without it your CSV will be full of leading spaces.

## Step 4: write it to CSV

Python's built-in `csv` module handles this without pandas:

python
import csv

with open("quotes.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["quote", "author", "tags"])
    writer.writeheader()
    writer.writerows(records)

print(f"Saved {len(records)} records to quotes.csv")


`newline=""` prevents blank lines between rows on Windows. `encoding="utf-8"` keeps curly quotes and non-Latin characters intact; skip it and you'll get a `UnicodeEncodeError` on the first Chinese author name.

`DictWriter` maps your dictionary keys to columns in the order you list them in `fieldnames`. If a dictionary is missing a key, `writerows` raises by default, so keep the record shape consistent in the loop above.

## Step 5: scraping more than one page

Hacker News is the standard second exercise: 30 stories per page, each with a title, rank and URL.

python
import httpx
from bs4 import BeautifulSoup

html = httpx.get("https://news.ycombinator.com/", timeout=10).text
soup = BeautifulSoup(html, "html.parser")


Parse the story rows into a list of `{"title", "rank", "url"}` dictionaries and you get 30 records starting with "Bypassing airport security via SQL injection" at rank 1.

Page two is where the real work starts. Pagination means reading the page's own "More" link, following it, and stopping when it disappears. The loop itself is easy. The problems that show up at scale are not: you need throttling so you don't hammer the server, retry logic for the page that inevitably times out at item 800, and a way to resume without redoing 800 pages.

For a crawl that size, Scrapy is worth the learning curve. It gives you concurrency limits, automatic retries and a scheduler out of the box, and it scales better than a hand-rolled pagination loop with `time.sleep()` in it.

## What to do when BeautifulSoup returns an empty list

If your selectors look right and `soup.select("div.product")` returns `[]`, the data probably isn't in the HTML you downloaded. BeautifulSoup only reads static HTML; anything rendered by JavaScript after page load is invisible to it.

Confirm this before rewriting code: open the page in a browser, right-click the element, choose View Page Source (not Inspect), and search for a value you can see on screen. If it's not in the source, it's rendered client-side.

The local fix is browser automation. Selenium with headless Chrome:

python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.service import Service
from webdriver_manager.chrome import ChromeDriverManager
import time

options = webdriver.ChromeOptions()
options.add_argument("--headless")
options.add_argument("--no-sandbox")
options.add_argument("--disable-dev-shm-usage")

service = Service(ChromeDriverManager().install())
driver = webdriver.Chrome(service=service, options=options)

for page in range(1, 3):
    driver.get(f"https://webscraper.io/test-sites/e-commerce/static/computers/laptops?page={page}")
    time.sleep(2)
    titles = driver.find_elements(By.CLASS_NAME, "title")
    prices = driver.find_elements(By.CLASS_NAME, "price")
    for t, p in zip(titles, prices):
        print(t.text, p.text)

driver.quit()


`--headless` runs the browser with no visible window. `ChromeDriverManager().install()` downloads the driver version that matches your installed Chrome, which saves you from the version-mismatch error that eats an afternoon every few months. `driver.quit()` releases the process; forget it inside a loop and you'll have a dozen zombie browsers holding memory.

Playwright does the same job and waits for elements more gracefully, but it controls a real browser, so it's slower and more resource-hungry than BeautifulSoup. Use it when you need it, not as a default.

Running Chrome on a server for hours is its own project: memory leaks, driver updates, container images. ScraperAPI covers that case by rendering the page for you. Add `render=true` and the request goes through a headless Chrome instance on their side:

python
import requests

payload = {
    "api_key": "YOUR_API_KEY",
    "url": "https://example.com/product/123",
    "render": "true",
}
r = requests.get("https://api.scraperapi.com", params=payload, timeout=60)
r.raise_for_status()
html = r.text


You parse `html` with BeautifulSoup exactly as before. Rendering is available on all plans, including the free one, so you can check whether it fixes your target before committing to a paid tier.

👉 [Test `render=true` against your target with the free credits](https://www.scraperapi.com/pricing?fp_ref=coupons)

## When the site starts blocking you

403s, CAPTCHAs and sudden timeouts after a few hundred requests mean the site has decided your traffic looks automated. The fixes escalate in cost: slower request rates, then rotating user agents, then a pool of proxies, then residential proxies, then a service that maintains all of it for you.

That last step is what ScraperAPI sells, and the billing model is worth understanding before you pick a plan, because a "1,000 credit" allowance doesn't mean 1,000 pages.

One credit is a standard page request. Harder targets cost more: Amazon is 5 credits, Google and Bing including their subdomains are 25, LinkedIn is 30, and sites behind bot protection like Cloudflare, Datadome or PerimeterX add 10 credits per request when the service bypasses them. So the free 1,000 credits is roughly 1,000 ordinary pages, or **40 Google searches**, or about 200 Amazon product pages. If your project is mostly one hard target, size your plan off that number rather than the headline credit count. The dashboard has a Domain Cost Estimator that gives the exact cost for any URL, and you can set `max_cost` on a request so a single expensive URL can't quietly drain the balance.

Exceeding your limits returns a 429 asking you to bring concurrency back within your plan's limit. There's no surcharge for going over; the request is turned away rather than queued, which means your code needs to catch 429 and back off, not treat it as a fatal error.

When credits run out, scraping pauses until they renew by default. Two mechanisms change that, and which ones you get depends on the tier:

| Plan | What happens when credits run out | Credits and price | Start |
| --- | --- | --- | --- |
| Free | Pauses until the cycle renews | 1,000 credits, max 5 concurrent connections, $0 | [Start on the free plan](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | Auto-upgrade to the next plan | On the live pricing page | [Check the Hobby tier](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | Auto-upgrade to the next plan | On the live pricing page | [Check the Startup tier](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | Auto-upgrade to the next plan | On the live pricing page | [Check the Business tier](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | Auto-upgrade or pay-as-you-go up to a monthly cap | On the live pricing page | [Check the Scaling tier](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | Auto-upgrade or pay-as-you-go up to a monthly cap | On the live pricing page | [Check the Professional tier](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | Pay-as-you-go up to a monthly cap | On the live pricing page | [Check the Advanced tier](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | Pay-as-you-go up to a monthly cap | Custom | [See the Enterprise options](https://www.scraperapi.com/pricing?fp_ref=coupons) |

ScraperAPI publishes the credit allowance and monthly price for each tier on its pricing page rather than in the FAQ, and those numbers change, so confirm them at checkout rather than trusting a figure quoted in a blog post. Annual plans can also renew early for a fresh batch of credits, which matters if your workload is seasonal.

Two practical notes if you're wiring this into Scrapy. Skip `DOWNLOAD_DELAY` and `RANDOMIZE_DOWNLOAD_DELAY` when routing through the API; they throttle your own crawl without adding any benefit, since concurrency is enforced on the service side.

👉 [Compare which tiers include pay-as-you-go before you pick a credit size](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Check the site's rules before you build

Scraping can violate a site's terms of use, so read the acceptable use policy of your target before you write the loop. Sites restrict scraping for two reasons that affect you directly: protecting data they consider proprietary, and preventing repeated requests from consuming bandwidth and slowing the site for everyone else. The legal status of scraping against a site's wishes is genuinely unclear in most jurisdictions, which is another reason not to build a business on one aggressive crawler. Rate-limit yourself, identify your bot where the site supports it, and stop when you get a clear "no."

## FAQ

**Why does my scraper work in the browser but return nothing in Python?** Almost always JavaScript rendering. View the page source rather than the DOM; if the data isn't there, BeautifulSoup can't see it.

**`requests`, `urllib` or `httpx`?** `requests` for anything you'll read again. `urllib` when you can't install third-party packages at all. `httpx` if you plan to move to async, since it shares an API surface with `requests`.

**Should I parse HTML with regular expressions?** Use them to clean text, not to extract structure. A pattern like `<title.*?>` works because `.*?` is non-greedy and stops at the first `>`, but drop the `?` and it swallows everything between the first opening bracket and the last closing one. Real pages nest tags in ways that make this fragile, and one markup change breaks the whole pattern. Reach for BeautifulSoup or `lxml` first.

**When is Scrapy worth learning?** When you're crawling thousands of pages, need resumable runs, or you're tired of writing your own retry and throttle logic. For 50 pages, it's overhead.

**Do I need a paid plan to start?** No. The free tier gives 1,000 credits and 5 concurrent connections, enough to confirm whether the service gets past whatever is blocking you. Upgrade once you know the per-page credit cost for your specific target.

## Which setup fits your project

One-off scrape of a few hundred static pages: `requests` plus BeautifulSoup, saved to CSV with `DictWriter`. Add `schedule` and a timestamped filename if it needs to run daily.

Content that only exists after JavaScript runs: try Playwright locally first if you're on your own machine and the volume is low. Above a few thousand pages a day, the browser infrastructure becomes the project, and a rendering API is the shorter path.

Repeat 403s, or a target that needs concurrency you can't get from one IP: hosted API. Start free, measure the real credit cost per page with the Domain Cost Estimator, then buy the tier that matches your monthly page count rather than the one with the biggest number.

Before you pay, confirm two things on the live pricing page: the credit allowance for the tier you're choosing, and whether your billing period matches how often you actually run the scraper. A monthly plan you use two weeks a year is the most common way to overspend on this.

👉 [Check the current ScraperAPI plans, credit allowances and billing options](https://www.scraperapi.com/pricing?fp_ref=coupons)
