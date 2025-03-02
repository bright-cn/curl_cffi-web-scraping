# 使用 curl_cffi 在 Python 中进行网络爬虫

[![Promo](https://github.com/luminati-io/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://www.bright.cn/)

本指南将展示如何使用 curl_cffi 来增强 Python 爬虫脚本，通过模拟真实浏览器的 TLS 指纹来绕过一些基于指纹检测的反爬机制。

- [什么是 `curl_cffi`？](#什么是-curl_cffi)
- [它是如何工作的](#它是如何工作的)
- [如何使用 `curl_cffi` 进行网络爬虫](#如何使用-curl_cffi-进行网络爬虫)
  - [步骤 #1：项目设置](#步骤-1项目设置)
  - [步骤 #2：安装 `curl_cffi`](#步骤-2安装-curl_cffi)
  - [步骤 #3：连接目标页面](#步骤-3连接目标页面)
  - [步骤 #4：添加数据爬取逻辑](#步骤-4添加数据爬取逻辑)
  - [步骤 #5：整合所有内容](#步骤-5整合所有内容)
- [`curl_cffi`：高级用法](#curl_cffi高级用法)
  - [浏览器模拟选择](#浏览器模拟选择)
  - [会话管理](#会话管理)
  - [代理整合](#代理整合)
  - [异步 API](#异步-api)
  - [WebSocket 连接](#websocket-连接)
- [`curl_cffi` 与 Requests、AIOHTTP、HTTPX 在网络爬虫方面的比较](#curl_cffi-与-requestsaiohttphttpx-在网络爬虫方面的比较)
- [`curl_cffi` 在网络爬虫方面的替代方案](#curl_cffi-在网络爬虫方面的替代方案)

## 什么是 `curl_cffi`？

[`curl_cffi`](https://github.com/lexiforest/curl_cffi) 通过 CFFI 为 `curl-impersonate` 提供 Python 绑定，从而能够模拟浏览器的 TLS/JA3/HTTP2 指纹。这有助于绕过很多基于 [TLS 指纹](https://www.bright.cn/blog/web-data/tls-fingerprinting) 的反爬检测。

它的主要特性包括：

- 支持 JA3/TLS 和 HTTP2 指纹模拟，包括最新的浏览器和自定义指纹  
- 性能比 `requests` 和 `httpx` 更快，与 `aiohttp` 相当  
- API 类似于 `requests`  
- 支持 `asyncio` 进行异步请求  
- 支持为每一个请求进行代理轮换  
- 支持 HTTP/2.0 和 `WebSocket`  

## 它是如何工作的

当你发送 HTTPS 请求时，会触发 [TLS 握手](https://www.cloudflare.com/learning/ssl/what-happens-in-a-tls-handshake/)，并产生唯一的 TLS 指纹。由于常规 HTTP 客户端与网页浏览器在握手细节上有差异，某些服务器能通过这些差异识别自动化流量并触发反爬。

`curl-impersonate`（也是 `curl_cffi` 的基础）通过定制 cURL 来复制真实浏览器的 TLS 指纹：

- **TLS 库微调**：使用与浏览器相同的 TLS 连接库，而不是传统 cURL 依赖的库。  
- **配置调整**：修改 TLS 扩展和 SSL 选项来模仿浏览器行为。  
- **HTTP/2 定制化**：匹配浏览器在 HTTP/2 握手环节的设置。  
- **非默认 cURL 选项**：使用 `--ciphers`、`--curves` 以及自定义请求头等提升模拟精度。  

这样发出的请求更像真正的浏览器流量，从而帮助绕过一些基于指纹侦测的反机器人机制。

## 如何使用 `curl_cffi` 进行网络爬虫

下面我们以爬取沃尔玛 (Walmart) 上“Keyboard”搜索页面为例：

![沃尔玛“Keyboard”产品页面](https://github.com/bright-cn/curl_cffi-web-scraping/blob/main/Images/s_5A13EDF6E0EA32867C0C89DFE864B4C8FA81CE91CC8CE80729F465B232BE7073_1737992236291_image.png)

如果你使用普通 HTTP 客户端访问该页面，就会收到下图所示的警告页面：

![服务器返回的提示](https://github.com/bright-cn/curl_cffi-web-scraping/blob/main/Images/s_5A13EDF6E0EA32867C0C89DFE864B4C8FA81CE91CC8CE80729F465B232BE7073_1737992185267_image.png)

即使在请求头中设置了 “User-Agent” 模拟常见浏览器，也会因为 TLS 指纹不同而被知乎为机器人。这时，`curl_cffi` 就派上用场了。

### 步骤 #1：项目设置

确保你已安装 Python 3 及以上版本。然后新建一个用于 `curl_cffi` 爬虫的目录：

```
mkdir curl-cfii-scraper
```

进入该目录并创建一个 [虚拟环境](https://docs.python.org/3/library/venv.html)：

```
cd curl-cfii-scraper
python -m venv env
```

在你喜欢的 Python IDE 中打开此项目文件夹，并在该文件夹中新建一个名为 `scraper.py` 的文件。

然后在终端中激活虚拟环境。若是 Linux 或 macOS：

```bash
./env/bin/activate
```

如果你在 Windows 上：

```bash
env/Scripts/activate
```

### 步骤 #2：安装 `curl_cffi`

在激活的虚拟环境中安装此 HTTP 客户端：

```
pip install curl-cffi
```

### 步骤 #3：连接目标页面

从 `curl_cffi` 中导入 `requests`：

```python
from curl_cffi import requests
```

`requests` 提供了一个类似于 `requests` 库的高级 API。可以用它对页面发起 GET 请求：

```python
response = requests.get("https://www.walmart.com/search?q=keyboard", impersonate="chrome")
```

这里的 `impersonate="chrome"` 参数会让 `curl_cffi` 在发起请求时模仿最新版本 Chrome 浏览器的 TLS 指纹，从而使沃尔玛不再把你的自动化请求识别为机器人请求。

获取页面的 HTML 内容：

```python
html = response.text
```

如果打印 `html`，你将看到：

```html
<!DOCTYPE html>
<html lang="en-US">
   <head>
      <meta charSet="utf-8"/>
      <meta property="fb:app_id" content="105223049547814"/>
      <meta name="viewport" content="width=device-width, initial-scale=1.0, minimum-scale=1, interactive-widget=resizes-content"/>
      <link rel="dns-prefetch" href="https://tap.walmart.com "/>
      <link rel="preload" fetchpriority="high" crossorigin="anonymous" href="https://i5.walmartimages.com/dfw/63fd9f59-a78c/fcfae9b6-2f69-4f89-beed-f0eeb4237946/v1/BogleWeb_subset-Bold.woff2" as="font" type="font/woff2"/>
      <link rel="preload" fetchpriority="high" crossorigin="anonymous" href="https://i5.walmartimages.com/dfw/63fd9f59-a78c/fcfae9b6-2f69-4f89-beed-f0eeb4237946/v1/BogleWeb_subset-Regular.woff2" as="font" type="font/woff2"/>
      <link rel="preconnect" href="https://beacon.walmart.com"/>
      <link rel="preconnect" href="https://b.wal.co"/>
      <title>Electronics - Walmart.com</title>
      <!-- 省略部分内容 ... -->
```

### 步骤 #4：添加数据爬取逻辑

要进行实际爬取，还需要一个 HTML 解析库，例如 [BeautifulSoup](https://www.crummy.com/software/BeautifulSoup/bs4/doc/)：

```bash
pip install beautifulsoup4
```

在 `scraper.py` 中导入它：

```python
from bs4 import BeautifulSoup
```

使用它来解析获取到的 HTML：

```python
soup = BeautifulSoup(response.text, "html.parser")
```

[`"html.parser"`](https://docs.python.org/3/library/html.parser.html) 是 Python 标准库内置的 HTML 解析器，可让 BeautifulSoup 使用多种方法检索选择器并提取数据。

举个例子，下面的示例只获取页面的 `<title>` 标签。我们可以用 [CSS 选择器](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_selectors) 中的 `find()` 方法，然后访问它的 `text` 属性：

```python
title_element = soup.find("title")
title = title_element.text
```

然后打印页面标题：

```python
print(title)
```

### 步骤 #5：整合所有内容

以下是最终的 `curl_cffi` 爬虫示例脚本：

```python
from curl_cffi import requests
from bs4 import BeautifulSoup

# 访问沃尔玛搜索“keyboard”页面
response = requests.get("https://www.walmart.com/search?q=keyboard", impersonate="chrome")

# 提取页面 HTML
html = response.text

# 用 BeautifulSoup 解析页面内容
soup = BeautifulSoup(response.text, "html.parser")

# 使用 CSS 选择器获取<title>标签，并获取其文本内容
title_element = soup.find("title")
title = title_element.text

# 这里可以加入更复杂的爬取逻辑...

# 打印爬取结果
print(title)
```

运行：

```bash
python3 scraper.py
```

若在 Windows 上：

```bash
python scraper.py
```

结果为：

```
Electronics - Walmart.com
```

如果省略 `impersonate="chrome"` 参数，则很可能会得到如下信息：

```
Robot or human?
```

## `curl_cffi`：高级用法

### 浏览器模拟选择

`curl_cffi` 支持使用多个浏览器的指纹进行模拟，只需在发起请求时通过 `impersonate` 参数指定：

```python
response = requests.get("<YOUR_URL>", impersonate="<BROWSER_LABEL>")
```

可用的模拟标签包括：

- `chrome99`, `chrome100`, `chrome101`, `chrome104`, `chrome107`, `chrome110`, `chrome116`, `chrome119`, `chrome120`, `chrome123`, `chrome124`, `chrome131`
- `chrome99_android`, `chrome131_android`
- `edge99`, `edge101`
- `safari15_3`, `safari15_5`, `safari17_0`, `safari17_2_ios`, `safari18_0`, `safari18_0_ios`

一些使用建议：

1. 想始终模拟最新浏览器版本，可直接使用 `chrome`、`safari` 或 `safari_ios`。  
2. [由于 Firefox 目前不被支持](https://github.com/lexiforest/curl_cffi/issues/59)，它使用 NSS，而其他浏览器使用 boringssl，curl 无法同时链接两个 TLS 库。  
3. 只有当浏览器指纹发生变化时，才会新增版本。如果某个版本未列出，你可用前一版本的指纹来模拟。  
4. 对非浏览器场景，可使用 `ja3`、`akamai` 等自定义的 TLS 指纹。更多细节可查阅 [impersonation 文档](https://curl-cffi.readthedocs.io/en/latest/impersonate.html)。  

### 会话管理

`curl_cffi` 可以使用 [`Session`](https://requests.readthedocs.io/en/latest/user/advanced/#session-objects) 对象来跨多个请求维持会话参数，例如 cookies、headers 或其他会话数据。

示例：

```python
# 创建一个新的会话
session = requests.Session()

# 该端点在服务器端设置一个名为 userId 的 cookie
session.get("https://httpbin.io/cookies/set/userId/5", impersonate="chrome")

# 打印会话中的 cookies，确认它们已被存储
print(session.cookies)
```

输出结果为：

```
<Cookies[<Cookie userId=5 for httpbin.org />]>
```

这说明会话对象可以保持请求间的状态，例如服务器定义的 cookies。

### 代理整合

与 `requests` 类似，`curl_cffi` 也可以通过 `proxies` 参数来设置代理：

```python
# 定义你的代理地址
proxy = "YOUR_PROXY_URL"

# 创建一个包含 HTTP 和 HTTPS 代理的字典
proxies = {"http": proxy, "https": proxy}

# 使用代理和浏览器模拟来发起请求
response = requests.get("<YOUR_URL>", impersonate="chrome", proxies=proxies)
```

### 异步 API

`curl_cffi` 通过 [`AsyncSession`](https://curl-cffi.readthedocs.io/en/latest/api.html#curl_cffi.requests.AsyncSession) 支持在 `asyncio` 中进行异步 HTTP 请求：

```python
from curl_cffi.requests import AsyncSession
import asyncio

# 定义异步函数
async def fetch_data():
    async with AsyncSession() as session:
        # 执行异步 GET 请求
        response = await session.get("https://httpbin.org/anything", impersonate="chrome")
        # 打印返回文本
        print(response.text)

# 运行异步函数
asyncio.run(fetch_data())
```

使用 `AsyncSession` 能更方便地并发执行多个请求。

### WebSocket 连接

`curl_cffi` 同样支持使用 [`WebSocket`](https://curl-cffi.readthedocs.io/en/latest/api.html#curl_cffi.requests.Session.ws_connect) 类来建立 WebSocket 连接：

```python
from curl_cffi.requests import WebSocket

# 定义一个回调函数，用来处理收到的消息
def on_message(ws, message):
    print(message)

# 使用回调函数初始化 WebSocket 连接
ws = WebSocket(on_message=on_message)

# 连接一个示例 WebSocket 服务并监听消息
ws.run_forever("wss://api.gemini.com/v1/marketdata/BTCUSD")
```

这在需要实时数据的场景下十分有用，比如那些通过 `WebSocket` 传输动态数据的网站或 API。

如果不需要渲染页面，而只是获取动态数据，可以直接抓取 `WebSocket` 数据来提高效率。

> **注意**：  
> 你还可以使用 [`AsyncWebSocket`](https://curl-cffi.readthedocs.io/en/latest/api.html#curl_cffi.requests.AsyncSession.ws_connect) 在异步场景下进行 WebSocket 通信。

## `curl_cffi` 与 Requests、AIOHTTP、HTTPX 在网络爬虫方面的比较

下表对比了 `curl_cffi` 和其他常见 Python HTTP 客户端在网络爬虫中的主要特性：

| **特性**                 | **curl_cffi** | **Requests** | **AIOHTTP** | **HTTPX** |
|--------------------------|---------------|--------------|------------|----------|
| **同步 API**            | ✔️            | ✔️           | ❌          | ✔️        |
| **异步 API**            | ✔️            | ❌           | ✔️          | ✔️        |
| **支持 `WebSocket`**    | ✔️            | ❌           | ✔️          | ❌        |
| **连接池**              | ✔️            | ✔️           | ✔️          | ✔️        |
| **支持 HTTP/2**         | ✔️            | ❌           | ❌          | ✔️        |
| **自定义 `User-Agent`** | ✔️            | ✔️           | ✔️          | ✔️        |
| **TLS 指纹伪装**        | ✔️            | ❌           | ❌          | ❌        |
| **速度**                | 高           | 中           | 高         | 中       |
| **重试机制**            | ❌            | 需通过 `HTTPAdapter` 实现 | 需第三方库 | 通过自定义 `Transport` |
| **代理整合**            | ✔️            | ✔️           | ✔️          | ✔️        |
| **Cookie 处理**         | ✔️            | ✔️           | ✔️          | ✔️        |

## `curl_cffi` 在网络爬虫方面的替代方案

`curl_cffi` 需要你手动编写所有爬虫逻辑，对于一般的静态页面足够，但若遇到复杂动态页面和深度安全检测的场景，实施起来可能并不轻松。

Bright Data 提供了多种替代方案：

- [抓取浏览器](https://www.bright.cn/products/scraping-browser)：基于 Puppeteer、Selenium 或 Playwright 的全托管云浏览器，内置了自动验证码识别和代理轮换。  
- [网络抓取 API](https://www.bright.cn/products/web-scraper)：预配置的 API，可直接获取超过百种热门站点的结构化数据。  
- [无代码爬虫](https://www.bright.cn/products/web-scraper/no-code)：简单易用的云端数据采集服务，无需编写代码。  
- [Datasets](https://www.bright.cn/products/datasets)：浏览并使用预构建的数据集，或自定义采集请求来获取你所需的特定数据。

立即创建一个免费的 Bright Data 帐号，体验我们的代理与爬虫解决方案吧！
