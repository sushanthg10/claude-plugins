---
description: Merge the current branch into dev and push — CI runs, CD deploys staging. No manual deploy.
---

Ship the branch you're on to **staging**. Merging into `dev` and pushing is the whole job — CI gates, CD (or the platform's branch tracking) deploys `dev` to staging. You never deploy by hand.

0. **Discover repo specifics** — read `CLAUDE.md` and, if present, `.harness/deploy.md` / `CONTRIBUTING.md` for: the gh account that has push (e.g. `gh auth switch --user …`), the staging URL, how to watch the deploy. If none is documented, default to `gh run list --branch dev --limit 1`.
1. **Check where you are** — `git status --short` must be clean (commit or stash first); note the branch with `git rev-parse --abbrev-ref HEAD`.
   - On **`main`** → STOP. `main` is production; never stage from it.
   - On **`dev`** → skip step 2.
2. **Merge into `dev`** — `git checkout dev && git pull --ff-only origin dev && git merge <branch> --no-edit`. Conflicts → STOP and report the conflicting paths; do not resolve silently.
3. **Push** — `git push origin dev`. A 404 means the wrong gh account — switch per step 0 and retry once.
   Nothing to push but want a redeploy? `gh workflow run <ci workflow> --ref dev` if the repo's CD chains off CI; otherwise say so.
4. **Watch it** — `gh run list --branch dev --limit 1` until green; then confirm the staging URL responds.

Ship to production with `/release:deploy-prod <version>`.
