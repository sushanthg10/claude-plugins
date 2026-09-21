---
description: Run what CI runs before a commit or PR. Reads the repo's CI workflow and package scripts; stops on the first failure.
---

Run the repo's CI gate locally, in order, and stop on the first failure. Otherwise give a clear pass summary.

1. **Discover the gate** — read `.github/workflows/ci.yml` (or the workflow that runs on push to `dev`/`main`) and `package.json` scripts. Run the same steps CI runs, in CI's order. Typical: `format:check`, `lint`, `typecheck`, `test`, `build`. If the repo has a documented local gate in `CONTRIBUTING.md` / `CLAUDE.md`, prefer that list.
2. **Heavy suites** (e2e / browser) only if the diff touches user-facing flows. Kill a stale dev server from another repo on the port first — test runners often reuse whatever is listening.
3. If anything fails, surface the actual error output — never just "failed".
