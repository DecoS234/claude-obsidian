# claude-obsidian — Claude + Obsidian Wiki Vault

This folder is both a Claude Code plugin and an Obsidian vault.

**Plugin name:** `claude-obsidian`
**Skills:** `/wiki`, `/wiki-ingest`, `/wiki-query`, `/wiki-lint`, `/wiki-merge`
**Vault path:** This directory (open in Obsidian directly)

## What This Vault Is For

This vault demonstrates the LLM Wiki pattern — a persistent, compounding knowledge base for Claude + Obsidian. Drop any source, ask any question, and the wiki grows richer with every session.

## Vault Structure

```
.raw/           source documents — immutable, Claude reads but never modifies
wiki/           Claude-generated knowledge base
_templates/     Obsidian Templater templates
_attachments/   images and PDFs referenced by wiki pages
```

## Session Initialization

At the start of every session, do exactly this — no more:

1. **Detect session type** from the first message:
   - Setup/scaffold request → read `skills/wiki/SKILL.md`, skip hot.md
   - Ingest request → read `wiki/hot.md` silently (orient on recent context), then proceed to `wiki-ingest`
   - Query request → read `wiki/hot.md` first; answer if possible, else read `wiki/index.md`
   - Lint/maintenance → read `wiki/hot.md` and `wiki/index.md`
   - Ambiguous → read `wiki/hot.md` only; proceed from there

2. **Read `wiki/hot.md` silently** (do not narrate the read). If it answers the question, respond immediately without reading further.

3. **Do NOT read `wiki/index.md` unless** the hot cache is insufficient and the query requires it.

**When to skip the wiki entirely:**
- General coding, language, or tool questions not specific to this domain
- Tasks already explained in the conversation
- Requests for syntax, definitions, or stable reference facts
- Any question the user has already provided full context for in their message

**Session page register:** In your working memory, maintain a list of wiki pages already read in this conversation. Never re-read a page you have already loaded in this session.

## How to Use

Drop a source file into `.raw/`, then tell Claude: "ingest [filename]".

Ask any question. Claude reads the index first, then drills into relevant pages.

Run `/wiki` to scaffold a new vault or check setup status.

Run "lint the wiki" every 10-15 ingests to catch orphans and gaps.

## Cross-Project Access

To reference this wiki from another Claude Code project, add to that project's CLAUDE.md:

```markdown
## Wiki Knowledge Base
Path: /path/to/this/vault

Wiki reading protocol (strict token budget):
1. Read wiki/hot.md first (~500 tokens). If it answers the question, stop there.
2. If not enough, read wiki/index.md (~1000 tokens).
3. If domain-specific, read wiki/<domain>/_index.md (~200 tokens).
4. Only then read individual wiki pages (~100-300 tokens each).

Do NOT consult the wiki for:
- General coding questions or language syntax
- Things already in this project's files or conversation
- Tasks unrelated to [your domain]
- Stable external facts Claude already knows
```

## Plugin Skills

| Skill | Trigger |
|-------|---------|
| `/wiki` | Setup, scaffold, route to sub-skills |
| `ingest [source]` | Single or batch source ingestion |
| `query: [question]` | Answer from wiki content |
| `lint the wiki` | Health check |
| `merge [[Page A]] into [[Page B]]` | Merge two semantically duplicate pages |
| `/save` | File the current conversation as a structured wiki note |
| `/autoresearch [topic]` | Autonomous research loop: search, fetch, synthesize, file |
| `/canvas` | Visual layer: add images, PDFs, notes to Obsidian canvas |

## Conversation Continuity Rules

- Queries within the same conversation on the same topic do NOT require re-reading hot.md or index.md again.
- Track which wiki pages were read this session. Reuse that context for follow-up questions.
- If the user asks a follow-up question, synthesize from already-loaded pages before reading new ones.
- Update hot.md and log.md **once at the end of the session**, not after every individual exchange.

## Token Discipline

| Operation | Max pages to read | Stop when |
|-----------|------------------|-----------|
| Quick query | 0 (hot.md + index only) | Answer is found in cache |
| Standard query | 3-5 | Answer is synthesized |
| Deep query | 10-15 | Full coverage achieved |
| Ingest | 5 existing pages | Cross-references placed |
| Lint | All (scan pass) | Checks complete |

## MCP (Optional)

If you configured the MCP server, Claude can read and write vault notes directly.
See `skills/wiki/references/mcp-setup.md` for setup instructions.
