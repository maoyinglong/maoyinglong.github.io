---
title: 飞书 Wiki 空间自动化：从权限获取到批量写入的 7 个实战坑
date: 2026-06-20 22:30:00
tags:
  - 飞书
  - Wiki
  - API
  - 自动化
categories:
  - 教程类
  - 开发工具
description: "飞书 Wiki 自动化写入从权限申请到批量操作，中间有 7 个经典坑位几乎每个开发者都会踩。这篇全部写清楚，建议收藏备查。"
cover: /images/feishu-wiki-automation-cover.webp
---

![封面图](/images/feishu-wiki-automation-cover.webp)

> 想用 API 自动管理飞书 Wiki 空间？你会遇到一连串奇怪的权限和参数问题。本文从权限获取到内容写入，把每个坑的真实原因和解法完整整理出来。

---

## 为什么要自动化 Wiki 空间

飞书的 Wiki 空间是团队知识沉淀的核心。但手动维护很麻烦：

- 想批量创建文档目录
- 想自动同步外部数据到 Wiki
- 想把日报 / 监控结果自动生成 Wiki 页面

这些事情靠人工太累，必须用 API。但飞书 Wiki API 的"权限模型 + 参数格式"比普通 Docx API 复杂得多，**文档没写明，社区资料也少**。

我把整套流程摸通了，记录下来。

---

## 第一步：权限获取（90% 的人卡在这里）

### 误区：以为 API 权限 = 空间权限

很多人以为：在飞书开发者后台开通了 Wiki 相关 API 权限，就能直接操作 Wiki 空间。**错**。

飞书的 Wiki 空间是独立的权限体系：

- **应用权限**（开发者后台）：能不能调 API
- **空间成员权限**（Wiki 后台）：能不能操作这个空间

**两者缺一不可**。

### 正确流程

1. 联系 Wiki 空间管理员（通常是创建者）
2. 让管理员进入 Wiki 空间 → **设置 → 成员管理**
3. 把你的应用（app_id）添加为**空间管理员**或**编辑者**
4. 拿到这两个 ID：

```
空间 ID: 7632282247265029317  (纯数字)
首页 token: XBF5wGQs6iJmNykLRBScM7oAnyc  (字母开头)
```

### OAuth scope 的坑

如果你用 `user_access_token`（OAuth 授权）：

```python
# ❌ 错误：wiki + drive 同时请求会报 20043
scope = "wiki:wiki:read drive:drive:readonly"

# ✅ 正确：分开授权，先 wiki 后 drive
scope = "wiki:wiki:read"
# 第一次授权完，再发起一次 drive 的授权
```

飞书的设计是：单次 OAuth 不能请求跨产品的 scope。**需要分两次授权**。

---

## 第二步：节点创建（创建目录、文档）

### 创建目录（根节点）

```bash
POST https://open.feishu.cn/open-apis/wiki/v2/spaces/{space_id}/nodes
```

请求体：

```json
{
  "obj_type": "docx",
  "node_type": "origin",
  "title": "我的新目录"
}
```

返回 `node_token`，记下来用于后续操作。

### 创建文档（踩坑点）

很多人尝试在指定目录里直接创建文档：

```bash
POST .../wiki/v2/spaces/{space_id}/nodes
Body: {
  "obj_type": "docx",
  "parent_node_token": "父目录token",
  "title": "新文档"
}
```

**报错：权限不足**。

原因：**创建子节点需要父节点的编辑权限**，而你可能只有目录的查看权限。

### 变通方案

**先创建到 space 根目录，再移动到目标目录**：

```python
import requests

headers = {"Authorization": f"Bearer {tenant_token}"}

# Step 1: 在根目录创建
resp = requests.post(
    f"https://open.feishu.cn/open-apis/wiki/v2/spaces/{space_id}/nodes",
    headers=headers,
    json={"obj_type": "docx", "node_type": "origin", "title": "新文档"}
)
new_node_token = resp.json()["data"]["node"]["node_token"]

# Step 2: 移动到目标目录
resp = requests.post(
    f"https://open.feishu.cn/open-apis/wiki/v2/spaces/{space_id}/nodes/{new_node_token}/move",
    headers=headers,
    json={
        "target_space_id": space_id,
        "target_parent_token": "目标目录的node_token"
    }
)
```

---

## 第三步：节点移动（又一个踩坑点）

```bash
POST /wiki/v2/spaces/{space_id}/nodes/{node_token}/move
```

### 第一个坑：参数位置

```json
// ❌ 错误：参数放 query string 会报 131002
?target_space_id=xxx&target_parent_token=yyy

// ✅ 正确：参数必须放 JSON body
{
  "target_space_id": "xxx",
  "target_parent_token": "yyy"
}
```

这个错误信息很模糊，很多人调试半天。

### 第二个坑：node_token vs obj_token

Wiki API 和 Docx API 用的是**不同的 token**：

| Token 类型 | 用途 | 示例 |
|---|---|---|
| `node_token` | Wiki API（移动、获取节点信息） | `XBF5wGQs6iJmNykLRBScM7oAnyc` |
| `obj_token` | Docx API（读写文档内容） | 不同！要通过 node API 转换 |

**写入文档时用 `obj_token`，不是 `node_token`**：

```python
# 先通过 Wiki API 获取 node 信息，里面有 obj_token
node_info = requests.get(
    f"https://open.feishu.cn/open-apis/wiki/v2/spaces/{space_id}/nodes/{node_token}",
    headers=headers
).json()
obj_token = node_info["data"]["node"]["obj_token"]

# 再用 obj_token 调 Docx API
requests.put(
    f"https://open.feishu.cn/open-apis/docx/v1/documents/{obj_token}/blocks",
    headers=headers,
    json={"items": [...]}
)
```

---

## 第四步：批量写入（限流处理）

### 限流错误

批量写内容时频繁遇到 `99992402` 错误。**这是飞书的限流保护**。

### 解决方案：分批 + 间隔

```python
import time

def batch_write_blocks(obj_token, blocks, headers, batch_size=5, interval=0.3):
    """分批写入文档内容"""
    for i in range(0, len(blocks), batch_size):
        batch = blocks[i:i + batch_size]
        resp = requests.put(
            f"https://open.feishu.cn/open-apis/docx/v1/documents/{obj_token}/blocks",
            headers=headers,
            json={"items": batch}
        ).json()

        if resp.get("code") != 0:
            raise Exception(f"写入失败: {resp}")

        time.sleep(interval)  # 关键：每批之间等 0.3 秒
```

**关键参数**：

- `batch_size = 5`：每批最多 5 个 block（实测稳定值）
- `interval = 0.3`：每批间隔 0.3 秒（保守值，避免 99992402）

---

## 第五步：特殊 Block 类型

飞书文档的 Block 类型比较多，常用的几个：

### 分割线

```json
{
  "block_type": 22,
  "divider": {}
}
```

**坑**：必须带 `divider: {}`，不能省略（即使它是空对象）。

### 二级标题

```json
{
  "block_type": 3,
  "heading2": {
    "elements": [{"text_run": {"content": "标题文字"}}],
    "style": {"bold": true}
  }
}
```

### 无序列表

```json
{
  "block_type": 12,
  "bullet": {
    "elements": [{"text_run": {"content": "列表项文字"}}]
  }
}
```

### 普通段落

```json
{
  "block_type": 2,
  "text": {
    "elements": [{"text_run": {"content": "段落文字"}}]
  }
}
```

---

## 实战封装

把所有步骤串起来，做成一个"创建 Wiki 文档 + 写入内容"的一站式函数：

```python
def create_wiki_doc_with_content(space_id, parent_node_token, title, content_blocks, headers):
    """在指定 Wiki 目录创建文档并写入内容"""
    # 1. 先在根目录创建
    resp = requests.post(
        f"https://open.feishu.cn/open-apis/wiki/v2/spaces/{space_id}/nodes",
        headers=headers,
        json={"obj_type": "docx", "node_type": "origin", "title": title}
    ).json()
    if resp.get("code") != 0:
        raise Exception(f"创建文档失败: {resp}")
    node_token = resp["data"]["node"]["node_token"]

    # 2. 移动到目标目录
    if parent_node_token:
        resp = requests.post(
            f"https://open.feishu.cn/open-apis/wiki/v2/spaces/{space_id}/nodes/{node_token}/move",
            headers=headers,
            json={
                "target_space_id": space_id,
                "target_parent_token": parent_node_token
            }
        ).json()
        if resp.get("code") != 0:
            raise Exception(f"移动文档失败: {resp}")

    # 3. 获取 obj_token
    node_info = requests.get(
        f"https://open.feishu.cn/open-apis/wiki/v2/spaces/{space_id}/nodes/{node_token}",
        headers=headers
    ).json()
    obj_token = node_info["data"]["node"]["obj_token"]

    # 4. 分批写入内容
    batch_write_blocks(obj_token, content_blocks, headers)

    return {
        "node_token": node_token,
        "obj_token": obj_token,
        "url": f"https://feishu.cn/wiki/{node_token}"
    }


# 用法：创建一份今日日报
content_blocks = [
    {"block_type": 3, "heading2": {"elements": [{"text_run": {"content": "今日日报"}}]}},
    {"block_type": 22, "divider": {}},
    {"block_type": 2, "text": {"elements": [{"text_run": {"content": "AI 行业有 3 条重要新闻..."}}]}},
    {"block_type": 12, "bullet": {"elements": [{"text_run": {"content": "新闻 1"}}]}},
    {"block_type": 12, "bullet": {"elements": [{"text_run": {"content": "新闻 2"}}]}},
]

result = create_wiki_doc_with_content(
    space_id="7632282247265029317",
    parent_node_token="目标目录的node_token",
    title="2026-06-20 每日情报",
    content_blocks=content_blocks,
    headers={"Authorization": "Bearer xxx"}
)
print(f"文档创建成功: {result['url']}")
```

---

## 写在最后

飞书 Wiki 自动化，**坑多在权限模型和参数格式**，业务逻辑本身并不复杂。几个建议：

1. **第一次配置时多花时间理清权限**：开发者后台 + 空间成员权限缺一不可
2. **善用"先创建再移动"模式**：避免子节点权限问题
3. **统一管理 token**：node_token 和 obj_token 别搞混，写一个工具函数专门做转换
4. **限流必须处理**：批量写一定要分批 + 间隔，否则 99992402 错误让你怀疑人生
5. **OAuth scope 分开请求**：跨产品权限不要一次性请求

如果你也在做飞书自动化，评论区交流一下你踩过的坑～

> 关联阅读：[飞书机器人开发踩坑实录](/2026/06/20/feishu-bot-permissions-and-mention/)、[飞书 Docx API 批量操作踩坑](/2026/06/20/feishu-docx-batch-edit/)