# AutoPARA — Obsidian 知识库自动整理插件设计文档

## 概述

一个 Claude Code 插件，自动整理 Obsidian vault 中的笔记。用户只需把素材丢进 inbox/，通过命令触发 AI 处理：归档原文、提炼概念、编译 wiki、生成可视化。解决"擅长捕获但不擅长整理"的痛点。

**项目目录**：`/Users/henry/Documents/1_Project/260405_obsidian_skills_autopara/`
**目标 vault**：`/Users/henry/Library/Mobile Documents/iCloud~md~obsidian/Documents/Wen/`
**发布渠道**：henry-hub 插件市场
**现有规模**：1754 篇 md 文件，1.1GB（含图片/附件）

---

## 1. Vault 目标结构

```
Vault/
├── inbox/              ← 唯一入口，用户丢素材的地方
├── archive/            ← 原文存档（按年月组织）
│   ├── 2026-01/
│   ├── 2026-02/
│   └── .../
├── wiki/               ← AI 编译维护的知识层
│   ├── _index.md       ← 总索引（所有文章列表 + 简述）
│   ├── _tags.md        ← 标签索引
│   ├── concepts/       ← 概念文章（AI 从多篇原文中提炼）
│   ├── topics/         ← 主题聚合页（多篇文章的汇总视图）
│   └── viz/            ← 可视化输出（Marp 幻灯片 / 图表）
├── output/             ← 查询结果输出
├── assets/             ← 图片/附件统一管理
├── .gitignore          ← 包含 .obsidian/
└── .gitattributes      ← git lfs 追踪规则
```

**核心原则**：
- inbox/ 是用户的，wiki/ 是 AI 的，archive/ 是桥梁
- 用户不需要理解或维护 archive/ 和 wiki/ 的内部结构
- 所有分类通过标签和 wiki 链接实现，不依赖文件夹层级

---

## 2. Frontmatter 规范

### 原文（archive/ 下的文件）

```yaml
---
title: SSH Key 配置指南
created: 2026-01-14
archived: 2026-04-05
tags: [ssh, devops, 服务器]
summary: 一句话摘要，描述这篇笔记的核心内容
concepts: [SSH密钥认证, 远程服务器管理]
source: original  # original | web-clip | import
---
```

### Wiki 文章（wiki/ 下的文件）

```yaml
---
title: SSH 密钥认证
type: concept  # concept | topic | viz
sources: [archive/2026-01/ssh-key-henrywen.md]
related: [远程服务器管理, Linux安全]
last_compiled: 2026-04-05
---
```

---

## 3. 命令体系

| 命令 | 用途 | 说明 |
|------|------|------|
| `/para ingest <文件或glob>` | 处理 inbox 中的指定文件 | 归档 + 打标签 + 编译进 wiki |
| `/para query <问题>` | 对知识库提问 | 输出到 output/ |
| `/para lint` | 健康检查 | 断链、孤页、标签不一致、缺摘要 |
| `/para viz <主题> [格式]` | 生成可视化 | 支持 marp / matplotlib / mermaid |
| `/para status` | 查看统计 | 文章数、标签分布、待处理数 |
| `/para migrate` | 一次性全量迁移 | 仅初始化时使用 |

### `/para ingest` 流程

1. 读取 inbox 中指定文件的内容
2. AI 生成 frontmatter（summary, tags, concepts）
3. 移动文件到 `archive/YYYY-MM/`（按 created 日期）
4. 图片/附件移到 `assets/`，脚本重写文内链接
5. 检查 wiki/ 是否有匹配的 concept，有则更新 sources 列表
6. 无匹配 concept 且 AI 判断值得新建 → 创建 concept 文章
7. 更新 `_index.md` 和 `_tags.md`

---

## 4. 全量迁移策略

现有 1754 篇笔记需要从 PARA 结构迁移到新结构。迁移分四个阶段，确保不丢信息。

### 阶段一：扫描（脚本，确定性）

Python 脚本遍历全部文件，生成 `manifest.json`：

```json
{
  "total_files": 1754,
  "total_size": "1.1GB",
  "files": [
    {
      "path": "00_Inbox/20260114-2042-SSH Key - henrywen.md",
      "size_bytes": 2048,
      "word_count": 350,
      "created": "2026-01-14",
      "has_frontmatter": false,
      "has_images": true,
      "images": ["img/ssh-key-1.png"],
      "preview": "前 200 字内容..."
    }
  ],
  "attachments": [
    {"path": "img/ssh-key-1.png", "size_bytes": 51200, "referenced_by": ["00_Inbox/..."]}
  ]
}
```

**产出**：完整的 vault 清单，每个文件都有记录。

### 阶段二：规划（AI 读 manifest）

AI 读取 manifest.json（含每篇的 preview），生成 `migration-plan.json`：

```json
{
  "actions": [
    {
      "source": "00_Inbox/20260114-2042-SSH Key - henrywen.md",
      "target": "archive/2026-01/ssh-key-henrywen.md",
      "frontmatter": {
        "title": "SSH Key 配置指南",
        "tags": ["ssh", "devops"],
        "summary": "...",
        "concepts": ["SSH密钥认证"]
      },
      "restructure": false
    }
  ],
  "duplicates": [],
  "outdated": [],
  "new_concepts": ["SSH密钥认证", "Docker部署", "Git工作流"]
}
```

**用户在此审阅计划后再执行。**

因为 manifest 本身可能很大，AI 的规划也按批次进行（每批 ~50 篇），每批产出一个 plan 片段，最后脚本合并成完整的 migration-plan.json。

### 阶段三：并行执行（teammates + worktree 隔离）

- 将 migration-plan.json 按批次分割（每批 30-50 篇）
- 每个 teammate 在独立的 git worktree 中工作
- 每个 teammate 的任务：
  - 读原文全文
  - 重写/补充 frontmatter
  - 重构内容（如需要：合并重复、更新过时信息）
  - 移动到目标路径
  - 更新图片/附件链接
- 完成后合并回主分支

### 阶段四：验证

脚本驱动，对比 manifest.json vs 迁移后状态：

- 文件数一致性检查（manifest 中每个文件在新结构中都有对应）
- 无空文件检查
- 所有 markdown 内链接有效性检查
- 所有图片/附件引用有效性检查
- frontmatter 完整性检查（每篇都有 title, tags, summary）
- 生成 `validation-report.md`，列出所有异常

---

## 5. 插件结构

遵循 henry-hub 规范：

```
plugins/autopara/
├── plugin.json                 ← 只写 name, version, description
├── skills/
│   ├── para-ingest/SKILL.md      ← ingest 命令
│   ├── para-query/SKILL.md       ← query 命令
│   ├── para-lint/SKILL.md        ← lint 命令
│   ├── para-viz/SKILL.md         ← viz 命令
│   ├── para-status/SKILL.md      ← status 命令
│   └── para-migrate/SKILL.md     ← 全量迁移命令
└── scripts/
    ├── scan.py                 ← 阶段一：扫描生成 manifest
    ├── validate.py             ← 阶段四：验证
    └── utils.py                ← 共享工具（链接重写、frontmatter 解析等）
```

### plugin.json

```json
{
  "name": "autopara",
  "version": "0.1.0",
  "description": "Obsidian 知识库自动整理 — inbox 丢素材，AI 编译 wiki"
}
```

---

## 6. 技术选型

| 组件 | 选择 | 理由 |
|------|------|------|
| 脚本语言 | Python (uv) | 用户已安装，文件操作和 JSON 处理方便 |
| Frontmatter 解析 | python-frontmatter | 成熟的 YAML frontmatter 读写库 |
| Markdown 链接重写 | 自写（正则） | Obsidian 链接格式固定，不需要重型库 |
| 可视化 - 幻灯片 | Marp | Obsidian 有 Marp 插件，markdown 原生 |
| 可视化 - 图表 | matplotlib / mermaid | matplotlib 生成图片，mermaid 嵌入 md |
| 版本管理 | git + git lfs | lfs 追踪 png/jpg/pdf 等二进制文件 |

---

## 7. Git 初始化

```bash
# .gitignore
.obsidian/
.DS_Store
.trash/

# .gitattributes (git lfs)
*.png filter=lfs diff=lfs merge=lfs -text
*.jpg filter=lfs diff=lfs merge=lfs -text
*.jpeg filter=lfs diff=lfs merge=lfs -text
*.gif filter=lfs diff=lfs merge=lfs -text
*.webp filter=lfs diff=lfs merge=lfs -text
*.pdf filter=lfs diff=lfs merge=lfs -text
*.mp4 filter=lfs diff=lfs merge=lfs -text
*.zip filter=lfs diff=lfs merge=lfs -text
```

---

## 8. 约束与边界

- **不做实时同步**：不监听文件变化，所有操作按需触发
- **不做 RAG**：靠索引文件和 AI 直接读取 markdown，不引入向量数据库
- **不做 Obsidian 插件**：这是 Claude Code 插件，Obsidian 仅作为查看器
- **不改非 markdown 文件**：AI 只处理 .md 文件，附件只移动不修改
- **迁移前必须 git 备份**：迁移命令会检查是否已 git init，未初始化则拒绝执行
