---
title: "让原生支持Agent操作的云笔记跑在Cloudflare上"
date: 2026-08-22 11:30:00
tags:
  - EdgeEver
  - Cloudflare
  - 自托管
  - MCP
  - 知识库
categories:
  - 教程类
  - 开源
  - 云服务
description: "笔记能导出，不等于真正属于你；AI能读一篇，也不等于能长期安全地管理整座知识库。我的旧笔记服务开始频繁掉授权后，我没有把全部数据押给下一家商业平台，而是在Cloudflare上搭了一套开源笔记库：不用养服务器，398篇历史文档完成迁移，AI通过细粒度权限读写，旧平台继续保留为第二份副本。"
cover: /images/edgeever-cloudflare-agent-notes-cover.webp
evidence: 亲测
---

> **笔记能导出，不等于真正属于你；AI能读一篇，也不等于能长期安全地管理整座知识库。我的旧笔记服务开始频繁掉授权后，我没有把全部数据押给下一家商业平台，而是在Cloudflare上搭了一套开源笔记库：不用养服务器，398篇历史文档完成迁移，AI通过细粒度权限读写，旧平台继续保留为第二份副本。**

我没有因为某次登录失败，就愤怒地把旧笔记全部删掉。

那样不叫数据主权，只是把依赖从一家公司搬到另一家公司。

真正让我改变架构的，是三个问题反复出现：命令行上传偶发报错、授权会过期、数据虽然能看，却很难被AI稳定、自动、细粒度地使用。

我需要的不是“另一款漂亮笔记软件”。

我需要一座能被人正常编辑、能被AI调用、能完整导出、还能保留第二份副本的知识库。

最后留下来的方案叫EdgeEver。

【官方文档】https://github.com/tianma-if/edgeever

## 我真正想解决的，不是记笔记

多数笔记产品解决的是“把文字放进去”。

Agent时代还多了四个问题：

1. AI如何找到正确的笔记；
2. AI能不能只读某些内容，而不是拿到整个账户；
3. 自动写入失败后，能否验证到底写没写；
4. 平台消失时，数据是否还能以普通文件恢复。

EdgeEver吸引我的地方不是“像印象笔记”，而是它把这些能力放在同一套开源系统里：

- Cloudflare Workers运行接口；
- D1保存结构化数据；
- R2保存图片和附件；
- MCP让Agent按工具读写；
- ZIP导出中包含Markdown和附件关系；
- 同一套程序也能部署到Docker、NAS或VPS。

![AI原生私人知识库架构](/images/edgeever-cloudflare-agent-notes-cover.webp)

项目官方说可以运行在Cloudflare免费额度内。我的实例截至2026年8月22日仍能正常返回健康状态：

```json
{"ok":true,"name":"edgeever","runtime":"cloudflare-workers","authMode":"required"}
```

“零月租”不是绝对承诺。超过Cloudflare免费额度、使用外部对象存储或接入收费模型，仍可能产生费用。对我的个人知识库规模，目前没有单独购买服务器。

## 我没有做单点迁移，而是双写

这套架构最重要的决定，不是部署，而是**不把旧平台立刻废掉**。

我的写入顺序是：

```text
新知识
  ├─ EdgeEver：主写，供AI搜索与操作
  └─ 旧笔记平台：追加备份，保留第二条恢复路径
```

EdgeEver优先，旧平台兜底。两边不互相替代。

为什么要多此一举？

因为自托管也会失败：Cloudflare接口可能变化，新版本可能覆盖本地修补，迁移脚本可能漏项，自己也可能误删，开源项目同样可能停止维护。

> **数据主权不是“只信自己”，而是任何一边坏掉，都有退出路线。**

## 部署前，我先审了一遍安全边界

我不会因为项目开源就自动相信它。

部署前检查了认证、会话、限流和数据库调用。基于当时版本的代码审计结果：

| 检查项 | 当时实测结果 |
|---|---|
| 密码存储 | PBKDF2-SHA256，10万次迭代并加盐 |
| 会话Cookie | HttpOnly、Secure、SameSite |
| MCP令牌 | Bearer认证，只存哈希 |
| 权限 | 8种细粒度Scope |
| 登录限流 | 用户名与IP分别限流 |
| 数据库调用 | D1参数化查询 |

这是2026年7月29日对当时版本的代码审计，不代表以后每个版本都自动安全。升级仍然要复查关键变更。

## 先按官方路线部署：浏览器就能完成

如果你只是想先把 EdgeEver 跑起来，最稳的路线不是在服务器上拉代码，而是直接走 Cloudflare 的在线构建。全程只需要 GitHub 和 Cloudflare 两个账号。

### 第一步：Fork 项目

打开 EdgeEver 官方仓库：

```text
https://github.com/tianma-if/edgeever
```

点右上角 `Fork`，把项目复制到自己的 GitHub 账号下面。

这一步的意义很简单：Cloudflare 后面要从你的仓库读取代码并自动部署。不要直接改官方仓库，也不要把密码写进代码文件。

### 第二步：在 Cloudflare 创建 D1 数据库

进入 Cloudflare 控制台：

```text
Workers & Pages → D1 → Create database
```

数据库名字必须填：

```text
edgeever
```

D1 可以理解成 Cloudflare 版 SQLite。EdgeEver 的笔记标题、正文索引、用户、笔记本、标签、修订记录，都靠它保存。

### 第三步：创建 R2 存储桶

继续在 Cloudflare 控制台里进入：

```text
Workers & Pages → R2 → Create bucket
```

存储桶名字必须填：

```text
edgeever-resources
```

R2 用来放图片和附件。这里有一个容易忽略的点：R2 虽然有免费额度，但首次启用通常需要绑定付款方式。个人笔记规模一般很难打穿免费额度，但这一步 Cloudflare 不会因为你“只想免费用”就跳过。

### 第四步：导入 GitHub 仓库

进入：

```text
Workers & Pages → Overview → Create application → Import Git Repository
```

选择刚才 Fork 出来的 `edgeever` 仓库。

关键配置保持这样：

```text
Production branch: main
Root directory: 留空或 /
Deploy command: npx wrangler deploy
```

新版 EdgeEver 会通过仓库里的部署入口自动完成构建、D1 迁移、R2 绑定和 Worker 发布。不要手动改 `wrangler.toml`，也不要在 Cloudflare 面板里重复添加一堆绑定。这里越“勤快”，越容易把配置搞乱。

### 第五步：设置管理员密码

在 Cloudflare 项目的设置里进入：

```text
Settings → Variables and Secrets
```

添加一个 Worker Secret：

```text
Name: EDGE_EVER_AUTH_PASSWORD
Value: 你自己的强密码
```

注意，是 `Secret`，不是普通变量。密码不要写进 GitHub，不要写进 `wrangler.toml`，也不要贴进文章或截图里。

默认管理员用户名是：

```text
admin
```

如果要改用户名，需要在第一次构建前添加 Builds 变量：

```text
EDGE_EVER_AUTH_USERNAME=你的用户名
```

已经创建过管理员账号后，再改这个变量不会自动重命名旧账号。

### 第六步：启动第一次部署

保存配置后点 `Save and Deploy`。

第一次构建会做几件事：

```text
安装依赖
→ 构建前端
→ 构建 Worker API
→ 查找名为 edgeever 的 D1
→ 绑定名为 edgeever-resources 的 R2
→ 执行数据库迁移
→ 发布 Worker
→ 请求健康检查
```

成功后，Cloudflare 会给你一个默认访问地址，类似：

```text
https://edgeever.你的子域.workers.dev
```

打开健康检查地址：

```text
https://你的EdgeEver地址/api/health
```

正常会看到类似结果：

```json
{"ok":true}
```

再检查接口文档：

```text
https://你的EdgeEver地址/api/openapi.json
```

这两个地址都能打开，再回到首页，用 `admin` 和刚才设置的密码登录。

### 第七步：打开自动更新

Fork 仓库之后，GitHub 默认会关闭公开 Fork 的定时工作流。你需要手动打开一次。

进入你的 Fork：

```text
Actions → I understand my workflows, go ahead and enable them
```

找到这个工作流：

```text
Update deployed EdgeEver
```

手动点一次 `Run workflow`。

它的作用不是“再部署一个新实例”，而是让你的 Fork 后续能跟随 EdgeEver 官方稳定版本更新。以后官方发新版本时，它会把更新同步到你的部署仓库，再触发 Cloudflare 重新构建。

## 如果你想用自定义域名

默认 `workers.dev` 地址能用，但我更建议给知识库单独准备一个子域名，例如：

```text
notes.example.com
```

在 EdgeEver 的 Cloudflare 构建变量里加：

```text
EDGE_EVER_CUSTOM_DOMAIN=notes.example.com
```

或者用完整路由：

```text
EDGE_EVER_ROUTE_PATTERN=notes.example.com/*
```

这里有个坑：如果用 Worker 路由接管域名，通常不要再额外创建一个指向 Worker 的 CNAME。重复配置可能让 Cloudflare 不知道该把请求交给谁，最后浏览器看到的不是你的笔记，而是奇怪的 DNS 或路由错误。

部署完成后，用这三个地址验收：

```text
https://notes.example.com/api/health
https://notes.example.com/api/openapi.json
https://notes.example.com/mcp
```

前两个用于确认 Web 和 REST API 活着；`/mcp` 是给 Agent 用的入口。

## 我自己的部署路线：重活不放主服务器

我最初把编译放在主服务器上。很快发现这是错误路径：带宽慢、依赖重、下载失败会污染长期运行环境。

后来我把编译挪到家里的 NAS 节点，只在临时目录里完成构建：

```text
临时目录拉源码
→ 安装 Bun 依赖
→ 构建
→ Wrangler 部署到 Cloudflare
→ 验证 /api/health 和 /mcp
→ 删除临时构建目录
```

主服务器只负责运行 Agent，不承担重型前端编译。这个取舍很重要：主 Agent 节点越干净，后续排障越简单。

如果走命令行部署，核心环境变量大概长这样：

```text
CLOUDFLARE_ACCOUNT_ID=你的Cloudflare账户ID
EDGE_EVER_WORKER_NAME=edgeever
EDGE_EVER_WORKERS_DEV=true
EDGE_EVER_D1_DATABASE_NAME=edgeever
EDGE_EVER_D1_DATABASE_ID=你的D1数据库ID
EDGE_EVER_R2_BUCKET_NAME=edgeever-resources
EDGE_EVER_AUTH_USERNAME=admin
EDGE_EVER_AUTH_PASSWORD=你的强密码
EDGE_EVER_SESSION_TTL_DAYS=400
EDGE_EVER_CUSTOM_DOMAIN=notes.example.com
```

本地部署前先安装依赖：

```bash
bun install
```

再跑部署检查：

```bash
bun run deploy:doctor
```

构建 Cloudflare 版本：

```bash
bun run build:cloudflare
```

最后部署：

```bash
bun run deploy:ci
```

这里不要把真实 `database_id`、密码、Token 写进仓库里的 `wrangler.toml`。新版 EdgeEver 会拒绝这类实例化配置进入受版本管理的文件。正确做法是放在 Cloudflare 面板变量、Worker Secret，或者本机不提交的 `.env.local` 里。

真实遇到的坑包括：

### Cloudflare认证报错

接口返回：

```text
Unknown X-Auth-Key [code: 9103]
```

根因不是Cloudflare服务坏了，而是凭据写入过程中被错误转义和截断。修复后重新部署才通过。

### 浏览器登录显示“连接不到实例”

API健康检查正常，浏览器却登录失败。

最终定位到CORS：服务端只允许本地开发地址，没有加入正式域名。补上生产域名后重新部署，登录恢复。

### 56KB正文把命令行撑爆

直接把长正文拼进命令参数，触发：

```text
Argument list too long
```

后来改为把正文写入临时文件，再让请求工具读取文件内容。这个坑对长笔记迁移尤其常见。

### Web接口被防护规则拦截

普通会话路径被拦截后，我没有关闭整套防护，而是改用带Bearer令牌的MCP接口。认证边界更明确，也更适合自动化。

## 398篇历史笔记怎么搬

我按原知识库目录映射成18个笔记本，然后批量写入。

第一次结果不是“398篇全绿”，而是：

```text
387 / 398 成功
11 篇失败
```

这里最容易出现的假闭环，是看到大部分成功就宣布迁移完成。

我没有这么做。剩下11篇逐个定位，修复长正文传参、接口偶发失败和格式问题，再补同步。最终历史数据全部处理，没有把失败项留在日志里装作不存在。

迁移验收至少要做三层：

1. 数量对比：源文件与目标笔记数；
2. 标题对比：有没有漏项或重名覆盖；
3. 正文抽查：首尾、图片、代码块是否完整。

如果只是看到接口返回200，不叫迁移完成。

## AI如何操作这座知识库

我没有把管理员密码交给Agent，而是签发单独的MCP令牌，并按用途限制Scope。

典型动作包括搜索标题和正文、读取指定笔记、新建和追加内容、移动笔记本、管理标签与获取附件。

MCP 的配置也不复杂。登录 EdgeEver 后进入：

```text
Profile → MCP settings
```

创建一个新的 API Token。不要直接把管理员密码交给 Agent，给 Agent 的应该是单独令牌，而且按用途拆权限。

只读检索型 Agent，可以只给：

```text
读取笔记
搜索笔记
读取笔记本
读取标签
```

需要自动沉淀复盘的 Agent，再额外给：

```text
创建笔记
更新笔记
移动笔记
添加标签
```

Agent 侧只需要记住三个东西：

```text
Base URL: https://你的EdgeEver域名
MCP endpoint: https://你的EdgeEver域名/mcp
Authorization: Bearer 你的MCP令牌
```

配置完以后，不要只看“工具列表能不能加载”。至少做一次完整回读：

```text
创建一条测试笔记
→ 搜索这条笔记
→ 读取正文
→ 检查标题、首段和结尾
→ 删除或移入测试笔记本
```

只有写入后能回读，才算 Agent 真正接上了这座云笔记。

这样做的价值不是“可以聊天”，而是Agent能把知识管理变成可靠流程：

```text
排障结束
→ 生成复盘
→ 写入EdgeEver
→ 回读首尾验证
→ 追加旧平台备份
```

读写权限可以拆开。只做检索的Agent，不需要拿写权限；只向收件箱追加的自动化，也不需要管理用户和系统设置。

## 我为什么没有直接换成Notion或Obsidian

Notion适合数据库和团队协作，我仍然保留它作为结构化总枢纽。但它不是我唯一的原始知识副本。

Obsidian的本地Markdown很干净，却需要自己解决移动端同步、接口和自动写入验证。

EdgeEver的价值刚好在中间：保留传统三栏笔记体验、数据可以导出为普通Markdown、原生支持Agent接口、不要求我长期维护一台新服务器。

它不是所有人的最终答案，但正好补上了我的缺口。

## 这套方案适合谁

适合：

- 有大量个人笔记，希望AI能长期整理的人；
- 不想为同步设备数持续付费；
- 已有Cloudflare账号和域名；
- 愿意理解备份、令牌和升级的人；
- 希望保留Markdown退出路线的人。

不适合：

- 完全不想碰部署和维护；
- 团队需要复杂审批、企业审计和成熟售后；
- 只记录几条临时便签；
- 认为“部署成功”等于“不再需要备份”。

我的最终判断不是“EdgeEver取代所有笔记”。

> **我保留了旧平台，也保留了Notion。EdgeEver解决的是最关键的一层：让原始知识有一份可迁移、可被AI调用、又不被单一商业平台锁死的主副本。**

398篇笔记搬完后，我得到的不是一个新玩具，而是一条退出路线。

> 「每天帮你踩一个 AI 的坑，省下一小时。」

如果你只能带走一种格式，会选数据库，还是普通Markdown？
