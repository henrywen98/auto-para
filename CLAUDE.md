# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AutoPARA is a Claude Code plugin that uses AI to automatically organize Obsidian vaults using the PARA methodology. Users drop files into inbox; AI archives them with frontmatter, compiles a wiki knowledge layer, and generates visualizations.

## Development Commands

```bash
# Install Python dependencies (requires uv, Python >=3.12)
cd plugins/autopara/scripts && uv sync

# Scripts — always run from plugins/autopara/scripts/
uv run scan.py <vault_path> [output_path]        # Vault inventory → manifest.json
uv run relink.py <md_file> <old_prefix> <new_prefix>  # Rewrite image links after file moves
uv run validate.py <vault_path> <manifest_path>   # Post-migration validation
```

No test suite or lint configuration exists.

## Architecture

### Two-Layer Design

**AI layer** (skills in `plugins/autopara/skills/<name>/SKILL.md`) handles content analysis, frontmatter generation, and wiki compilation. **Deterministic layer** (Python scripts in `plugins/autopara/scripts/`) handles file scanning, link rewriting, and validation. Design principle: AI decides *what* to do with content; scripts handle *precise file operations*.

Skills reference specs and scripts via `${CLAUDE_PLUGIN_ROOT}` — this variable resolves to `plugins/autopara/` at runtime. All script paths in SKILL.md files use this prefix (e.g., `${CLAUDE_PLUGIN_ROOT}/scripts/relink.py`).

### Key Cross-Cutting Patterns

- **Frontmatter contracts**: Archive files require 7 fields, wiki files require 5, query outputs require 4. Specs in `references/frontmatter-spec.md` — skills and validate.py both enforce these.
- **Vault ownership**: `0_inbox/` belongs to the user, `1_wiki/` belongs to AI, `3_archive/` is the bridge. AI never modifies inbox; users never edit wiki directly.
- **Image handling**: All images centralized in `4_assets/`. After moving a file, always run `relink.py` to update paths. Supports both `![](path)` and `![[path]]` Obsidian syntax.
- **Batch mode**: `para-ingest` has a `--batch` flag that skips user confirmation, wiki updates, and git commits — designed for teammate/parallel processing during `para-migrate`.

### para-migrate Parallel Execution

Migration is the most complex workflow, split into 4 phases:
1. **Scan** (scan.py) → manifest.json
2. **Plan** (AI reads previews in batches, generates migration-plan.json) → pauses for user review
3. **Execute** (~40 files per teammate, isolated via git worktrees, merged after completion)
4. **Validate** (validate.py) → validation-report.md

### Plugin Structure

Two-level nesting: marketplace root (`auto-para/`) contains the plugin (`plugins/autopara/`). Each level has its own `.claude-plugin/` manifest. Install via Claude Code plugin marketplace: search `henrywen98/auto-para`.

## Notes

- `manifest.json`, `validation-report.md`, `.venv/` are gitignored
- `relink.py` can be imported as a module (`from relink import relink_content`) or run standalone
- `scan.py` word count is bilingual: Chinese counted per character, English per word
