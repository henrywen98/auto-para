# MOC Rules

Shared rules for creating, updating, and validating Maps of Content. Referenced by zet-ingest and zet-lint.

## File Naming

- MOC filename = normalized tag (lowercase kebab-case) + `.md` (e.g. `ai.md`, `machine-learning.md`)
- When matching a tag to an existing MOC: Glob `2_maps/*.md`, strip `.md` suffix, lowercase the filename, then compare with the normalized tag. Tag `ai` matches `AI.md`, `ai.md`, or `Ai.md`.

## Creation Threshold

A MOC is created when a tag (after normalization per frontmatter-spec.md) appears in ≥3 notes across `1_zettel/`. Tags `AI` and `ai` count as the same tag.

## MOC Structure

```markdown
---
title: "Human-Readable Title"    # Capitalized: "AI", "Machine Learning"
type: map                        # map (topic-level) or hub (domain-level)
note_count: 7                    # MUST equal actual count of `- [[` entries below
last_updated: 2026-04-06
---

# Title

## Notes

- [[note-slug]] — one-line description from the note's summary

## Related Maps

- [[other-moc|Display Name]] — brief explanation of relationship
```

## Updating an Existing MOC

1. Read the MOC file
2. Collect the set of note titles already listed (all `- [[...]]` entries under `## Notes`)
3. For each note in `1_zettel/` with this tag that is NOT already listed, append: `- [[note-slug]] — summary`
4. **Recount note_count**: count all `- [[` lines under `## Notes`. Write this number to the `note_count` frontmatter field. NEVER increment the old value — always recount from actual entries.
5. Update `last_updated` to today's date

## Wikilink Display Format

When referencing a MOC whose filename is lowercase but the topic is conventionally uppercase, use the display text syntax: `[[ai|AI]]`, `[[llm|LLM]]`. This keeps wikilinks readable while filenames stay normalized.

## Related Maps Cross-Linking

Two MOCs are related when they share notes (a note tagged with both `ai` and `llm` means the `ai` and `llm` MOCs are related).

After creating or updating MOCs:
1. For each pair of MOCs that share at least one note, check if a cross-link exists in each MOC's `## Related Maps`
2. If not, append `- [[other-moc|Display Name]] — brief relationship description`
3. Do not add duplicate cross-links

## MOC Frontmatter

Per frontmatter-spec.md:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| title | string | yes | Topic name (capitalized for display) |
| type | enum: map, hub | yes | map = topic-level, hub = domain-level |
| note_count | integer | yes | Count of `- [[` entries under `## Notes` (auto-updated, always recounted) |
| last_updated | date | yes | Last update date |
