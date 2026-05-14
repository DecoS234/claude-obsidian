---
name: wiki-scan
description: "Scan one or more inbox folders for files that haven't been ingested into the Obsidian wiki vault yet, classify them by module/topic, and route the confident matches into raw/ + source pages automatically. Use this whenever the user says things like 'scan my downloads', 'what's new in Downloads', '/wiki-scan', 'find new files to ingest', 'check for files to add', 'pick up the new stuff', 'inbox sweep', or any phrasing where the goal is bulk-triaging unstructured files into the wiki rather than ingesting one named source. Prefer this skill over wiki-ingest when the user hasn't told you which specific file to process — wiki-ingest is for a single named source; wiki-scan is for 'figure out what's worth ingesting and do it'."
---

# wiki-scan: Inbox Sweep for the Wiki

Read the inboxes. Triage everything. Pull the wheat into `raw/` and the wiki; leave the chaff alone. The user shouldn't have to name files one by one when there's a clear backlog sitting in `~/Downloads`.

This skill is the discovery half of ingestion. `wiki-ingest` handles a single source the user names. `wiki-scan` walks folders, decides what's new, classifies each file, auto-ingests the confident matches, and hands you a short triage list for the ambiguous ones.

---

## Scan Targets

Default scan roots (in order):

1. `~/Downloads/`
2. `~/Documents/`
3. `<vault>/raw/_inbox/` if it exists — the user's intentional staging area.

The user can override with positional args: `/wiki-scan ~/Desktop/temp ~/Documents/2026-syllabus` scans those two paths instead.

If the user has a CLAUDE.md schema that says `.raw/` (hidden) instead of `raw/`, use that. Read the vault's `CLAUDE.md` first; treat its layout description as authoritative over this skill's defaults.

---

## What "new" means

A file is considered new if **all** are true:

1. Its cleaned destination filename (see *Filename cleanup*) does not already exist under any `raw/` subfolder.
2. It does not match a `(N)`-suffix duplicate of an existing canonical filename — e.g., `01_DGTIN-Assignment 1 and 2-2510 (1).docx` is a dup of `01_DGTIN-Assignment-1-and-2-2510.docx`.
3. If `.raw/.manifest.json` exists (created by `wiki-ingest`'s Delta Tracking), the file's content hash is not already recorded as ingested.

When in doubt about duplication, prefer to skip and report — re-ingesting is harder to undo than missing one ingest.

---

## Noise filtering

Drop without prompting:

| Category | Extensions / patterns |
|---|---|
| Executables / installers | `.exe`, `.msi`, `.dmg`, `.pkg`, `.deb`, `.rpm`, `.appimage` |
| Archives | `.zip`, `.tar`, `.gz`, `.tgz`, `.bz2`, `.7z`, `.rar`, `.iso` |
| Code packages | `.jar`, `.war`, `.vsix`, `.crx`, `.whl`, `.egg` |
| Media | `.mp3`, `.mp4`, `.mov`, `.mkv`, `.webm`, `.avi`, `.wav`, `.m4a`, `.flac` |
| Generic images | `.png`, `.jpg`, `.jpeg`, `.gif`, `.bmp`, `.webp` — **unless** the file came from Obsidian Web Clipper (filename pattern `<slug>-YYYY-MM-DD.{md,png}` paired) or the user explicitly says to keep images |
| Torrents / partial | `.torrent`, `.part`, `.crdownload`, `.tmp` |
| Dev artifacts | `node_modules/`, `__pycache__/`, `.venv/`, `dist/`, `build/`, `.next/`, `target/`, anything beginning with `.` other than `.md` |
| Already extracted | If a folder named `<name>-extracted/` and an archive `<name>.zip` both exist, treat the zip as noise. |

When you drop noise, **report counts not names** — "skipped 47 noise files (24 images, 15 archives, 8 executables)" — unless something looks borderline (e.g., a `.zip` that might be a course bundle), in which case ask.

---

## Classification

### Step 1: extension routes to a `raw/` subfolder

| Extension | Destination | Rationale |
|---|---|---|
| `.pdf` | `raw/pdfs/` | Direct copy + optional extracted `.md` sibling (use `tools/extract_pdf.py` if present, or `pdftotext`). |
| `.docx`, `.doc`, `.txt`, `.md`, `.html` | `raw/notes/` | Text-shaped sources. For `.docx`/`.doc`, run `tools/extract_docx.py` if available to get a sibling `.md`. |
| `.pptx`, `.ppt` | `raw/slides/` | Lectures, decks. |
| `.xlsx`, `.xls`, `.csv`, `.tsv` | `raw/sheets/` | Data tables, gradesheets. |
| `.py`, `.sql`, `.asm`, `.c`, `.cpp`, `.cs`, `.java`, `.js`, `.ts`, `.go`, `.rs` | `raw/practicals/code/` | Source files. |
| `.pkt`, `.pka` | `raw/practicals/networking/` | Packet Tracer artifacts. |
| `.json` (Claude export) | `raw/claude-ai/` | Recognized by name patterns like `claude-export-*.json` or `conversations.json`. |
| Anything else | report and ask |

Create folders only when needed. Don't pre-make empty ones.

### Step 2: filename pattern → module / topic

Match against the project pages currently in `wiki/projects/`. Read each project page's frontmatter `aliases` field if present — those are your match terms.

**Match against whole words, not substrings.** This is the lesson from the 2026-05-10 ingest: the substring `ics` matched `eth**ICS**`, `analyt**ICS**`, and `aquapon**ICS**`. Use word boundaries — in regex terms, `\b(ics|sscc|ct123)\b`, case-insensitive. Module codes (`CT123-3-1`, `MPU2342`, `DGTIN`) match as-is. Aliases listed without word boundaries need them added before use.

Confidence tiers:

- **High** — filename contains an explicit module code (`CT044-3-1-PYP`, `MPU2342`, `KIAR`, `DGTIN`) or an alias listed in a project page's frontmatter. Auto-ingest.
- **Medium** — filename matches a common topic phrase (e.g., "Assembly Language", "Visual Programming") that's mentioned across ≥2 wiki pages but not a confirmed alias. Auto-ingest into the most-mentioned project, flag the inference in the source page.
- **Low** — no clear match, or two equally-likely matches. Triage: ask user.

### Step 3: filename cleanup

Apply to the destination filename only — preserve the original in the source page's manifest.

- Spaces → `-`
- `%` → `-`
- `&` → `-and-`
- Drop any non-alphanumeric character except `-`, `.`, `_`
- Collapse repeated `-` to single `-`
- Strip `(N)` duplicate suffixes (`-1`, `-2` artifacts left over from browser downloads)
- Preserve extension case-folded to lowercase (`.PDF` → `.pdf`)

Examples:

- `01_DGTIN-Assignment 1 and 2-2510 (1).docx` → `01-DGTIN-Assignment-1-and-2-2510.docx` (and recognized as dup of the existing `(0)` copy)
- `Lecture 03 - Von Neumann & Friends.pptx` → `Lecture-03-Von-Neumann-and-Friends.pptx`
- `CT108-3-1-PYP [Group Assignment] 2509.pdf` → `CT108-3-1-PYP-Group-Assignment-2509.pdf`

---

## Auto-ingest flow (per file, high/medium confidence)

For every file that passes new + classified-high-or-medium:

1. **Copy** the file to its `raw/` destination with the cleaned filename.
2. **Extract** to a sibling `.md` if the file is `.docx`/`.pdf` and the relevant `tools/extract_*.py` is available, or `pandoc` / `pdftotext` are installed. Extraction is best-effort — if it fails, log it and move on; the source page still gets created.
3. **Update or create the source page** at `wiki/sources/Source - <Title>.md`. For module-cluster files coming in as a batch, prefer one source page per *cluster* (the 2026-05-10 ingest pattern) rather than per file — a source page is a manifest of files plus brief metadata, not one-per-file. If a cluster source page already exists for this module, append the new file to its file list; bump `ingested:` to today's date in frontmatter.
4. **Touch the project page** (`wiki/projects/<module> coursework.md`): extend its `sources:` frontmatter if the source page is newly created; otherwise no edit needed.

If the project page doesn't exist yet, **create a stub** before adding the source — three lines is fine. Source pages must always link to a real project page.

---

## Triage flow (per file, low confidence)

After the auto-ingest pass, surface the unresolved files as a single grouped report:

```
Low-confidence files (need your call):

  raw/_inbox/quiz prep.pdf
    - No module code in filename. "Quiz prep" is generic.
    - Candidates: [[ICS coursework]] (last modified 2 days ago), [[Database coursework]]
    - Default if you say nothing: ICS coursework

  raw/_inbox/IMG_2026_05_11.pdf
    - Probably a photo of notes. No textual hint.
    - Candidates: any project. Default: ask after others are resolved.
```

Wait for the user to respond. Don't ingest low-confidence files silently — that's how the misc bucket grew to 58 files in the 2026-05-10 run.

If the user says "skip them all" or doesn't respond, leave low-confidence files in place (do not move them) and note the count in the log entry.

---

## Output

After the scan completes, produce a single grouped summary:

```
Scan summary — 2026-05-11

Scanned: ~/Downloads (114), ~/Documents (37), raw/_inbox (0). 151 files seen.

Skipped: 47 noise files (24 images, 15 archives, 8 executables).
Already ingested: 89 files (matched against raw/ + .raw/.manifest.json).

Auto-ingested: 12 files into 4 source pages —
  - [[Source - OOP - IOOP bulk material]] (+3 files)
  - [[Source - Networks bulk material]] (+2 files)
  - [[Source - Database - IDB bulk material]] (+5 files)
  - [[Source - Quiz prep — week 8]] (new, +2 files)
Project pages touched: [[OOP coursework]], [[Networks coursework]], [[Database coursework]] (sources: extended)

Low-confidence (triage below):
  - 3 files. See list.
```

Then the triage list, then wait.

---

## Log entry

After the scan, append one entry to `log.md`:

```markdown
## [YYYY-MM-DD] ingest | wiki-scan sweep — <N> files auto-ingested, <M> triaged
- Summary: <one paragraph: roots scanned, file counts in/out, what auto-ingest matched and to which sources, what was skipped as noise vs already-ingested, what the triage list contained and how it was resolved>
- Pages touched: <list of source pages touched (new/extended) + project pages whose `sources:` was bumped + log of cluster pages>
```

`<op>` is `ingest` per the CLAUDE.md format — wiki-scan sweeps are a flavor of ingest.

---

## Failure modes to remember

- **Substring classifier bug.** `if 'ics' in name.lower()` will match `ethics`, `analytics`, `aquaponics`. Use `re.search(r'\bics\b', name, re.I)` instead. Same for any 3-letter module code.
- **Empty raw/ assumption.** If the vault is new (no `raw/` yet), don't crash — create the subdirectories you need as you go.
- **Filesystem case sensitivity.** Windows treats `Source - Foo.md` and `source - foo.md` as the same file; case-fold when checking duplicates.
- **Cluster vs single source pages.** Bulk module material (lecture decks, exam papers, lab briefs) goes into one cluster page per module. One-off documents (a specific assignment brief, a single Claude session, an article) get their own source page. When unsure, prefer single — clusters are easy to merge later; un-clustering a wad of files is harder.
- **The vault's CLAUDE.md is authoritative.** If it says `wiki/people/` exists but you find no people pages, that's not an error — concepts pages and people pages are often empty in early-stage vaults. Don't try to "fix" missing categories during a scan.

---

## Quick reference: minimal scan algorithm

```python
# Pseudocode of the happy path
for root in scan_roots:
    for path in walk(root):
        if is_noise(path): drop()
        if already_ingested(path): skip()
        kind = classify_extension(path)
        project, confidence = classify_filename(path, project_aliases)
        if confidence in ("high", "medium"):
            cleaned = clean_filename(path)
            dest = vault / "raw" / kind / cleaned
            copy(path, dest)
            extract_to_md_if_supported(dest)
            append_to_source_page(project, dest)
            touch_project_page_sources(project)
        else:
            queue_for_triage(path, candidates=project_candidates)
report_summary()
process_triage_with_user()
append_log_entry()
```

That's the whole skill.
