## Crawler/Scraper (WIP)

### Using Scrapy - a spike to crawl a site and return list of PDF files URLs (WIP)

- currently define a list if domains to ignore (mainly social media)
- exclude crawl path by patterns (e.g. file format .jpeg)
- keep track of list of webpages already crawled

#### Need to be improved:

- handle cases where PDF download are hidden as javascript event e.g. click event
- implement with playwright
- handling IP blocking, rate limiting, captcha, GDPR consent popup etc

```py
import re

from scrapy.linkextractors import LinkExtractor
from scrapy.spiders import CrawlSpider, Rule

from ..items import Item


class CrawlerV2(CrawlSpider):
    name = "crawler_v2"
    start_urls = ["website_url_here"]
    deny_domains = [
        "instagram",
        "facebook",
        "youtube",
        "tiktok",
        "linkedin",
        "twitter",
        "google",
        "vimeo",
        "disqus",
        "slack",
    ]
    must_include = ["website_domain"]

    allowed_mime_type = [b"application/pdf"]
    exclude_patterns = [".*\.(css|js|gif|jpg|jpeg|png)"]
    rules = [
        Rule(
            LinkExtractor(
                # deny=[r"^(tel:)", r"^(mailto:)", r"^javascript"],
                deny_domains=[
                    "instagram.com",
                    "facebook.com",
                    "youtube.com",
                    "tiktok.com",
                    "linkedin.com",
                    "twitter.com",
                    "google.com",
                    "vimeo.com",
                    "disqus.com",
                    "slack.com",
                ],
            ),
            callback="parse_item",
            follow=True,
        )
    ]

    def set_playwright_true(self, request, response):
        request.meta["playwright"] = True
        return request

    def ignore_domains(self, url: str) -> bool:
        cond = ("|").join(self.deny_domains)
        matched = re.search(r"({})(\.com)".format(cond), url, re.IGNORECASE)
        return bool(matched)

    def parse_item(self, response, data=None):
        data = data if data else {}
        if response.headers["Content-Type"] in self.allowed_mime_type:
            self.logger.info({"url": response.url, **data})
            item = Item()
            item["inner_text"] = data["inner_text"]
            item["download_attr"] = data["download_attr"]
            item["title_attr"] = data["title_attr"]
            item["alt"] = data["alt"]
            item["src_url"] = response.url
            yield item
            return

        if not any(x in response.url for x in self.must_include):
            self.logger.info("SKIP %s", response.url)
            return

        self.logger.info(response.url)
        for link_tag in response.css("a"):
            href = link_tag.css("::attr(href)").get()
            if not href:
                continue
            if self.ignore_domains(href):
                self.logger.info("ignore_domains SKIP %s", href)
                continue
            if re.search(r"^(tel:|mailto:|fax:|javascript)", href):
                self.logger.info("re.match SKIP %s", href)
                continue

            req = response.follow(
                href,
                callback=self.parse_item,
                cb_kwargs={
                    "data": {
                        "inner_text": link_tag.css("::text").get(),
                        "download_attr": link_tag.css("::attr(download)").get(),
                        "title_attr": link_tag.css("::attr(title)").get(),
                        "alt": link_tag.css("::attr(alt)").get(),
                    }
                },
            )
            yield req

    # async def errback(self, failure):
    #     page = failure.request.meta["playwright_page"]
    #     await page.close()

```
