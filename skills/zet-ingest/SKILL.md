---
name: zet-ingest
description: >-
  This skill should be used when the user asks to "process inbox", "ingest notes",
  "organize inbox", "zet ingest", "digest notes", "process new notes",
  "archive inbox files", "handle new materials", or has new files in 0_inbox/ to process.
  Orchestrates Zettelkasten ingestion: atomizes inbox content into permanent notes,
  builds wikilinks, and maintains MOC navigation. The most frequently used daily command.
user-invocable: true
arguments:
  - name: target
    description: "File path or glob pattern, e.g. 0_inbox/*.md or a specific filename. Omit to process all .md files in 0_inbox/"
    required: false
---

# Zettelkasten Ingest — Process Inbox

Atomize inbox content into permanent notes in 1_zettel/, build links, and update MOC navigation in 2_maps/.

## Reference Specs

Read before processing:
- Frontmatter: ${CLAUDE_PLUGIN_ROOT}/references/frontmatter-spec.md
- Vault structure: ${CLAUDE_PLUGIN_ROOT}/references/vault-structure.md

## Workflow

### 1. Scan Inbox

Parse the target argument to find files. If no target specified, default to all .md files in 0_inbox/.
Use Glob to expand patterns. Count the files found.

If 0_inbox/ is empty or no files match, inform the user and stop.

### 2. Route by Volume

- **≤10 files** → process directly in current conversation (Section 3)
- **>10 files** → batch dispatch via zet-worker agents (Section 4)

### 3. Direct Processing (≤10 files)

For each file:

**a. Read and Analyze**
- Read full content
- Determine source type: original (user-written) / web-clip (URLs, clipped formatting) / import (from other tools, has existing frontmatter)
- Determine created date: filename pattern (e.g. 20260114) > existing frontmatter > file mtime

**b. Atomize**
- Identify independent concepts in the content
- Multiple unrelated topics → split into separate notes
- Single coherent topic → keep as one note
- Each resulting note must be self-contained

**c. Rewrite**
- Reformulate in clear prose, preserving all core information
- Maintain original language (Chinese stays Chinese, English stays English)
- Improve structure: add headings, clean formatting, fix markdown

**d. Generate Frontmatter**
- id: YYYYMMDDHHmm timestamp (increment minutes for splits from same source)
- title, tags, summary, source, created, processed (today)
- Follow ${CLAUDE_PLUGIN_ROOT}/references/frontmatter-spec.md

**e. Build Links**
- Search 1_zettel/ using Grep for semantically related existing notes
- Add `## Links` section with contextual wikilinks
- Every new note must link to ≥1 existing note
- Format: `- Related to [[note-title]]: explanation of connection`
- Prefer cross-domain connections over obvious same-topic links

**f. Write and Move**
- Write to 1_zettel/YYYY-MM/<slug>.md (create YYYY-MM directory if needed)
- Move images/attachments to 4_assets/, update references to `![[image.png]]`
- Delete source file from 0_inbox/

### 4. Batch Dispatch (>10 files)

Split files into batches of ~10. For each batch, dispatch a zet-worker agent with the file list as prompt input. Workers share the working directory.

Workers handle: read → atomize → rewrite → frontmatter → build links → write to 1_zettel/ → move images.
Workers skip: MOC updates, git commit, inbox deletion.

After all workers complete, proceed to Section 5.
Then delete all processed source files from 0_inbox/.

### 5. Update MOCs

Scan all newly created notes in 1_zettel/. For each tag that appears in ≥3 total notes across 1_zettel/:

- If a MOC exists in 2_maps/ for that topic → read it, append new entries with one-line descriptions, update note_count and last_updated
- If no MOC exists → create one in 2_maps/<topic>.md with proper frontmatter
- Add "Related Maps" links between MOCs that share notes

MOC frontmatter follows ${CLAUDE_PLUGIN_ROOT}/references/frontmatter-spec.md (type: map, note_count, last_updated).

### 6. Commit

```bash
git add 1_zettel/ 2_maps/ 4_assets/
git commit -m "zet: ingest N files, created M notes"
```

### 7. Report

Display to user:
- Files processed from inbox
- Atomic notes generated (may exceed file count if splitting occurred)
- Links created
- MOCs created or updated
- Any files that could not be processed (with reason)
