---
name: zet-lint
description: >-
  This skill should be used when the user asks to "check knowledge base", "zet lint",
  "vault health", "any broken links", "check frontmatter", "find orphan notes",
  "what's wrong with the vault", "knowledge base health", or wants a structural
  health check of their Zettelkasten.
user-invocable: true
---

# Zettelkasten Lint — Health Check

Scan the knowledge base for structural problems: orphan notes, broken links, incomplete frontmatter, MOC gaps.

## Reference Specs

- Frontmatter: ${CLAUDE_PLUGIN_ROOT}/references/frontmatter-spec.md
- Vault structure: ${CLAUDE_PLUGIN_ROOT}/references/vault-structure.md

## Checks

Run all checks below, collecting issues into three categories: errors, warnings, suggestions.

### 1. Broken Links

Scan all .md files in 1_zettel/ and 2_maps/ for wikilinks `[[...]]`.
For each wikilink, verify the target note exists somewhere in 1_zettel/ or 2_maps/.
Report any links pointing to non-existent notes.

**Category**: Error

### 2. Orphan Notes

Find notes in 1_zettel/ that have:
- Zero outbound wikilinks in their `## Links` section, OR
- Zero inbound wikilinks (no other note links to them)

Use Grep to search for `[[note-title]]` across all files to detect inbound links.

**Category**: Warning (zero outbound), Error (zero inbound — note is completely disconnected)

### 3. Frontmatter Completeness

Read ${CLAUDE_PLUGIN_ROOT}/references/frontmatter-spec.md for required field lists.

For each file in 1_zettel/:
- Check all 7 required frontmatter fields are present: id, title, created, processed, source, tags, summary
- Check `## Links` section exists in the body

For each file in 2_maps/:
- Check all 4 required fields: title, type, note_count, last_updated

**Category**: Warning

### 4. MOC Coverage

Count how many notes in 1_zettel/ are referenced by at least one MOC in 2_maps/.
Report notes not referenced by any MOC.

**Category**: Suggestion (if the note has tags that match an existing MOC but isn't listed)

### 5. Tag-MOC Alignment

Collect all tags from 1_zettel/ frontmatter. For tags with ≥3 notes:
- Check if a corresponding MOC exists in 2_maps/
- If not, suggest creating one

**Category**: Suggestion

### 6. Empty Files

Find files in 1_zettel/ and 2_maps/ with no content (only frontmatter or completely empty).

**Category**: Error

### 7. Stale MOC Counts

For each MOC in 2_maps/, count actual inbound note links and compare to the `note_count` frontmatter field.
Report mismatches.

**Category**: Warning

## Output

Generate 3_output/lint-report-YYYY-MM-DD.md:

```markdown
# Zettelkasten Lint Report — YYYY-MM-DD

## Summary
- Total notes: N
- Total MOCs: N
- Errors: N
- Warnings: N
- Suggestions: N

## Errors
- [list each error with file path and description]

## Warnings
- [list each warning]

## Suggestions
- [list each suggestion]
```

## Auto-Fix

After presenting the report, offer to auto-fix:
- Create missing MOCs for tags with ≥3 notes
- Update stale MOC note_counts
- Add missing `## Links` sections (with placeholder for user to fill)

Ask user for confirmation before applying fixes. If fixes are applied, commit:

```bash
git add 1_zettel/ 2_maps/
git commit -m "zet: lint auto-fix — N issues resolved"
```
