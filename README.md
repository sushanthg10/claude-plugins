# claude-plugins

Claude Code marketplace `sushanthg10`. Install in any repo:

```
/plugin marketplace add sushanthg10/claude-plugins
/plugin install release@sushanthg10
```

## release

dev = staging, main = production from squash commits, CI/CD deploys on push.

| Command                        | What                                                                 |
| ------------------------------ | -------------------------------------------------------------------- |
| `/release:preflight`           | run the repo's CI gate locally                                       |
| `/release:deploy-stage`        | merge current branch → `dev`, push → CD staging                      |
| `/release:deploy-prod vX.Y.Z`  | squash `dev`→`main` as `release: vX.Y.Z`, push → CD prod, tag, back-merge |

Repo specifics (push account, URLs, platform) are read from the repo's `CLAUDE.md`
/ `.harness/deploy.md` at run time — document them there. The `releasing` skill
auto-loads when an agent is asked to ship/release/squash/tag.

Update after edits: `/plugin marketplace update sushanthg10`.
