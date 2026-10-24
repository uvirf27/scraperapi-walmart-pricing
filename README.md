# ScraperAPI Walmart：拉取商品与搜索数据用官方结构化端点就够，积分消耗、套餐选择一次说清

搜 "ScraperAPI Walmart" 的人，基本都是同一件事：想稳定拿到 Walmart 的商品价格、库存、评论或搜索结果，又不想自己养代理池、写验证码破解和解析器。这篇直接回答三件事：ScraperAPI 的 Walmart 端点能拿到什么数据、一次请求实际烧多少积分、哪个套餐配你的抓取量最划算。

先给结论：ScraperAPI 提供 Walmart 的结构化数据端点（Product 和 Search），传一个商品 ID 就能拿回解析好的 JSON，单次请求按 5 积分计费；7 天免费试用给 5,000 积分，不用信用卡。如果你每个月要抓的 Walmart 商品在一两万次以内，Hobby（$49/月）大概率够用；需要国家级地理定位得从 Business（$299/月）起；积分用量难以预估的团队，要看支持按量超额付费的 Scaling 档（$475/月）。

> **选购速览**：多数个人和小团队场景，从免费试用开始验证数据字段是否满足需求，再上 Hobby。想直接看全部套餐和年付价格，👉 [查看 ScraperAPI 全部套餐与年付价格](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## ScraperAPI 的 Walmart 端点能拿到什么

商品详情一次调用就能拿全。同步端点是 GET `https://api.scraperapi.com/structured/walmart/product`，必填参数只有两个：你的 `api_key` 和 `product_id`。商品 ID 就藏在 Walmart 链接的 `/ip/` 后面，比如 `walmart.com/ip/5253396052` 里那串数字。

返回的是解析好的 JSON，字段包括 `product_name`、`product_description`、`brand`、`image` 和 `offers`。官方文档里的示例响应是一台 "AT&T Samsung Galaxy S24 Ultra"，`offers` 里带 `availability`（InStock）、配送方式和商品成色这些字段——做价格监控或库存监控，这层结构基本直接可用。

几个可选参数值得记住：

- **`tld`**：只有两个合法值，`com`（walmart.com）和 `ca`（walmart.ca）。想抓 Walmart 其他国家的站点，这个端点不支持。
- **`country_code`**：两位国家代码（如 `au`、`es`、`it`），控制出口 IP 的地理位置。注意这依赖套餐的地理定位权限，后文会讲这个限制卡在哪个档位。
- **`output_format`**：`json`（默认）或 `csv`。

需要批量抓的时候，异步端点更合适。POST `https://async.scraperapi.com/structured/walmart/product`，请求体里放 `productIds` 数组就能一次提交多个商品，每个商品返回一个独立作业对象，带 `statusUrl` 可以轮询进度，也可以设 `callback` 用 webhook 接收结果。作业完成后，返回的数据结构和同步端点完全一致，代码不用写两套。

搜索数据走另一个端点（`/structured/walmart/search`），返回搜索结果里的商品列表。如果你不想用结构化端点，传统路线也还在：向 `api.scraperapi.com` 传目标 Walmart 页面 URL 加 `render=true`，拿回渲染后的 HTML，自己用 Beautiful Soup 之类的库解析。ScraperAPI 自己的教程就是这么演示的，还专门提示了 Walmart 页面数据藏在 `__NEXT_DATA__` JSON 里、解析器容易随页面改版失效——这也是为什么多数人现在直接用结构化端点。

## 全套餐价格表：从免费到企业级一共九档

以下为 ScraperAPI 定价页当前公开展示的套餐（月付价，括号内为选择年付后折算到每月的价格，年付统一 9 折）：

| 套餐 | 价格/月 | 积分额度 | 并发线程 | 地理定位 | 购买链接 |
| --- | --- | --- | --- | --- | --- |
| Free | $0 | 1,000/月 | 5 | — | [免费注册](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | $49（$44.10） | 100,000 | 20 | 仅美国和欧洲 | [查看 Hobby 套餐](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | $149（$134.10） | 1,000,000 | 50 | 仅美国和欧洲 | [查看 Startup 套餐](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | $299（$269.10） | 3,000,000 | 100 | 全球（国家级） | [查看 Business 套餐](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | $475（$427.50） | 5,000,000 | 200 | 全球（国家级） | [查看 Scaling 套餐](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | $975（$877.50） | 10,500,000 | 300 | 全球（国家级） | [查看 Professional 套餐](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | $1,975（$1,777.50） | 21,500,000 | 500 | 全球（国家级） | [查看 Advanced 套餐](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | 定制 | 22M+ 起 | 500+ | 全球（国家级） | [联系销售](https://www.scraperapi.com/?fp_ref=coupons) |

所有付费套餐都包含 JS 渲染、高级代理、JSON 自动解析、CAPTCHA 与反爬检测、自定义请求头和会话、自动重试、不限带宽，以及 99.9% 在线率 SLA。也就是说，功能不是按档位拆开卖的，档位差的是量、并发和两项权限（下面细说）。

## 积分消耗才是真实成本，标称额度会骗人

这是大多数测评没算清楚的一笔账。ScraperAPI 按 API 积分计费，而积分消耗随请求类型变化很大：普通 HTML 请求 1 积分，JS 渲染 10 积分，超高级代理 25 积分，两者叠加最高 75 积分。对 Walmart 用户来说最关键的一条是：**电商类目标站（Walmart、Amazon、eBay）的结构化端点每次固定消耗 5 积分**。

所以真实账要这么算：

- Hobby 的 100,000 积分，按每次 5 积分算，大约是 **20,000 次 Walmart 商品查询**，不是 10 万次。
- 如果你走传统 HTML 路线还开了 `render=true`，一次请求烧 10 积分以上，同样额度只剩约 1 万次。
- 失败请求不扣积分，只对成功响应计费，这点对 Walmart 这种会间歇性拦截的站点算个安慰。

一份第三方定价分析的原话是：积分额的实际产出"最多能差约 75 倍"——同样 10 万积分的两个客户，一个能做 10 万次简单请求，另一个只够一千多次复杂抓取。看中哪个套餐之前，先按你自己的请求类型把额度除一遍。

## 三个写在套餐细则里的限制

第一个限制是地理定位。Hobby 和 Startup 的地理定位只覆盖美国和欧洲，需要抓其他国家的 Walmart 数据（配合 `country_code` 参数），得升到 Business 起的档位。想抓 walmart.ca 的话不用升档，`tld=ca` 在低档套餐就能用，但出口 IP 所在地受套餐限制。

第二个限制是超额处理。Hobby、Startup、Business 三档的积分用完后没有按量付费选项，只能升级；从 Scaling 开始才支持按量超额付费（PAYG），月度额度用完可以按固定单价继续抓。如果你的抓取量季节性波动大、很难预估，这个差异比积分数字本身更重要。

第三个限制是并发。Hobby 只有 20 个并发线程，跑大批量任务会明显变慢。一份 2026 年 3 月的独立测评把这一点列为主要短板：如果你做的是时效敏感的价格监控，20 并发可能撑不住你的轮询频率。

免费额度也顺便说清楚：注册后先有 7 天试用，送 5,000 积分，不需要信用卡；试用期结束后有一个永久免费账户，每月循环补充 1,000 积分，最多 5 个并发。1,000 积分按 Walmart 端点 5 积分一次算，就是每月 200 次商品查询——够验证字段和写代码，不够跑生产。

试用是验证"这个工具适不适合你"成本最低的方式，👉 [领取 7 天 5,000 积分免费试用](https://www.scraperapi.com/?fp_ref=coupons)，注册后后台就能看到你的 API Key。

## 和其他 Walmart 抓取方案比，它处在什么位置

搜索 Walmart 抓取方案时你会撞见好几类产品，它们解决的是不同的问题。按已核验的信息摆在一起：

| 方案 | 接入方式 | 数据形态 | 定价模式 |
| --- | --- | --- | --- |
| ScraperAPI | 一个 API key | 结构化 JSON（商品/搜索） | 积分包月 |
| Walmart 专用 scraper API（如 walmartscraperapi 一类） | 一个 API key | 专用端点返回 JSON | 按成功请求计费，免费 1,000 次 |
| Walmart 官方联盟 API | 需通过联盟计划审批 | feed JSON，有门槛 | 对已批准联盟免费 |
| SerpApi Walmart API | API key | 搜索+商品 JSON | 按搜索量订阅 |
| Oxylabs / Bright Data | 企业合同或数据集配置 | 解析后电商结构/数据集记录 | 订阅制，起订门槛高 |
| Apify | 每类任务一个 actor | actor 输出 JSON | 按结果计费，有月度免费额度 |

几个诚实的判断：如果你的需求只有 Walmart 一个站点、量又大，专用的 Walmart scraper API 按成功请求计费的模式确实更省——有厂商报出 Pro 档约 $0.60/1,000 次的价格。如果你已经被批准进入 Walmart 的联盟计划，官方 API 本身免费，只是拿不到评论数据且审批流程不在你控制内。反过来，如果你的抓取清单以后会加 Amazon、Google 或其他站点，一个通用平台的积分可以跨目标复用，这也是 ScraperAPI 这类产品的核心卖点——它在 2026 年 4 月收购了 Traject Data（Rainforest API 和 SerpWow 背后的公司），结构化端点已经覆盖十种实时 SERP 与电商数据源。

## 实际成功率：第三方基准怎么说

公开测评对 ScraperAPI 的评价有分歧，且分歧本身有信息量。

一份 2026 年 9 月的横向基准把 ScraperAPI 排在 7 个受测抓取 API 的第 4 名：11 个目标网站的整体成功率 52%，平均响应 7.5 秒，折算成本约 $4.05/1,000 请求。另一份 2026 年 7 月的测评引用的独立基准则给出 62–64% 的成功率区间，并明确指出它在 Cloudflare 这类强反爬目标上表现偏弱。

两组数字不矛盾，它们测的目标站组合不同。翻译成决策语言：抓 Walmart 这类主流电商站，属于这类产品的舒适区，成功率大概率高于它们在硬核目标上的平均线；但如果你同时要抓 Instagram、Booking.com 这种级别的防护目标，多家测评都建议另做打算。还有一条来自测评的实用信息：有用户遇到 CAPTCHA 页被误计积分，联系客服后获得修正——积分异常时值得去后台核对用量明细。

## 两段可以直接抄的代码

同步拿单个商品：

bash
curl --request GET \
  --url "https://api.scraperapi.com/structured/walmart/product?api_key=API_KEY&product_id=5253396052&tld=com"


批量异步提交（Python 风格的请求体）：

json
{
  "apiKey": "API_KEY",
  "productIds": ["5253396052", "616074177"],
  "tld": "com",
  "output_format": "json",
  "callback": { "type": "webhook", "url": "https://你的服务/回调地址" }
}


发送到 `https://async.scraperapi.com/structured/walmart/product`，立即返回作业 ID 和状态查询地址，结果通过 webhook 或轮询拿取。批量场景下这套流程比逐个同步请求省事，也更容易控制并发。

## FAQ

**抓 Walmart 商品一次请求到底花多少积分？** 结构化端点对电商目标站固定收 5 积分一次；走 HTML 解析路线则是普通请求 1 积分、开 JS 渲染 10 积分起。做大规模商品监控，优先用结构化端点。

**支持 Walmart 加拿大站吗？** 支持，`tld` 参数设为 `ca` 即可。其他国家站点目前不在支持列表里。

**积分用完会被直接扣费吗？** 不会。Hobby 及以下档位积分用尽后请求会被限制，需要升级；Scaling 及以上可以按量超额付费继续使用。

**有没有退款或免费试用？** 有 7 天免费试用（5,000 积分，无需信用卡），试用期结束后还有每月 1,000 积分的永久免费账户。具体退款条款以官网条款页为准。

## 按你的情况选套餐

- 每月 Walmart 商品查询在 2 万次以内、只需要美国/欧洲数据：Hobby（$49/月，年付 $44.10）够用，先用免费试用验证字段。
- 每日监控、每周竞品分析这类常规任务，量在 10 万–60 万次普通请求：Startup 到 Business 之间选，差价换的是 10 倍积分。
- 需要美国和欧洲以外的国家级地理定位：直接从 Business 起，低档没有这个权限。
- 抓取量波动大、不想每月盯着额度：Scaling 是支持按量超额的最低档位。
- 只抓 Walmart 一个站、追求单价最低：专用 Walmart scraper API 的按成功计费模式更省，可以直接按上面表格里的信息去比较。

下单前确认两件事就行：把你预期的请求类型（结构化端点还是渲染 HTML）套进积分倍率算一遍真实月用量；再确认你要抓的站点数据在 `tld` 支持范围内。都确认了，👉 [开通 ScraperAPI 并开始 7 天免费试用](https://www.scraperapi.com/?fp_ref=coupons)。
