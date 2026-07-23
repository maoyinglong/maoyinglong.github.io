---
title: OneDrive 全量备份踩坑实录：2.5GB 打包压缩到 312MB 的全过程
date: 2026-06-20 20:30:00
tags:
  - OneDrive
  - 备份
  - 优化
  - Python
categories:
  - 教程类
  - 运维
description: "2.5GB 数据怎么压到 312MB 再推 OneDrive？全过程踩坑记录：差异备份、压缩策略、断点续传，走一遍就懂了。"
cover: /images/onedrive-backup-optimization-cover.webp
---

![封面图](/images/onedrive-backup-optimization-cover.webp)

> 服务器上的脚本和数据越来越多，每天备份到 OneDrive 越来越慢，跨国上传经常超时断线。这篇记录我把 2.5GB 备份压缩到 312MB、传输从超时变成秒传的完整过程。

---

## 痛点：备份越来越慢，最后干脆失败

我的服务器每天凌晨自动跑全量备份到 OneDrive（用 Microsoft Graph API）。最开始一切正常，但随着时间推移：

- `/root/.hermes` 目录越来越大
- Python 虚拟环境、Node 依赖、系统快照、回收站……这些**备份根本不需要的东西**也一起被打包
- 打包文件膨胀到 2.5GB+
- 跨国直连上传，碰上大文件经常超时，被网关掐断
- 即使没超时，大分片（10MB+）一抖动就丢包，重试率高得离谱

最后索性直接失败。

---

## 第一步：分析为什么这么大

我先看了下备份内容到底是啥：

```bash
tar -tzf /tmp/last_backup.tar.gz | head -50
du -sh /root/.hermes/* | sort -hr | head -10
```

发现真正占空间的是这些"垃圾"：

| 目录 | 大小 | 是不是真的需要备份？ |
|---|---|---|
| `venv` | ~500 MB | ❌ 重新装就行 |
| `node_modules` | ~800 MB | ❌ 重新 `npm install` 就行 |
| `.hermes/node` | ~300 MB | ❌ 系统级二进制 |
| `.hermes/lsp` | ~150 MB | ❌ 语言服务器 |
| `.hermes/state-snapshots` | ~1 GB | ❌ 运行时快照 |
| `.hermes/trash` | ~50 MB | ❌ 回收站 |
| `.hermes/checkpoints` | ~200 MB | ❌ 历史 checkpoint |
| **真实业务数据** | **~300 MB** | ✅ 必须备份 |

结论：**真正需要备份的只有约 12%**，其他全是"环境 + 缓存 + 历史"。

---

## 第二步：手术刀式排除（核心优化）

在 tar 命令里精准加入 `--exclude` 参数：

```bash
tar -czf backup.tar.gz \
  --exclude='*venv*' \
  --exclude='*node_modules*' \
  --exclude='.hermes/node' \
  --exclude='.hermes/lsp' \
  --exclude='.hermes/checkpoints' \
  --exclude='.hermes/state-snapshots' \
  --exclude='.hermes/trash' \
  /root/.hermes /root/scripts
```

**实测效果**：

| 项目 | 优化前 | 优化后 | 缩减率 |
|---|---|---|---|
| 打包体积 | 2.5 GB | 312 MB | **88%** |
| 打包时间 | ~30 秒 | < 5 秒 | 83% |
| 上传耗时 | 超时失败 | 8 分钟 | ∞ |

体积直接打了 1 折，效果立竿见影。

**几个细节**：

1. `*venv*` 用通配符匹配各种虚拟环境（venv、.venv、myenv 等）
2. `--exclude` 可以写多次，每次一个模式
3. **注意 exclude 的顺序**：tar 是按命令行顺序匹配的，靠前的规则先生效
4. 排除 `state-snapshots` 后，**再加个清理逻辑**：保留最近 3 个快照，旧的删掉

---

## 第三步：解决上传超时

打包体积小了，但跨国上传还是不稳定。继续优化。

### 关键点 1：免代理直连

我服务器上配置了 Xray 代理（`127.0.0.1:10808`），Python 的 `requests` 库默认会读环境变量 `HTTP_PROXY`，**导致 OneDrive 走代理上传，反而更慢**。

解法：建一个专属的 Session，**关掉 trust_env**：

```python
import requests

direct_session = requests.Session()
direct_session.trust_env = False  # 不读系统代理环境变量
```

这样 OneDrive 的 Graph API 调用 100% 走物理直连。

### 关键点 2：5MB 黄金分片

OneDrive 上传大文件需要分片，原本我用的是 10MB 分片。改成 5MB 后：

- 单个分片小 → 抖动丢包的影响小
- 重试成本低 → 单分片重传只要几秒
- 并发友好 → 可以同时上传多个 5MB 分片

```python
CHUNK_SIZE = 5 * 1024 * 1024  # 5MB
```

### 关键点 3：重试 + 抖动退避

加上重试逻辑：

```python
import time

def upload_with_retry(session, url, data, max_retries=5):
    for attempt in range(max_retries):
        try:
            response = session.put(url, data=data)
            if response.status_code in (200, 201, 202):
                return response
        except requests.exceptions.RequestException as e:
            print(f"上传失败 (第 {attempt+1} 次): {e}")

        # 退避：每次重试前多等一会
        time.sleep(3 + attempt)

    raise Exception(f"上传失败，已重试 {max_retries} 次")
```

退避时间设置也很重要：

- 第 1 次重试：等 3 秒
- 第 2 次重试：等 4 秒
- 第 3 次重试：等 5 秒
- ...

避免重试风暴给服务端压力。

---

## 第四步：完整的优化脚本

下面是核心上传逻辑（简化版）：

```python
import os
import tarfile
import requests
import time

ONEDRIVE_BACKUP_PATH = "backups/full_backup"
CHUNK_SIZE = 5 * 1024 * 1024  # 5MB

def create_backup_tar(source_dirs, output_file):
    """打包 + 排除垃圾"""
    excludes = [
        "*venv*",
        "*node_modules*",
        ".hermes/node",
        ".hermes/lsp",
        ".hermes/checkpoints",
        ".hermes/state-snapshots",
        ".hermes/trash",
    ]

    with tarfile.open(output_file, "w:gz") as tar:
        for source in source_dirs:
            tar.add(source, arcname=os.path.basename(source),
                    exclude=lambda name: any(ex in name for ex in excludes))
    return output_file


def upload_to_onedrive(file_path, access_token):
    """分片上传到 OneDrive"""
    session = requests.Session()
    session.trust_env = False  # 关键：不读代理
    session.headers.update({
        "Authorization": f"Bearer {access_token}",
        "Content-Type": "application/octet-stream"
    })

    file_size = os.path.getsize(file_path)
    file_name = os.path.basename(file_path)

    # 1. 创建上传会话
    create_url = f"https://graph.microsoft.com/v1.0/me/drive/root:/{ONEDRIVE_BACKUP_PATH}/{file_name}:/createUploadSession"
    resp = session.post(create_url, json={"item": {"@microsoft.graph.conflictBehavior": "replace"}})
    upload_url = resp.json()["uploadUrl"]

    # 2. 分片上传
    with open(file_path, "rb") as f:
        offset = 0
        while offset < file_size:
            chunk = f.read(CHUNK_SIZE)
            end = offset + len(chunk) - 1
            headers = {
                "Content-Length": str(len(chunk)),
                "Content-Range": f"bytes {offset}-{end}/{file_size}"
            }

            for attempt in range(5):
                resp = session.put(upload_url, data=chunk, headers=headers)
                if resp.status_code in (200, 201, 202):
                    break
                print(f"分片 {offset}-{end} 重试 {attempt+1}/5")
                time.sleep(3 + attempt)
            else:
                raise Exception(f"分片 {offset}-{end} 上传失败")

            offset = end + 1
            print(f"已上传 {offset}/{file_size} bytes ({100*offset//file_size}%)")

    return True


# 主流程
tar_file = create_backup_tar(["/root/.hermes", "/root/scripts"], "/tmp/backup.tar.gz")
print(f"打包完成: {os.path.getsize(tar_file) / 1024 / 1024:.1f} MB")

upload_to_onedrive(tar_file, access_token="...")
print("✅ 上传成功")
```

---

## 实测结果

| 指标 | 优化前 | 优化后 |
|---|---|---|
| 打包体积 | 2.5 GB | **312 MB** |
| 打包时间 | 30 秒 | **< 5 秒** |
| 上传耗时 | 超时失败 | **8 分钟** |
| 重试次数 | 50+ | 0 |
| 成功率 | 60% | **100%** |

打包时间从 30 秒缩到 5 秒，传输成功率 100%。

---

## 一句话总结

**备份不只是"把东西打包传上去"，更重要的是"只备份值得备份的东西"**。先做排除规则（按目录/模式），再做传输优化（直连 + 小分片 + 重试），这两步做完一般都能把跨国备份从"经常失败"变成"稳定秒传"。

如果你也在搞自动化备份，评论区聊聊你踩过的坑～