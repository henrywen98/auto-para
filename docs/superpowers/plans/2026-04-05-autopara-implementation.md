# AutoPARA Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code plugin that auto-organizes an Obsidian vault — inbox materials are archived, tagged, and compiled into a wiki by AI.

**Architecture:** Plugin with 6 skills (ingest/query/lint/viz/status/migrate) + Python scripts for deterministic file operations (scan, validate, link rewriting). Skills orchestrate AI reasoning; scripts handle bulk file I/O. Migration uses teammates + worktrees for parallel processing of 1754 existing files.

**Tech Stack:** Claude Code plugin (SKILL.md), Python 3.14 (uv), python-frontmatter, matplotlib, Marp

---

## File Structure

```
plugins/autopara/
├── .claude-plugin/
│   └── plugin.json                      # Plugin manifest (name, version, description only)
├── README.md                            # Chinese documentation
├── skills/
│   ├── para-ingest/SKILL.md               # Process inbox files → archive + wiki
│   ├── para-query/SKILL.md                # Q&A against knowledge base
│   ├── para-lint/SKILL.md                 # Health check (broken links, orphans, missing summaries)
│   ├── para-viz/SKILL.md                  # Generate visualizations (marp/mermaid/matplotlib)
│   ├── para-status/SKILL.md               # Knowledge base statistics
│   └── para-migrate/SKILL.md              # One-time full vault migration
├── scripts/
│   ├── pyproject.toml                   # uv project config with dependencies
│   ├── scan.py                          # Phase 1: scan vault → manifest.json
│   ├── validate.py                      # Phase 4: post-migration validation
│   └── relink.py                        # Rewrite markdown image/attachment links
└── references/
    ├── frontmatter-spec.md              # Frontmatter schema for archive + wiki files
    └── vault-structure.md               # Target vault directory structure reference
```

**Vault target** (after migration):
```
/Users/henry/Library/Mobile Documents/iCloud~md~obsidian/Documents/Wen/
├── inbox/
├── archive/YYYY-MM/
├── wiki/ (_index.md, _tags.md, concepts/, topics/, viz/)
├── output/
├── assets/
├── .gitignore
└── .gitattributes
```

---

### Task 1: Project Scaffolding + Git Init for Vault

Set up the plugin project structure and initialize git + lfs in the Obsidian vault.

**Files:**
- Create: `plugins/autopara/.claude-plugin/plugin.json`
- Create: `plugins/autopara/README.md`
- Create: `plugins/autopara/references/frontmatter-spec.md`
- Create: `plugins/autopara/references/vault-structure.md`
- Create: (in vault) `.gitignore`
- Create: (in vault) `.gitattributes`

**Plugin root:** `/Users/henry/Documents/1_Project/260405_obsidian_skills_autopara/plugins/autopara/`
**Vault root:** `/Users/henry/Library/Mobile Documents/iCloud~md~obsidian/Documents/Wen/`

- [ ] **Step 1: Create plugin.json**

```json
{
  "name": "autopara",
  "version": "0.1.0",
  "description": "Obsidian 知识库自动整理 — inbox 丢素材，AI 归档 + 编译 wiki + 可视化"
}
```

Write to `plugins/autopara/.claude-plugin/plugin.json`.

- [ ] **Step 2: Create reference docs**

Write `plugins/autopara/references/frontmatter-spec.md`:

```markdown
# Frontmatter 规范

## 原文 (archive/ 下的文件)

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| title | string | yes | 文章标题 |
| created | date (YYYY-MM-DD) | yes | 原始创建日期 |
| archived | date (YYYY-MM-DD) | yes | 归档日期 |
| tags | string[] | yes | 标签列表 |
| summary | string | yes | 一句话摘要 |
| concepts | string[] | yes | 关联的概念列表（对应 wiki/concepts/ 下的文章） |
| source | enum: original, web-clip, import | yes | 来源类型 |

## Wiki 文章 (wiki/ 下的文件)

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| title | string | yes | 概念/主题名称 |
| type | enum: concept, topic, viz | yes | 文章类型 |
| sources | string[] | yes | 引用的 archive 文件路径列表 |
| related | string[] | yes | 相关概念/主题列表 |
| last_compiled | date (YYYY-MM-DD) | yes | 最后编译日期 |
```

Write `plugins/autopara/references/vault-structure.md`:

```markdown
# Vault 目标结构

```
Vault/
├── inbox/          — 唯一入口，用户丢素材
├── archive/        — 原文存档（按年月: archive/YYYY-MM/）
├── wiki/           — AI 编译维护
│   ├── _index.md   — 总索引
│   ├── _tags.md    — 标签索引
│   ├── concepts/   — 概念文章
│   ├── topics/     — 主题聚合页
│   └── viz/        — 可视化输出
├── output/         — 查询结果
└── assets/         — 图片/附件
```

## 原则

- inbox/ 是用户的，wiki/ 是 AI 的，archive/ 是桥梁
- 分类靠标签和 wiki 链接，不靠文件夹层级
- assets/ 统一管理所有图片/附件，md 文件用相对路径引用
```

- [ ] **Step 3: Create README.md**

Write `plugins/autopara/README.md`:

```markdown
# AutoPARA

Obsidian 知识库自动整理插件。

## 功能

- `/para ingest <文件>` — 处理 inbox 中的文件，归档 + 编译进 wiki
- `/para query <问题>` — 对知识库提问
- `/para lint` — 健康检查
- `/para viz <主题>` — 生成可视化
- `/para status` — 查看统计
- `/para migrate` — 全量迁移（仅初始化时用）

## 安装

```bash
/plugins install autopara@henry-hub
```
```

- [ ] **Step 4: Initialize git + lfs in vault**

```bash
cd "/Users/henry/Library/Mobile Documents/iCloud~md~obsidian/Documents/Wen/"
git init
git lfs install
```

- [ ] **Step 5: Create .gitignore in vault**

Write `.gitignore` to the vault root:

```
.obsidian/
.DS_Store
.trash/
node_modules/
```

- [ ] **Step 6: Create .gitattributes in vault**

Write `.gitattributes` to the vault root:

```
*.png filter=lfs diff=lfs merge=lfs -text
*.jpg filter=lfs diff=lfs merge=lfs -text
*.jpeg filter=lfs diff=lfs merge=lfs -text
*.gif filter=lfs diff=lfs merge=lfs -text
*.webp filter=lfs diff=lfs merge=lfs -text
*.pdf filter=lfs diff=lfs merge=lfs -text
*.mp4 filter=lfs diff=lfs merge=lfs -text
*.zip filter=lfs diff=lfs merge=lfs -text
*.pptx filter=lfs diff=lfs merge=lfs -text
*.docx filter=lfs diff=lfs merge=lfs -text
*.xlsx filter=lfs diff=lfs merge=lfs -text
```

- [ ] **Step 7: Initial commit of vault (current state as backup)**

```bash
cd "/Users/henry/Library/Mobile Documents/iCloud~md~obsidian/Documents/Wen/"
git add -A
git commit -m "chore: initial vault backup before AutoPARA migration"
```

- [ ] **Step 8: Commit plugin scaffolding**

```bash
cd "/Users/henry/Documents/1_Project/260405_obsidian_skills_autopara"
git add plugins/
git commit -m "feat: scaffold autopara plugin structure with reference docs"
```

---

### Task 2: Python Scripts (scan + validate + relink)

Deterministic scripts for file scanning, link rewriting, and post-migration validation. These run without AI involvement.

**Files:**
- Create: `plugins/autopara/scripts/pyproject.toml`
- Create: `plugins/autopara/scripts/scan.py`
- Create: `plugins/autopara/scripts/relink.py`
- Create: `plugins/autopara/scripts/validate.py`
- Test: all scripts tested inline with assertions

**All paths below relative to:** `plugins/autopara/scripts/`

- [ ] **Step 1: Create pyproject.toml**

```toml
[project]
name = "autopara-scripts"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "python-frontmatter>=1.1.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

- [ ] **Step 2: Install dependencies**

```bash
cd "/Users/henry/Documents/1_Project/260405_obsidian_skills_autopara/plugins/autopara/scripts"
uv sync
```

- [ ] **Step 3: Write scan.py**

```python
#!/usr/bin/env python3
"""扫描 Obsidian vault，生成 manifest.json。

用法: uv run scan.py <vault_path> [output_path]
输出: manifest.json，包含每个 md 文件的元数据和 preview
"""

import json
import re
import sys
from datetime import datetime
from pathlib import Path

import frontmatter


def extract_date_from_filename(name: str) -> str | None:
    """从文件名中提取日期，如 '20260114-2042-xxx.md' → '2026-01-14'"""
    m = re.match(r"(\d{4})(\d{2})(\d{2})", name)
    if m:
        return f"{m.group(1)}-{m.group(2)}-{m.group(3)}"
    return None


def extract_images(content: str) -> list[str]:
    """提取 markdown 和 obsidian 格式的图片引用"""
    patterns = [
        r"!\[.*?\]\((.+?)\)",        # ![alt](path)
        r"!\[\[(.+?)\]\]",           # ![[path]]
    ]
    images = []
    for p in patterns:
        images.extend(re.findall(p, content))
    return images


def count_words(text: str) -> int:
    """中英文混合字数统计"""
    chinese = len(re.findall(r"[\u4e00-\u9fff]", text))
    english = len(re.findall(r"[a-zA-Z]+", text))
    return chinese + english


def scan_vault(vault_path: Path) -> dict:
    skip_dirs = {".obsidian", ".git", ".trash", "node_modules"}
    files = []
    attachments: dict[str, list[str]] = {}

    md_files = sorted(
        f for f in vault_path.rglob("*.md")
        if not any(part in skip_dirs for part in f.parts)
    )

    for md_file in md_files:
        rel = str(md_file.relative_to(vault_path))
        content = md_file.read_text(encoding="utf-8", errors="replace")
        post = frontmatter.loads(content)
        images = extract_images(content)

        created = extract_date_from_filename(md_file.name)
        if not created and "created" in post.metadata:
            created = str(post.metadata["created"])
        if not created:
            ts = md_file.stat().st_mtime
            created = datetime.fromtimestamp(ts).strftime("%Y-%m-%d")

        preview = post.content[:200].replace("\n", " ").strip()

        files.append({
            "path": rel,
            "size_bytes": md_file.stat().st_size,
            "word_count": count_words(post.content),
            "created": created,
            "has_frontmatter": bool(post.metadata),
            "has_images": len(images) > 0,
            "images": images,
            "preview": preview,
        })

        for img in images:
            attachments.setdefault(img, []).append(rel)

    # 扫描非 md 附件
    all_attachments = []
    for f in vault_path.rglob("*"):
        if f.is_file() and f.suffix.lower() != ".md" and not any(
            part in skip_dirs for part in f.parts
        ):
            rel = str(f.relative_to(vault_path))
            all_attachments.append({
                "path": rel,
                "size_bytes": f.stat().st_size,
                "extension": f.suffix.lower(),
                "referenced_by": attachments.get(f.name, []),
            })

    return {
        "scan_date": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
        "vault_path": str(vault_path),
        "total_md_files": len(files),
        "total_attachments": len(all_attachments),
        "files": files,
        "attachments": all_attachments,
    }


def main():
    if len(sys.argv) < 2:
        print("用法: uv run scan.py <vault_path> [output_path]")
        sys.exit(1)

    vault_path = Path(sys.argv[1])
    output_path = Path(sys.argv[2]) if len(sys.argv) > 2 else Path("manifest.json")

    if not vault_path.is_dir():
        print(f"错误: {vault_path} 不是有效目录")
        sys.exit(1)

    manifest = scan_vault(vault_path)
    output_path.write_text(json.dumps(manifest, ensure_ascii=False, indent=2))
    print(f"扫描完成: {manifest['total_md_files']} 篇 md, {manifest['total_attachments']} 个附件")
    print(f"输出: {output_path}")


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: Test scan.py against the real vault**

```bash
cd "/Users/henry/Documents/1_Project/260405_obsidian_skills_autopara/plugins/autopara/scripts"
uv run scan.py "/Users/henry/Library/Mobile Documents/iCloud~md~obsidian/Documents/Wen/" manifest.json
```

Expected: prints `扫描完成: ~1754 篇 md, ~1400+ 个附件` and creates `manifest.json`.

Verify:
```bash
python3 -c "import json; d=json.load(open('manifest.json')); print(d['total_md_files'], d['total_attachments'])"
```

- [ ] **Step 5: Write relink.py**

```python
#!/usr/bin/env python3
"""重写 markdown 文件中的图片/附件链接，指向新的 assets/ 目录。

用法: uv run relink.py <md_file> <old_base> <new_base>
示例: uv run relink.py note.md "img/" "../assets/"

也可作为模块导入: from relink import relink_content
"""

import re
import sys
from pathlib import Path


def relink_content(content: str, path_map: dict[str, str]) -> str:
    """将 content 中引用的路径按 path_map 替换。

    path_map: {"old/path/img.png": "assets/img.png", ...}
    处理两种格式:
      - ![alt](old/path/img.png) → ![alt](assets/img.png)
      - ![[old/path/img.png]] → ![[assets/img.png]]
    """
    def replace_md(m: re.Match) -> str:
        alt = m.group(1)
        old_path = m.group(2)
        # 尝试匹配完整路径或仅文件名
        new_path = path_map.get(old_path)
        if not new_path:
            basename = Path(old_path).name
            new_path = path_map.get(basename, old_path)
        return f"![{alt}]({new_path})"

    def replace_wiki(m: re.Match) -> str:
        old_path = m.group(1)
        new_path = path_map.get(old_path)
        if not new_path:
            basename = Path(old_path).name
            new_path = path_map.get(basename, old_path)
        return f"![[{new_path}]]"

    content = re.sub(r"!\[([^\]]*)\]\(([^)]+)\)", replace_md, content)
    content = re.sub(r"!\[\[([^\]]+)\]\]", replace_wiki, content)
    return content


def main():
    if len(sys.argv) < 4:
        print("用法: uv run relink.py <md_file> <old_prefix> <new_prefix>")
        sys.exit(1)

    md_file = Path(sys.argv[1])
    old_prefix = sys.argv[2]
    new_prefix = sys.argv[3]

    content = md_file.read_text(encoding="utf-8")

    # 简单前缀替换模式
    path_map = {}
    for m in re.finditer(r"!\[.*?\]\((.+?)\)", content):
        old = m.group(1)
        if old.startswith(old_prefix):
            path_map[old] = old.replace(old_prefix, new_prefix, 1)
    for m in re.finditer(r"!\[\[(.+?)\]\]", content):
        old = m.group(1)
        if old.startswith(old_prefix):
            path_map[old] = old.replace(old_prefix, new_prefix, 1)

    new_content = relink_content(content, path_map)

    if new_content != content:
        md_file.write_text(new_content, encoding="utf-8")
        print(f"重写了 {len(path_map)} 个链接: {md_file}")
    else:
        print(f"无需修改: {md_file}")


if __name__ == "__main__":
    main()
```

- [ ] **Step 6: Test relink.py**

```bash
cd "/Users/henry/Documents/1_Project/260405_obsidian_skills_autopara/plugins/autopara/scripts"
python3 -c "
from relink import relink_content

# 测试 markdown 格式
content = '![截图](img/test.png)\n正文\n![[img/other.jpg]]'
path_map = {'img/test.png': '../assets/test.png', 'img/other.jpg': '../assets/other.jpg'}
result = relink_content(content, path_map)
assert '![截图](../assets/test.png)' in result, f'md relink failed: {result}'
assert '![[../assets/other.jpg]]' in result, f'wiki relink failed: {result}'

# 测试无匹配不修改
result2 = relink_content('![x](keep.png)', {})
assert result2 == '![x](keep.png)', 'should not modify unmatched'

print('relink tests passed')
"
```

Expected: `relink tests passed`

- [ ] **Step 7: Write validate.py**

```python
#!/usr/bin/env python3
"""迁移后验证：对比 manifest.json 与实际 vault 状态。

用法: uv run validate.py <vault_path> <manifest_path>
输出: validation-report.md
"""

import json
import re
import sys
from pathlib import Path

import frontmatter

REQUIRED_ARCHIVE_FIELDS = {"title", "created", "archived", "tags", "summary", "concepts", "source"}
REQUIRED_WIKI_FIELDS = {"title", "type", "sources", "related", "last_compiled"}


def validate(vault_path: Path, manifest_path: Path) -> str:
    manifest = json.loads(manifest_path.read_text())
    original_count = manifest["total_md_files"]

    skip_dirs = {".obsidian", ".git", ".trash", "node_modules"}
    current_files = [
        f for f in vault_path.rglob("*.md")
        if not any(part in skip_dirs for part in f.parts)
    ]

    issues: list[str] = []
    warnings: list[str] = []
    stats = {
        "original_count": original_count,
        "current_count": len(current_files),
        "archive_count": 0,
        "wiki_count": 0,
        "inbox_count": 0,
        "empty_files": 0,
        "missing_frontmatter": 0,
        "broken_links": 0,
    }

    for f in current_files:
        rel = str(f.relative_to(vault_path))
        content = f.read_text(encoding="utf-8", errors="replace")

        # 空文件检查
        if len(content.strip()) == 0:
            issues.append(f"空文件: {rel}")
            stats["empty_files"] += 1
            continue

        # 分区统计
        if rel.startswith("archive/"):
            stats["archive_count"] += 1
            post = frontmatter.loads(content)
            missing = REQUIRED_ARCHIVE_FIELDS - set(post.metadata.keys())
            if missing:
                issues.append(f"archive frontmatter 缺失字段 {missing}: {rel}")
                stats["missing_frontmatter"] += 1

        elif rel.startswith("wiki/") and not rel.startswith("wiki/_"):
            stats["wiki_count"] += 1
            post = frontmatter.loads(content)
            missing = REQUIRED_WIKI_FIELDS - set(post.metadata.keys())
            if missing:
                issues.append(f"wiki frontmatter 缺失字段 {missing}: {rel}")
                stats["missing_frontmatter"] += 1

        elif rel.startswith("inbox/"):
            stats["inbox_count"] += 1

        # 图片链接有效性
        for img in re.findall(r"!\[.*?\]\((.+?)\)", content):
            if img.startswith("http"):
                continue
            img_path = (f.parent / img).resolve()
            if not img_path.exists():
                issues.append(f"断链: {rel} → {img}")
                stats["broken_links"] += 1

        for img in re.findall(r"!\[\[(.+?)\]\]", content):
            candidates = list(vault_path.rglob(Path(img).name))
            if not candidates:
                issues.append(f"断链: {rel} → ![[{img}]]")
                stats["broken_links"] += 1

    # 文件数量对比
    if stats["current_count"] < original_count:
        diff = original_count - stats["current_count"]
        issues.append(f"文件数减少: 原始 {original_count}, 当前 {stats['current_count']} (差 {diff})")

    # 生成报告
    report = f"""# 迁移验证报告

## 统计

| 指标 | 值 |
|------|-----|
| 原始文件数 | {stats['original_count']} |
| 当前文件数 | {stats['current_count']} |
| archive/ | {stats['archive_count']} |
| wiki/ | {stats['wiki_count']} |
| inbox/ | {stats['inbox_count']} |
| 空文件 | {stats['empty_files']} |
| 缺失 frontmatter | {stats['missing_frontmatter']} |
| 断链 | {stats['broken_links']} |

## 问题 ({len(issues)} 项)

"""
    if issues:
        for issue in issues:
            report += f"- {issue}\n"
    else:
        report += "无问题。\n"

    if warnings:
        report += f"\n## 警告 ({len(warnings)} 项)\n\n"
        for w in warnings:
            report += f"- {w}\n"

    return report


def main():
    if len(sys.argv) < 3:
        print("用法: uv run validate.py <vault_path> <manifest_path>")
        sys.exit(1)

    vault_path = Path(sys.argv[1])
    manifest_path = Path(sys.argv[2])
    report = validate(vault_path, manifest_path)

    output = vault_path / "validation-report.md"
    output.write_text(report, encoding="utf-8")
    print(f"验证报告已生成: {output}")

    if "文件数减少" in report or "空文件" in report:
        print("⚠ 发现问题，请查看报告")
        sys.exit(1)
    else:
        print("验证通过")


if __name__ == "__main__":
    main()
```

- [ ] **Step 8: Commit scripts**

```bash
cd "/Users/henry/Documents/1_Project/260405_obsidian_skills_autopara"
git add plugins/autopara/scripts/
git commit -m "feat: add scan, relink, and validate Python scripts"
```

---

### Task 3: para-migrate Skill

The one-time migration skill that orchestrates the 4-phase vault migration. This is the most critical skill — it coordinates scan → plan → teammates execute → validate.

**Files:**
- Create: `plugins/autopara/skills/para-migrate/SKILL.md`

- [ ] **Step 1: Write para-migrate SKILL.md**

```markdown
---
name: para-migrate
description: >-
  一次性全量迁移 Obsidian vault 到 AutoPARA 结构。
  当用户说「迁移知识库」「para migrate」「初始化 autopara」「重构 vault」时触发。
user-invocable: true
arguments:
  - name: vault_path
    description: "Obsidian vault 路径"
    required: true
---

# KB Migrate — 全量迁移

将现有 Obsidian vault 迁移到 AutoPARA 结构（inbox/archive/wiki/assets）。

**前置条件**: vault 已 git init + git lfs install，且有初始 commit 作为备份。

## 迁移流程

```
Phase 1: 扫描 (脚本) → manifest.json
Phase 2: 规划 (AI) → migration-plan.json
Phase 3: 执行 (teammates) → 文件迁移 + frontmatter
Phase 4: 验证 (脚本) → validation-report.md
```

## Phase 1: 扫描

运行扫描脚本，生成 vault 的完整清单：

```bash
cd "${CLAUDE_PLUGIN_ROOT}/scripts"
uv run scan.py "<vault_path>" "<vault_path>/manifest.json"
```

产出 `manifest.json`，向用户报告统计数据（文件数、附件数、大小）。

## Phase 2: 规划

manifest.json 可能很大（1754 个文件条目），按以下流程处理：

1. 先创建目标目录结构：

```bash
cd "<vault_path>"
mkdir -p inbox archive wiki/concepts wiki/topics wiki/viz output assets
```

2. 读取 manifest.json，按原始文件夹分批（每批 ~50 个文件条目）
3. 对每批：读取文件的 preview，生成该批的 migration-plan 片段：
   - target: 目标路径（`archive/YYYY-MM/<slug>.md`）
   - frontmatter: AI 根据 preview 生成的 tags, summary, concepts
   - restructure: 是否需要重构内容（true/false）
4. 脚本合并所有片段为完整的 `migration-plan.json`
5. **暂停，请用户审阅 migration-plan.json**

migration-plan.json 格式：

```json
{
  "actions": [
    {
      "source": "00_Inbox/20260114-2042-SSH Key - henrywen.md",
      "target": "archive/2026-01/ssh-key-henrywen.md",
      "frontmatter": {
        "title": "SSH Key 配置指南",
        "tags": ["ssh", "devops"],
        "summary": "SSH Key 生成与配置的完整指南",
        "concepts": ["SSH密钥认证"],
        "source": "original"
      },
      "restructure": false,
      "images": ["img/ssh-1.png"],
      "image_targets": ["assets/ssh-1.png"]
    }
  ],
  "concept_candidates": ["SSH密钥认证", "Docker部署", "Git工作流"],
  "duplicate_groups": [],
  "outdated": []
}
```

## Phase 3: 并行执行 (teammates)

用户审阅通过后，按以下步骤执行：

1. 将 migration-plan.json 按 ~40 个 action 分批
2. 为每个批次派发一个 teammate（使用 worktree 隔离）
3. 每个 teammate 的任务：
   - 读取分配到的 action 列表
   - 对每个 action：
     a. 读取源文件全文
     b. 写入 frontmatter（来自 plan）
     c. 如果 restructure=true，重构 markdown 内容（改善结构、补充缺失信息）
     d. 移动到 target 路径（`git mv` 或新建 + 删除）
     e. 移动关联图片到 assets/
     f. 运行 relink 更新图片链接
   - 完成后 commit 本批次的变更
4. 所有 teammate 完成后，合并回主分支
5. 清理旧的空目录

**teammate 执行模板**：

```
你是 AutoPARA 迁移 worker。你的任务是处理一批文件迁移。

vault 路径: <vault_path>
relink 脚本路径: ${CLAUDE_PLUGIN_ROOT}/scripts/relink.py

你的批次 actions (JSON):
<batch_actions>

对每个 action：
1. 读取 source 文件
2. 在文件顶部写入 frontmatter（保留原有内容）
3. 如果 restructure=true，改善 markdown 结构（不丢失任何信息）
4. 创建 target 目录（如不存在）
5. 移动文件到 target 路径
6. 移动 images 到 image_targets
7. 用 relink.py 更新链接

全部完成后 commit:
git add -A && git commit -m "migrate: batch N (M files)"
```

## Phase 4: 验证

```bash
cd "${CLAUDE_PLUGIN_ROOT}/scripts"
uv run validate.py "<vault_path>" "<vault_path>/manifest.json"
```

向用户展示 validation-report.md 的内容。如果有问题，列出并协助修复。

## Phase 5: Wiki 编译（首次）

验证通过后，进入首次 wiki 编译：

1. 读取所有 archive 文件的 frontmatter
2. 统计 concepts 和 tags 的频率
3. 为高频 concepts（出现 >= 2 次）生成 wiki/concepts/ 文章
4. 为高频 tags（出现 >= 5 次）生成 wiki/topics/ 聚合页
5. 生成 wiki/_index.md（所有 archive 文件列表 + 简述）
6. 生成 wiki/_tags.md（所有标签及其关联文件）
7. commit wiki

## 安全机制

- 迁移前检查 git status，必须有初始 commit
- 每个 phase 之间暂停确认
- manifest.json 保留作为回滚参考
- 任何 phase 失败可 git reset 回到上一个 commit
```

- [ ] **Step 2: Commit**

```bash
cd "/Users/henry/Documents/1_Project/260405_obsidian_skills_autopara"
git add plugins/autopara/skills/para-migrate/
git commit -m "feat: add para-migrate skill for full vault migration"
```

---

### Task 4: para-ingest Skill

The daily-use skill for processing new files from inbox.

**Files:**
- Create: `plugins/autopara/skills/para-ingest/SKILL.md`

- [ ] **Step 1: Write para-ingest SKILL.md**

```markdown
---
name: para-ingest
description: >-
  处理 inbox 中的文件，归档到 archive 并编译进 wiki。
  当用户说「处理笔记」「整理 inbox」「para ingest」「归档这篇」时触发。
user-invocable: true
arguments:
  - name: target
    description: "要处理的文件路径或 glob 模式，如 inbox/*.md 或具体文件名"
    required: true
---

# KB Ingest — 处理 inbox 文件

将 inbox/ 中的指定文件归档到 archive/，同时更新 wiki。

## 流程

### 1. 定位文件

解析 target 参数，找到要处理的文件列表。如果是 glob 模式，展开匹配。
向用户确认待处理文件列表。

### 2. 逐文件处理

对每个文件：

**a. 分析内容**
- 读取文件全文
- 判断 source 类型（original / web-clip / import）
- 生成 frontmatter: title, tags, summary, concepts
- 确定 created 日期（从文件名提取 > frontmatter > 文件修改时间）
- 确定目标路径: `archive/YYYY-MM/<slug>.md`

**b. 写入 frontmatter**
- 在文件顶部插入 YAML frontmatter
- 遵循 `${CLAUDE_PLUGIN_ROOT}/references/frontmatter-spec.md` 规范

**c. 移动文件**
- 移动到 `archive/YYYY-MM/` 目录
- 移动关联图片/附件到 `assets/`
- 运行 relink 更新链接：
```bash
cd "${CLAUDE_PLUGIN_ROOT}/scripts"
uv run relink.py "<file>" "<old_prefix>" "../assets/"
```

**d. 更新 wiki**
- 读取 `wiki/_index.md`，追加新条目
- 读取 `wiki/_tags.md`，更新标签索引
- 对每个 concept：
  - 如果 `wiki/concepts/<concept>.md` 存在 → 追加 source 到 sources 列表
  - 如果不存在且 AI 判断值得新建 → 创建 concept 文章

### 3. Commit

```bash
git add -A
git commit -m "kb: ingest N files from inbox"
```

### 4. 报告

向用户展示处理结果：
- 处理了几篇
- 生成/更新了哪些 concepts
- 新增了哪些 tags
```

- [ ] **Step 2: Commit**

```bash
cd "/Users/henry/Documents/1_Project/260405_obsidian_skills_autopara"
git add plugins/autopara/skills/para-ingest/
git commit -m "feat: add para-ingest skill for inbox processing"
```

---

### Task 5: para-query Skill

**Files:**
- Create: `plugins/autopara/skills/para-query/SKILL.md`

- [ ] **Step 1: Write para-query SKILL.md**

```markdown
---
name: para-query
description: >-
  对知识库提问，AI 研究并生成回答。
  当用户说「查知识库」「para query」「问知识库」「在笔记里找」时触发。
user-invocable: true
arguments:
  - name: question
    description: "要查询的问题"
    required: true
  - name: depth
    description: "查询深度: quick（快速，只看索引）| deep（深入，读原文）"
    required: false
---

# KB Query — 知识库问答

基于 wiki 和 archive 回答用户的问题。

## 流程

### 1. 读取索引

先读取以下文件建立全局概览：
- `wiki/_index.md` — 所有文章列表
- `wiki/_tags.md` — 标签索引

### 2. 定位相关内容

根据问题关键词：
- 在 _index.md 中匹配相关文章
- 在 _tags.md 中匹配相关标签
- 用 Grep 在 archive/ 和 wiki/ 中搜索关键词

### 3. 深入阅读

根据 depth 参数：
- **quick**（默认）：只读 wiki/concepts/ 和 wiki/topics/ 中的匹配文章
- **deep**：还读取 archive/ 中的原文

### 4. 生成回答

综合所有读到的内容，生成结构化回答。

### 5. 保存输出

将回答保存到 `output/YYYY-MM-DD-<slug>.md`：

```yaml
---
title: <问题简述>
query: "<原始问题>"
date: YYYY-MM-DD
sources: [引用的文件路径列表]
---
```

告知用户输出文件位置，询问是否要将此输出归档到 wiki（作为新的 topic 或 concept 文章）。
```

- [ ] **Step 2: Commit**

```bash
cd "/Users/henry/Documents/1_Project/260405_obsidian_skills_autopara"
git add plugins/autopara/skills/para-query/
git commit -m "feat: add para-query skill for knowledge base Q&A"
```

---

### Task 6: para-lint Skill

**Files:**
- Create: `plugins/autopara/skills/para-lint/SKILL.md`

- [ ] **Step 1: Write para-lint SKILL.md**

```markdown
---
name: para-lint
description: >-
  知识库健康检查：断链、孤页、标签不一致、缺失摘要。
  当用户说「检查知识库」「para lint」「知识库健康」时触发。
user-invocable: true
---

# KB Lint — 知识库健康检查

扫描知识库，发现并报告数据完整性问题。

## 检查项

### 1. 断链检查
- 扫描所有 md 文件中的图片链接（`![](path)` 和 `![[path]]`）
- 检查链接目标是否存在
- 扫描 wiki 文件中的 sources 字段，检查引用的 archive 文件是否存在

### 2. 孤页检查
- 找出 archive/ 中没有被任何 wiki 文章引用的文件
- 这些文件需要编译进 wiki

### 3. Frontmatter 完整性
- archive/ 文件必须有: title, created, archived, tags, summary, concepts, source
- wiki/ 文件必须有: title, type, sources, related, last_compiled
- 列出缺失字段的文件

### 4. 标签一致性
- 对比 `_tags.md` 索引与实际文件中的 tags
- 找出索引中有但文件中没有的标签（幽灵标签）
- 找出文件中有但索引中没有的标签（遗漏标签）

### 5. Concept 覆盖度
- 统计 archive 文件 frontmatter 中引用的 concepts
- 检查对应的 wiki/concepts/ 文章是否存在
- 建议为高频但缺失的 concept 创建文章

## 输出

生成 `output/lint-report-YYYY-MM-DD.md`，按严重程度分类：

- **错误**: 断链、空文件、文件数不一致
- **警告**: 缺失 frontmatter 字段、孤页
- **建议**: 新 concept 候选、标签清理

询问用户是否要自动修复可修复的问题（如更新索引、补全缺失的 concept 文章）。
```

- [ ] **Step 2: Commit**

```bash
cd "/Users/henry/Documents/1_Project/260405_obsidian_skills_autopara"
git add plugins/autopara/skills/para-lint/
git commit -m "feat: add para-lint skill for knowledge base health check"
```

---

### Task 7: para-viz Skill

**Files:**
- Create: `plugins/autopara/skills/para-viz/SKILL.md`

- [ ] **Step 1: Write para-viz SKILL.md**

```markdown
---
name: para-viz
description: >-
  生成知识库可视化：Marp 幻灯片、Mermaid 图、Matplotlib 图表。
  当用户说「生成幻灯片」「para viz」「可视化」「知识图谱」时触发。
user-invocable: true
arguments:
  - name: topic
    description: "主题或问题"
    required: true
  - name: format
    description: "输出格式: marp | mermaid | matplotlib（默认 marp）"
    required: false
---

# KB Viz — 知识库可视化

基于知识库内容生成可视化输出。

## 格式

### Marp (默认)
生成 Marp 格式的 markdown 幻灯片，保存到 `wiki/viz/<slug>.md`。

Marp 文件格式：
```yaml
---
marp: true
theme: default
paginate: true
---
```

适用于：主题概览、学习笔记、知识分享

### Mermaid
生成 Mermaid 图表，嵌入到 markdown 文件中。

适用于：概念关系图、流程图、知识图谱

### Matplotlib
生成 Python matplotlib 脚本，运行后输出 PNG 到 `assets/viz/`。

```bash
cd "${CLAUDE_PLUGIN_ROOT}/scripts"
uv run viz_script.py
```

适用于：标签分布统计、时间线、词频分析

## 流程

1. 读取 wiki/_index.md 和 wiki/_tags.md 定位相关内容
2. 读取相关 wiki 文章和 archive 原文
3. 根据 format 生成可视化
4. 保存到 wiki/viz/ 或 assets/viz/
5. 在 wiki/_index.md 中记录新的 viz 条目
6. Commit
```

- [ ] **Step 2: Commit**

```bash
cd "/Users/henry/Documents/1_Project/260405_obsidian_skills_autopara"
git add plugins/autopara/skills/para-viz/
git commit -m "feat: add para-viz skill for knowledge visualization"
```

---

### Task 8: para-status Skill

**Files:**
- Create: `plugins/autopara/skills/para-status/SKILL.md`

- [ ] **Step 1: Write para-status SKILL.md**

```markdown
---
name: para-status
description: >-
  查看知识库统计信息：文章数、标签分布、待处理数。
  当用户说「知识库状态」「para status」「知识库统计」时触发。
user-invocable: true
---

# KB Status — 知识库统计

快速展示知识库的当前状态。

## 统计内容

1. **文件数统计**
   - inbox/ 待处理数
   - archive/ 已归档数
   - wiki/concepts/ 概念文章数
   - wiki/topics/ 主题文章数
   - wiki/viz/ 可视化数
   - output/ 查询输出数

使用 Bash 统计：
```bash
VAULT="<vault_path>"
echo "inbox: $(find "$VAULT/inbox" -name '*.md' 2>/dev/null | wc -l)"
echo "archive: $(find "$VAULT/archive" -name '*.md' 2>/dev/null | wc -l)"
echo "concepts: $(find "$VAULT/wiki/concepts" -name '*.md' 2>/dev/null | wc -l)"
echo "topics: $(find "$VAULT/wiki/topics" -name '*.md' 2>/dev/null | wc -l)"
echo "viz: $(find "$VAULT/wiki/viz" -name '*.md' 2>/dev/null | wc -l)"
echo "output: $(find "$VAULT/output" -name '*.md' 2>/dev/null | wc -l)"
```

2. **标签分布**（Top 20）
   - 读取 wiki/_tags.md，展示最常用的 20 个标签及其文件数

3. **最近活动**
   - 最近 5 篇归档的文件
   - 最近 5 篇更新的 wiki 文章

4. **健康概览**
   - inbox 积压数量
   - 孤页数量（archive 中未被 wiki 引用的）
   - 如果积压 > 10 或孤页 > 20，建议运行 `/para lint`

以表格形式向用户展示。
```

- [ ] **Step 2: Commit**

```bash
cd "/Users/henry/Documents/1_Project/260405_obsidian_skills_autopara"
git add plugins/autopara/skills/para-status/
git commit -m "feat: add para-status skill for knowledge base statistics"
```

---

### Task 9: Integration Test — Full Dry Run

End-to-end test using a small sample of real vault files.

**Files:** No new files, tests run against a temp copy of vault data.

- [ ] **Step 1: Create a test sample**

```bash
VAULT="/Users/henry/Library/Mobile Documents/iCloud~md~obsidian/Documents/Wen"
TEST_DIR="/tmp/autopara-test"
rm -rf "$TEST_DIR"
mkdir -p "$TEST_DIR/00_Inbox"

# 复制 5 篇 inbox 文件作为测试样本
cp "$VAULT/00_Inbox/"*.md "$TEST_DIR/00_Inbox/" 2>/dev/null
# 复制几篇其他目录的文件
mkdir -p "$TEST_DIR/03 Resources📚"
ls "$VAULT/03 Resources📚/"*.md 2>/dev/null | head -5 | xargs -I{} cp "{}" "$TEST_DIR/03 Resources📚/"

cd "$TEST_DIR" && git init && git lfs install && git add -A && git commit -m "test: initial"
```

- [ ] **Step 2: Run scan.py against test sample**

```bash
cd "/Users/henry/Documents/1_Project/260405_obsidian_skills_autopara/plugins/autopara/scripts"
uv run scan.py /tmp/autopara-test /tmp/autopara-test/manifest.json
cat /tmp/autopara-test/manifest.json | python3 -c "import json,sys; d=json.load(sys.stdin); print(f'Files: {d[\"total_md_files\"]}, Attachments: {d[\"total_attachments\"]}')"
```

Expected: prints file and attachment counts matching the copied files.

- [ ] **Step 3: Verify scan output structure**

```bash
python3 -c "
import json
d = json.load(open('/tmp/autopara-test/manifest.json'))
for f in d['files']:
    assert 'path' in f and 'preview' in f and 'created' in f, f'bad entry: {f}'
print(f'All {len(d[\"files\"])} entries valid')
"
```

- [ ] **Step 4: Run validate.py (pre-migration, should report issues)**

```bash
cd "/Users/henry/Documents/1_Project/260405_obsidian_skills_autopara/plugins/autopara/scripts"
uv run validate.py /tmp/autopara-test /tmp/autopara-test/manifest.json
cat /tmp/autopara-test/validation-report.md
```

Expected: reports missing frontmatter on all files (since no migration has happened yet). This confirms the validator works.

- [ ] **Step 5: Cleanup**

```bash
rm -rf /tmp/autopara-test
```

- [ ] **Step 6: Final commit of any adjustments**

```bash
cd "/Users/henry/Documents/1_Project/260405_obsidian_skills_autopara"
git add -A
git diff --cached --stat
# Only commit if there are changes
git diff --cached --quiet || git commit -m "fix: adjustments from integration test"
```

---

## Self-Review Checklist

- [x] **Spec coverage**: All 6 commands (ingest/query/lint/viz/status/migrate) have dedicated skills. Migration strategy covers all 4 phases (scan/plan/execute/validate). Git + LFS setup included.
- [x] **Placeholder scan**: No TBD/TODO. All code blocks are complete. All file paths are exact.
- [x] **Type consistency**: frontmatter field names match across spec, reference docs, scan.py, validate.py, and skill instructions. `concepts` (not `concept`), `sources` (not `source` in wiki), `source` (not `sources` in archive).
