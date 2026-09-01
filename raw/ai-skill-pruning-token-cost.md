---
title: "我删掉一半AI技能，它反而更聪明了"
date: 2026-08-22 10:00:00
tags:
  - AI Agent
  - Skills
  - Token优化
  - 提示词治理
categories:
  - 教程类
  - AI工程
description: "给AI装技能很爽：搜索、写作、运维、画图，恨不得一次塞满。真正长期运行后，我却发现它越来越重——新会话更贵，注意力被无关能力分走，备份目录甚至会伪装成可用技能。我没有换模型，只做了一次可回滚的减法：启用技能从192个降到94个，常驻技能索引缩小48%，每次缓存失效少背约4000～5500个"
cover: /images/ai-skill-pruning-token-cost-cover.webp
evidence: 亲测
---

> **给 AI 装技能很爽：搜索、写作、运维、画图，恨不得一次塞满。真正长期运行后，我却发现它越来越重——新会话更贵，注意力被无关能力分走，备份目录甚至会伪装成可用技能。我没有换模型，只做了一次可回滚的减法：启用技能从192个降到94个，常驻技能索引缩小48%，每次缓存失效少背约4000～5500个输入Token。**

我原来相信一件很直觉的事：

**AI会的技能越多，能力就越强。**

直到我用官方测量命令把提示词拆开，才发现自己养的不是全能助手，而是一只每天出门都背着整个工具仓库的骆驼。

问题不是某个技能写得差。

问题是太多技能在任务开始前就已经占据了它的视野。

## 技能正文没有全塞进去，为什么仍然会变重

Hermes官方把技能设计成“渐进式披露”：先让Agent看到技能名称和简介，真正匹配任务时，再读取完整`SKILL.md`和引用文件。

【官方文档】https://hermes-agent.nousresearch.com/docs/user-guide/features/skills

这套设计已经比“开局把所有操作手册都塞进上下文”节省得多。但它不等于零成本。

每个启用技能至少要在常驻索引中留下技能名称和简介。几十个时不明显，到一两百个，索引本身就成了一篇长文。

更麻烦的是，模型还要在一堆相似能力里做选择：通用运维、专用运维、旧版运维、恢复备份里的运维，描述稍有重叠，注意力就会被稀释。

> **渐进加载解决了“正文全量注入”，没有替你解决“索引无限膨胀”。**

## 我测到的真实数字

【亲测】我先运行：

```bash
hermes prompt-size --json
hermes skills list --enabled-only
```

治理前后的结果如下：

| 指标 | 治理前 | 治理后 | 变化 |
|---|---:|---:|---:|
| 启用技能 | 192 | 94 | -51.0% |
| 技能索引 | 33,901B | 17,618B | -48.0% |
| 系统提示词 | 61,139B | 44,856B | -26.6% |
| 稳定提示词前缀 | 48,749B | 32,466B | -16,283B |

按常见字符与Token比例粗略换算，新会话或前缀缓存失效时，少输入约4000～5500个Token。真实账单仍以模型服务商的usage为准。

2026年8月22日我再次测量，当前技能索引是17,812B，系统提示词45,178B，说明瘦身结果仍然维持，没有悄悄反弹回原来的规模。

![AI技能瘦身实测对比](/images/ai-skill-pruning-token-cost-cover.webp)

这不是一次“模型突然变聪明”的跑分神话。我能确认的是三件事：常驻输入明显下降、技能候选冲突减少、每次缓存失效需要重新发送的稳定前缀变短。

至于回答质量提高多少，没有同一批盲测样本，我不会编一个百分比。

## 真正的污染源，居然是备份目录

这次最离谱的发现，不是技能多，而是**备份也被当成了技能**。

本机的`.restore-backups/`下保留了历史副本。文件本来只是保险，但扫描后发现：

- 备份目录里有169个`SKILL.md`；
- 最终技能列表中，有70个备份来源技能仍处于启用状态；
- 同一个能力出现多个版本，名称与描述高度相似。

这类问题很隐蔽。你看磁盘，会以为那只是备份；Agent看索引，会以为那是七十种现在就能调用的能力。

所以我现在判断技能数量时，不再只数磁盘文件。文件数量不等于最终索引数量，真正该看的是运行时识别后的列表：

```bash
hermes skills list --enabled-only
```

## 我没有删文件，只做可逆禁用

最危险的做法是看到“192个”就开始批量删除。

技能里可能有低频但关键的灾备流程、安全审计、升级恢复和凭据操作。它们一个月不用，不等于没有价值。

我的处理顺序是先做基线备份：

```bash
stamp=$(date +%Y%m%d-%H%M%S)
out="$HOME/.hermes/backups/skill-prune-$stamp"
mkdir -p "$out"
cp "$HOME/.hermes/config.yaml" "$out/config.yaml"
tar -C "$HOME/.hermes" -czf "$out/skills.tar.gz" skills
hermes prompt-size --json > "$out/prompt-size-before.json"
sha256sum "$out/config.yaml" "$out/skills.tar.gz"
```

这一步不是形式主义。后面如果禁错了，能直接回滚配置和技能目录。

我保留基础设施、网关、安全、备份、知识库、搜索、发布和高频排障能力。优先禁用长期没有使用记录的垂直技能、被专用能力覆盖的通用技能、恢复目录和历史副本。

**重点是禁用，不是物理删除。**

文件还在。发现误伤时，只要从禁用列表移出即可。

## 别用一条配置命令硬塞复杂列表

我踩过最危险的坑，是把列表当成字符串写进配置：

```bash
hermes config set skills.disabled '["a","b"]'
```

命令显示成功，但最终可能写成：

```yaml
skills:
  disabled: '["a","b"]'
```

它不是列表，而是一整段字符串。

后续脚本如果对它执行`len()`，得到的是字符数，不是技能数。那次我看到一个荒唐的`DISABLED_COUNT=4977`，才意识到配置类型已经坏了。

复杂YAML我的安全写法是：完整读取、在内存修改、重新解析验证、原子替换。

```python
from pathlib import Path
import os, tempfile, yaml

path = Path.home() / ".hermes/config.yaml"
config = yaml.safe_load(path.read_text())
config["skills"]["disabled"] = merged_list

rendered = yaml.safe_dump(
    config,
    allow_unicode=True,
    sort_keys=False,
    width=100000,
)
check = yaml.safe_load(rendered)
assert isinstance(check["skills"]["disabled"], list)

fd, tmp = tempfile.mkstemp(prefix="config.yaml.", dir=str(path.parent))
os.close(fd)
Path(tmp).write_text(rendered)
os.replace(tmp, path)
```

写完还要运行：

```bash
hermes config check
hermes prompt-size --json
hermes skills list --enabled-only
```

同时确认配置能解析、核心技能还在、当前模型和服务商没有漂移。

## description也该减肥

一个技能的简介不应该承担完整说明书的职责。

好的description只回答两件事：它能做什么，什么情况下应该加载。

安装步骤、命令、失败处理和案例都应该放进正文，等任务真正命中后再读。

把五百字简介压到五十字，不会削弱技能；反而让匹配更干净，也让常驻索引更轻。

## 这套方法适合谁

适合：

- 已经安装几十到几百个Skills的Agent用户；
- 新会话首轮Token长期偏高；
- 同类技能很多，经常选错工具；
- 升级、迁移后遗留大量备份目录；
- 希望保留资产，但不想让所有资产常驻的人。

不适合：

- 只有十来个技能的小型环境；
- 没有测量命令，只凭感觉删文件；
- 所有技能确实高频且边界清晰的专用Agent。

我的判断很明确：

> **技能库应该像仓库，不应该像每天背在身上的行李。能力可以很多，常驻索引必须克制。**

下一次想装“500个神级Skills”之前，先测一次提示词。数字会比收藏夹冷静得多。

> 「每天帮你踩一个 AI 的坑，省下一小时。」

你现在的AI，背了多少它根本用不到的技能？
