---
title: 飞书机器人开发踩坑实录：权限配置、群聊互@和那些我绕过的弯路
date: 2026-06-20 11:30:00
tags:
  - 飞书
  - 机器人
  - Bot开发
  - API
categories:
  - 教程类
  - 开发工具
description: "飞书机器人开发里最头疼的不是代码，是权限配置和群聊互 @ 那些玄学。整理了开发全程踩过的弯路，让你少折腾半天。"
cover: /images/feishu-bot-permissions-and-mention-cover.webp
---

![封面图](/images/feishu-bot-permissions-and-mention-cover.webp)

> 折腾飞书机器人，最大的坎不是写代码，是权限和事件订阅的迷宫。本文把我用过的一份「一键导入权限 JSON」和互@机制的坑整理出来，帮你少走 2-3 天弯路。

---

## 写在前面：为什么写这篇

如果你正在做飞书机器人 / Bot 开发，大概率已经踩过这些坑：

- 文档说"权限已开通"，但接口就是返回 `99991672: permission denied`
- 机器人在群里疯狂响应无关消息，被同事吐槽
- 多个机器人之间无法互相@唤醒，编排逻辑断了

这篇不写完整教程（官方文档有），只讲**实战中真正会遇到的关键配置**和 JSON 模板，复制就能用。

---

## 一、飞书 Bot 权限配置：一键导入 JSON

### 痛点

飞书开发者后台的权限管理页，默认只列出"高频权限"。想要用全功能（消息、文档、日历、任务等）必须一个个手动找，一个个勾。**一个应用配完经常要花半小时**，还容易漏。

### 解法：批量导入

在开发者后台 → **权限管理** → 顶部找到 **批量导入/导出权限** 按钮 → 切换到 **导入** Tab → 把下面这段 JSON 完整粘贴进去，覆盖默认示例：

```json
{
  "scopes": {
    "tenant": [
      "contact:contact.base:readonly",
      "docx:document:readonly",
      "im:chat:create",
      "im:chat:read",
      "im:chat:update",
      "im:message.group_at_msg:readonly",
      "im:message.p2p_msg:readonly",
      "im:message.pins:read",
      "im:message.pins:write_only",
      "im:message.reactions:read",
      "im:message.reactions:write_only",
      "im:message:readonly",
      "im:message:recall",
      "im:message:send_as_bot",
      "im:message:send_multi_users",
      "im:message:send_sys_msg",
      "im:message:update",
      "im:resource",
      "application:application:self_manage",
      "cardkit:card:write",
      "cardkit:card:read",
      "drive:drive.metadata:readonly",
      "docs:document.comment:create",
      "docs:document.comment:delete",
      "docs:document.comment:read",
      "docs:document.comment:update",
      "docs:document.comment:write_only",
      "docx:document:create",
      "docx:document:write_only",
      "docx:document.block:convert"
    ],
    "user": [
      "contact:user.employee_id:readonly",
      "offline_access",
      "base:app:copy",
      "base:field:create",
      "base:field:read",
      "base:field:update",
      "base:record:create",
      "base:record:retrieve",
      "base:record:update",
      "base:table:create",
      "base:table:read",
      "base:table:update",
      "base:view:read",
      "base:view:write_only",
      "base:app:create",
      "base:app:update",
      "base:app:read",
      "sheets:spreadsheet.meta:read",
      "sheets:spreadsheet:read",
      "sheets:spreadsheet:create",
      "sheets:spreadsheet:write_only",
      "docs:document:export",
      "docs:document.media:upload",
      "calendar:calendar:read",
      "calendar:calendar.event:create",
      "calendar:calendar.event:read",
      "calendar:calendar.event:update",
      "contact:contact.base:readonly",
      "contact:user.base:readonly",
      "contact:user:search",
      "docs:document.media:download",
      "drive:file:download",
      "drive:file:upload",
      "im:chat.members:read",
      "im:message",
      "im:message.group_msg:get_as_user",
      "im:message.p2p_msg:get_as_user",
      "im:message:readonly",
      "search:docs:read",
      "search:message",
      "space:document:move",
      "space:document:retrieve",
      "task:comment:read",
      "task:comment:write",
      "task:task:read",
      "task:task:write",
      "task:tasklist:read",
      "task:tasklist:write",
      "wiki:node:create",
      "wiki:node:read",
      "wiki:node:retrieve",
      "wiki:space:read",
      "wiki:space:retrieve",
      "wiki:space:write_only",
      "contact:user.basic_profile:readonly"
    ]
  }
}
```

**两个关键步骤**：

1. 粘贴后点击 **确认新增权限**
2. 一定要点击 **申请开通/发布应用**，否则权限不会真正生效（很多人栽在这步，配置完了还是报权限错）

### 容易踩的坑

- **部分权限需要审批**：比如 `contact:user.employee_id:readonly`（读取员工 ID）属于敏感权限，导入后还需要在 **权限管理 → 我的应用权限** 页面提交审批，管理员通过后才能用
- **scope 类型别搞混**：`tenant` 是租户级（应用身份调用），`user` 是用户级（代表用户操作）。同样是「读消息」，调用场景不同要选不同的 scope
- **写完代码才配权限**是反过来的。建议先配权限 → 写代码 → 联调，能少排查很多"明明逻辑没问题"的玄学问题

---

## 二、群聊互@与机器人响应控制

### 痛点：机器人疯狂响应无关消息

默认情况下，飞书机器人会监听群里的所有消息并尝试响应（如果你的逻辑没做好过滤）。结果就是——群里讨论点别的，它都要插一句"收到！"或者更糟，乱回复业务无关的内容。

### 解法：`requireMention` 配置

在飞书机器人适配的配置中，加一个开关：

```json
{
  "channels": {
    "feishu": {
      "enabled": true,
      "appId": "cli_你的AppID",
      "appSecret": "你的AppSecret",
      "requireMention": true,
      "groupPolicy": "open"
    }
  }
}
```

**两个关键参数**：

| 参数 | 取值 | 含义 |
|------|------|------|
| `requireMention` | `true` / `false` | `true` = 必须 @机器人才响应（推荐） |
| `groupPolicy` | `open` / `allowlist` | `open` = 所有群都响应；`allowlist` = 仅白名单群 |

**推荐组合**：

- 内部测试 / 个人玩具 Bot：`open` + `requireMention: true`
- 正式上线 / 多人协作：`allowlist` + `requireMention: true`（双保险，避免被恶意群加进去骚扰）

### 进阶：多机器人互@唤醒

这个场景很多人问：**两个机器人之间如何互相 @ 触发对方？**

实现逻辑其实不复杂：

1. **机器人 A** 在回复时，主动构造一个消息体，包含对 **机器人 B** 的 @ 标记
2. 飞书的消息体内 `@` 标记格式是：`<at user_id="ou_xxx">机器人B名字</at>`
3. **机器人 B** 配置了对应的 Webhook 事件订阅和回调逻辑，就会被触发
4. 关键是 **user_id** 要正确（`ou_` 开头的 open_id，可以在机器人 B 的事件回调中拿到）

注意这里有个细节：飞书的 `user_id`（`ou_`）和 `open_id`（`on_`）不是同一个东西，互@的时候需要用 `open_id`。

---

## 三、一些零碎但救命的小贴士

### 1. 调试阶段看实时事件流

飞书开放平台 → 你的应用 → **事件订阅** → **事件日志**：能看到所有 webhook 推送的原始数据，调试 Bot 行为最快的地方。

### 2. 本地调试不用每次部署

飞书的 webhook 回调 URL 必须公网可访问。本地开发推荐用 **frp** 或者 **Cloudflare Tunnel** 暴露 80/443 端口，然后填到 webhook URL 配置里。

### 3. Token 过期提前 5 分钟刷新

`tenant_access_token` 默认有效期 2 小时，但实际很多场景会提前失效。建议做一个定时器，每 1.5 小时主动刷新一次，比依赖"调用失败再重试"稳得多。

### 4. 卡片消息调试

调试飞书消息卡片时，先用纯文本消息测试逻辑，卡片模板放在最后做。卡片 JSON 写错的话，飞书返回的错误信息很模糊，定位起来很费劲。

---

## 写在最后

飞书 Bot 开发，**60% 的时间花在权限和事件配置上，真正写业务逻辑的时间反而不多**。把上面这套权限 JSON 收藏起来，新开应用的时候直接导入，能省下不少时间。

如果你是第一次做飞书机器人，建议按这个顺序推进：
1. 先开权限 → 发布应用
2. 跑通一个最简单的「收到消息回复 hello」
3. 加入 `requireMention` 控制
4. 接入卡片消息做交互
5. 最后才考虑多 Bot 编排

这样每一步都能跑通再往下走，不会陷入"写了一堆逻辑，不知道哪步出问题"的泥潭。

有问题欢迎评论区交流～