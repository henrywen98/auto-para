# Zettelkasten

AI-powered Obsidian vault organizer — drop materials into inbox, AI atomizes into permanent notes, builds wikilinks, and maintains MOC navigation.

> A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin. No Python, no dependencies — pure skills + agents.

## Why

Manual note organization doesn't scale. You clip articles, jot ideas, save snippets — and they pile up in a flat folder, unlinked and forgotten.

This plugin applies the [Zettelkasten method](https://zettelkasten.de/introduction/) automatically:

- **Atomize**: One concept per note. Multi-topic files get split.
- **Rewrite**: Clean prose with proper structure, preserving all information.
- **Link**: Every note connects to existing knowledge via contextual wikilinks.
- **Navigate**: MOCs (Maps of Content) auto-generated when topics accumulate.

## Quick Start

### 1. Install

```bash
# In Claude Code
/plugins install github:henrywen98/zettelkasten
```

### 2. Initialize your vault

```bash
cd /path/to/your/obsidian/vault
mkdir -p 0_inbox 1_zettel 2_maps 3_output 4_assets
git init  # recommended for backup safety
```

### 3. Use

Drop files into `0_inbox/`, then open Claude Code in the vault:

```bash
cd /path/to/your/vault
claude
```

```
> /zet-ingest
```

That's it. Your notes appear in `1_zettel/`, organized by year-month, interlinked, with MOCs in `2_maps/`.

## Commands

| Command | Description |
|---------|-------------|
| `/zet-ingest [target]` | Process inbox into atomic Zettelkasten notes |
| `/zet-query <question>` | Q&A against your knowledge base |
| `/zet-lint` | Health check: orphans, broken links, frontmatter |

### /zet-ingest

The main workflow. Scans `0_inbox/`, dispatches AI workers to process files in batches of ~10, then updates MOCs and commits.

Each file goes through: **read → classify → atomize → rewrite → frontmatter → link → write**.

```
0_inbox/weekly-learning.md          →  1_zettel/2026-04/ssh-key-authentication.md
  (SSH + Python decorators + Docker)    1_zettel/2026-04/python-decorator-patterns.md
                                        1_zettel/2026-04/docker-compose-networking.md
```

- Multi-topic files are split into independent atomic notes
- Each note links to ≥1 existing note (connection forcing)
- Original language preserved (Chinese stays Chinese)
- Inbox files deleted after processing

### /zet-query

Ask questions against your accumulated knowledge. Navigates MOCs and note links, reads relevant notes, synthesizes an answer with citations.

```
> /zet-query What do I know about authentication?
```

Output saved to `3_output/` with source references.

### /zet-lint

Structural health check:

- Broken wikilinks
- Orphan notes (no inbound/outbound links)
- Incomplete frontmatter
- MOC coverage gaps
- Stale MOC counts

Offers auto-fix for common issues.

## Vault Structure

```
Vault/
├── 0_inbox/        Drop materials here (deleted after processing)
├── 1_zettel/       Permanent notes, atomic & linked
│   └── YYYY-MM/    Organized by year-month
├── 2_maps/         MOC navigation (auto-maintained)
├── 3_output/       Query results, lint reports
└── 4_assets/       Images and attachments
```

## Note Format

Every permanent note in `1_zettel/` follows this structure:

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

## Architecture

```
/zet-ingest
    │
    ▼
zet-ingest (skill)          ← Orchestrator: scan, batch, dispatch, MOC, commit
    │
    ├── zet-worker (agent)  ← Batch 1: read → atomize → rewrite → link → write
    ├── zet-worker (agent)  ← Batch 2: can link to batch 1's notes
    └── ...                 ← Sequential, each batch commits before next
```

- **zet-ingest**: Pure orchestrator. Scans inbox, splits into batches, dispatches workers, updates MOCs, commits.
- **zet-worker**: File processor. Handles all the reading, atomizing, rewriting, linking, and writing. ~10 files per batch.
- **zet-query**: Independent skill. Navigates MOCs + grep to answer questions.
- **zet-lint**: Independent skill. Scans for structural issues.

Batches are sequential — each commits before the next starts. This means later batches can discover and link to earlier notes, and you can interrupt between batches without losing work.

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- An Obsidian vault (or any markdown folder)
- Git initialized in the vault (recommended)

## Model Recommendations

| Model | Best for |
|-------|----------|
| **Sonnet** | Daily ingestion — fast, reliable, good link quality |
| **Opus** | High-value content — best atomization judgment and cross-domain linking |

## Inspiration

Inspired by [Andrej Karpathy](https://karpathy.ai/)'s ideas on building a personal knowledge base with AI — letting machines handle the organizing so you can focus on thinking.

## License

MIT
