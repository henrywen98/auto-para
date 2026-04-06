# Zettelkasten — AI-Powered Note Organizer for Obsidian

[![Claude Code Plugin](https://img.shields.io/badge/Claude_Code-Plugin-blue)](https://docs.anthropic.com/en/docs/claude-code)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

**Automatically organize your Obsidian vault using the Zettelkasten method.** Drop files into inbox, AI atomizes content into permanent atomic notes, builds wikilinks, and maintains Maps of Content (MOC) — powered by Claude Code.

> Zero dependencies. No Python, no scripts — pure Claude Code skills + agents.

[中文版](#中文版)

## The Problem

You clip articles, jot ideas, save snippets — and they pile up in a folder, **unlinked and forgotten**. Manual note organization doesn't scale. Your second brain stays dumb.

## The Solution

This plugin turns your Obsidian vault into a self-organizing knowledge base:

- **Atomize** — One concept per note. Multi-topic files are automatically split.
- **Rewrite** — Clean prose with proper structure, preserving all original information.
- **Link** — Every note connects to existing knowledge via contextual wikilinks. Cross-domain connections preferred.
- **Navigate** — Maps of Content (MOC) auto-generated when topics accumulate ≥3 notes.

## Quick Start

### 1. Install the Plugin

```bash
# In Claude Code
/plugins install github:henrywen98/zettelkasten
```

### 2. Set Up Your Vault

```bash
cd /path/to/your/obsidian/vault
mkdir -p 0_inbox 1_zettel 2_maps 3_output 4_assets
git init  # recommended for version safety
```

### 3. Ingest Your First Notes

Drop any files (notes, web clips, article exports) into `0_inbox/`, then:

```bash
cd /path/to/your/vault
claude
```

```
> /zet-ingest
```

That's it. Atomic notes appear in `1_zettel/`, organized by year-month, interlinked, with MOC navigation in `2_maps/`.

## Features

### `/zet-ingest` — Inbox to Knowledge Base

The core workflow. Scans `0_inbox/`, dispatches AI agents to process files in batches of ~10, then updates MOCs and commits via git.

Each file goes through: **read → classify → atomize → rewrite → frontmatter → link → write**.

```
0_inbox/weekly-learning.md          →  1_zettel/2026-04/ssh-key-authentication.md
  (SSH + Python decorators + Docker)    1_zettel/2026-04/python-decorator-patterns.md
                                        1_zettel/2026-04/docker-compose-networking.md
```

- Multi-topic files split into independent atomic notes
- Each note links to ≥1 existing note (connection forcing)
- Original language preserved (Chinese stays Chinese, English stays English)
- Source files deleted after processing — inbox stays clean

### `/zet-query` — Ask Your Knowledge Base

Query your accumulated knowledge in natural language. The AI navigates MOCs and note links, reads relevant notes, and synthesizes an answer with `[[wikilink]]` citations.

```
> /zet-query What do I know about authentication?
```

Output saved to `3_output/` with full source references.

### `/zet-lint` — Vault Health Check

Structural integrity scanner:

- Broken wikilinks pointing to non-existent notes
- Orphan notes with no inbound or outbound links
- Incomplete frontmatter (missing required fields)
- MOC coverage gaps — uncategorized notes
- Stale MOC counts

Auto-fix offered for common issues.

## Vault Structure

```
Vault/
├── 0_inbox/        Drop materials here (deleted after processing)
├── 1_zettel/       Permanent notes — atomic, linked, by year-month
│   └── YYYY-MM/    e.g. 2026-04/
├── 2_maps/         Maps of Content — auto-maintained topic navigation
├── 3_output/       Query results, lint reports
└── 4_assets/       Images and attachments
```

## Note Format

Every permanent note follows a consistent structure with YAML frontmatter:

```yaml
---
id: "202604061430"
title: "SSH Key Authentication"
created: 2026-04-06
processed: 2026-04-06
source: web-clip          # original | web-clip | import
tags: [ssh, security, devops]
summary: "How SSH key-based authentication works — key generation, exchange, and verification flow"
---

# SSH Key Authentication

[Note content in clear prose...]

## Links
- Related to [[tls-handshake-protocol]]: both use asymmetric cryptography for initial authentication
- See [[server-hardening-checklist]]: SSH key auth is a key step in hardening
```

## How It Works

```
/zet-ingest
    │
    ▼
zet-ingest (skill)          ← Orchestrator: scan, batch, dispatch, MOC, commit
    │
    ├── zet-worker (agent)  ← Batch 1: read → atomize → rewrite → link → write
    ├── zet-worker (agent)  ← Batch 2: can link to batch 1's notes
    └── ...                 ← Sequential processing, each batch commits before next
```

| Component | Role |
|-----------|------|
| **zet-ingest** | Orchestrator — scans inbox, batches files, dispatches workers, updates MOCs, git commits |
| **zet-worker** | File processor — reads, atomizes, rewrites, links, writes. ~10 files per batch |
| **zet-query** | Knowledge Q&A — navigates MOCs + full-text search to answer questions |
| **zet-lint** | Health checker — finds structural issues, offers auto-fix |

Batches run sequentially — each commits before the next starts. Later batches discover and link to earlier notes, improving link quality progressively. You can interrupt between batches without losing work.

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (CLI, desktop app, or IDE extension)
- An Obsidian vault (or any markdown folder)
- Git initialized in the vault (recommended)

## Model Recommendations

| Model | Best for |
|-------|----------|
| **Sonnet** | Daily ingestion — fast, reliable, good link quality |
| **Opus** | High-value content — best atomization judgment and cross-domain linking |

## Related Concepts

- [Zettelkasten Method](https://zettelkasten.de/introduction/) — The note-taking methodology behind this plugin
- [Atomic Notes](https://notes.andymatuschak.org/Evergreen_notes_should_be_atomic) — Andy Matuschak on why one concept per note matters
- [Maps of Content](https://www.linkingyourthinking.com/) — Nick Milo's MOC concept for navigating linked notes
- [Building a Second Brain](https://www.buildingasecondbrain.com/) — Tiago Forte's PARA method for organizing digital knowledge

## Inspiration

Inspired by [Andrej Karpathy](https://karpathy.ai/)'s ideas on building a personal knowledge base with AI — letting machines handle the organizing so you can focus on thinking.

## Contact

Questions, suggestions, or feedback? Reach out at **henrywen98@gmail.com**

## License

MIT

---

# 中文版

## Zettelkasten — AI 驱动的 Obsidian 笔记整理工具

**用 AI 自动整理你的 Obsidian 知识库。** 把文件丢进收件箱，AI 自动拆分为原子笔记、构建双向链接、维护 MOC（内容地图）导航 — 基于 Claude Code 插件。

> 零依赖。不需要 Python、不需要脚本 — 纯 Claude Code skills + agents 实现。

## 痛点

你收藏文章、记录灵感、保存代码片段 — 然后它们就堆在文件夹里，**没有链接、逐渐遗忘**。手动整理笔记无法规模化，你的"第二大脑"始终是死的。

## 解决方案

这个插件把你的 Obsidian 仓库变成一个自组织的知识库：

- **原子化** — 每条笔记只包含一个概念，多主题文件自动拆分
- **改写** — 清晰的文字表述，保留所有原始信息
- **链接** — 每条笔记通过上下文 wikilink 连接到已有知识，偏好跨领域关联
- **导航** — 当某个标签积累 ≥3 条笔记时，自动生成 MOC（内容地图）

## 快速开始

### 1. 安装插件

```bash
# 在 Claude Code 中执行
/plugins install github:henrywen98/zettelkasten
```

### 2. 初始化仓库

```bash
cd /path/to/your/obsidian/vault
mkdir -p 0_inbox 1_zettel 2_maps 3_output 4_assets
git init  # 推荐，用于版本安全
```

### 3. 处理第一批笔记

把任意文件（笔记、网页剪藏、文章导出）放入 `0_inbox/`，然后：

```bash
cd /path/to/your/vault
claude
```

```
> /zet-ingest
```

完成。原子笔记出现在 `1_zettel/` 中，按年月组织，互相链接，MOC 导航在 `2_maps/`。

## 功能

### `/zet-ingest` — 收件箱 → 知识库

核心工作流。扫描 `0_inbox/`，分批派发 AI agent 处理（每批 ~10 个文件），然后更新 MOC 并 git 提交。

每个文件经历：**读取 → 分类 → 原子化 → 改写 → 元数据 → 建链接 → 写入**

```
0_inbox/每周学习笔记.md              →  1_zettel/2026-04/ssh-key-authentication.md
  (SSH + Python装饰器 + Docker)         1_zettel/2026-04/python-decorator-patterns.md
                                        1_zettel/2026-04/docker-compose-networking.md
```

- 多主题文件自动拆分为独立的原子笔记
- 每条笔记至少链接到 1 条已有笔记（强制连接）
- 保持原始语言（中文写的笔记保持中文）
- 处理后删除源文件 — 收件箱保持清洁

### `/zet-query` — 向知识库提问

用自然语言查询你积累的知识。AI 通过 MOC 和笔记链接导航，阅读相关笔记，生成带 `[[wikilink]]` 引用的综合回答。

```
> /zet-query 我关于认证方面知道什么？
```

输出保存到 `3_output/`，附完整来源引用。

### `/zet-lint` — 仓库健康检查

结构完整性扫描：

- 指向不存在笔记的断链
- 没有入站或出站链接的孤立笔记
- 不完整的元数据（缺少必填字段）
- MOC 覆盖空白 — 未被分类的笔记
- 过时的 MOC 计数

可自动修复常见问题。

## 仓库结构

```
Vault/
├── 0_inbox/        在此放入素材（处理后删除）
├── 1_zettel/       永久笔记 — 原子化、已链接、按年月组织
│   └── YYYY-MM/    如 2026-04/
├── 2_maps/         内容地图 — 自动维护的主题导航
├── 3_output/       查询结果、健康检查报告
└── 4_assets/       图片和附件
```

## 模型推荐

| 模型 | 适用场景 |
|------|----------|
| **Sonnet** | 日常处理 — 速度快、可靠、链接质量好 |
| **Opus** | 高价值内容 — 原子化判断和跨领域链接质量最佳 |

## 灵感来源

受 [Andrej Karpathy](https://karpathy.ai/) 关于用 AI 构建个人知识库的理念启发 — 让机器处理整理工作，你专注于思考。

## 联系方式

问题、建议或反馈？请联系 **henrywen98@gmail.com**

## 许可证

MIT
