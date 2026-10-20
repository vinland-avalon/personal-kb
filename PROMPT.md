# Prompt

Personal rule text, installed into every agent surface (Claude Code, Cortex
Code, Cursor) by `SETUP.md` step 1. **This file is the single source of truth** —
every installed copy is derived from it, so edit here, never there.

Everything after the `---` is copied verbatim. It must therefore:

- be self-contained — it is read from other repos, so "this repo" means nothing;
- use absolute paths;
- open each topic with its own `##` heading. `SETUP.md` adds only the file title
  above the block, so these headings are the rule file's whole structure.

The only rule shipped below the line is the knowledge base. Other personal
rules — branch names, review habits, tools — are not wiki content. Add them
here: append a `##` section after the `---`, then re-run `SETUP.md` step 1 and
re-paste the same body into Cursor User Rules by hand (`SETUP.md` step 3). Do
the same when the keywords in `AGENTS.md` change.

Keep each section short. Every line after the `---` sits in the context of
every session in every repo. A rule that only matters in one repo belongs in
that repo's `CLAUDE.md` instead.

---

## Personal knowledge base

I keep a personal knowledge base at `/home/repo/personal-kb`.
Schema: `/home/repo/personal-kb/AGENTS.md` — read it before any KB write, and read `/home/repo/personal-kb/wiki/index.md` on questions.

`raw/` is immutable sources. `wiki/` is compiled knowledge you maintain.
`wiki/index.md` is the catalog. `log.md` is append-only.

**Query is default.** On any question, read `/home/repo/personal-kb/wiki/index.md` then relevant pages before searching the repo or guessing. Cite. If empty or stale, say so and continue. Do not edit wiki.

Keywords (first token, case-insensitive):

- `KB-QUERY` … — explicit query; same as default. Do not edit wiki.
- `KB-SYNC` — git pull/merge, fold new files under `/home/repo/personal-kb/raw/` into wiki (default: all not yet in `raw/processed/`; or the paths I list), health-check, commit and push, then reinstall the rules by re-running `SETUP.md` step 1 (the pull may carry a `PROMPT.md` from another workspace). Never force. Never commit snowflake. New source wins on conflict; note it in `log.md`.
- `KB-PERSIST` — write this chat to `/home/repo/personal-kb/raw/sessions/YYYY-MM-DD-<slug>.md` (durable bits only, not a transcript). Do not sync. Do not edit wiki.

Write the KB in English — wiki pages, `log.md`, session notes — even when the session or the source is in Chinese; quote identifiers and titles verbatim. Sources under `raw/` keep their original language.

Rules: I pick what goes in `raw/`. You own `wiki/`. Every wiki claim needs a pointer (Google Doc URL, code path, or `raw/...`). No future plans, sprint dates, or meeting debate. Newer evidence beats older pages; note overwrites in `log.md`. This KB is private — never commit it into the Snowflake monorepo.
