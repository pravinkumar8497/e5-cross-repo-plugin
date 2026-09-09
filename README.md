# e5-dev-workflow

Claude Code plugin marketplace: the mandatory harness for any feature or fix
across e5 service repos. Bundles pinned copies of `superpowers`, `ponytail`,
a cherry-picked `grilling` skill, and the `cross-repo-feature` skill that
drives Plan → Design → Implement → Test → Review → Validation.

See `docs/superpowers/specs/2026-09-06-cross-repo-feature-harness-design.md`
for the full design.

## Setup (once per teammate)

```
/plugin marketplace add <git-url-of-this-repo>
/plugin install superpowers
/plugin install ponytail
/plugin install e5-dev-workflow-extras
/plugin install e5-platform-plugin
```

## Updating

After this repo gets new commits, run `/plugin marketplace update` (or
reinstall the affected plugin) to pick up changes.

## Known services

Edit `plugins/e5-platform-plugin/services.json` when a repo is added or removed.
