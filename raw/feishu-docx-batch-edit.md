---
title: "飞书API能创建但不能批量删除，这是设计不是bug"
date: 2026-07-27 11:30:00
evidence: "亲测"
tags:
  - 飞书
  - Docx API
  - 踩坑
  - 开发
categories:
  - 教程类
  - 开发工具
description: "飞书Docx API的1770001 invalid param错误排障全记录：tenant_access_token批量删除被拒绝的根因、三种绕过方案。"
cover: /images/feishu-docx-batch-edit-cover.webp
---

![封面图](/images/feishu-docx-batch-edit-cover.webp)

两个小时。我花了两个小时才搞明白为什么飞书 Docx API 删不掉一段文字。

事情是这样的：我做内容自动化，需要把日报写入飞书文档。理想流程很简单——拉数据、清旧内容、写新内容。三个步骤，听起来 10 分钟搞定。

结果呢？批量删除直接报 `invalid param (1770001)`。单个删能成功，批量删就炸。更新 Block 也炸。文档上还没写这个限制。

下面是踩坑全记录和最终解法，直接抄。

## 现象：什么能跑、什么不行

我用 `tenant_access_token`（应用身份）测了一轮：

| 操作 | 结果 |
|---|---|
| 创建 Block | ✅ 成功 |
| 删除单个 Block（`start_index=N, end_index=N+1`）| ✅ 成功 |
| 批量删除（`start_index=4, end_index=14`）| ❌ `1770001` |
| 更新 Block | ❌ `invalid param` |

规律很清晰：**单个操作能过，批量操作直接拒绝**。

## 根因：应用身份的隐藏限制

文档上没明说，我推测是：

- `tenant_access_token` 是应用级身份，权限受限
- 批量删除涉及"破坏性操作 + 多 block 影响范围"，平台对应用身份做了风控
- 想搞复杂编辑，可能需要 `user_access_token`（用户身份 OAuth）

但获取 `user_access_token` 需要完整 OAuth 流程，做纯后台自动化的时候不现实。我先在 `tenant_access_token` 下找出路。

## 三种解法，从稳到快

### 解法一：逐块删除（最稳）

每次只删一个 block，从后往前删（避免索引变化），删完一个再删下一个：

```python
def delete_blocks_one_by_one(doc_token, parent_id, headers, start, end):
    current_end = end
    while current_end > start:
        # 每次重新拿最新 revision_id
        rev_resp = requests.get(
            f"https://open.feishu.cn/open-apis/docx/v1/documents/{doc_token}",
            headers=headers
        )
        rev = rev_resp.json()["data"]["document"]["revision_id"]

        del_resp = requests.delete(
            f"https://open.feishu.cn/open-apis/docx/v1/documents/{doc_token}"
            f"/blocks/{parent_id}/children/batch_delete",
            params={"document_revision_id": rev},
            json={"start_index": current_end - 1, "end_index": current_end},
            headers=headers
        )
        if del_resp.json().get("code") != 0:
            raise Exception(f"删除失败: {del_resp.json()}")
        current_end -= 1
```

缺点：删 N 个 block 要 N 次请求。但稳。

### 解法二：先删后插（推荐，效率高）

关键发现：**POST 创建 Block 支持批量，DELETE 不支持批量**。

所以最佳组合：用"逐个删除 + 批量创建"：

```python
def replace_content(doc_token, parent_id, headers, new_blocks):
    # 1. 获取最新 revision
    rev = get_revision(doc_token, headers)

    # 2. 从后往前逐个删
    block_count = get_block_count(doc_token, headers)
    for i in range(block_count - 1, 0, -1):
        rev = get_revision(doc_token, headers)
        requests.delete(
            f".../blocks/{parent_id}/children/batch_delete",
            params={"document_revision_id": rev},
            json={"start_index": i, "end_index": i + 1},
            headers=headers
        )

    # 3. 批量插入新内容
    requests.post(
        f".../blocks/{parent_id}/children",
        json={"children": new_blocks},
        headers=headers
    )
```

### 解法三：换 user_access_token（终极方案）

如果一定要批量删除，走 OAuth 拿用户身份 token。但这只适合用户产品（用户授权后代表自己操作），不适合纯后台自动化。

## 其他容易踩的坑

### revision_id 每次都在变

飞书文档每改一次，`revision_id` 就变。用老的删直接报"revision 不匹配"。

**铁律**：每次操作前先 GET 一次拿最新 revision。

### 索引动态变化

你删第 5 个 block 后，原本索引 6 的变成索引 5。所以**逐个删除必须从后往前**——这个要是搞反了，删到一半索引全乱。

### 文档有占位 block

新建飞书文档至少有一个占位 block。删除时**别把整个文档删空**，否则后续插入找不到 parent。

### Wiki 文档 ≠ Docx 文档

操作的如果是 Wiki 节点（`/wiki/...`），API 路径前缀是 `/wiki/v2`，不是 `/docx/v1`。这是两个不同的 API 体系，权限模型也不同。

## 最终可用代码（生产环境验证过）

```python
import requests

FEISHU_BASE = "https://open.feishu.cn/open-apis/docx/v1"

def clear_and_write_doc(doc_token, parent_id, tenant_token, new_blocks):
    headers = {
        "Authorization": f"Bearer {tenant_token}",
        "Content-Type": "application/json"
    }

    # 1. 获取文档信息
    doc_resp = requests.get(
        f"{FEISHU_BASE}/documents/{doc_token}", headers=headers
    ).json()
    rev = doc_resp["data"]["document"]["revision_id"]
    block_count = len(doc_resp["data"]["document"]["blocks"])

    # 2. 从后往前逐个删（保留索引 0 占位）
    if block_count > 1:
        for i in range(block_count - 1, 0, -1):
            rev = requests.get(
                f"{FEISHU_BASE}/documents/{doc_token}", headers=headers
            ).json()["data"]["document"]["revision_id"]
            del_resp = requests.delete(
                f"{FEISHU_BASE}/documents/{doc_token}"
                f"/blocks/{parent_id}/children/batch_delete",
                params={"document_revision_id": rev},
                json={"start_index": i, "end_index": i + 1},
                headers=headers
            ).json()
            if del_resp.get("code") != 0:
                print(f"删除第 {i} 个 block 失败: {del_resp}")
                break

    # 3. 批量创建新内容
    return requests.post(
        f"{FEISHU_BASE}/documents/{doc_token}/blocks/{parent_id}/children",
        headers=headers,
        json={"children": new_blocks}
    ).json()
```

## 适合谁与不适合谁

**适合**：
- 用飞书做自动化日报/周报/内容推送的
- 需要程序化编辑飞书文档的开发者

**不适合**：
- 只需要手动编辑飞书文档的——别折腾 API
- 需要高性能批量编辑的——逐块删除的 N 次请求模式不适合大批量场景

> 飞书 Docx API 的限制比想象的多，文档没明写，官方论坛也很少人讨论。但只要掌握"逐个删除 + 批量创建 + 从后往前"三板斧，自动化内容推送完全能跑通。

> 「每天帮你踩一个 AI 的坑，省下一小时。」

你做过飞书文档自动化吗？评论区聊聊踩过的坑。

---

**关联阅读：**

- [飞书机器人哑了？90%的人不知道查这个地方](/feishu-bot-connectivity-fix/)
- [飞书机器人开发踩坑实录：权限配置、群聊互@和那些我绕过的弯路](/feishu-bot-permissions-and-mention/)
- [飞书 Wiki 空间自动化：从权限获取到批量写入的 7 个实战坑](/feishu-wiki-automation/)

