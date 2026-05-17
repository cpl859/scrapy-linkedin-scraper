# Scrapy LinkedIn 爬虫实战指南：用 ScraperAPI 突破反爬封锁，稳定抓取职位与人才数据

上个月一个做猎头的朋友找我，说他们团队自己写了个 LinkedIn 爬虫，跑了三天就全被封了，IP 换了一批又一批，最后什么数据都没拿到。我问他用的什么方案，他说"就是普通 requests 加代理池"。

我当时就知道问题在哪了。

LinkedIn 的反爬不是普通代理能对付的。它有行为指纹检测、TLS 指纹识别、JavaScript 渲染验证，还有一套专门针对数据中心 IP 的黑名单。你换 IP 没用，因为它认的不只是 IP。

这篇文章我把自己踩过的坑都写出来，包括为什么我最后选了 ScraperAPI 来处理这层麻烦，以及怎么把它接进 Scrapy 的爬虫里。

---

## LinkedIn 爬虫为什么这么难搞

说到这个，很多人第一反应是"加个 User-Agent 不就行了"。我以前也这么想。

LinkedIn 的反爬机制大概分三层：

**第一层是 IP 层面。** 数据中心 IP 基本上一请求就封，住宅 IP 好一点，但高频请求同样撑不住。我自己测过，同一个住宅 IP 对 LinkedIn 的职位搜索页连续请求 20 次左右，就开始出现 999 错误或者跳验证码。

**第二层是请求指纹。** TLS 握手的 cipher suite 顺序、HTTP/2 的帧结构、请求头的字段顺序——这些东西 Python 的 requests 库和真实浏览器差异很大。LinkedIn 的服务器会做指纹比对，发现不像浏览器就直接返回空数据或者 403。

**第三层是行为模式。** 真实用户不会每隔 0.5 秒请求一次，不会每次都从同一个 referer 进来，不会 session 里没有任何 cookie 积累。

这三层叠在一起，自己维护一套绕过方案的成本非常高。我算过，光是维护一个稳定的住宅代理池，每个月的成本就不低，还要自己处理 IP 轮换逻辑、重试逻辑、指纹伪装——这些时间花出去，不如把精力放在数据处理上。

---

## ScraperAPI 在这里能解决什么

ScraperAPI 本质上是一个代理中间层，但它做的事情比普通代理多得多。

我用它处理 LinkedIn 请求的时候，最直接的感受是：我不需要再管 IP 轮换了。它背后有住宅 IP 池，自动处理轮换，失败自动重试，还会根据目标网站自动调整请求头和 TLS 指纹。

对 Scrapy 来说，接入方式也很干净——通过它的代理端点，或者直接用 API 请求模式，两种都能用。

---

## 全套餐对比

| 套餐名称 | 核心配置 | 月价格 | 开通链接 |
| ------ | ------ | --- | --- |
| Hobby | 100,000 API 积分 / 月，并发 5 线程，支持基础 JS 渲染 | $49/月 | [ 开通 Hobby 套餐](https://www.scraperapi.com/?fp_ref=coupons) |
| Startup | 1,000,000 API 积分 / 月，并发 25 线程，支持 JS 渲染 + 地理定位 | $149/月 | [ 开通 Startup 套餐](https://www.scraperapi.com/?fp_ref=coupons) |
| Business | 3,000,000 API 积分 / 月，并发 50 线程，支持高级 JS 渲染 + 住宅 IP | $299/月 | [ 开通 Business 套餐](https://www.scraperapi.com/?fp_ref=coupons) |
| Enterprise | 积分量 / 并发按需定制，专属支持，SLA 保障 | 联系报价 | [ 咨询 Enterprise 方案](https://www.scraperapi.com/?fp_ref=coupons) |

注意：LinkedIn 这类高反爬目标，每次请求消耗的积分会比普通网站多（通常是 10–25 积分/次，具体取决于是否开启 JS 渲染和住宅 IP）。选套餐前先估算一下你的月请求量。

免费试用提供 5,000 次 API 积分，够跑一个小规模测试。[👉 免费注册拿试用积分](https://www.scraperapi.com/?fp_ref=coupons)

---

## 核心亮点深度展开

### 1. 代理模式接入 Scrapy，改动最小

ScraperAPI 提供一个代理端点：`proxy-server.scraperapi.com:8001`，认证方式是用 API Key 作为用户名。

这意味着你不需要改 Spider 的任何逻辑，只需要在 Scrapy 的 `settings.py` 里加几行：

```python
# settings.py

SCRAPERAPI_KEY = 'your_api_key_here'

DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.httproxy.HttpProxyMiddleware': 110,
    'myproject.middlewares.ScraperAPIMiddleware': 100,
}
```

然后写一个简单的中间件：

```python
# middlewares.py

import base64
from scrapy import signals

class ScraperAPIMiddleware:
    def __init__(self, api_key):
        self.api_key = api_key
        # 构造代理认证头
        credentials = base64.b64encode(
            f'scraperapi:{api_key}'.encode()
        ).decode()
        self.proxy_auth = f'Basic {credentials}'

    @classmethod
    def from_crawler(cls, crawler):
        return cls(
            api_key=crawler.settings.get('SCRAPERAPI_KEY')
        )

    def process_request(self, request, spider):
        request.meta['proxy'] = 'http://proxy-server.scraperapi.com:8001'
        request.headers['Proxy-Authorization'] = self.proxy_auth
        return None
```

我第一次接进去跑的时候，原来那个一直 403 的 LinkedIn 职位搜索页，直接就通了。就这么简单。

### 2. API 请求模式，适合需要 JS 渲染的页面

LinkedIn 的很多页面是动态渲染的，职位详情页尤其明显。这时候代理模式不够用，需要让 ScraperAPI 帮你跑一个无头浏览器。

API 请求模式的写法是把目标 URL 作为参数传给 ScraperAPI 的端点：

```python
# spider.py

import scrapy
from urllib.parse import urlencode, quote_plus

class LinkedInJobsSpider(scrapy.Spider):
    name = 'linkedin_jobs'
    SCRAPERAPI_KEY = 'your_api_key_here'
    SCRAPERAPI_ENDPOINT = 'https://api.scraperapi.com/'
    
    def build_scraperapi_url(self, target_url, render_js=True, country='us'):
        """构造 ScraperAPI 请求 URL"""
        params = {
            'api_key': self.SCRAPERAPI_KEY,
            'url': target_url,
            'render': 'true' if render_js else 'false',
            'country_code': country,
            'premium': 'true',  # 启用住宅 IP，LinkedIn 必须
        }
        return f"{self.SCRAPERAPI_ENDPOINT}?{urlencode(params)}"
    
    def start_requests(self):
        # LinkedIn 职位搜索示例
        search_queries = [
            'python developer',
            'data engineer',
            'machine learning engineer',
        ]
        
        for query in search_queries:
            linkedin_url = (
                f'https://www.linkedin.com/jobs/search/'
                f'?keywords={quote_plus(query)}&location=China'
            )
            api_url = self.build_scraperapi_url(linkedin_url)
            yield scrapy.Request(
                url=api_url,
                callback=self.parse_job_list,
                meta={'original_query': query},
                # 关闭 Scrapy 自带的重试，让 ScraperAPI 处理
                dont_filter=True,
            )
    def parse_job_list(self, response):
        """解析职位列表页"""
        jobs = response.css('div.job-search-card')
        for job in jobs:
            job_url = job.css('a.job-search-card__title-link::attr(href)').get()
            if job_url:
                yield {
                    'title': job.css(
                        'h3.job-search-card__title::text'
                    ).get(').strip(),
                    'company': job.css(
                        'h4.job-search-card__company-name::text'
                    ).get('').strip(),
                    'location': job.css(
                        'span.job-search-card__location::text'
                    ).get('').strip(),
                    'job_url': job_url,
                }
                # 抓取职位详情
                detail_api_url = self.build_scraperapi_url(job_url)
                yield scrapy.Request(
                    url=detail_api_url,
                    callback=self.parse_job_detail,
                    meta={'job_url': job_url},
                )
        # 翻页逻辑
        next_page = response.css(
            'button[aria-label="Next"]::attr(data-offset)'
        ).get()
        if next_page:
            query = response.metaget('original_query', '')
            next_url = (
                f'https://www.linkedin.com/jobs/search/'
                f'?keywords={quote_plus(query)}&start={next_page}'
            )
            yield scrapy.Request(
                url=self.build_scraperapi_url(next_url),
                callback=self.parse_job_list,
                meta={'original_query': query},
            )
    def parse_job_detail(self, response):
        """解析职位详情页"""
        yield {
            'job_url': response.meta.get('job_url'),
            'description': response.css(
                'div.show-more-less-html__markup'
            ).get('').strip(),
            'seniority_level': response.css(
                'span.description__job-criteria-text--criteria::text'
            ).getall(),
            'employment_type': response.css(
                'li.description__job-criteria-item:nth-child(2) span::text'
            ).get('').strip(),
        }
```

这里有个坑我后来才发现：`premium=true` 这个参数对 LinkedIn 来说几乎是必须的。不开的话，即使 JS 渲染开了，还是会碰到 999 错误。开了之后稳定多了，代价是每次请求消耗的积分更多。

### 3. 并发控制，别把自己的账号搞进黑名单

Scrapy 默认并发是 16，对 LinkedIn 来说太激进了。就算 ScraperAPI 帮你处理了 IP 问题，LinkedIn 还是会对同一个 session 的请求频率做检测。

我用的配置：

```python
# settings.py

# 并发控制
CONCURRENT_REQUESTS = 4
CONCURRENT_REQUESTS_PER_DOMAIN = 2

# 请求延迟，加随机抖动
DOWNLOAD_DELAY = 2
RANDOMIZE_DOWNLOAD_DELAY = True  # 实际延迟在 0.5x 到 1.5x 之间随机

# 关闭 cookies，避免 session 追踪
COOKIES_ENABLED = False

# 重试配置
RETRY_TIMES = 3
RETRY_HTTP_CODES = [500, 502, 503, 504, 408, 429, 999]

# AutoThrottle 自动限速
AUTOTHROTLE_ENABLED = True
AUTOTHROTTLE_START_DELAY = 2
AUTOTHROTTLE_MAX_DELAY = 10
AUTOTHROTTLE_TARGET_CONCURRENCY = 2.0
```

跑起来之后，我的成功率从之前自己维护代理池的 40% 左右，提升到了 85% 以上。剩下的失败基本都是 LinkedIn 的登录墙——那些需要登录才能看的内容，ScraperAPI 也没法绕过，这是正常的。

### 4. 结构化数据输出与去重

抓回来的数据要做去重，LinkedIn 的职位 ID 在 URL 里，可以用它做唯一键：

```python
# pipelines.py

import hashlib
import json
from itemadapter import ItemAdapter

class LinkedInDeduplicationPipeline:
    def __init__(self):
        self.seen_ids = set()
    def process_item(self, item, spider):
        adapter = ItemAdapter(item)
        job_url = adapter.get('job_url', '')
        
        # 从 URL 提取职位 ID
        job_id = self._extract_job_id(job_url)
        
        if job_id in self.seen_ids:
            from scrapy.exceptions import DropItem
            raise DropItem(f'重复职位: {job_id}')
        
        self.seen_ids.add(job_id)
        return item
    
    def _extract_job_id(self, url):
        """从 LinkedIn URL 提取职位 ID"""
        import re
        match = re.search(r'/jobs/view/(\d+)', url)
        if match:
            return match.group(1)
        # 备用： URL 的 hash
        return hashlib.md5(url.encode()).hexdigest()

class JsonLinesExportPipeline:
    def open_spider(self, spider):
        self.file = open('linkedin_jobs.jsonl', 'w', encoding='utf-8')
    
    def close_spider(self, spider):
        self.file.close()
    
    def process_item(self, item, spider):
        line = json.dumps(dict(item), ensure_ascii=False) + '\n'
        self.file.write(line)
        return item
```

### 5. 错误监控，知道哪里在掉链子

ScraperAPI 的响应里会带状态信息，我习惯把它记录下来，方便后续分析哪类页面失败率高：

```python
# middlewares.py（补充到之前的中间件里）

import logging

logger = logging.getLogger(__name__)

class ScraperAPIMonitorMiddleware:
    def process_response(self, request, response, spider):
        # ScraperAPI 会在响应头里带上实际状态码
        real_status = response.headers.get(
            'Sa-Final-Status-Code', b''
        ).decode()
        if real_status and real_status != '200':
            logger.warning(
                f'ScraperAPI 返回非 200 状态: {real_status} | '
                f'URL: {request.url[:100]}'
            )
        
        # 记录积分消耗（如果响应头里有的话）
        credits_used = response.headers.get(
            'Sa-Credits-Used', b''
        ).decode()
        if credits_used:
            spider.crawler.stats.inc_value(
                'scraperapi/credits_used', 
                int(credits_used)
            )
        
        return response
```

---

## FAQ

**Q：ScraperAPI 能抓需要登录的 LinkedIn 内容吗？**

不能直接绕过登录墙。ScraperAPI 处理的是 IP 和请求指纹层面的问题，登录态需要你自己提供 cookie。如果你有合法的 LinkedIn 账号，可以把 cookie 注入到请求头里，但这涉及账号安全风险，需要自己权衡。

**Q：Scrapy 接入 ScraperAPI 之后，原来的中间件还能用吗？**

大部分可以正常用。需要注意的是，ScraperAPI 代理模式下，Scrapy 的 `HttpCompressionMiddleware` 有时会和 ScraperAPI 的响应压缩冲突，如果遇到乱码，可以尝试禁用它。

**Q：积分不够用怎么办，能临时升级套餐吗？**

可以随时升级，按月计费，不锁定合同。我自己的习惯是先用 Hobby 套餐跑小规模测试，确认爬虫逻辑没问题再升到 Startup。

**Q：ScraperAPI 有没有专门针对 LinkedIn 的优化？**

它有一个 Structured Data Endpoint，专门针对 LinkedIn 职位页面做了解析，可以直接返回结构化 JSON，不需要自己写 CSS 选择器。对于只需要标准字段（职位名、公司、地点、描述）的场景，这个比自己写 Spider 省事很多。[👉 查看 Structured Data Endpoint 文档](https://www.scraperapi.com/?fp_ref=coupons)

**Q：免费试用够不够用来测试 LinkedIn 爬虫？**

5,000 积分，开启 JS 渲染和住宅 IP 的情况下大概能跑 200–500 次请求，够验证基本流程。建议先用职位搜索列表页测，比详情页便宜。

**Q：ScraperAPI 支持异步框架吗，比如 aiohttp？**

支持。API 请求模式本质上就是一个 HTTP 请求，任何支持 HTTP 的框架都能用。Scrapy 是最顺手的，因为它的中间件机制让接入非常干净。

[👉 免费注册，用 5,000 积分测试你的 LinkedIn 爬虫](https://www.scraperapi.com/?fp_ref=coupons)

---

## 写在最后

如果你只是偶尔抓一次 LinkedIn 数据，自己维护代理池也不是不行，就麻烦。但如果你需要持续稳定地跑，或者团队里没有专门维护爬虫基础设施的人，ScraperAPI 这类服务能帮你把精力集中在数据本身，而不是和反爬机制死磕。

适合用它的场景：需要持续监控职位变化的招聘团队、做人才市场分析的研究者、构建竞品情报系统的产品团队。如果你只是跑一次性的数据采集，Hobby 套餐完全够用。

[👉 免费注册 ScraperAPI，开始稳定抓取 LinkedIn 数据](https://www.scraperapi.com/?fp_ref=coupons)
