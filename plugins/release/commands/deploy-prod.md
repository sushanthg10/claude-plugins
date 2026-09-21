---
description: Ship to production — squash dev→main as "release: <version>", push, tag after CI is green, back-merge to dev. CI/CD deploys main. No manual deploy.
disable-model-invocation: true
argument-hint: "<version> e.g. v1.2.0"
---

Release to production as **`$ARGUMENTS`**. Pushing `main` runs CI and CD deploys production — you don't deploy by hand. Confirm with the user before pushing `main` (it ships live). Stop and report if any check fails.

0. **Discover repo specifics** — read `CLAUDE.md` and, if present, `.harness/deploy.md` / `CONTRIBUTING.md` for the gh push account, the production URL, and how the repo watches deploys. Also check whether `dev` CI may skip heavy suites (some repos allow `[skip tests]` on `dev` but never on `main`).
1. **Version** — `$ARGUMENTS` must look like `v1.2.3`. Missing or malformed → STOP and ask. `git tag -l "$ARGUMENTS"` must be empty and it must be higher than `git describe --tags --abbrev=0` (if any tag exists).
2. **Preflight** — on `dev`, `git status --short` clean, and **CI green on `dev`** (`gh run list --branch dev --limit 1`). Red or pending → STOP; never squash a red `dev`.
3. **Safety** — `git fetch origin`, then `git merge-base --is-ancestor origin/main dev` (exit 0 = nothing on `main` is lost; else STOP — someone committed to `main` directly; reconcile first).
4. **Squash to main** — `git checkout main && git reset --hard origin/main && git merge --squash dev && git commit -m "release: $ARGUMENTS"` (end with the repo's `Co-Authored-By:` line).
5. **Tree check** — `git diff --stat main dev` MUST be empty (trees identical). If not, STOP.
6. **Push main** — `git push origin main` (fast-forward, never force; 404 → switch gh account per step 0). Watch: `gh run list --branch main --limit 1`. On green, CD deploys production.
7. **Tag it** — once CI is green: `git tag -a "$ARGUMENTS" -m "release: $ARGUMENTS" && git push origin "$ARGUMENTS"`. Tag after green, so a tag never points at a failed build.
8. **Back-merge** — `git checkout dev && git merge main --no-edit && git push origin dev`. This keeps `dev`'s ancestry linked so the next `merge-base` check passes.
9. **Verify** — hit the production URL; confirm the change is live (build + CDN differ from dev).

Staging is a separate trigger (`/release:deploy-stage`).
