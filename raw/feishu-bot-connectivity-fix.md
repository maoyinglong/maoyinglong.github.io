---
title: "飞书机器人哑了？90%的人不知道查这个地方"
date: 2026-07-27 12:00:00
evidence: "亲测"
tags:
  - 飞书
  - 机器人
  - 排障
  - 网关
categories:
  - 教程类
  - 开发工具
description: "飞书Bot突然不回复的完整诊断流程：gateway进程挂掉的根因（Executor shutdown）、三步诊断法、一键恢复脚本。"
cover: /images/feishu-bot-connectivity-fix-cover.webp
---

![封面图](/images/feishu-bot-connectivity-fix-cover.webp)

某天下午，我给飞书 Bot 发了条消息——石沉大海。

打开飞书开放平台，点"验证连接状态"——红色，失败。日志里躺着一条：`RuntimeError: Executor shutdown has been called`。

然后我花了 5 分钟定位、10 秒修好。这篇把完整的诊断链路写下来，下次你对号入座，不用从零排查。

## 根因一句话

飞书 Bot 用 **WebSocket 长连接** 保持在线的。这个长连接进程叫 `gateway`。当 gateway 被 SIGTERM 杀掉后，asyncio 的线程池进入关闭状态。这时候即使 WebSocket 重连成功、消息收到了，**回复时调用线程池直接抛 `Executor shutdown`，消息发不出去**。

表现就是：飞书后台显示连接异常，实际 Bot 能收到消息但发不出，你以为 Bot 死了。

## 三步诊断

### 第一步：看进程

```bash
ps aux | grep -E 'hermes|gateway' | grep -v grep
```

- 看到 `hermes gateway run` → gateway 还活着，问题不在这
- 没看到 → **gateway 已挂**，往下走

### 第二步：看日志

```bash
tail -50 ~/.hermes/logs/gateway.log
```

关键日志模式：

| 日志 | 含义 |
|---|---|
| `Executor shutdown has been called` | executor 崩了，消息发不出 |
| `Received SIGTERM` | gateway 被信号杀掉了 |
| `✓ feishu connected` | WebSocket 连接正常 |
| 最后一行是 shutdown，没有新启动 | gateway 已停 |

### 第三步：看配置（仅在前两步正常但不回复时）

```bash
cat ~/.hermes/config.yaml | grep -A 10 feishu
```

确认：`enabled: true`、`connection_mode: websocket`、没有 `proxy:` 配置（**飞书必须直连**，走中转会破坏 WebSocket）、`app_id` 和 `app_secret` 正确。

## 解法

### 90% 的情况：重启 gateway

```bash
nohup hermes gateway run > ~/.hermes/logs/gateway.log 2>&1 &
tail -20 ~/.hermes/logs/gateway.log
```

看到这两行就是活了：

```
✓ feishu connected
Gateway running with 2 platform(s)
```

### gateway 反复崩溃

可能有多个进程冲突：

```bash
pkill -f 'hermes gateway'
sleep 2
nohup hermes gateway run > ~/.hermes/logs/gateway.log 2>&1 &
```

### 一键恢复

```bash
pkill -f 'hermes gateway' 2>/dev/null; sleep 2
nohup hermes gateway run > ~/.hermes/logs/gateway.log 2>&1 &
sleep 10 && tail -20 ~/.hermes/logs/gateway.log
```

## 几个常见误区

### "验证连接状态"按钮失败 ≠ 真的失败

飞书开放平台的"验证连接状态"按钮点失败不用慌。一些 SDK 库对平台侧的 PONG 探测不回复，属于设计如此。**判断标准**：看 gateway 日志有没有 `✓ feishu connected`，加实际发条消息看能不能回。

### 飞书必须直连

`.feishu.cn` 和 `.larksuite.com` **不要走中转**。中转引入的额外握手延迟会破坏 WebSocket 长连接。如果你有全局中转（如 Clash），环境变量加 `no_proxy=*.feishu.cn,*.larksuite.com`。

### 别重启 hermes chat

`hermes chat` 是 CLI 客户端，不是后台服务。重启它**对飞书连接毫无帮助**。真正管飞书连接的是 `hermes gateway`。

> 飞书服务端极少出问题。99% 是本地 gateway 进程或 executor 状态异常。

## 防患于未然

### systemd 守护

```ini
[Unit]
Description=Hermes Gateway
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/hermes gateway run
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload && systemctl enable --now hermes-gateway
```

### Cron 兜底

```bash
*/5 * * * * pgrep -f 'hermes gateway' > /dev/null || \
  (nohup hermes gateway run > ~/.hermes/logs/gateway.log 2>&1 &)
```

## 适合谁与不适合谁

**适合**：用飞书 Bot 的开发者，尤其是自部署 Agent 网关的。

**不适合**：用飞书官方托管 Bot 的——那个 gateway 不用你管。

> 「每天帮你踩一个 AI 的坑，省下一小时。」

你的飞书 Bot 挂过吗？是什么原因？评论区聊聊。

---

**关联阅读：**

- [飞书机器人开发踩坑实录：权限配置、群聊互@和那些我绕过的弯路](/feishu-bot-permissions-and-mention/)
- [飞书API能创建但不能批量删除，这是设计不是bug](/feishu-docx-batch-edit/)
- [飞书 Wiki 空间自动化：从权限获取到批量写入的 7 个实战坑](/feishu-wiki-automation/)

