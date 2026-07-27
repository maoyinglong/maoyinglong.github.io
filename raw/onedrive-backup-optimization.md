---
title: 你花天价加硬盘，我零成本25TB OneDrive白养虾
date: 2026-07-27 10:00:00
evidence: "亲测"
tags:
  - OneDrive
  - E3
  - Microsoft Graph
  - 备份
  - Agent
categories:
  - 教程类
  - 运维
description: "微软E3开发者订阅白给25TB OneDrive，用Client Credentials Flow（免交互）两步让Agent自动备份接管。从注册应用到分片上传，每一步都能直接抄。"
cover: /images/onedrive-backup-optimization-cover.webp
---

![封面图](/images/onedrive-backup-optimization-cover.webp)

你给 VPS 扩容 100GB 多少钱？月付三四十块吧。

微软 E3 开发者订阅白给 25TB OneDrive。不花钱。

我让我的 Agent 每天凌晨自动打包、自动上传、自动接管这 25TB。已经跑了两个月，零故障。这篇把完整方案拆给你——从拿 Token 到 Agent 全自动备份，每一步都能直接抄。

## 为什么 Agent 需要一块自己的硬盘

Agent 跑久了，数据比你想象的多：cron 日志、会话快照、状态备份、脚本、配置文件。VPS 那几十 GB 的 SSD 塞满只是时间问题。

E3 开发者订阅你可能早就拿过了——注册微软开发者计划（Microsoft 365 Developer Program），送 25 个 E3 席位。每个账号 OneDrive 默认 1TB，管理员能调到 5TB。还不够？联系微软支持直接拉到 25TB。

> 25TB ≈ 300 个 80GB 的 VPS 系统盘。白给的。

关键问题是：怎么让 Agent 自动用起来？不能用那种需要浏览器弹窗的 OAuth——Agent 没人给它在浏览器里点"同意"。

这时候就需要 **Client Credentials Flow**。它是 Microsoft Graph API 专门给后台服务设计的鉴权方式：**不需要用户交互，不需要浏览器登录，纯靠 Client ID + Client Secret 直接拿 Token**。对 Agent 来说就是两个字符串的事。

## 第一步：注册一个后台应用（App-only）

登录 Azure Portal（E3 订阅附带的），进 Azure Active Directory → 应用注册 → 新注册。

- 名称随便填，比如 `AgentBackup`
- 账户类型选"仅此组织目录"
- 不需要重定向 URI

注册完后记下两个东西：
- **Application (Client) ID**：应用的身份证
- **Directory (Tenant) ID**：你 E3 组织的租户 ID

然后点"证书和密码" → 新建客户端密码 → 记下密码值（只显示一次）。这就是 **Client Secret**。

最后给权限：API 权限 → 添加权限 → Microsoft Graph → 应用程序权限 → 勾选 `Files.ReadWrite.All` → 管理员同意。

> Files.ReadWrite.All 是应用级权限，不是用户委托。这意味着这个应用能读写整个组织所有人的 OneDrive——但既然是自己的 E3 组织，读写自己的完全没问题。

## 第二步：Client Credentials Flow 拿 Token

三个参数发一个 POST，Token 就到手了：

```python
import requests

def get_graph_token(tenant_id, client_id, client_secret):
    resp = requests.post(
        f"https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token",
        data={
            "client_id": client_id,
            "client_secret": client_secret,
            "scope": "https://graph.microsoft.com/.default",
            "grant_type": "client_credentials"
        },
        timeout=20
    )
    return resp.json()["access_token"]
```

**全程没有任何弹窗、没有浏览器、不需要点"同意"**。Agent 自己就能调。

拿到 Token 后，Graph API 直接访问 OneDrive。完整路径是：

```
https://graph.microsoft.com/v1.0/users/{user_id}/drive/root:/
```

其中 `user_id` 是你 E3 账号的邮箱（如 `your-email@yourdomain.onmicrosoft.com`），也可以用 `/me` 简化。

## 第三步：Agent 自动备份脚本（抄作业）

下面是生产环境在用的完整逻辑——打包 + 瘦身 + 分片上传：

```python
import os
import time
import tarfile
import requests

CHUNK_SIZE = 5 * 1024 * 1024           # 5MB 黄金分片
TENANT_ID = "你的租户ID"
CLIENT_ID = "你的应用ID"
CLIENT_SECRET = "你的ClientSecret"
USER_ID = "你的E3邮箱"

# 直连 Session（不走中转，OneDrive 能直连）
direct = requests.Session()
direct.trust_env = False  # 不读系统中转环境变量

def get_token():
    resp = direct.post(
        f"https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/token",
        data={
            "client_id": CLIENT_ID,
            "client_secret": CLIENT_SECRET,
            "scope": "https://graph.microsoft.com/.default",
            "grant_type": "client_credentials"
        },
        timeout=20
    )
    return resp.json()["access_token"]

def agent_backup_to_onedrive():
    """Agent cron 调用的备份入口"""

    # 1. 打包 ← 精准排除垃圾
    tar_path = "/tmp/agent_backup.tar.gz"
    excludes = [
        "*venv*", "*node_modules*", ".agent/node", ".agent/lsp",
        ".agent/checkpoints", ".agent/state-snapshots", ".agent/trash"
    ]
    with tarfile.open(tar_path, "w:gz") as tar:
        for path in ["/root/.agent", "/root/scripts"]:
            tar.add(path, arcname=os.path.basename(path),
                    exclude=lambda n: any(ex in n for ex in excludes))

    size_mb = os.path.getsize(tar_path) / 1024 / 1024
    print(f"📦 打包完成: {size_mb:.1f} MB")

    # 2. 拿 Token
    token = get_token()

    # 3. 创建上传会话
    file_name = f"backup_{time.strftime('%Y%m%d_%H%M%S')}.tar.gz"
    session_url = f"https://graph.microsoft.com/v1.0/users/{USER_ID}/drive/root:/backups/{file_name}:/createUploadSession"
    resp = direct.post(session_url, headers={"Authorization": f"Bearer {token}"}, timeout=20)
    upload_url = resp.json()["uploadUrl"]

    # 4. 分片上传（5MB 分片 + 5 次重试 + 递增退避）
    file_size = os.path.getsize(tar_path)
    with open(tar_path, "rb") as f:
        offset = 0
        while offset < file_size:
            chunk = f.read(CHUNK_SIZE)
            end = offset + len(chunk) - 1
            headers = {"Content-Range": f"bytes {offset}-{end}/{file_size}"}
            for attempt in range(5):
                try:
                    r = direct.put(upload_url, headers=headers, data=chunk, timeout=60)
                    if r.status_code in (200, 201, 202):
                        break
                except requests.exceptions.RequestException:
                    pass
                time.sleep(3 + attempt)
            else:
                raise Exception(f"分片 {offset}-{end} 上传失败")
            offset = end + 1
            print(f"  ⬆ {offset}/{file_size} ({100*offset//file_size}%)")

    print(f"✅ OneDrive 备份完成: {file_name}")
    return file_name
```

把这个脚本挂到 Agent 的 cron 里，每天凌晨 4 点自动触发。Agent 自己打包、自己瘦身、自己上传。全程零人工。

## 效果：从 2.5GB 到 312MB，从频繁超时到零重试

排除规则生效前，打包文件 2.5GB，跨国上传经常在 60% 左右被网关掐断。排除后体积打了一折，传输从超时变秒传。

| 指标 | 优化前 | 优化后 |
|---|---|---|
| 打包体积 | 2.5 GB | **312 MB** |
| 打包时间 | ~30 秒 | **< 5 秒** |
| 上传耗时 | 超时失败 | **~8 分钟** |
| 单次重试次数 | 50+ | **0** |
| 成功率 | 60% | **100%** |

三个关键设计让这成为可能：

1. **Client Credentials Flow**：不要浏览器、不要用户交互——Agent 拿两个字符串就能读写 OneDrive 上的任何文件。这是整个自动化的基石。

2. **trust_env = False**：强制物理直连，绕开系统的中转通道。OneDrive 本来就对直连友好，走中转反而慢。

3. **5MB 分片 + 5 次重试**：小分片抖动影响小，重试成本低。10MB 分片一抖就丢半个，5MB 丢了一个秒补。

> 备份的精髓不是"把东西打包传上去"，而是"只备份值得备份的东西，用 Agent 能理解的方式连接"。Client Credentials Flow 是让 OneDrive 从"手动上传的网盘"变成"Agent 自带硬盘"的那把钥匙。

## 适合谁与不适合谁

**适合**：
- 手上有 E3 开发者订阅的——25TB 吃灰不如给 Agent 当私有云盘
- 跑 Agent / 自动化任务需要自动备份的
- 想要"设定一次、永久运行"的零交互方案

**不适合**：
- 只有个人版 OneDrive（5GB）的——方案依然能用，但空间扛不住 Agent 的日志量
- 网络环境无法直连微软服务的——Client Credentials 拿 Token 也需要连 `login.microsoftonline.com`
- 处理高度敏感数据的——虽然走直连，但数据存在微软服务器

> 「每天帮你踩一个 AI 的坑，省下一小时。」

你有 E3 开发者订阅吗？25TB 在用还是吃灰？评论区聊聊。
