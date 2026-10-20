# SETUP

Bring a **fresh cloud workspace** up to speed. Human/agent-followed runbook —
nothing here is auto-loaded by any agent, unlike `AGENTS.md` / `CLAUDE.md`.

Scope: this KB, the personal rules that ride along with it, and the agent
surfaces that must know about both. Add future machine setup here too (see
[Conventions](#conventions-for-this-file)).

- **Rule text** — the KB rule *and every other personal rule* — lives in
  `PROMPT.md`, one `##` section each. That is the single source of truth.
- **This file** only *installs* that text into each surface. Never retype the
  rule text here — every step derives it from `PROMPT.md`, so there is no third
  copy to drift.

---

## 0. Clone the KB

Chicken-and-egg: you cannot read this file until the KB exists. This is the one
step you must know without the KB.

```bash
git clone <your-personal-kb-remote> /home/repo/personal-kb
```

`/home/repo/personal-kb` is hardcoded in the rule text — clone it to exactly
that path or every step below breaks.

Then continue by pointing your agent at this file:
`follow /home/repo/personal-kb/SETUP.md`

---

## 1. Install the rules into agent surfaces

Paste this whole block once. Idempotent — safe to re-run after `PROMPT.md`
changes, and re-run is how you resync. It installs **all** of `PROMPT.md`, so a
new `##` section there needs no change here.

`KB-SYNC` re-runs this block as its last step (`AGENTS.md`, step 5), so on a
workspace that syncs you rarely run it by hand — a sync that pulls someone else's
`PROMPT.md` reinstalls it for you.

Both outputs are **generated files**: the block overwrites them with `>`. Never
hand-edit `~/.claude/CLAUDE.md` or the `.mdc` — the next sync silently discards
it. Rules go in `PROMPT.md`.

```bash
KB=/home/repo/personal-kb
# everything after the first "---" in PROMPT.md, leading blank lines trimmed
rule_body() { sed -n '/^---$/,$p' "$KB/PROMPT.md" | tail -n +2 | sed '/./,$!d'; }
# section titles, for the Cursor description — derived, so it never drifts
rule_topics() { rule_body | sed -n 's/^## //p' | paste -sd ';' - | sed 's/;/; /g'; }
# single-quoted: the "!" in "<!--" must never sit inside double quotes (see Conventions)
kb_origin='<!-- Source of truth: /home/repo/personal-kb/PROMPT.md. Regenerate: see /home/repo/personal-kb/SETUP.md -->'

# --- Claude Code: user-level memory, all projects, every session ---
# PROMPT.md owns its own "##" headings; only the file title is added here.
mkdir -p ~/.claude
{ echo "# Personal rules (user level — applies to every project)"; echo
  echo "$kb_origin"; echo
  rule_body; } > ~/.claude/CLAUDE.md

# --- Cursor: project rule, gitignored, always-apply ---
# guarded: without the check, a workspace that has not cloned the monorepo yet
# gets a fabricated /home/repo/snowflake tree containing only this rule
if [ -d /home/repo/snowflake/.git ]; then
  mkdir -p /home/repo/snowflake/.cursor/rules/personal
  { echo "---"
    echo "description: Personal rules (source: $KB/PROMPT.md) — $(rule_topics)"
    echo "alwaysApply: true"; echo "---"; echo
    echo "$kb_origin"; echo
    rule_body; } > /home/repo/snowflake/.cursor/rules/personal/personal-rules.mdc
  # superseded name from when the only rule was the KB one; both are gitignored
  rm -f /home/repo/snowflake/.cursor/rules/personal/personal-kb.mdc
else
  echo "skipped the Cursor rule: /home/repo/snowflake is not a git clone yet"
fi
```

Verify:

```bash
head -3 ~/.claude/CLAUDE.md   # prints the "Personal rules" heading

# every "##" section of PROMPT.md must appear in the installed copy
diff <(sed -n '/^---$/,$p' /home/repo/personal-kb/PROMPT.md | sed -n 's/^## //p') \
     <(sed -n 's/^## //p' ~/.claude/CLAUDE.md) && echo "all rule sections installed"

# run from the monorepo, not the KB — must print the .gitignore line that
# protects the file. Silence + exit 1 means it is NOT ignored: stop and fix,
# or you will commit the KB path into snowflake.
git -C /home/repo/snowflake check-ignore -v .cursor/rules/personal/personal-rules.mdc
```

### Why these two paths

| Surface | Path | Scope |
|---|---|---|
| Claude Code | `~/.claude/CLAUDE.md` | user level → **all** repos, every session |
| Cortex Code (CoCo) | `~/.claude/CLAUDE.md` — same file | user level → all workspaces, every session |
| Cursor | `snowflake/.cursor/rules/personal/personal-rules.mdc` | that repo, always-apply |
| Cursor (in this KB) | `AGENTS.md` + `CLAUDE.md` at KB root | already committed, nothing to do |

`.cursor/rules/personal` is gitignored by the monorepo on purpose —
`snowflake/.gitignore` has `# allow a place for personal rules that won't be
checked in`. That is the sanctioned local-only spot, so this satisfies "never
commit the KB into the Snowflake monorepo".

Claude Code memory precedence, for reference: `~/.claude/CLAUDE.md` (user) →
`<repo>/CLAUDE.md` (project) → `<repo>/CLAUDE.local.md` (project, uncommitted).
Only the user level reaches every session, which is why the KB rule goes there.

New sessions pick the rule up at startup; a session already running does not.

---

## 2. Cortex Code (CoCo) — already covered by step 1, verify only

Cortex Code Desktop discovers **user-scope instruction files** by *name* in three
directories: `~/.snowflake/cortex/`, `~/`, and `~/.claude/`. Recognized names
include `AGENTS.md`, `CLAUDE.md`, `CORTEX.md`, `SNOWFLAKE.md`, `RULES.md`,
`.cursor/rules/*.mdc`, `.claude/rules/*.md` — matched case-insensitively.

So the `~/.claude/CLAUDE.md` that step 1 already writes **is** the Cortex Code
user rule. Nothing to install. Do **not** add a second copy (see below).

Verify — from a *new* CoCo chat, ask:

```
Which instruction files did you load, and do you know about my personal KB?
```

Expected: it names `~/.claude/CLAUDE.md` as a **User**-scope file and can restate
the `KB-QUERY` / `KB-SYNC` / `KB-PERSIST` keywords. Or check the UI:
Agent Settings → **Rules** → **Instruction Files** tab lists every discovered
file with its scope; `~/.claude/CLAUDE.md` must appear under *User*.

### Why not also write `~/.snowflake/cortex/AGENTS.md`

Two reasons, both real:

1. **Double injection.** CoCo deduplicates instruction files *by path*, not by
   content. Two files with the same rule text = the rule in context twice,
   burning tokens against the ~100k combined-instruction cap for zero gain.
2. **It is not ours to overwrite.** `~/.snowflake/cortex/AGENTS.md` is the file
   behind Agent Settings → Personalization → **Custom instructions**. The editor
   and the file stay in sync, so a scripted `>` here would silently clobber
   anything typed into that box. Leave it for hand-written CoCo-specific
   preferences.

If step 1's file ever stops being discovered, the cause is the
`chat.instructionFilePatterns` setting having been narrowed — fix the setting
rather than adding a copy. Only if that is not possible, fall back to appending
(not overwriting) the rule into `~/.snowflake/cortex/CORTEX.md`, which is a
recognized name that no UI writes to.

---

## 3. Cursor User Rules — manual, cannot be scripted

Cursor's account-level **User Rules** are not a file. On a remote-SSH workspace
the server side (`~/.cursor-server/.../globalStorage/`) holds no `state.vscdb`
at all — the rules live in the desktop client on the Mac and sync via the
account. Nothing on the workspace can read or write them.

So: paste `PROMPT.md` (everything after the `---`) into Cursor → Settings →
Rules → User Rules **by hand**, on the Mac, once per account. It survives
workspace destroy; no agent here can verify it, so treat step 1 as the load-bearing
copy and this as a bonus.

> `PROMPT.md` says these "follow the account, so they survive a cloud workspace
> destroy." True for Cursor User Rules only. `~/.claude/CLAUDE.md` is a local
> file in the workspace home — a workspace rebuild wipes it, which is exactly why
> step 1 exists and why `PROMPT.md` stays the git-backed original.

---

## 4. When `PROMPT.md` changes — including adding a new rule

Adding a rule is a `PROMPT.md` edit, never a `SETUP.md` edit: append a `##`
section there and re-run the step-1 block (it overwrites both files and picks up
new sections automatically), then re-paste into Cursor User Rules by hand.
Cortex Code needs no separate action — it reads the same `~/.claude/CLAUDE.md`
step 1 just rewrote. Commit `PROMPT.md` so other workspaces get the rule on
their next `KB-SYNC` + re-run of step 1.

Nothing in step 1 mentions any individual rule, so the only reasons to touch it
are a new *surface* or a changed path.

---

## Conventions for this file

Keep future entries in the same shape so this stays followable:

1. One numbered section per setup concern, ordered by dependency.
2. Every step is an **idempotent** paste-able block — re-running must be safe.
3. Every step ends with a **verify** command whose expected output is stated.
4. Never duplicate content that lives elsewhere in the KB; derive it.
5. Note anything that **cannot** be automated, and say why (see step 3).
6. **No `!` inside double quotes** in any paste-able block. Every interactive
   shell here does history expansion inside double quotes, so
   `echo "<!-- x -->"` dies on paste even though it is fine in a script.
   Measured on this workspace 2026-09-11: `bash: !-: event not found` and
   `zsh: event not found: -`. The login shell is `/bin/bash` (`$SHELL=/bin/bash`)
   and zsh behaves identically, so this is **not** a zsh quirk to be waved off by
   a bash user. Single-quote such strings. A `!` inside a `#` comment happens to
   be safe (both shells have `interactive_comments` on), but single-quoting is
   the habit worth keeping.
7. Test a block by *pasting* it, not by running it as a file — `bash script.sh`
   has history expansion off and will hide exactly the class of bug in 6.

This file is not wiki content: it is not in `wiki/index.md` and needs no
source pointer, because it documents this machine, not knowledge.
