# Personal knowledge wiki — schema

This directory is a personal knowledge base. Do not commit it to snowflake-eng/snowflake.

**Human:** pick sources, ask questions, reject wrong pages.
**Agent:** write and maintain `wiki/`. Never edit files under `raw/` except (1) writing a session note on `KB-PERSIST`, and (2) moving a folded file to `raw/processed/`.

## Layout

| Path | Who writes | Role |
|---|---|---|
| `raw/` | human | Immutable sources (Google Doc exports, pasted notes). |
| `raw/sessions/` | agent, only on `KB-PERSIST` | One markdown file per session the human chose to keep. |
| `raw/processed/` | agent (move only) | Sources already folded into the wiki. Never rewrite content. |
| `wiki/` | agent | Compiled, interlinked pages. Humans rarely edit. |
| `wiki/index.md` | agent | Catalog. Rebuild when pages change. |
| `log.md` | agent | Append-only. Never delete entries. |
| `AGENTS.md` | both | This schema. Update when a convention actually changes. |

Do not pre-create domain folders. On `KB-SYNC`, add pages (and folders only when a cluster of pages needs one) and update `wiki/index.md`. Prefer patching an existing page over creating a new one. Reorganize during the health-check if the layout has drifted.

## Page rules

- One idea per page. Prefer < 200 lines.
- Every non-trivial claim needs a pointer: Google Doc URL, code path in the snowflake repo, or `raw/...`.
- Do not copy future plans, meeting debate, or sprint dates into wiki pages.
- If a new source contradicts an old page, update the old page and note the conflict in `log.md`.
- **Write in English.** Everything you put in the KB — wiki pages, `log.md`, and `KB-PERSIST` session notes — is English, even when the session or the source is in Chinese. Quote titles, identifiers, parameter names, and error strings verbatim in their original language; translate the prose around them. Sources under `raw/` keep whatever language the human wrote them in — you never rewrite those. Keep filenames kebab-case ASCII.

Wiki page header:

```markdown
# <Title>

**Updated**: YYYY-MM-DD
**Sources**: <url or raw/ path>
```

## Operations

**Query is the default.** On any question, read `wiki/index.md` then relevant pages before searching the repo or guessing. Cite pages. If empty or stale, say so and continue with code/docs. Do not edit `wiki/` on query.

Keywords (case-insensitive, first token):

| Keyword | Does |
|---|---|
| `KB-QUERY` | Same as default query, but explicit |
| `KB-SYNC` | Git + fold new `raw/` into wiki + health-check |
| `KB-PERSIST` | Write this chat into `raw/sessions/` |

Without `KB-SYNC` / `KB-PERSIST`, do not write the KB.

This directory is `/home/repo/personal-kb`. Git commands run here. If there is no `.git` or no remote, skip git steps and say so.

### `KB-QUERY` …

Same as default query. Rest of the line is the question. Do not edit `wiki/`.

### `KB-PERSIST`

Write `raw/sessions/YYYY-MM-DD-<short-slug>.md`. Keep only durable content: question, conclusion, decisions and why, code paths, Jira/GDoc links, gotchas. Drop tool traces, dead ends, and chatter. Do not dump the transcript. Do not `KB-SYNC`. Do not edit `wiki/`. Reply with the file path after writing.

Session file header:

```markdown
# <Title>

**Date**: YYYY-MM-DD
**Tool**: cursor | claude-code | cortex-code
```

### `KB-SYNC` [path ...]

Always: git, then fold, then health-check, then commit/push, then reinstall the rules.

- No paths: fold **every new** source under `raw/` (including `raw/sessions/`). New = not yet in `raw/processed/`, skip `README.md` and `.gitkeep`.
- One or more paths: fold only those files. They must be under `raw/`.
- If nothing new to fold, still run git and health-check.

#### 1. Git (before folding)

Run all git in this directory. Never `--force`. Never change git config. Never commit into snowflake-eng/snowflake.

1. `git status` + `git fetch`.
2. If no remote: skip remaining git steps with a one-line note, continue.
3. If uncommitted local work: leave it in the working tree; include it in the merge/commit at the end.
4. `git pull --no-rebase` (merge). If already up to date, continue.
5. On conflict, resolve automatically using the rules below. Do not leave conflict markers.

**Conflict rules** (prefer keeping knowledge, not dropping it):

| Path | How to merge |
|---|---|
| `log.md` | Keep every unique `## [...]` entry from both sides. Sort by the date in the heading. Never delete an old entry. |
| `wiki/index.md` | After page files are resolved, rebuild the index from the pages that exist. Do not hand-merge the catalog. |
| `wiki/**/*.md` | Union of sections and `Sources`. If two claims contradict, keep the newer `Updated:` and add one line noting the discarded claim. If `Updated:` is tied, keep both sentences. |
| `raw/**` | Never drop a source. Same path, different bytes: keep local as-is and write the incoming copy next to it as `<name>.remote.md`. |
| `AGENTS.md` / `PROMPT.md` / `CLAUDE.md` | Union of sections. Never silently drop a keyword (`AGENTS.md`) or a `##` rule section (`PROMPT.md`) that exists on either side. Within a section that both sides changed, prefer the fuller text; if unsure, keep the local copy and note the other version in `log.md`. |

Do not use `git merge -X ours` / `-X theirs` as a blanket strategy.

#### 2. Fold

For each selected source, in a stable order (oldest first if dates are in the filename):

1. Read the source. Do not modify `raw/` content.
2. Patch existing pages when they fit. Create a page only if the concept will be queried again. Create a folder only when several pages belong together.
3. If the source supplements an old page, merge it in and add the source to `Sources`.
4. If it contradicts an old page, the new source wins: update the page and note the conflict in `log.md`.
5. Put the Google Doc URL (if any) on the wiki page `Sources` line, together with the processed path.
6. Move the file to `raw/processed/`, keeping the relative path under `raw/` (e.g. `raw/sessions/foo.md` → `raw/processed/sessions/foo.md`).

Update `wiki/index.md` one-liners. Group by whatever structure actually exists — do not invent empty sections.

#### 3. Health-check

Look for: contradictions, claims superseded by newer sources, orphans (no inbound links from index or other pages), important concepts mentioned but missing a page, missing source links.
Fix safe bookkeeping (links, index). Ask before rewriting a thesis.

#### 4. Commit and push

`git add` every file that belongs to this KB, commit if there is anything to commit:

```
git commit -m "$(cat <<'EOF'
kb-sync: fold raw and health-check

EOF
)"
```

`git push`. If rejected (another workspace pushed), `git pull --no-rebase`, resolve again, commit if needed, push again. Repeat until push succeeds or you hit a non-conflict error (auth, no remote). Report what happened.

Log:

```markdown
## [YYYY-MM-DD] sync
- Git: pulled / merged / pushed / skipped (reason)
- Folded: <paths or none>
- Pages: ...
- Health-check: ...
- Rules: reinstalled / unchanged (say so either way)
```

#### 5. Reinstall the rules

The pull may have brought in a `PROMPT.md` written on another workspace, and this sync may have edited it here. Either way the installed copies are now stale: `~/.claude/CLAUDE.md` and the Cursor rule are generated files, and nothing else regenerates them.

So after the push, run the install block in [SETUP.md](SETUP.md) step 1. Do not retype it — it is idempotent, and re-running it is exactly how a resync works. Then report whether the installed copy actually changed:

```bash
diff <(sed -n '/^---$/,$p' /home/repo/personal-kb/PROMPT.md | sed -n 's/^## //p') \
     <(sed -n 's/^## //p' ~/.claude/CLAUDE.md) && echo "all rule sections installed"
```

This only fixes **this** workspace, and only for sessions started afterwards — a session already running keeps the rules it loaded at startup. Other workspaces need their own `KB-SYNC` (or a manual step 1). Cursor's account-level User Rules still have to be pasted by hand; see SETUP.md step 3.

## Tooling

Cursor, Claude Code, and Cortex Code all follow this file. Do not copy Snowflake monorepo skills into this wiki.

This directory (`/home/repo/personal-kb`) must be a standalone git repo with a remote so `KB-SYNC` can pull/merge/push across workspaces.
