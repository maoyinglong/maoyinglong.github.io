---
title: 飞书机器人突然不回复了？90% 是这个原因（gateway 排障实录）
date: 2026-06-20 19:30:00
tags:
  - 飞书
  - 机器人
  - 排障
  - 网关
categories:
  - 教程类
  - 开发工具
description: "飞书机器人突然不回复，90% 都是 gateway 进程挂了。这篇用一次真实排障还原了完整诊断链路，以后遇到同类问题直接对号入座。"
cover: /images/feishu-bot-connectivity-fix-cover.webp
---

![封面图](/images/feishu-bot-connectivity-fix-cover.webp)

> 飞书 Bot 跑得好好的，某次系统更新 / 容器重启后突然不回复了？开放平台"验证连接状态"还显示失败？别急着怀疑飞书平台，**90% 是本地 gateway 进程挂了**。本文记录完整的诊断和恢复流程。

---

## 现象

- 给 bot 发消息，没任何回复
- 飞书开放平台"验证连接状态"按钮变红、显示失败
- 日志里出现 `RuntimeError: Executor shutdown has been called`

---

## 根因（一句话说清）

飞书 Bot 通常是用 **WebSocket 长连接**保持在线的（不用 webhook 回调，免公网 IP）。这个长连接进程一般叫 `gateway`。

当 gateway 进程被 SIGTERM 杀掉后，asyncio 的线程池 executor 进入关闭状态。这时候即使 WebSocket 重连成功、消息也收到了，**回复消息时调用线程池的代码会直接抛 `Executor shutdown`，导致消息发不出去**。

表现就是：

- 飞书后台显示"连接异常"或"验证失败"
- 实际 bot 能收到消息，但发不出去
- 用户以为 bot 死了

---

## 完整诊断步骤

### 第一步：确认 gateway 进程是否存活

```bash
ps aux | grep -E 'hermes|gateway' | grep -v grep
```

**怎么看**：

- 看到 `hermes gateway run` 进程 → gateway 还活着，问题不在这里
- 只看到 `hermes chat` 之类的 CLI 进程，没有 gateway → **gateway 已挂**，继续下一步

### 第二步：看 gateway 日志

```bash
tail -50 ~/.hermes/logs/gateway.log
```

**关键日志模式**：

| 日志 | 含义 |
|---|---|
| `Executor shutdown has been called` | executor 崩了，消息发不出 |
| `Received SIGTERM — initiating shutdown` | gateway 被信号杀掉了 |
| `✓ feishu connected` | 飞书 WebSocket 连接正常 |
| 最后一行是 shutdown，没有新启动记录 | gateway 已停 |

### 第三步：检查配置（仅在 gateway 正常仍不回复时）

```bash
cat ~/.hermes/config.yaml | grep -A 10 feishu
```

确认几件事：

- `enabled: true`
- `connection_mode: websocket`
- 没有 `proxy:` 配置（**飞书必须直连**，代理会破坏 WebSocket）
- `app_id` 和 `app_secret` 正确

---

## 解法

### 方案 A：重启 gateway（90% 的情况用这个）

```bash
# 后台启动
nohup hermes gateway run > ~/.hermes/logs/gateway.log 2>&1 &

# 等 5-10 秒看启动日志
tail -20 ~/.hermes/logs/gateway.log
```

看到这两行就是启动成功：

```
✓ feishu connected
Gateway running with 2 platform(s)
```

然后去飞书给 bot 发条消息测试。

### 方案 B：gateway 反复崩溃

检查是否有多个 gateway 进程冲突：

```bash
ps aux | grep 'hermes gateway' | grep -v grep
```

如果有多个，全杀掉再重启：

```bash
pkill -f 'hermes gateway'
sleep 2
nohup hermes gateway run > ~/.hermes/logs/gateway.log 2>&1 &
```

### 方案 C：一键恢复命令

```bash
# 杀掉残留进程 + 重启 + 验证
pkill -f 'hermes gateway' 2>/dev/null; sleep 2
nohup hermes gateway run > ~/.hermes/logs/gateway.log 2>&1 &
sleep 10 && tail -20 ~/.hermes/logs/gateway.log
```

---

## 几个常见误区

### 1. "验证连接状态"按钮失败 ≠ 真的失败

飞书开放平台的"验证连接状态"按钮，**点失败不用慌**。这是因为一些 SDK 库（比如 lark-oapi）实现细节问题，对平台侧的 PONG 探测不回复，属于设计如此。

**判断真正状态的标准**：看 gateway 日志有没有 `✓ feishu connected`，加上实际发条消息看 bot 能不能回复。

### 2. 飞书必须直连，不要走代理

`.feishu.cn` 和 `.larksuite.com` 域名**不要配代理**。代理引入的额外握手延迟会破坏 WebSocket 长连接。

**如果你用了全局代理**（如 Xray、Clash）：

- 环境变量加 `no_proxy=*.feishu.cn,*.larksuite.com`
- 或在 config.yaml 的 feishu 配置段明确不要 proxy

### 3. 别反复重启 `hermes chat`

`hermes chat` 是 CLI 客户端（你用来跟 AI 聊天的命令行界面），不是后台服务。重启它**对飞书连接毫无帮助**。

真正管飞书连接的是 `hermes gateway`。

### 4. 怀疑飞书平台前先自查

飞书服务端极少出问题。**99% 的情况是本地 gateway 进程或 executor 状态异常**。先按上面流程自查，确认本地没问题再去提工单。

---

## 防患于未然：如何避免 gateway 反复挂

### 1. 用 systemd 守护（推荐）

如果你在 Linux VPS 上跑：

```ini
# /etc/systemd/system/hermes-gateway.service
[Unit]
Description=Hermes Gateway
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/hermes gateway run
Restart=always
RestartSec=10
StandardOutput=append:/var/log/hermes-gateway.log
StandardError=append:/var/log/hermes-gateway.log

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable hermes-gateway
systemctl start hermes-gateway
```

这样 gateway 挂了 systemd 会自动拉起，不用人工干预。

### 2. 用 Docker 跑（更隔离）

```yaml
# docker-compose.yml
services:
  hermes-gateway:
    image: your-hermes-image
    command: hermes gateway run
    restart: unless-stopped
    volumes:
      - ./config:/root/.hermes/config.yaml
      - ./logs:/root/.hermes/logs
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

`restart: unless-stopped` + Docker 自带的日志轮转，比裸跑稳得多。

### 3. 加监控和告警

```bash
# 每 5 分钟检查一次 gateway 是否存活
*/5 * * * * pgrep -f 'hermes gateway' > /dev/null || \
  (nohup hermes gateway run > ~/.hermes/logs/gateway.log 2>&1 &)
```

或者用更高级的：写一个 watchdog 脚本，gateway 挂了自动重启 + 飞书消息通知你。

---

## 一句话总结

**飞书 bot 不回复 → 先看 gateway 日志 → 有 Executor shutdown 或进程不在 → 重启 gateway → 完事。**

如果你也踩过这个坑，或者有更高级的排查思路，评论区聊聊～

> 关联阅读：[飞书机器人开发踩坑实录：权限配置、群聊互@](/2026/06/20/feishu-bot-permissions-and-mention/)