---
title: 我的博客丑了三个月，换了个主题当场封神：Hexo博客换NexT主题全流程
date: 2026-07-22 20:30:00
tags:
  - Hexo
  - NexT
  - 博客
  - 教程
  - 小白向
categories:
  - 教程类
description: 从默认landscape到NexT主题，10分钟让Hexo博客颜值从30分飙到90分。侧边栏、暗色模式、代码复制、全文搜索、RSS——50行YAML全搞定，附3个必看踩坑实录。
cover: /images/hexo-next-theme-cover.webp
---

![封面图](/images/hexo-next-theme-cover.webp)

我的 Hexo 博客上线好几个月了，一直用默认的 landscape 主题。说实话，丑得我都不想打开第二眼——没有侧边栏、没有分页、文章列表一拉到底，旧文章永远沉在底部不见天日。

上周我终于忍不了了，花 10 分钟换了 **NexT 主题**，顺便把品牌定制也做了。现在打开博客，我自己都想多刷几遍。

下面是把整个过程拆成 8 步的完整教程，每一步都有命令，小白也能照抄。

---

## 第〇步：先搞清楚你现在的状况

首先，进入你的博客项目目录：

```bash
cd 你的博客目录
```

看一下当前用的是不是默认主题：

```bash
grep 'theme:' _config.yml
```

如果输出 `theme: landscape`，说明你跟我一样在用默认主题，这篇文章就是为你写的。

---

## 第一步：为什么选 NexT？

Hexo 生态里有几千个主题，我列三个最主流的选择：

| 主题 | GitHub 标星 | 特点 |
|------|------------|------|
| **NexT** | 3 万+ | 最老牌、文档最全、踩坑基本有前人填过 |
| Butterfly | 7 千+ | 国产热门，颜值高，配置门槛稍高 |
| Fluid | 9 千+ | Material Design 风格，简洁 |

我选 NexT 的理由很简单：**3 万 star 的社区冗余。** 一个主题用的人越多，你遇到的每个问题基本都能在搜索引擎里找到答案。这对小白来说就是最大的安全感。

---

## 第二步：安装 NexT

如果你是 pnpm 用户（和我一样），一行搞定：

```bash
pnpm add hexo-theme-next --save
```

如果是 npm 用户：

```bash
npm install hexo-theme-next --save
```

装好后，修改站点配置文件 `_config.yml`，把主题名换成 `next`：

```yaml
theme: next
```

就这么简单，现在运行 `hexo server` 看看效果——你的博客已经换皮了。

> 默认效果其实就能打 60 分，但我们要的是 90 分——不急，后面一步步调。

---

## 第三步：创建主题配置文件

**NexT 8.x 最核心的一个概念：不要在 `node_modules` 里直接改主题文件。**

为什么？因为 `node_modules` 里的一切改动用 `npm install` 或 `pnpm install` 后会被覆盖，你的定制全丢。

正确做法：在你的项目根目录新建一个文件叫 `_config.next.yml`，NexT 会自动读取这个文件来覆盖默认配置。

```bash
touch _config.next.yml
```

下面我把 `_config.next.yml` 里要写的内容逐项拆开讲。

---

## 第四步：基础配置（照抄就行）

打开 `_config.next.yml`，填入以下内容：

```yaml
# 布局方案：Gemini 是双栏宽侧边模式，信息密度高
scheme: Gemini

# 暗色模式：自动跟随系统
darkmode: true

# 菜单栏：你想让读者看到哪些入口
menu:
  home: / || fa fa-home
  archives: /archives/ || fa fa-archive
  tags: /tags/ || fa fa-tags
  categories: /categories/ || fa fa-th
  about: /about/ || fa fa-user

# 侧边栏：靠左放，文章页面自动展开
sidebar:
  position: left
  display: post

# 头像：圆形显示
avatar:
  url: /images/avatar.png
  rounded: true

# 阅读进度条：一眼看到阅读进度
reading_progress:
  enable: true
  color: "#4fc3f7"

# 代码块：圆角 + 一键复制按钮
codeblock:
  border_radius: 8
  copy_button:
    enable: true

# PJAX：浏览器无刷新跳转，体验丝滑
pjax: true

# 图片功能三件套
fancybox: true      # 点击放大
mediumzoom: true    # 缩放
lazyload: true      # 懒加载（图片多了不卡）

# 书签功能
bookmark:
  enable: true
  color: "#4fc3f7"
```

每一项我都加了注释说明它是干什么的。你现在只需复制粘贴，以后想调再回来看注释。

> 这一步走完，你的博客已经能打到 75 分了——有侧边栏、有进度条、代码块有了复制按钮。剩下的 15 分，我们留给"品牌定制"。

---

## 第五步：品牌定制——把博客变成"你的"

### 5.1 准备一张头像

找一张正方形的图片（推荐 512×512 以上），放进 `source/images/` 目录。

```bash
mkdir -p source/images
cp /你的图片路径/你的头像.png source/images/avatar.png
```

### 5.2 改站点信息

打开 `_config.yml`（注意，这个是站点主配置，不是主题配置），修改以下字段：

```yaml
title: 你的博客名
subtitle: 一句话介绍你的博客
description: 你的博客是干什么的（写清楚，搜索引擎要靠这个收录）
keywords: 关键词1,关键词2,关键词3
author: 你的名字
```

### 5.3 添加"关于"页面

因为菜单里配了 `about: /about/`，你必须建一个对应的页面，否则点进去是 404：

```bash
hexo new page about
```

这会在 `source/about/index.md` 生成一个文件。打开它，用第一人称写一段话告诉读者你是谁、这个博客写什么。

### 5.4 定制页脚

在 `_config.next.yml` 中加上页脚设置：

```yaml
footer:
  since: 2026
  icon:
    name: fa fa-snowflake
    color: "#4fc3f7"
  copyright: 你的名字 — 一句话介绍
```

---

## 第六步：装两个实用插件

### 6.1 本地搜索

让你的读者能在博客里搜关键词：

```bash
pnpm add hexo-generator-searchdb --save
```

然后在 `_config.yml` 中添加：

```yaml
search:
  path: search.xml
  field: post
  content: true
```

### 6.2 RSS 订阅

让你的博客能被 RSS 阅读器抓取：

```bash
pnpm add hexo-generator-feed --save
```

然后在 `_config.yml` 中添加：

```yaml
feed:
  enable: true
  type: atom
  path: atom.xml
  limit: 20
```

---

## 第七步：本地预览 + 部署上线

### 7.1 本地预览

```bash
hexo clean && hexo generate && hexo server
```

打开浏览器访问 `http://localhost:4000`，确认一切正常。

### 7.2 部署

```bash
git add -A
git commit -m "主题迁移：从 landscape 到 NexT"
git push
```

GitHub Actions 构建完成后，打开你的博客域名验证效果。

---

## 第八步（必看）：三个坑，踩进去再爬出来太疼了

### 坑一：pnpm 严格模式下 NexT 功能失灵

**现象**：`hexo generate` 报一堆 `Cannot find module 'hexo-util'` 错误，首页文章显示全文（摘要机制失效）。

**原因**：pnpm 默认不提升依赖（strict hoisting），NexT 的脚本找不到它依赖的模块。

**修复（必须做，不能跳过）**：

```bash
echo 'shamefully-hoist=true' > .npmrc
CI=true pnpm install
```

然后验证修复效果：

```bash
hexo generate 2>&1 | grep -c "Cannot find module"
```

输出必须为 `0`，否则说明还有问题。

### 坑二：Hexo 8 的摘要标签已经废了

Hexo 8.1.2 彻底移除了 `<!-- more -->` 和 `excerpt_length` 的摘要截断功能。

**唯一替代方案**：每篇文章的 front-matter 里手动加 `description` 字段：

```yaml
---
title: "文章标题"
description: "80-150 字的中文摘要，告诉读者这篇文章讲什么"
---
```

没有 `description` 的文章会显示全文——首页瞬间被长文淹没。

### 坑三：改完部署了，打开网页还是旧的？

**别慌，这是 CDN 缓存。**

你 curl 验证看到的可能是旧页面。正确的验证姿势：

```bash
# ❌ 错：可能拿到 CDN 缓存
curl -sL "https://你的域名/"

# ✅ 对：强制跳过缓存
curl -sL "https://你的域名/?nocache=$(date +%s)"
```

---

## 总结

走完上面 8 步，你的博客已经脱胎换骨：

- ✅ 有侧边栏、分页、暗色模式
- ✅ 代码块一键复制、图片点击放大
- ✅ 本地全文搜索
- ✅ RSS 订阅
- ✅ 品牌定制——标题、头像、页脚都是你自己的味道
- ✅ 3 个坑提前帮你填平了

整个过程不用写一行 CSS，不用改一行模板代码。NexT 把 90% 的事情都替你做好了，你只需要写一个 50 行的 YAML 文件。

> 你的博客不应该只是"能用"——它值得"好看"。
