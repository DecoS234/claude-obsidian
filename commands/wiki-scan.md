---
description: Scan inbox folders (~/Downloads, ~/Documents, vault staging) for new files, classify each by module/topic, auto-ingest the confident matches into raw/ and source pages, and triage the rest. Use whenever the user wants to bulk-process unstructured files into the wiki without naming them one by one.
---

Read the `wiki-scan` skill. Then run the inbox sweep.

Usage:
- `/wiki-scan` — scan the default roots: `~/Downloads`, `~/Documents`, and `<vault>/raw/_inbox/` if it exists.
- `/wiki-scan [path...]` — scan the given paths instead. One or more, space-separated. Paths can be absolute or `~`-relative.

Before scanning, read the vault's `CLAUDE.md` so the layout (whether the source folder is `raw/` or `.raw/`, what `wiki/` subfolders exist, what frontmatter the project pages use) is honored. The CLAUDE.md is the source of truth — this skill's defaults are only a fallback.

After scanning, produce the summary block defined in the skill (counts of seen / skipped / already-ingested / auto-ingested, plus the low-confidence triage list). Wait for the user to resolve the triage before continuing.

Append one `## [YYYY-MM-DD] ingest | wiki-scan sweep — …` entry to `log.md` when the sweep is complete (after any triage decisions are applied).

If no vault is set up in the current working directory, say: "No wiki vault found here. Run /wiki first to set one up, or `cd` into the vault and re-run."

If the vault is set up but `wiki/projects/` is empty (no project aliases to match against), say so and ask the user to either (a) name a default project to file everything under, or (b) let the scan run and dump the auto-classifiable files into per-extension source pages without project linking.
