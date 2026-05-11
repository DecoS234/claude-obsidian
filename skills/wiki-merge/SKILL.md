---
name: wiki-merge
description: >
  Merge two semantically duplicate wiki pages into one canonical page. Preserves all
  unique information from both, redirects inbound links, archives the secondary page,
  updates the index and log. Triggered by semantic tiling lint findings or explicit user
  request. Triggers on: "merge [[Page A]] into [[Page B]]", "combine these pages",
  "these pages are duplicates", "consolidate", "wiki merge".
allowed-tools: Read Write Edit Glob Grep Bash
---

# wiki-merge: Merge Duplicate Pages

When lint flags two pages as semantic duplicates (tiling violation) or the user identifies redundant pages, this skill merges them without losing any information.

The result is one canonical page, one archived redirect stub, and zero orphan links.

---

## When to Merge vs. When to Keep Both

**Merge when:**
- Two pages cover the exact same concept under different names
- One page is a strict subset of the other with no unique content
- Semantic tiling lint reports cosine similarity >= 0.90

**Keep both when:**
- The pages are related but distinct (e.g., "RAG" vs. "Semantic Search")
- One page is a historical snapshot that should remain as a source record
- Cosine similarity is 0.80-0.89 (review band, not merge band)

If unsure, ask the user before touching anything.

---

## Merge Procedure

### 0. Identify which page is canonical

Decide before writing anything:
- **Primary** (canonical): the page to keep. Usually the one with more inbound links.
- **Secondary** (to archive): the page to retire.

```bash
grep -r "\[\[Primary Page\]\]" wiki/ --include="*.md" -l | wc -l
grep -r "\[\[Secondary Page\]\]" wiki/ --include="*.md" -l | wc -l
```

Report to the user: "I'll merge [[Secondary]] into [[Primary]]. [[Primary]] has N inbound links, [[Secondary]] has M. Confirm?" Wait for confirmation before proceeding.

---

### 1. Read both pages completely

Read both pages in full before writing anything.

---

### 2. Extract unique content from secondary

Identify content that exists ONLY in the secondary page: sections, examples, data points, wikilinks, frontmatter fields (tags, aliases, related) not in the primary.

---

### 3. Merge into primary

Edit `wiki/[path]/Primary.md`:

1. **Frontmatter merge**: union of tags, aliases (add secondary title as alias), related links. Keep primary's `address:`. Set `updated` to today.
2. **Content merge**: integrate unique content from secondary into appropriate sections. Write as coherent prose, not a paste.
3. **Add merge notice** at the bottom:
   ```markdown
   ---
   > [!note] Merged
   > Content from [[Secondary Page Name]] was merged into this page on YYYY-MM-DD.
   > The secondary page is archived at `wiki/archive/Secondary Page Name.md`.
   ```

---

### 4. Archive and redirect the secondary

Create `wiki/archive/[Secondary Page Name].md` with the full original content preserved, plus an archive frontmatter header.

Replace `wiki/[path]/Secondary.md` with a redirect stub:
```markdown
---
type: redirect
title: "[Secondary Page Name]"
redirects_to: "[[Primary Page Name]]"
archived_date: YYYY-MM-DD
---

# [Secondary Page Name]

> [!note] Redirected
> This page was merged into [[Primary Page Name]] on YYYY-MM-DD.
> Archived version: [[archive/Secondary Page Name]].
```

Do NOT delete the secondary page — archive stubs preserve history and prevent broken external links.

---

### 5. Update all inbound links

```bash
grep -r "\[\[Secondary Page Name\]\]" wiki/ --include="*.md" -l
```

For each file found (except the archive page, redirect stub, and log.md), replace `[[Secondary Page Name]]` with `[[Primary Page Name]]`. Preserve aliased link display text.

---

### 6. Update index, sub-indexes, log, and hot cache

- `wiki/index.md`: remove secondary entry; primary entry stays
- Any `_index.md` that listed secondary: update to primary
- `wiki/log.md` (append at TOP):
  ```markdown
  ## [YYYY-MM-DD] merge | [[Secondary]] → [[Primary]]
  - Secondary archived at: wiki/archive/Secondary Page Name.md
  - Unique content transferred: [list]
  - Inbound links updated: N pages
  - Trigger: [tiling lint / user request]
  ```
- `wiki/hot.md` Recent Changes: `- Merged: [[Secondary]] into [[Primary]] (YYYY-MM-DD)`

---

## Batch Merge

Handle one pair at a time. Each merge changes the link graph, affecting subsequent decisions. After each merge, re-run tiling check on affected pages before moving to the next pair.

---

## What Not to Do

- Do not delete any page. Archive stubs preserve history.
- Do not merge without user confirmation when pages are in different domains.
- Do not silently drop any content from the secondary page.
- Do not edit past entries in `wiki/log.md`.
