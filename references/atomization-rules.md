# Atomization Rules

Decision rules for splitting inbox files into atomic notes. Every rule serves one goal: **one note = one independently understandable concept**.

## Decision Flow

```
Read inbox file
    │
    ├─ Title test: Can a single atomic title cover the entire content?
    │   ├─ Yes → lean toward keeping as one note
    │   └─ No (requires two+ unrelated titles) → lean toward splitting
    │
    ├─ Tag test: Do all sections share the same tag set?
    │   ├─ Yes → lean toward keeping as one note
    │   └─ No (different sections need entirely different tags) → lean toward splitting
    │
    ├─ Independence test: Can each split part be understood without the original?
    │   ├─ Yes → eligible for splitting
    │   └─ No (incomplete without surrounding context) → do not split
    │
    └─ Final decision: at least two tests point toward splitting → execute split
```

## Split vs Keep Decision Table

| Dimension | Keep as one note | Split into multiple notes |
|-----------|-----------------|--------------------------|
| **Topic relationship** | Multiple aspects of one topic (e.g. "Docker networking" covering bridge/host/overlay) | Unrelated domains combined (e.g. one file covering both Docker and Git) |
| **Independent comprehension** | Split parts would be incomplete without original context | Each part is self-contained |
| **Tag ownership** | All content shares the same tag set | Different sections require entirely different tags |
| **Title test** | A single atomic title covers the whole file | Two+ unrelated titles needed to cover the content |
| **Length signal** | Short text or single-topic long-form | >1000 words with ≥2 independent sections on different concepts |

## Examples

### Keep (do not split)

**Example 1**: An article on "React useEffect" covering deps array, cleanup function, common pitfalls
- Reason: All aspects of useEffect, sharing `[react, hooks, useEffect]` tags
- Title: "React useEffect complete guide" — one title covers it all

**Example 2**: A note on "Git rebase vs merge" with comparison table and usage scenarios
- Reason: Rebase and merge are two sides of the same topic (branch merging strategy)
- Splitting would leave the rebase part needing merge context to be complete

**Example 3**: A 2000-word deep dive on "Kubernetes Pod scheduling"
- Reason: Long but single-topic, with progressive structure between sections

### Split

**Example 1**: "Weekly learning notes" containing SSH config, Python decorators, Docker compose
- Split into 3 notes: each topic is independent, tags are completely different
- SSH → `[ssh, devops]`, Python decorators → `[python, design-patterns]`, Docker → `[docker, devops]`

**Example 2**: A web-clip article — first half on "OAuth2 protocol flow", second half on "JWT token structure"
- Split into 2 notes: related (both authentication) but each independently understandable
- OAuth2 does not require JWT implementation details; JWT does not depend on OAuth2 flow

**Example 3**: A personal note mixing "meeting minutes" and "technical design discussion"
- Split into 2 notes: meeting minutes are event records, technical design is a concept note — different purposes

## Edge Cases

### Gray area: related but independent

When two topics are related but each self-contained (e.g. "HTTP/2" and "HTTP/3"), prefer splitting and connect them via `## Links` cross-references. The power of Zettelkasten lies in links, not in keeping related content in the same file.

### Long single-topic articles

Articles over 2000 words on a single topic should **not be split due to length**. Atomicity refers to conceptual atomicity, not word count. A 3000-word deep dive on "TCP three-way handshake" is a perfectly valid atomic note.

### List-type content

"10 Git tips" style lists:
- If tips are independent and each worth expanding → split into individual notes
- If tips are brief and too thin on their own → keep as one note

Threshold: whether each extracted item has enough substance to form a valuable standalone note (typically ≥200 words of expanded content).

### Incomplete source material

If an inbox file is fragmentary or too short (<100 words), keep as one note without attempting to split. Mark in the frontmatter summary that content may be incomplete.
