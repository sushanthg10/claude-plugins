---
description: Reconcile .flow/boards/main.yml with the repo's Linear project. Disk → Linear for status; Linear → board for new issues. Mechanical — reports, applies, never plans. `--dry-run` to preview.
---

# Flow Linear — Mirror the Board to Linear

Linear is a **read-mostly mirror** of Flow state for people who do not open the repo. The board (`.flow/boards/main.yml`) is the identity registry; progress is derived from `.flow/changes/` and `.flow/archive/` per the harness `board.md`. Linear never becomes a source of truth for progress — this command writes Linear state *from* disk, and only ever reads Linear for **new issues** that should join the board.

## Config

Read `.flow/linear.yml` in the repo root:

```yaml
team: Edac Software Labs                          # Linear team name or id
project_id: 36da64af-88ac-4bb6-a0f7-d70e3055dba7  # Linear project UUID — always the id, names collide
project_url: https://linear.app/<ws>/project/<slug>
```

Missing file → stop and print that block as a template. Ask for the team; then offer to create the project via `save_project` (name = repo folder name) and write the file. Wait.

Labels `flow:brief` and `flow:task` mirror the board's `track`. Create them with `create_issue_label` if the first `save_issue` reports them missing.

Join key: Linear issue **title** == board `name` (exact, lowercase-kebab-case).

## Session setup

Parse `$ARGUMENTS`. Optional single token `--dry-run` → compute and report, apply nothing.

Linear MCP server `linear-server` (tools `mcp__linear-server__*`) must be connected. If absent or unauthenticated, stop: "Run `claude mcp add --transport http linear-server https://mcp.linear.app/mcp`, then `/mcp` → linear-server → authenticate, restart, and re-run."

Read all four sources before touching anything:

1. `.flow/boards/main.yml` → `board[]` (`name`, `description`, `track`). If it fails to parse, report the line and stop — a bare `: ` inside an unquoted description is the usual cause; fix it as a `>-` folded scalar.
2. `ls .flow/changes/` → `inflight` (folder names).
3. `ls .flow/archive/` → `archived` (strip the leading `YYYY-MM-DD-` from each folder).
4. `list_issues` with `project: <project_id>`, `limit: 250`, `fields: [id, title, status, labels, description]`; follow `cursor` while `hasNextPage`. → `issues[]`.

## Derived state (disk → Linear)

For each board entry, one state, in this precedence:

| on disk | Linear state |
|---|---|
| `name` ∈ `archived` | `Done` |
| `name` ∈ `inflight` | `In Progress` |
| otherwise (board only) | `Backlog` |

`track: task` → label `flow:task`; anything else → `flow:brief`.

## Step 1 — Board → Linear

For each board entry:

- **No issue with that title** → `save_issue` create: `team`, `project: <project_id>`, `title: name`, `description`, `state` per table, `addLabels: [flow:<track>]`.
- **Issue exists, state ≠ derived** → `save_issue` update `id`, `state`. Never downgrade `Done` → anything: if Linear says Done and disk says otherwise, report it as a conflict (someone closed it by hand) and leave it.
- **Issue exists, state matches** → nothing.
- Never overwrite an existing issue's description or labels — humans may enrich them in Linear.

Apply these without asking; they are idempotent and derived. Batch many `save_issue` calls per turn.

## Step 2 — Linear → board

For each issue in the project whose title matches no board `name`:

- Status `Canceled` or `Done` → skip (report only). The board is not a place for work nobody means to do.
- Otherwise it is a **new intent raised in Linear**. Do not invent a name silently:
  1. Sanitize the title into a candidate `name` per the Flow name convention (lowercase-kebab-case `[a-z][a-z0-9]*(-[a-z0-9]+)*`, 2–5 content-bearing words, no `add-`/`fix-`/`new-`/`update-`/`remove-` prefixes).
  2. Present all candidates at once — `EDA-NN "<title>" → <candidate-name>` — via a choice prompt. Wait for the user to confirm, rename, or skip each.
  3. For each confirmed one: append to `.flow/boards/main.yml` — `name`, `description` (the issue's description first paragraph, or the title if empty; use a `>-` folded scalar), `track` (`task` if labelled `flow:task`, else `brief`). Keyed on `name`; never duplicate.
  4. `save_issue` update the issue: `title` → the confirmed `name`, `addLabels: [flow:<track>]`. From here on the title is the join key.

The board entry is identity only. No status, no Linear id — per the harness `board.md`.

Also report, without acting: any folder in `.flow/changes/` or `.flow/archive/` with no board entry (a planner ran without backfilling). Suggest the user add it to the board so it can be mirrored.

## Step 3 — Report

```
FLOW LINEAR — <date> — <project_url>
Board → Linear: <n> created, <n> state updates, <n> unchanged
Linear → board: <n> pulled (<names>), <n> skipped
Off-board folders: <list, or none>
Conflicts: <list, or none>
Next: /flow-brief <name>   (for each pulled entry)
```

`--dry-run` prints the same report with "would" verbs and writes nothing.

## Out of scope

- Writing `.flow/changes/`, `.flow/states/`, application code, git.
- Two-way status. Progress flows disk → Linear only.
- Deleting or archiving Linear issues or projects.

## When to run

After `/flow-brief`, `/flow-task`, `/flow-archive`, or whenever someone added issues in Linear. Cheap to run; safe to run twice.
