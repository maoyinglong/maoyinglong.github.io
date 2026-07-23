---
title: 手把手教你部署 CLI Proxy API——统一管理所有 AI 大模型的神器
date: 2026-07-14 12:00:00
tags:
  - Docker
  - CLI Proxy API
  - AI 工具
  - 大模型
  - 自托管
  - 中转
  - VPS
categories:
  - 教程类
  - AI工程
  - 开源
description: 手把手教你用 Docker Compose 部署 CLI Proxy API，把所有大模型的 Key 统一管理起来。面向零基础小白，每一步都解释清楚为什么这么做，配置字段逐一说明，附带踩坑提醒。
cover: /images/cli-proxy-api-cover.webp
---

![CLI Proxy API 封面](/images/cli-proxy-api-cover.webp)

你手头是不是有好几个 AI 大模型的 Key？Claude 一个、OpenAI 一个、Gemini 一个……每次换工具都要重新填一遍 Key，还要时刻担心 API 被封、不知道哪个 Key 还剩多少额度。说真的，用不了几天就头大了。

今天这篇就带你部署一个神器，名字叫 **CLI Proxy API**。

> 它像一个智能调度台：你只需要把所有大模型的 Key 一次性交给他，之后不管用 Claude Code、OpenAI 客户端还是 Gemini 工具，全部接到这个统一的入口上。它帮你做反向中转防封、自动负载均衡、实时用量监控——你再也不用东一个 Key 西一个 Key 地到处填了。

这篇文章面向**零基础小白**。每一步我都会解释清楚"为什么这么做、这一步是在干什么"。跟着走就行。

---

## 动手之前，你需要准备什么

只需要两样东西：

**一台有公网 IP 的云服务器**，海外的就行，已经装好了 Docker 和 Docker Compose。如果你还没装 Docker，可以看看我们之前的那篇 Docker 安装教程。

**一个能 SSH 连上服务器的终端工具**。Mac 用户用自带的终端，Windows 可以用 PowerShell 或者 Xshell，随便哪个都行。

---

## 部署开始

### 第一步：连上你的服务器

打开你的终端工具，输入下面这行命令，把"你的服务器IP"换成你真实的服务器地址：

```bash
ssh root@你的服务器IP
```

回车之后输入密码就进去了。看到一串欢迎文字，说明连接成功。

### 第二步：创建一个专门的文件夹

在 Linux 里，我们习惯把每个服务放在 `/opt` 目录下。先建一个文件夹，然后进去：

```bash
mkdir -p /opt/cli-proxy-api && cd /opt/cli-proxy-api
```

`mkdir -p` 的意思是"如果上级目录不存在就自动创建"，防止报错。`&&` 代表前面命令成功了才执行后面的。现在你应该在 `/opt/cli-proxy-api` 这个目录下了。

### 第三步：创建 docker-compose.yml

Docker Compose 是 Docker 官方出的一个编排工具。你只需要写一个 `docker-compose.yml` 配置文件，告诉它"我要用什么镜像、开哪个端口、挂哪些文件夹"，然后一条命令就能把整个服务拉起来。比自己手动敲 `docker run` 十几行参数要省心得多。

用 nano 编辑器新建这个文件：

```bash
nano docker-compose.yml
```

把下面这段原封不动粘贴进去：

```yaml
services:
  cli-proxy-api:
    image: eceasy/cli-proxy-api:latest
    container_name: cli-proxy-api
    ports:
      - "8317:8317"
    volumes:
      - ./config.yaml:/CLIProxyAPI/config.yaml
      - ./auths:/root/.cli-proxy-api
      - ./logs:/CLIProxyAPI/logs
    restart: unless-stopped
```

我给你逐行解释一下这些字段是什么意思：

**`services:`** 表示"下面开始定义服务"。一个 docker-compose.yml 可以同时跑好几个服务，这里我们只定义一个。

**`cli-proxy-api:`** 是这个服务的名字，你可以随便起，但要跟下面 `container_name` 一致。

**`image: eceasy/cli-proxy-api:latest`** 告诉 Docker 去 Docker Hub 上拉取这个镜像。`eceasy/cli-proxy-api` 是镜像名，`latest` 表示最新版本。

**`container_name: cli-proxy-api`** 给跑起来的容器起个名字，方便之后用 `docker logs cli-proxy-api` 之类的命令查看它。

**`ports:`** 配置端口映射。`"8317:8317"` 的意思是：把容器里面的 8317 端口，映射到服务器的 8317 端口上。这样你访问 `服务器IP:8317` 就等于访问容器里的服务了。

**`volumes:`** 配置目录挂载。Docker 容器删除后，里面的所有数据都会丢失。所以我们必须把重要的目录"挂"到服务器本地：
- `./config.yaml` 是核心配置文件，挂在本地不怕丢。
- `./auths` 是存放各大模型登录凭证的目录。
- `./logs` 是日志目录，方便排查问题。

**`restart: unless-stopped`** 让容器在服务器重启后自动启动，除非你手动执行了 `docker stop` 命令。

粘贴完成后，按 `Ctrl+O` 保存文件，再按 `Ctrl+X` 退出 nano。

### 第四步：创建 config.yaml

这是 CLI Proxy API 运行所必需的核心配置文件：

```bash
nano config.yaml
```

把下面内容粘进去：

```yaml
host: "0.0.0.0"
port: 8317

remote-management:
  allow-remote: true
  secret-key: "你设定的管理面板密码"
  disable-control-panel: false

auth-dir: "~/.cli-proxy-api"

logging-to-file: true
logs-max-total-size-mb: 10
error-logs-max-files: 10
```

关键字段说明：

**`host: "0.0.0.0"`** 表示监听服务器上的所有网络接口。如果改成 `127.0.0.1`，就只能在本机访问，外网打不开。

**`port: 8317`** 服务监听的端口号。你可以改成别的，但要和上面 docker-compose.yml 里的端口映射保持一致。

**`remote-management:`** 管理面板的配置块：
- `allow-remote: true` **必须设为 true**，否则从外网浏览器打不开管理面板，只能在本机用 `curl` 访问。
- `secret-key` 是管理面板的登录密码。填一个你能记住的就行，服务启动后会自动把你的明文密码加密成哈希值存进文件，很安全。
- `disable-control-panel: false` 表示启用 Web 管理面板（默认就是 false）。

**`auth-dir: "~/.cli-proxy-api"`** 存放各大模型认证信息的目录，后面你在面板里登录 Claude、OpenAI 等账号时，凭据就存在这里。

**`logging-to-file: true`** 开启日志写入文件，方便以后排查问题。

**`logs-max-total-size-mb: 10`** 日志总大小上限为 10MB，超出后自动滚动。

保存并退出（`Ctrl+O` 然后 `Ctrl+X`）。

### 第五步：一键启动

在项目目录下执行：

```bash
docker compose up -d
```

`-d` 是 detached 的缩写，意思是"放到后台运行"。等大约 10 秒钟，服务就完全启动了。

### 第六步：打开管理面板

打开你的浏览器，在地址栏输入：

```
http://你的服务器IP:8317/management.html
```

用你刚才在 `secret-key` 里设置的密码登录。登录后就能看到一个 Web 管理面板，在这里你可以：
- 添加各大模型的 API Key（Claude、OpenAI、Gemini 等等）
- 配置反向中转规则
- 查看每个 Key 的剩余额度和调用次数
- 管理 OAuth 登录凭据

然后在你本地的 Claude Code 或其他 AI 工具里，把 API 地址改成 `http://你的服务器IP:8317/v1`，Key 填你在管理面板里设置的 `api-keys`。这样所有请求都会经过这个统一的中转站，再也不用来回切换了。

---

## 一个重要踩坑提醒

有个小陷阱，我踩过了，你就不用再踩：**不要用 `curl -I` 去检测这个服务是否正常运行**。

`curl -I` 发送的是 HTTP HEAD 请求（只问服务器"你在不在"，不要实际内容）。但 CLI Proxy API 的 Go 后端对 HEAD 请求处理有 bug，会永远返回 404。

如果你部署完后习惯性地跑一句 `curl -I http://IP:8317/management.html`，看到 404，然后以为服务挂了、反复重启排查，那就浪费半天时间了。

**正确做法**始终用 GET 请求：

```bash
curl -s -o /dev/null -w '%{http_code}' http://你的服务器IP:8317/management.html
```

这行命令的意思是：静默请求这个页面，只输出 HTTP 状态码。返回 `200` 就说明一切正常。

---

## 总结

整个部署流程就是三步：
- 写一个 `docker-compose.yml`
- 写一个 `config.yaml`
- 跑一句 `docker compose up -d`

跑起来之后，你所有的大模型 API Key 全部交给这一个面板统一管理。再也不用到处填 Key、担心封 IP、猜哪个 Key 快没额度了。

> 工具越少，心智负担越小。把复杂度关进一个容器里，你就自由了。
