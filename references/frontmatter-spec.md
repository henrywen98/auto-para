# Frontmatter Spec

## Permanent Notes (1_zettel/)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | string | yes | Timestamp ID in YYYYMMDDHHmm format, unique identifier |
| title | string | yes | Atomic title — one concept per note |
| created | date (YYYY-MM-DD) | yes | Original material creation date |
| processed | date (YYYY-MM-DD) | yes | Date AI processed this note |
| source | enum: original, web-clip, import | yes | Source type |
| tags | string[] | yes | Tag list |
| summary | string | yes | One-line summary of the note's core idea |

### Tag Normalization

All tags MUST be lowercase kebab-case. Normalize during processing:

| Input | Normalized |
|-------|-----------|
| `AI` | `ai` |
| `LLM` | `llm` |
| `RLHF` | `rlhf` |
| `Machine Learning` | `machine-learning` |
| `product_management` | `product-management` |
| `CI/CD` | `ci-cd` |

Rules:
1. Lowercase everything
2. Replace spaces and underscores with hyphens
3. Replace slashes with hyphens
4. Collapse consecutive hyphens
5. Strip leading/trailing hyphens

Tags are the key that maps notes to MOCs. Case-inconsistent tags cause MOC threshold miscounts and duplicate MOCs.

### Smart Quote Normalization

All frontmatter string values MUST use straight quotes, not smart/curly quotes. Normalize during processing:

| Input | Unicode | Normalized |
|-------|---------|-----------|
| `\u201c` | U+201C left double curly | `"` |
| `\u201d` | U+201D right double curly | `"` |
| `\u2018` | U+2018 left single curly | `'` |
| `\u2019` | U+2019 right single curly | `'` |

Apply this normalization to the raw frontmatter text **before** YAML parsing. Smart quotes inside YAML string values cause `yaml.safe_load()` to fail, which can lead to data loss if the parsing error is not handled safely.

### Source Type Detection

| Type | Signals |
|------|---------|
| original | User-authored content, personal notes, reflections |
| web-clip | URLs, "Source:" headers, clipped formatting, blockquotes from articles |
| import | Structured data, exported from other tools, has existing frontmatter |

### Required Body Structure

Every permanent note must end with a `## Links` section containing contextual wikilinks:

```
## Links
- Related to [[note-title]]: explanation of why these are connected
- See [[other-note]]: explanation of the relationship
```

Rules:
- Every note must link to ≥1 existing note (connection forcing)
- Links must include context — never bare `[[]]` without explanation
- Prefer cross-domain "surprise" connections over obvious same-topic links

## MOC Structure Notes (2_maps/)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| title | string | yes | Topic name |
| type | enum: map, hub | yes | map = topic-level, hub = domain-level |
| note_count | integer | yes | Number of linked notes (auto-updated) |
| last_updated | date (YYYY-MM-DD) | yes | Last update date |

## Query Output (3_output/)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| title | string | yes | Brief question summary |
| query | string | yes | Original query text |
| date | date (YYYY-MM-DD) | yes | Query date |
| sources | string[] | yes | Referenced file path list |
