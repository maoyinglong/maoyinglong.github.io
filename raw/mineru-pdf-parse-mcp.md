---
title: 把 PDF 变成可读 Markdown：MinerU 文档解析 MCP 部署实战
date: 2026-06-20 17:30:00
tags:
  - PDF解析
  - MCP
  - AI工具
  - 自动化
categories:
  - 教程类
  - AI工程
description: "MinerU 能把任意 PDF 解析成干净的 Markdown，还能作为 MCP 工具被 AI Agent 直接调用。这篇记录完整部署过程和实测效果。"
cover: /images/mineru-pdf-parse-mcp-cover.webp
---

![封面图](/images/mineru-pdf-parse-mcp-cover.webp)

> 想让 AI 读 PDF 论文、合同、扫描件？MinerU 这个国产开源工具的解析精度堪比专业 OCR，还能直接以 MCP 协议挂到 Claude / GPT 工作流里。这篇记录我从零部署到稳定使用的完整过程。

---

## 背景：为什么需要文档解析 MCP

我经常要把 arXiv 论文、行业报告、PDF 合同喂给 AI 处理。问题是：

- 直接把 PDF 路径给 GPT？它读不动扫描版、表格、复杂公式
- 用现成的 OCR 服务？要么贵、要么精度不行、要么是闭源
- 自己跑 PaddleOCR？需要 GPU 环境，部署成本高

直到发现了 **MinerU**（OpenDataLab 出品），发现这是当前最好的开源方案：

- ✅ 国产开源，支持本地/云端双模式
- ✅ 解析精度高（公式、表格、版面都能识别）
- ✅ 支持 PDF/DOCX/PPTX 多格式
- ✅ 输出标准 Markdown，可以直接喂给大模型
- ✅ 官方提供 MCP 服务端，开箱即用

---

## 第一步：环境准备

### Node.js 版本要求

MinerU 的 MCP 包基于 Node.js 18+，但实际跑起来**强烈建议用 Node 20+**（我用 22 完全没问题）。

### 关键陷阱：用绝对路径，不要用 `node`

直接写 `command: node` 在很多 MCP 客户端会出诡异错误——`spawn node ENOENT`。原因可能是：

- 系统的 PATH 里 `node` 不存在
- 或者 MCP 客户端用的是另一个虚拟环境

**正确做法**：用 Node 二进制的**绝对路径**：

```yaml
command: /root/.hermes/node/bin/node   # 或你的实际路径
args:
  - /root/.hermes/node/lib/node_modules/mineru-mcp/dist/index.js
```

不同系统的路径：
- Linux 自定义安装：`/opt/node/bin/node`
- macOS Homebrew：`/opt/homebrew/bin/node` 或 `/usr/local/bin/node`
- nvm：`~/.nvm/versions/node/v22.0.0/bin/node`

可以用 `which node` 查实际路径。

---

## 第二步：安装 MCP 包

```bash
npm install -g mineru-mcp
```

装完后会输出包路径，类似：

```
/usr/local/lib/node_modules/mineru-mcp
# 或 /root/.hermes/node/lib/node_modules/mineru-mcp（隔离环境）
```

记住这个路径，待会要写到配置里。

---

## 第三步：获取 API Key

到 https://mineru.net 注册开发者账号，在控制台拿到 API Token。

**安全提示**：

- 不要把 API Key 直接写在命令行里调试（容易被 shell 历史、监控日志捕获）
- 不要写到公共配置仓库
- 推荐做法：写入 MCP 配置文件的 `env` 字段，用权限 600 保护

---

## 第四步：配置 MCP 服务

不管你是用 Claude Desktop、Cursor、Cline 还是自建的 Hermes Agent，配置方式大同小异：

```yaml
mcp_servers:
  mineru-mcp:
    command: /root/.hermes/node/bin/node
    args:
      - /root/.hermes/node/lib/node_modules/mineru-mcp/dist/index.js
    env:
      MINERU_API_KEY: your_api_key_here
    enabled: true
```

加载配置后（一般是 `/reload` 或重启客户端），AI 工具列表里会多出 6 个工具：

| 工具 | 功能 |
|---|---|
| `mineru_parse` | 提交单个文档 URL 解析 |
| `mineru_status` | 查询解析状态、拿结果 |
| `mineru_batch` | 批量提交（最多 200 个）|
| `mineru_batch_status` | 批量任务进度查询 |
| `mineru_upload_batch` | 上传本地文件批量解析 |
| `mineru_download_results` | 下载结果并保存为 Markdown |

---

## 第五步：实战用法

### 单个 PDF 解析

让 AI 帮你调用：

> "帮我解析这篇论文 https://arxiv.org/pdf/2401.00001.pdf，提取核心方法和实验结果"

AI 会自动：

1. 调用 `mineru_parse` 提交任务
2. 调用 `mineru_status` 轮询状态
3. 拿到 Markdown 结果后做总结

### 批量解析（一键处理一堆论文）

```
帮我批量解析这几篇 PDF：https://arxiv.org/pdf/A.pdf, https://arxiv.org/pdf/B.pdf, ...
结果保存到 /root/knowledge_vault/raw/ 目录
```

### 本地文件解析

```
/tmp/contracts/ 目录里有几份扫描版合同，帮我提取关键条款（金额、期限、违约责任）
```

---

## 第六步：踩坑记录

### 坑 1：解析状态长时间 Pending

**现象**：提交了一个 100 页的扫描版 PDF，过了 1 分钟还在 Pending 状态。

**原因**：复杂排版/扫描件需要更长时间处理，**这是正常现象不是 bug**。

**正确做法**：

- 设置合理的轮询间隔（5-10 秒一次）
- 单次响应慢不要立即重试或杀进程
- 给 MCP 调用设置 5 分钟左右的 timeout

```python
import time

for i in range(60):  # 最多轮询 5 分钟
    status = call_tool("mineru_status", {"task_id": task_id})
    if status["state"] == "completed":
        return status["result"]
    time.sleep(5)
```

### 坑 2：`spawn node ENOENT` 错误

**根因**：配置里用了裸 `node` 命令，但 MCP 客户端的 PATH 环境变量里找不到。

**解法**：改用 Node 二进制的绝对路径（上面第一步已经强调）。

### 坑 3：API Key 被日志泄露

**现象**：调试时为了方便，把 Key 直接写在 shell 命令里，结果被某段日志脚本捕获后传到云端监控。

**解法**：

- 调试时用环境变量：`export MINERU_API_KEY=xxx && call_tool ...`
- 配置完成立即 `unset MINERU_API_KEY`
- 长期配置写在 MCP 配置文件的 `env` 字段，并加权限 `chmod 600 config.yaml`

---

## 第七步：实测效果

我用 MinerU 解析了 50 篇 arXiv 论文（平均 15 页，含公式和图表），对比直接用 GPT-4 Vision 处理：

| 维度 | MinerU | GPT-4 Vision |
|---|---|---|
| 公式识别准确率 | ~95% | ~70% |
| 表格识别 | 完整保留结构 | 经常错位 |
| 处理速度（单页）| ~3 秒 | ~8 秒 |
| 成本（50 篇）| **免费额度** | 约 $5 |
| Token 消耗（喂给后续 LLM）| 显著更少 | 较高 |

MinerU 输出的是干净的 Markdown，结构化程度高，**喂给后续 LLM 的 token 消耗能减少 70% 左右**。

---

## 第八步：高级用法（自动化场景）

### 配合定时任务：每天自动解析 RSS 里的论文

```bash
# 1. RSS 抓取器发现新论文链接
# 2. 调用 mineru_batch 批量解析
# 3. mineru_download_results 保存到本地
# 4. 自动入库到知识库
```

### 配合 RAG：文档向量化前预处理

MinerU 输出 Markdown → 拆分成 chunk → embedding 入向量库。整个链路无需任何人工干预。

### 配合论文阅读助手

把 MinerU + Claude / GPT 组合成"论文精读机器人"：

1. 用户给 PDF URL
2. MinerU 解析成 Markdown
3. 大模型读 Markdown，按"摘要/方法/实验/结论"四段总结
4. 用户问细节时，针对性引用论文段落回答

---

## 写在最后

MinerU 是我目前用过的国产开源文档解析里**精度最高、部署最简单、成本最低**的方案。配合 MCP 协议，几乎所有 AI IDE 和 Agent 框架都能零成本接入。

**几个建议**：

- **个人学习用**：免费额度完全够用，每天解析 100 篇论文都没问题
- **生产环境**：建议买企业版 + 自己部署开源版做兜底
- **大批量任务**：用 `mineru_batch` 比单篇循环快 5-10 倍
- **保留原始 PDF**：解析结果虽然好，但 PDF 原件还是有保留价值

你在用什么工具解析 PDF？评论区聊聊～