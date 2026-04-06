---
name: zet-query
description: >-
  This skill should be used when the user asks to "search knowledge base", "query notes",
  "zet query", "find in my notes", "what did I write about", "search my notes",
  "find notes about", "what do I know about", or wants to ask questions against their
  Zettelkasten knowledge base. Supports quick (MOC + index only) and deep (reads full notes) modes.
user-invocable: true
arguments:
  - name: question
    description: "The question to query against the knowledge base"
    required: true
  - name: depth
    description: "Query depth: quick (MOC + grep only) | deep (reads full notes). Default: deep"
    required: false
---

# Zettelkasten Query — Knowledge Base Q&A

Answer questions by navigating the Zettelkasten through MOCs and note links. This is the "thinking stage" — where the user engages with their accumulated knowledge.

## Reference Specs

- Frontmatter: ${CLAUDE_PLUGIN_ROOT}/references/frontmatter-spec.md
- Vault structure: ${CLAUDE_PLUGIN_ROOT}/references/vault-structure.md

## Workflow

### 1. Read MOC Index

Read all MOC files in 2_maps/ to build a topic overview. Identify which MOCs are relevant to the question based on titles and content.

If 2_maps/ is empty, skip to step 2 and rely entirely on Grep.

### 2. Locate Relevant Notes

Use multiple strategies to find relevant content:

- **MOC navigation**: Follow links from relevant MOCs to 1_zettel/ notes
- **Keyword search**: Use Grep to search 1_zettel/ for question keywords and related terms
- **Link traversal**: When reading a relevant note, follow its `## Links` section to discover connected notes

Collect a set of candidate notes (aim for 5-20 depending on topic breadth).

### 3. Read Notes

Based on depth parameter:

- **quick**: Read only the frontmatter (title, summary, tags) of candidate notes. Synthesize from summaries.
- **deep** (default): Read full content of the most relevant candidate notes (up to ~15). Follow links to read additional notes if needed for completeness.

### 4. Synthesize Answer

Compose a structured answer that:

- Directly addresses the question
- Cites specific notes using wikilinks: [[note-title]]
- Identifies gaps — topics the knowledge base doesn't cover
- Suggests related notes the user might want to explore
- Maintains the language of the question (Chinese question → Chinese answer)

### 5. Save Output

Save the answer to 3_output/YYYY-MM-DD-<slug>.md with frontmatter:

```yaml
---
title: "Brief question summary"
query: "Original question text"
date: YYYY-MM-DD
sources: ["1_zettel/path/to/note1.md", "1_zettel/path/to/note2.md"]
---
```

Inform the user of the output file location.

### 6. Suggest Follow-ups

Based on the query results, suggest:
- Related questions the user might want to explore
- Notes that are tangentially related but might spark new connections
- Gaps in the knowledge base that could be filled
