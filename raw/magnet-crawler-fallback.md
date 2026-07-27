---
title: "我放弃了磁力爬虫，但这4套分级降级与反爬方案你拿走"
date: 2026-07-27 12:30:00
evidence: "亲测"
tags:
  - 爬虫
  - 磁力
  - Python
  - 反爬
categories:
  - 教程类
  - 开发工具
description: "磁力搜索爬虫的4级降级方案：从纯静态正则到Headless浏览器再到DrissionPage破Cloudflare盾，含完整Python代码和实战决策树。"
cover: /images/magnet-crawler-fallback-cover.webp
---
我先说结论：**我放弃了自建磁力爬虫**。

但这不是一个失败故事。在放弃之前，我跑通了从纯静态正则会到 Headless 浏览器破 Cloudflare 盾的全套技术栈，整理出了一套 4 级降级方案。你可以直接拿走用，省掉我踩过的所有坑。

这篇不是完整的爬虫教程，而是一份**选型决策树**——面对一个资源站，你能立刻判断该用什么级别去打。

## 4 级降级方案：按需升级，绝不越级

每升一级，资源开销成倍增长。我一共设计了 4 个级别：

| 级别 | 适用场景 | 开销 | 技术栈 |
|---|---|---|---|
| 🟢 1 静态正则 | 老 HTML 站，无反爬 | 极低 | requests + BeautifulSoup + re |
| 🟡 2 复杂静态 | 链接藏 JS/隐藏 input 里 | 中等 | requests + Base64 解码 |
| 🟠 3 Headless 浏览器 | 必须点 DOM 才出链接 | 高 | Playwright |
| 🔴 4 终极破盾 | Cloudflare 5 秒盾 | 极高 | DrissionPage |

**核心原则：一旦当前级别能搞定，立刻返回，绝不无故触发下一级。**

为什么？Headless 浏览器一启动就吃 500MB+ 内存。并发 3 个直接 OOM。你得对宿主机负责。

## 级别 1：纯静态正则（80% 的站靠它）

### 什么时候用

上古论坛、下载站，HTML 直接写磁力链接，没有任何反爬。

典型代表：`dygod.net`（电影天堂）、各种老 PT 论坛。

### 代码

```python
import requests, re
from bs4 import BeautifulSoup

MAGNET_RE = re.compile(r'magnet:\?xt=urn:btih:[a-zA-Z0-9]+', re.I)
FTP_RE = re.compile(r'ftp://[^\s"\'<>]+', re.I)

def scrape_level1(url):
    resp = requests.get(url, headers={"User-Agent": "Mozilla/5.0..."}, timeout=15)
    # 老站常用 GB2312
    if 'gb2312' in (resp.apparent_encoding or '').lower():
        resp.encoding = 'gb2312'
    html = resp.text
    magnets = set(MAGNET_RE.findall(html))
    ftps = set(FTP_RE.findall(html))
    return list(magnets) + list(ftps)
```

### 实战经验

1. **绕开主页**：用站内搜索或 Google `site:xxx 关键字` 定位到详情页，链接密度更高
2. **编码处理**：老站常用 GB2312，不显式声明会乱码
3. **提取元数据**：别只返回磁力链接——用户需要知道文件大小、清晰度、格式、音轨

```python
def extract_metadata(context_text):
    meta = {}
    if res := re.search(r'\b(2160p|1080p|720p|4K)\b', context_text, re.I):
        meta['resolution'] = res.group(1)
    if fmt := re.search(r'\b(BluRay|WEB-DL|HDTV|Remux|BDRip)\b', context_text, re.I):
        meta['source'] = fmt.group(1)
    if size := re.search(r'(\d+\.?\d*)\s*(GB|MB)', context_text, re.I):
        meta['size'] = f"{size.group(1)} {size.group(2)}"
    return meta
```

> 这个级别全程不到 50MB 内存，宿主机毫无压力。80% 的老站用它就够了。

## 级别 2：复杂静态解析

### 什么时候用

页面返回 200 OK，但磁力链接被 Base64 编码藏在 JS 变量里，或者放在隐藏 `<input>`、`data-*` 属性中。

### 代码

```python
import base64

def scrape_level2(html):
    magnets = []
    soup = BeautifulSoup(html, 'html.parser')

    # 从隐藏 input 提取
    for inp in soup.find_all('input', type='hidden'):
        if 'magnet:?' in inp.get('value', ''):
            magnets.append(inp['value'])

    # 从 data-* 属性提取
    for elem in soup.find_all(attrs={'data-magnet': True}):
        magnets.append(elem['data-magnet'])

    # Base64 解码 JS 变量
    b64_pattern = re.compile(r'["\']([A-Za-z0-9+/=]{40,})["\']')
    for match in b64_pattern.findall(html):
        try:
            decoded = base64.b64decode(match).decode('utf-8', errors='ignore')
            if 'magnet:?' in decoded:
                magnets.extend(MAGNET_RE.findall(decoded))
        except Exception:
            continue

    return list(set(magnets))
```

这个级别解决 60% 的"看起来很难其实静态"的网站。

## 级别 3：Headless 浏览器（高开销，按需用）

必须点击真实 DOM 节点才触发 XHR 返回链接的网站——比如 React/Vue 渲染的页面。

```python
from playwright.sync_api import sync_playwright

def scrape_level3(url):
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        try:
            page = browser.new_page()
            page.goto(url, timeout=30000)
            page.wait_for_load_state('networkidle')
            page.click('a.download-btn', timeout=5000)
            page.wait_for_timeout(2000)
            html = page.content()
            return MAGNET_RE.findall(html)
        finally:
            browser.close()  # 🔥 必须 close，否则内存泄漏
```

**铁律**：严禁并发。一个 browser 吃 500MB+，开 5 个直接 OOM。

## 级别 4：DrissionPage 终极破盾（核武器）

Cloudflare 5 秒盾、521 Anti-Bot 拦截——这时候只有 DrissionPage 能破。

```python
from DrissionPage import ChromiumPage, ChromiumOptions

def scrape_level4(url):
    options = ChromiumOptions()
    options.headless(True)
    page = None
    try:
        page = ChromiumPage(options)
        page.get(url, retry=3, timeout=30)
        page.wait.load_start()
        for _ in range(30):  # 等 CF 挑战完成
            if 'just-a-moment' not in page.html.lower():
                break
            page.wait(1)
        return MAGNET_RE.findall(page.html)
    finally:
        if page:
            page.quit()  # 🔥 必须 quit，销毁 Chromium 内核
```

## 实战决策树

```
新站 → 级别 1 试
  ↓ 成功？搞定
  ↓ 失败 → 级别 2 试
    ↓ 成功？搞定
    ↓ 失败 → 级别 3 试
      ↓ 成功？搞定
      ↓ 失败 → 级别 4（最后手段）
        ↓ 失败？放弃，换站
```

每一级用 `try/except`，失败自动降级下一级。**别一上来就 Headless 浏览器，杀鸡用牛刀**。

## 资源保护红线

不管用哪一级：

1. **严禁并发**：Headless 浏览器吃 500MB+，并发 3 个就可能 OOM
2. **强制资源释放**：`try...finally` 中执行 `browser.close()` 或 `page.quit()`
3. **超时必须设**：每个 HTTP 请求 `timeout=15`
4. **频率限制**：两次请求之间 `time.sleep(1-3)`
5. **优先静态**：能静态正则绝不用浏览器

## 适合谁与不适合谁

**适合**：偶尔需要搜资源的，想把爬虫当工具而非事业来做的。

**不适合**：想做大规模聚合站的——这个方案是"够用就行"级别，不是商业级。需要长期维护大量站点爬虫的——站点变动快，维护成本高。

> 爬虫的精髓不是"用最牛的技术"，而是"用最低的成本搞定目标"。80% 的老站静态正则就够，真碰到 JS 渲染再上 Headless，CF 盾最后才动 DrissionPage。按这个分级，用最少的资源跑最多的站。

> 「每天帮你踩一个 AI 的坑，省下一小时。」

你做过爬虫吗？踩过什么反爬坑？评论区聊聊。
