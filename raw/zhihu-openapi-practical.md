---
title: "三步接入知乎API，Agent立刻会查资料：知乎比Google更适合做中文AI内容源"
date: 2026-07-27 10:30:00
evidence: "亲测"
tags:
  - 知乎
  - API
  - MCP
  - AI应用
categories:
  - 教程类
  - AI工程
description: "知乎开放平台API完整接入指南：站内搜索、全网搜索、直答大模型、热榜四大接口，从鉴权到OpenAI兼容调用，三步让Agent拥有中文内容检索能力。"
cover: /images/zhihu-openapi-practical-cover.webp
---

![封面图](/images/zhihu-openapi-practical-cover.webp)

我跟你讲，知乎开放平台是被严重低估的东西。

大多数人的印象还停在"知乎不就是个问答社区"，但它的开放平台藏了一套完整的 AI 内容基础设施——站内搜索、全网搜索、大模型问答、热榜数据。而且鉴权简单到离谱：一个 AccessSecret + 时间戳就搞定。

最让我意外的是：**直答 API 直接兼容 OpenAI 协议**。现有的 GPT/LangChain 工作流改一行 base_url 就能用知乎自家的模型。说真的，这东西接入成本是我用过最低的。

下面是抄作业时间。

## 我为什么需要这个？

做 AI 应用最头疼的就是内容源。

自己爬？反爬累死人。买数据？贵还不一定合规。接搜索引擎 API？Google/Bing 的中文支持一团糟。

知乎开放平台刚好卡在这个缝隙：中文高质量内容 + 合规 API + 几乎零成本。我跑通了三个场景：

- 给 AI Agent 装个"知乎大脑"——让 Agent 能搜知乎、引用回答
- 做舆情追踪——热榜 API 每分钟都有新数据
- 搭智能问答——直答 API 三个模型档位，从快到深全有

## 鉴权：一个 Secret，一个时间戳

注册开发者（https://developer.zhihu.com/profile），拿 AccessSecret。所有 API 统一 Bearer 鉴权：

```bash
curl -G 'https://developer.zhihu.com/api/v1/content/zhihu_search' \
  --data-urlencode 'Query=AI Agent' \
  -H 'Authorization: Bearer 你的AccessSecret' \
  -H "X-Request-Timestamp: $(date +%s)" \
  -H 'Content-Type: application/json'
```

两个必传 Header：`Authorization` 和 `X-Request-Timestamp`。

> 我踩的第一个坑：忘带 X-Request-Timestamp，直接被拒。文档里这一行位置不起眼，但它是必传的。

## 四大接口，各有用处

### 站内搜索

```
GET https://developer.zhihu.com/api/v1/content/zhihu_search?Query=关键词&Count=10
```

`Query` 必填不能空。`Count` 最大 10。返回字段很全：标题、内容类型、赞同数、评论数、作者、权威等级、排序分。

一个被低估的字段：**`AuthorityLevel`（权威等级）**。做严肃问答用它过滤，比只看赞同数靠谱。

### 全网搜索

```
GET https://developer.zhihu.com/api/v1/content/global_search?Query=关键词&Count=20
```

`Filter` 参数是灵魂。比如锁定特定网站 + 时间范围：

```
host=="example.com" AND publish_time>=1778494631
```

支持 `host`（域名）、`publish_time`（时间戳）、`AND`/`OR` 和括号。传入时注意 **URL 编码**。

隐藏限制：`host` 不支持搜 zhihu.com 及子域名——搜站内请用上一个接口。

### 知乎直答（兼容 OpenAI 协议！）

真香警告：

```python
import openai, time

client = openai.OpenAI(
    api_key="你的AccessSecret",
    base_url="https://developer.zhihu.com/v1",
    default_headers={"X-Request-Timestamp": str(int(time.time()))}
)

response = client.chat.completions.create(
    model="zhida-thinking-1p5",
    messages=[{"role": "user", "content": "什么是 RAG？"}]
)
# response.choices[0].message.reasoning_content  → 推理过程
# response.choices[0].message.content             → 最终回答
```

三个档位：

| 模型 | 特点 | 什么时候用 |
|---|---|---|
| `zhida-fast-1p5` | 快速回答 | 日常够用 |
| `zhida-thinking-1p5` | 深度思考，含推理过程 | 需要解释"为什么" |
| `zhida-agent` | 智能检索+回答 | 类似 RAG 模式 |

支持流式 SSE。前端可以直接渲染 `reasoning_content` 做"AI 在思考"动画。

> 【亲测】直答 MCP 用的是 Streamable HTTP，默认会阻塞等完整输出。需要流式增量或看推理过程，别走 MCP，直接调原生 completions API。

### 知乎热榜

```
GET https://developer.zhihu.com/api/v1/content/hot_list?Limit=30
```

返回 `Title`、`Url`、`ThumbnailUrl`、`Summary`。`ThumbnailUrl` 是免费封面图——做聚合产品直接用。

注意频率：别高频调用，会被限流（错误码 30001）。

## 别问我为啥知道，问就是踩过

### API、Skill、MCP 三种形式别乱选

| 形式 | 适合谁 | 坑 |
|---|---|---|
| RESTful API | 后端开发 | 最灵活 |
| Skill 包 (.zip) | 想开箱即用 | 定制空间有限 |
| MCP | Claude/Cursor/Cline | 直答 MCP 不支持流式 |

我选原生 API——自建 Agent 和工作流需要最大控制权。

### AccessSecret 别暴露在前端

所有 API 调用走后端做中转，前端只调自己的后端。把 AccessSecret 写在前端代码里，浏览器 DevTools 几秒钟就能看到。

### 三种接入形式的 MCP 端点各不同

站内搜索和热榜用 MCP over SSE（两个 URL：SSE + Message），直答用 Streamable HTTP（单一 stream 端点）。配 MCP 时注意区分。

## 效果：10 分钟搭一个热榜解读

把热榜 API + 直答 API 串起来：

```python
# 1. 拉热榜
hot = requests.get(
    "https://developer.zhihu.com/api/v1/content/hot_list",
    headers=HEADERS, params={"Limit": 10}
).json()

# 2. 直答生成百字解读
for item in hot["Data"]["Items"]:
    response = client.chat.completions.create(
        model="zhida-fast-1p5",
        messages=[{"role": "user", "content": f"100字解读：{item['Title']}"}],
        stream=False
    )
    print(f"🔥 {item['Title']}\n   {response.choices[0].message.content}\n")
    time.sleep(1)
```

10 分钟，"今日知乎热议 Top 10 解读"就出来了。

## 适合谁与不适合谁

**适合**：
- 做 AI 应用需要中文内容源的开发者
- 用 Claude/Cursor/Cline 想接入知乎搜索的
- 做舆情监控/内容聚合的

**不适合**：
- 想批量抓数据做训练的——API 有频率限制，不是干这个的
- 需要实时毫秒级响应的——不是设计目标
- 期望完全免费的——有额度限制，大量调用需要付费

> 知乎开放平台最良心的一点：API 设计规范、鉴权简单、直答兼容 OpenAI 协议。这三个加起来，接入体验吊打大多数大厂的开放平台。

> 「每天帮你踩一个 AI 的坑，省下一小时。」

你用什么东西给你 AI Agent 喂内容？评论区聊聊，说不定有更好的我还没试。

---

**关联阅读：**

- [AI Agent 到底是个啥？小白也能看懂的入门指南](/ai-agent-beginner-guide/)
- [我让AI管AI，结果它把数据库清空了](/ai-agent-governance-scale-os/)
- [我的 AI 助手疯了，30 分钟停不下来](/ai-agent-sunk-cost-trap/)
- [手把手教你部署 CLI Proxy API——统一管理所有 AI 大模型的神器](/cli-proxy-api-docker-deploy-guide/)
- [GPT-4读PDF错到离谱，这个工具却对了95%：PDF转Markdown，让所有AI工具都能读PDF](/mineru-pdf-parse-mcp/)
- [SkillOpt 在 AI Agent 中的硬核落地：如何让 Agent 自己长记性？](/skillopt-agent-self-evolution/)
- [腾讯元宝 API 提取实战：把它的非标接口包装成标准 OpenAI 兼容](/yuanbao-api-pure-proxy/)

