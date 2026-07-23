---
title: 自建磁力搜索爬虫：从入门到放弃再到放弃得明明白白
date: 2026-06-20 21:30:00
tags:
  - 爬虫
  - 磁力
  - Python
  - 反爬
categories:
  - 教程类
  - 开发工具
description: "自建磁力爬虫这条路我走完了——从入门到放弃，再到放弃得明明白白。复盘了整个技术路线和最终决策，省得你重蹈覆辙。"
cover: /images/magnet-crawler-fallback-cover.webp
---

![封面图](/images/magnet-crawler-fallback-cover.webp)

> 想自己写个磁力搜索的爬虫？看完市面上大大小小的资源站，我总结出一套"分级降级"的实战规约：从纯静态正则到 Headless 浏览器破盾，按需升级，**用最低成本搞定 80% 的网站**。

---

## 写在前面：为什么写这篇

磁力搜索这件事，国内的站基本都活不长，今天能用的站明天就挂；国外的站又普遍有强反爬。**纯靠 requests 写爬虫，十有八九会失败**。

我摸索了一段时间，把"什么站用什么级别、什么级别用什么技术栈、怎么保护宿主机资源"整理出一套实战规约。这篇不是完整的爬虫教程，而是**一份选型决策树**——让你面对一个新站时，能快速判断该用什么级别去打。

---

## 核心原则：分级降级，按需升级

爬虫的级别从低到高，开销也越来越大。我设计了 4 个级别：

| 级别 | 适用场景 | 资源开销 | 技术栈 |
|---|---|---|---|
| 🟢 1 - 静态正则 | 上古 HTML 站，无反爬 | 极低 | requests + BeautifulSoup + re |
| 🟡 2 - 复杂静态 | 链接在 JS/隐藏 input 里 | 中等 | requests + Base64 解码 |
| 🟠 3 - Headless 浏览器 | 必须点真实 DOM 才出链接 | 高 | Playwright |
| 🔴 4 - 终极破盾 | Cloudflare 5秒盾 | 极高 | DrissionPage |

**核心原则**：**一旦当前级别能搞定，立刻返回，绝不无故触发下一级**。这是为了最大化保护宿主机——Headless 浏览器一启动就吃 500MB+ 内存，跑 10 个 VPS 就可能把你服务器干爆。

---

## 第一级：纯静态正则（80% 的简单站靠这个）

### 适用场景

- 上古时代的论坛、下载站
- HTML 直接把磁力链接写在页面上
- 0 反爬措施，没有 Cloudflare、没有 JS 渲染

典型代表：`dygod.net`（电影天堂）、各种老 PT 论坛

### 技术栈

```python
import requests
import re
from bs4 import BeautifulSoup

HEADERS = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
}

MAGNET_RE = re.compile(r'magnet:\?xt=urn:btih:[a-zA-Z0-9]+', re.I)
FTP_RE = re.compile(r'ftp://[^\s"\'<>]+', re.I)
```

### 实战代码

```python
def scrape_level1(search_url, headers=None):
    """级别1：纯静态正则"""
    h = headers or HEADERS
    resp = requests.get(search_url, headers=h, timeout=15)

    # 老旧网站常用 GB2312 编码，必须显式声明
    if 'gb2312' in (resp.apparent_encoding or '').lower():
        resp.encoding = 'gb2312'

    html = resp.text

    # 一次性提取所有磁力和 FTP
    magnets = set(MAGNET_RE.findall(html))
    ftps = set(FTP_RE.findall(html))

    return list(magnets) + list(ftps)
```

### 实战经验

1. **绕开主页**：不要在主页白费力气。先用站内搜索或 Google `site:xxx 关键字` 定位到具体的**详情页**，详情页的链接密度更高
2. **编码处理**：老站常用 GB2312，不显式声明会乱码
3. **去重**：同一页面可能有重复链接，用 `set` 去重
4. **资源消耗**：全程不到 50MB 内存，宿主机毫无压力

### 提取元数据（重要！）

**不要只返回磁力链接！** 用户需要知道这个链接是什么资源。提取这些信息：

- 文件大小（Size）
- 清晰度（1080p / 4K / 720p）
- 文件格式（MKV / MP4 / TS）
- 音轨字幕信息
- 版本说明（Remux / Web-DL / BDRip）

```python
import re

def extract_metadata(context_text):
    """从磁力链接的上下文文本提取元数据"""
    metadata = {}

    # 清晰度
    if res := re.search(r'\b(2160p|1080p|720p|4K)\b', context_text, re.I):
        metadata['resolution'] = res.group(1)

    # 格式
    if fmt := re.search(r'\b(BluRay|WEB-DL|HDTV|DVDRip|Remux|BDRip)\b', context_text, re.I):
        metadata['source'] = fmt.group(1)

    # 文件大小（GB/MB）
    if size := re.search(r'(\d+\.?\d*)\s*(GB|MB)', context_text, re.I):
        metadata['size'] = f"{size.group(1)} {size.group(2)}"

    return metadata
```

拿到磁力后，在 HTML 里找它的父节点或相邻文本节点，把这些信息一起返回给用户。

---

## 第二级：复杂静态解析

### 适用场景

- 页面返回 200 OK，但磁力链接被 **Base64 编码**藏在 JS 变量里
- 链接放在隐藏的 `<input>` 或 `data-*` 属性中
- 需要简单的 Token 拼接

### 实战代码

```python
import base64

def scrape_level2(html):
    """级别2：从 JS 变量和隐藏 input 提取"""
    magnets = []

    # 1. 从 <input> value 提取
    soup = BeautifulSoup(html, 'html.parser')
    for inp in soup.find_all('input', type='hidden'):
        value = inp.get('value', '')
        if 'magnet:?' in value:
            magnets.append(value)

    # 2. 从 data-* 属性提取
    for elem in soup.find_all(attrs={'data-magnet': True}):
        magnets.append(elem['data-magnet'])

    # 3. 从 JS 变量 Base64 解码提取
    b64_pattern = re.compile(r'["\']([A-Za-z0-9+/=]{40,})["\']')
    for match in b64_pattern.findall(html):
        try:
            decoded = base64.b64decode(match).decode('utf-8', errors='ignore')
            if 'magnet:?' in decoded:
                # 再用级别1的正则提取
                magnets.extend(MAGNET_RE.findall(decoded))
        except Exception:
            continue

    return list(set(magnets))
```

这个级别解决 60% 的"看起来很难其实静态"的网站。

---

## 第三级：Headless 浏览器（高开销，按需用）

### 适用场景

- 必须**点击真实 DOM 节点**（如下载按钮）才会触发 XHR 请求返回链接
- 网站用了 React/Vue 渲染，纯 HTML 看不到内容

### 技术栈

Playwright（推荐）或 Puppeteer。

### 实战代码

```python
from playwright.sync_api import sync_playwright

def scrape_level3(url):
    """级别3：Headless 浏览器模拟点击"""
    magnets = []

    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        try:
            page = browser.new_page()
            page.goto(url, timeout=30000)
            page.wait_for_load_state('networkidle')

            # 模拟点击下载按钮
            page.click('a.download-btn', timeout=5000)
            page.wait_for_timeout(2000)  # 等 XHR 回来

            # 提取渲染后的 HTML
            html = page.content()
            magnets = MAGNET_RE.findall(html)

        finally:
            browser.close()  # 🔥 关键：必须 close，否则内存泄露

    return magnets
```

### 护栏铁律

**严禁并发！** 一个浏览器实例吃 500MB+ 内存，开 5 个并发 VPS 直接 OOM。

```python
# ❌ 错误写法
async def scrape_many(urls):
    tasks = [scrape(url) for url in urls]
    await asyncio.gather(*tasks)  # 并发 5 个 Playwright 直接 OOM

# ✅ 正确写法
def scrape_many(urls):
    for url in urls:
        result = scrape_level3(url)
        # 串行执行，每个跑完立刻销毁
```

---

## 第四级：DrissionPage 终极破盾（核武器）

### 适用场景

- Cloudflare "Just a moment" 5 秒盾
- 521 强力 Anti-Bot 拦截
- eztv.ag、therarbg.com 等重度 CF 站

### 技术栈

DrissionPage（基于 Chromium，能自动处理 CF 挑战）。

### 实战代码

```python
from DrissionPage import ChromiumPage, ChromiumOptions

def scrape_level4(url):
    """级别4：DrissionPage 破 CF 盾"""
    magnets = []

    options = ChromiumOptions()
    options.headless(True)

    page = None
    try:
        page = ChromiumPage(options)
        page.get(url, retry=3, timeout=30)

        # CF 盾通常需要 5-8 秒等待
        page.wait.load_start()

        # 等待页面正常加载（CF 挑战完成后页面会有变化）
        for _ in range(30):
            if 'just-a-moment' not in page.html.lower():
                break
            page.wait(1)

        magnets = MAGNET_RE.findall(page.html)
    finally:
        if page:
            page.quit()  # 🔥 关键：销毁 Chromium 内核进程

    return magnets
```

### 护栏铁律

DrissionPage 启动的 Chromium 比 Playwright 更重，**必须在 finally 中强制 quit**，否则后台残留进程吃满内存。

---

## 实战决策树

面对一个新站，按这个顺序试：

```
新站 → 用级别1试
  ↓ 成功？搞定
  ↓ 失败
  → 用级别2试
    ↓ 成功？搞定
    ↓ 失败
    → 用级别3试
      ↓ 成功？搞定
      ↓ 失败
      → 用级别4（最后手段）
        ↓ 失败？放弃，换别的站
```

**关键**：每一级都用 `try/except`，失败了自动降级下一级；不要一上来就 Headless 浏览器，杀鸡用牛刀。

---

## 资源保护红线

不管用哪一级，**一定要遵守**：

1. **严禁并发**：Headless 浏览器一启动就吃 500MB+，并发 3 个就可能 OOM
2. **强制资源释放**：`try...finally` 块中执行 `browser.close()` 或 `page.quit()`
3. **超时必须设**：每个 HTTP 请求必须 `timeout=15`，否则卡死会拖垮整个流程
4. **频率限制**：两次请求之间加 `time.sleep(1-3)`，避免被目标站拉黑
5. **优先静态**：能静态正则搞定的绝不用浏览器

---

## 站梯队清单（实测）

### 🏆 梯队一：影视动漫资源站

| 站点 | 难度 | 推荐级别 |
|---|---|---|
| `dygod.net` | ⭐ 极其简单 | 级别 1 |
| `seedhub.cc` | ⭐⭐ 需要策略 | 级别 2 |
| `bttwoo.com` | ⭐⭐ 需要策略 | 级别 2 |
| `btbtla.com` | ❌ 强 CF | 级别 4 |

### 🥈 梯队二：磁力 Telegram 搜索引擎

| 站点 | 难度 | 推荐级别 |
|---|---|---|
| `thepiratebay.org` | ⭐⭐ | 级别 2-3 |
| `ciliku.net` | ⭐⭐ | 级别 2 |
| `eztv.ag` | ❌ 极强 Anti-Bot | 级别 4 |

---

## 一句话总结

**爬虫的精髓不是"用最牛的技术"，而是"用最低的成本搞定目标"**。

- 80% 的老站用静态正则就够
- 真碰到 JS 渲染再上 Headless
- CF 盾最后才动用 DrissionPage

按这个分级思路，**用最少的资源，跑最多的网站**。

如果你也在做爬虫，评论区聊聊你踩过的反爬坑～