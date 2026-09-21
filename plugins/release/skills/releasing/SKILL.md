---
name: releasing
description: Use when asked to release, ship, deploy, squash to main, back-merge, tag a version, or push to staging/production in any repo that uses dev = staging and main = production with CI/CD deploying on push. Encodes the safe squash-release workflow and its invariants.
---

# Releasing: dev → staging, squash → main → production

**Model.** Work on `dev`. `main` is built only from **squash** commits — one
`release: vX.Y.Z` per ship, tagged after CI is green. CI/CD deploys on push:
`dev` → staging, `main` → production. **Nobody runs a deploy by hand.**

**Commands.** `/release:preflight` (CI gate locally), `/release:deploy-stage`
(merge into `dev`, push), `/release:deploy-prod vX.Y.Z` (squash, push, tag,
back-merge). `deploy-prod` is never auto-invoked — the user fires it.

## Invariants (STOP if any fails)

- `git status --short` clean before any release step.
- **CI green on `dev`** before squashing. A red `dev` blocks the release.
- `git merge-base --is-ancestor origin/main dev` → exit 0. Proves nothing on
  `main` is lost: the squash sits on top of `origin/main`, the push is a
  fast-forward, never a force.
- After the squash commit, `git diff --stat main dev` is **empty** (trees
  identical). `git log main..dev` is useless here — squash means no shared
  lineage; use the diff.
- Tag only after `main`'s CI is green.
- Always back-merge `main` → `dev` after a release, or the next ancestry check fails.

## Repo specifics live in the repo

Read `CLAUDE.md`, `.harness/deploy.md`, `CONTRIBUTING.md` for: which gh account
has push (`gh auth switch --user …` — a push 404 is almost always this), staging /
production URLs, CD workflow name (for `gh workflow run … --ref dev` redeploys),
platform (Cloudflare Workers via `wrangler` — needs `--env`; Railway — branch
tracking; etc.). Never guess these; if undocumented, ask or default to
`gh run list --branch <b> --limit 1`.

## Gotchas seen in the wild

- `workflow_run`-triggered CD reads its workflow file from the **default
  branch** — a new `cd.yml` on `dev` is dormant until it lands on `main`.
- Commit-message hooks / guards may match strings like `wrangler deploy` inside
  the message; anchor guards to command boundaries.
- Committing to `dev` mid-squash (another session, an editor) breaks the tree
  check — abort with `git branch -f main origin/main`, wait, redo.
- Test runners that `reuseExistingServer` pick up a stale dev server from another
  repo on the same port.
