---
title: EvoMap 白嫖 GitHub 仓库积分：1W 积分保姆级攻略（含一次失败修复）
date: 2026-06-20 14:30:00
tags:
  - GitHub
  - EvoMap
  - 白嫖
  - 开发工具
categories:
  - 教程类
  - 开发工具
description: "EvoMap 能把 GitHub PR 贡献换成真实积分，攻略很简单但有几个经典坑。这篇保姆级步骤 + 一次失败修复全记录，看完能直接抄作业。"
cover: /images/evomap-bounty-guide-cover.webp
---

> 最近发现一个叫 **EvoMap** 的活动，上传 GitHub 仓库就能换 API 积分，最高 1W。我跑通了一遍流程，踩了一个大坑，本文把我验证过的正确路径完整分享出来。

---

## EvoMap 是什么

简单说：它是一个面向开发者的 AI API 积分平台，最近在做拉新活动——你提交一个 GitHub 仓库 URL，声明你在里面的角色（owner / maintainer），系统按仓库 star 数给你发积分，最高 1W。

看起来很爽对吧？但规则细节很微妙，**解读错一个字就会被拒**。

---

## 规则精确解读（一次失败的教训）

EvoMap 资格验证原文：

> 上传你的公开 GitHub 仓库 URL 并声明角色，奖励按仓库 star 数确定。

被拒的原因原文：

> 仓库属于你的 GitHub 账号（owner，或在活动开始前对其有提交贡献）

拆开看其实就两条路径：

| 角色 | 通过条件 | 难度 |
|---|---|---|
| **Owner** | 仓库 owner 是你的 GitHub 账号 | 简单（用 fork 即可）|
| **Maintainer** | 在 EvoMap 活动开始**之前**就对仓库有 commit 历史（已被 merge 的 PR）| 困难 |

注意 Maintainer 那条要求有个**很坑的时间限制**：必须是活动**开始之前**的 commit 历史。活动期间提的 PR 不算。

---

## 我第一次是怎么失败的

我看到 NodeLoc 上有个教程用户说"过几天也就基本有了，然后就可以用这个维护者身份爽拿1W积分"。这句话的隐含意思是：他**在活动开始之前**就已经向某个开源仓库提过 PR 并被 merge 了。

我跟着操作，向 `timqian/chinese-independent-blogs`（23k⭐）提交了 PR，状态是 Open（还在等维护者合并），然后去 EvoMap 申请 maintainer 角色。

结果：**被拒**。

失败根因（双重否定）：

1. ❌ 仓库 owner 是 `timqian`，不是我的账号 → 不能选 owner
2. ❌ 我的 PR 是活动期间才提的，未 merge，活动前没 commit 历史 → 也不能选 maintainer

---

## 正确的三条路径

失败后我梳理了三条可行方案：

| 方案 | 操作 | 优势 | 劣势 |
|---|---|---|---|
| **A ⭐推荐** | 改用 fork 仓库 + 选 owner | 100% 通过 | fork 通常 0⭐，只能拿保底积分 |
| **B** | 等 PR 被 timqian merge → 重提上游 + maintainer | 23k⭐ 满积分 | 依赖上游 merge，不可控 |
| **C** | 用其他 star 高的自有仓 | star 高 | 公开仓 star 一般都很低 |

**结论**：

- **想 100% 通过？** 选方案 A，立刻拿到保底积分
- **想拿满积分？** 选方案 B，但要等 merge，且 merge 之后还得再来 EvoMap 重新提交一次

我先用了方案 A（保险），方案 B 等 PR merge 之后再补一次。

---

## 实操步骤（方案 A）

### Step 1：Fork 目标仓库

到 https://github.com/timqian/chinese-independent-blogs 页面，点右上角 **Fork** 按钮。

Fork 完成后，你账号下会有一个 `你的账号/chinese-independent-blogs`，owner 是你。

### Step 2：准备你 Fork 的内容

可以小改一下 fork 仓库的内容（比如更新自己的博客条目），让 commit 历史活跃起来。**不做任何修改直接提交，EvoMap 可能会因为无 commit 记录而质疑"你是真的 owner 吗"**。

### Step 3：在 EvoMap 提交

填入：

- **仓库 URL**：`https://github.com/你的账号/chinese-independent-blogs`
- **角色**：Owner
- **声明**：例如 "这是我从上游 fork 的个人副本，我在我的博客中添加了条目"

提交后基本秒过。

---

## 实操步骤（方案 B，等 merge 后再补）

### 关键点：Fine-grained PAT 不行！

这一步特别容易踩坑：**跨 fork PR 必须用 Classic PAT，不能用 Fine-grained**。

| Token 类型 | 跨 fork PR | EvoMap 适用 |
|---|---|---|
| **Fine-grained PAT** | ❌（对非自己账号的公共仓库永远是 read-only）| ❌ |
| **Classic PAT + `public_repo` scope** | ✅ | ✅ |

Fine-grained PAT 不管你怎么勾选，对非自己账号的公共仓库权限永远是 read-only。**这是 GitHub 的安全设计，没有绕过办法**。

### 创建 Classic PAT 的步骤

1. 访问 https://github.com/settings/tokens/new
2. Note 随便写，比如 "EvoMap-PR"
3. Expiration 建议 90 天或自定义
4. 勾选 `public_repo` scope
5. 生成后**立即复制 token**（只显示一次）

### 用 API 创建 PR

```bash
curl -X POST \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer 你的ClassicPAT" \
  https://api.github.com/repos/timqian/chinese-independent-blogs/pulls \
  -d '{
    "title": "Add 你的博客 to 中文独立博客列表",
    "head": "你的账号:你的分支",
    "base": "master",
    "body": "添加我的独立博客..."
  }'
```

权限要求：

- 对 fork 仓库：`Contents: Write` + `Pull requests: Write`
- 对上游仓库：只需 `Pull` 权限（PR 是通过 fork 仓库发起的，不是直接修改上游）

---

## 注意事项

### 1. PR 提交后等多久被 merge？

热门仓库的 PR merge 周期：1-30 天不等。`chinese-independent-blogs` 这种社区仓库一般 1-3 天。

如果 PR 一直没人看，去项目 issues 区礼貌 ping 一下，或者在 PR 评论里 @ 维护者。

### 2. 重新提交的限制

EvoMap 没有明确的"重提交次数限制"，但也不建议刷——同一仓库多次提交可能被识别为滥用。

### 3. 多个仓库可以多次提交

理论上你可以用多个 fork 仓库分别提交，但 **EvoMap 似乎按"总 star 数"计算奖励**，所以分散到多个低 star 仓库不如集中到一个 fork。

---

## 总结：最优策略

如果你只是想保底拿积分：

1. Fork 任意一个你看上的开源仓库
2. 在 fork 里做点小修改（提交一个 commit）
3. 提交到 EvoMap，选 Owner
4. 拿到保底积分

如果你想最大化收益：

1. 选一个你熟悉的、star 数高的开源项目
2. 在活动**开始之前**就提 PR 并被 merge（这样后期才能用 maintainer 身份）
3. 维护贡献历史，积累到一定权重再申请

我已经用方案 A 拿到保底积分，等 PR merge 后再来方案 B 补一次。如果有进展我会在评论区更新。

---

*你也在玩 EvoMap 吗？欢迎评论区交流你的姿势～*