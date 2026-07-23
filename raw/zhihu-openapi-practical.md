---
title: 知乎开放平台 API 实战：用它搭一个内容聚合 / 智能体问答系统
date: 2026-06-20 16:00:00
tags:
  - 知乎
  - API
  - MCP
  - AI应用
categories:
  - 教程类
  - AI工程
description: "知乎开放平台 API 文档写得很散，实战起来还有很多隐藏坑。这篇系统整理了内容聚合和智能体问答场景下的完整接入姿势。"
cover: /images/zhihu-openapi-practical-cover.webp
---

![封面图](/images/zhihu-openapi-practical-cover.webp)

> 知乎官方开放平台提供了站内搜索、全网搜索、知乎直答（问答）、热榜四大类 API/Skill/MCP。这篇从实操角度，把每一类接口"怎么用、什么时候用、有什么坑"完整梳理一遍。

---

## 先说定位：知乎 API 适合什么场景

简单理解：知乎开放平台把它的内容检索、问答和热榜能力，以 API/MCP/Skill 三种形式开放出来。**不是给"刷数据"用的，而是给 AI 应用做"内容源"用的**。

常见用法：

| 场景 | 用哪个 API |
|---|---|
| 给 AI 助手加上"查知乎"的能力 | 直答 API / 全网搜索 |
| 做内容聚合（自动追踪话题）| 站内搜索 + 热榜 |
| 做舆情监控或趋势分析 | 热榜 + 全网搜索的 `filter` 参数 |
| 在 Claude / GPT / LangChain 工作流里嵌入 | MCP 协议（标准接口）|

---

## 第一步：拿到 Access Secret（鉴权）

### 注册和拿 key

到 https://developer.zhihu.com/profile 注册并创建应用。审核通过后（一般很快），在个人中心能看到 **`Access Secret`**。

### 调用格式

所有 API 统一用 Bearer 鉴权 + 时间戳防重放：

```bash
curl -G 'https://developer.zhihu.com/api/v1/content/zhihu_search' \
  --data-urlencode 'Query=怎么理解rave文化' \
  -H 'Authorization: Bearer 你的AccessSecret' \
  -H "X-Request-Timestamp: $(date +%s)" \
  -H 'Content-Type: application/json'
```

**两个必填 Header**：

- `Authorization: Bearer <你的AccessSecret>`
- `X-Request-Timestamp`: 秒级 Unix 时间戳（`date +%s`），**必须传当前时间**，服务端会校验时间偏差

**最容易踩的坑**：忘记带 `X-Request-Timestamp`，直接被拒。文档里这一行很多人没注意。

---

## 第二步：四大类接口的使用场景

### 1. 站内搜索（zhihu_search）

**什么时候用**：你想在 AI 应用里加"查知乎站内"的按钮或指令。

```bash
GET https://developer.zhihu.com/api/v1/content/zhihu_search
```

参数：

- `Query`（必填）：关键词
- `Count`（选填）：返回数量，默认 10，**最大 10**（多了服务端会截断）

返回字段：`Title`、`ContentType`（Article/Answer）、`ContentID`、`Url`、`VoteUpCount`、`CommentCount`、`AuthorName`、`AuthorAvatar`、`EditTime`、`AuthorityLevel`、`RankingScore`。

**实战注意**：

- 关键词必须是非空字符串，空字符串直接报错
- 想根据赞同数排序，自己拿到结果后做后处理；接口本身只按 `RankingScore` 排序（综合分）
- `AuthorityLevel`（权威等级）是个被低估的字段——做严肃问答时可以用它做过滤

### 2. 全网搜索（global_search）

**什么时候用**：你想搜全网（不只是知乎），但要中文优先、知乎生态过滤。

```bash
GET https://developer.zhihu.com/api/v1/content/global_search
```

参数：

- `Query`（必填）
- `Count`（选填）：默认 10，**最大 20**
- `SearchDB`：`all`（默认）/ `realtime`（实时）/ `static`（静态索引）
- `Filter`：高级语法筛选，**必须 URL 编码**

**`Filter` 是这个接口的灵魂**，比如：

```
host=="example.com" AND publish_time>=1778494631
```

支持 `host`（站点域名过滤）+ `publish_time`（时间过滤）+ `AND`/`OR` 逻辑。

**重要提示**：`host` 不支持搜索知乎站内（`zhihu.com` 及子域名会被排除），搜站内必须用第一个接口。

### 3. 知乎直答（zhida）- 问答大模型

**什么时候用**：你想让 AI 应用直接调用知乎自家的大模型（基于知乎内容训练）。

```bash
POST https://developer.zhihu.com/v1/chat/completions
```

**这个接口是 OpenAI 兼容的**，可以直接套 OpenAI SDK：

```python
import openai

client = openai.OpenAI(
    api_key="你的AccessSecret",
    base_url="https://developer.zhihu.com/v1",
    default_headers={"X-Request-Timestamp": str(int(time.time()))}
)

response = client.chat.completions.create(
    model="zhida-thinking-1p5",  # 深度思考模式
    messages=[{"role": "user", "content": "什么是 RAG？"}]
)
```

**模型档位**：

| 模型 | 特点 |
|---|---|
| `zhida-fast-1p5` | 快速回答，日常推荐 |
| `zhida-thinking-1p5` | 深度思考，输出包含 `reasoning_content`（推理过程） |
| `zhida-agent` | 智能检索 + 回答（类似 RAG 模式）|

**流式响应**：支持 SSE 格式，包含 `reasoning_content` 和 `content` 两段。前端可以分别渲染"思考过程"和"最终答案"。

### 4. 知乎热榜（hot_list）

**什么时候用**：做内容聚合、舆情监控、趋势追踪。

```bash
GET https://developer.zhihu.com/api/v1/content/hot_list?Limit=30
```

参数：`Limit`（选填），默认 30，最大 30。

返回结构非常清晰：

```json
{
  "Total": 30,
  "Items": [
    {
      "Title": "如何看待当前 AI Agent 的发展趋势？",
      "Url": "https://www.zhihu.com/question/123456789",
      "ThumbnailUrl": "...",
      "Summary": "..."
    }
  ]
}
```

**实战技巧**：

- 把热榜数据 + 直答 API 组合：自动生成"今日热议话题解读"
- 用 `ThumbnailUrl` 做封面图，比你自己爬知乎省事得多
- 注意频率限制，**别高频调用**，会被限流（错误码 30001）

---

## 第三步：选 API、Skill 还是 MCP？

知乎开放平台针对每个接口提供了**三种形式**：

| 形式 | 适合谁 | 优点 |
|---|---|---|
| **RESTful API** | 后端开发、自己写代码 | 最灵活 |
| **Skill 包**（.zip）| 想要开箱即用的 Agent 框架 | 预制好 Agent 调用逻辑 |
| **MCP** | Claude Desktop、Cursor 等支持 MCP 的工具 | 标准协议，接入即用 |

### 推荐用法

**如果你用 Claude Desktop / Cursor / Cline 等 AI IDE**：

直接配 MCP。比如直答 MCP 的配置：

```json
{
  "mcpServers": {
    "zhihu-zhida": {
      "url": "https://developer.zhihu.com/api/mcp/zhida/v1/stream",
      "headers": {
        "Authorization": "Bearer 你的AccessSecret"
      }
    }
  }
}
```

**注意**：直答 MCP 用了 Streamable HTTP 传输，单一 stream 端点处理所有 RPC 请求。它默认会阻塞等待完整响应，**如果你需要流式增量输出或看推理过程，建议直接调原生 completions API 而不是 MCP**。

**如果你自建 Agent / 工作流**：

直接调原生 API 更可控，特别是 `reasoning_content` 这种深度推理字段。

---

## 第四步：常见错误码

```json
{
  "code": 20001,
  "message": "鉴权失败"
}
```

| 错误码 | 说明 | 排查方向 |
|---|---|---|
| 0 | 成功 | - |
| 10001 | 参数错误 | 检查 query 字段是否为空、格式是否正确 |
| 20001 | 鉴权失败 | 检查 AccessSecret 是否过期、是否带 `Bearer` 前缀 |
| 30001 | 频率限制 | 降低调用频率、加退避 |

---

## 实战：一个简单的"知乎热榜 + 解读"工作流

把上面几个 API 串起来：

```python
import requests
import time
import openai

ACCESS_SECRET = "你的AccessSecret"
HEADERS = {
    "Authorization": f"Bearer {ACCESS_SECRET}",
    "X-Request-Timestamp": str(int(time.time())),
    "Content-Type": "application/json"
}

# 1. 拉热榜
hot = requests.get(
    "https://developer.zhihu.com/api/v1/content/hot_list",
    headers=HEADERS,
    params={"Limit": 10}
).json()

# 2. 用直答 API 给每个热点生成一段解读
client = openai.OpenAI(
    api_key=ACCESS_SECRET,
    base_url="https://developer.zhihu.com/v1",
    default_headers=HEADERS
)

for item in hot["Data"]["Items"]:
    response = client.chat.completions.create(
        model="zhida-fast-1p5",
        messages=[{
            "role": "user",
            "content": f"请用 100 字解读这个知乎热榜话题：{item['Title']}"
        }],
        stream=False
    )
    print(f"🔥 {item['Title']}")
    print(f"   {response.choices[0].message.content}\n")
    time.sleep(1)  # 避免频率限制
```

10 分钟就能搭一个"今日知乎热议 Top 10 解读"的脚本。

---

## 写在最后

知乎开放平台相比其他大厂的开放平台，**最良心的是 API 设计得规范、文档清晰、鉴权简单**。特别是直答 API 直接兼容 OpenAI 协议，做应用接入几乎零成本。

几个建议：

- **不要高频调用**：热榜 1 小时拉一次足够了，搜索按需调用
- **优先用站内搜索 API 而不是爬知乎**：合规、稳定、有官方支持
- **直答 API 的 `reasoning_content` 字段**：做"思考过程可视化"特别有用，很多前端直接渲染这个字段做"AI 在思考"的效果
- **AccessSecret 不要暴露在前端**：所有 API 调用走后端代理，前端只调自己的后端

有问题欢迎评论区交流，或者告诉我你打算用知乎 API 做什么项目～