---
name: cross-repo-feature
description: Use for any feature or fix that touches one or more e5 service repos. Runs Plan+Design once (shared across affected repos), then Implement+Test, then Review, independently per repo, then a shared Validation pass. Use when the user describes a problem statement that needs to be turned into work across e5 services, or says "cross-repo feature", "new feature/fix across services", or names multiple e5-* repos.
---

# Cross-Repo Feature Harness

Full design rationale: see this plugin repo's
`docs/superpowers/specs/2026-09-06-cross-repo-feature-harness-design.md`.

Known services live in `services.json` next to this skill. If a repo the
user mentions isn't listed there, ask them to confirm it and add it to that
manifest before continuing — never silently scan the filesystem for repos.

Specs and status for every feature/fix live in a separate `e5-feature-specs`
repo, cloned alongside the service repos, one directory per feature:
`e5-feature-specs/<YYYY-MM-DD>-<feature-slug>/design.md` and `.../status.md`.

## Repo knowledge notes

Each known service has running notes under `knowledge/<repo-name>/`, next to this
skill in this plugin (see "OKF pattern" below for the folder/file shape). It exists
so this skill doesn't re-explore the same code from scratch every time — it records
things that took real investigation to find (which file actually owns a piece of
logic, non-obvious schema/join facts, gotchas, format assumptions, transitional
shortcuts and their ceiling) rather than things a directory listing would show. Not
a changelog and not a design doc — keep entries short, dated, and file/symbol-specific.

**Read it first:** before exploring a repo in Phase 1 (or before touching a repo in
Phase 2), read `knowledge/<repo-name>/_overview.md` if it exists, then whichever
`<feature-slug>.md` in that folder matches the work at hand. Treat it as a
fast-start hint, not ground truth — if it looks stale or contradicts what you read
in the actual code, trust the code and fix the note.

**Update it last:** as the final action of Phase 3 for this feature (after
`status.md` is marked complete), append or update the matching
`knowledge/<repo-name>/<feature-slug>.md` for every repo that was actually touched
or investigated (create the file, and add it to `_overview.md`'s `features` list, if
this is a new feature for that repo), capturing anything non-obvious this feature
surfaced (new file/symbol locations, schema facts, format assumptions, transitional
shortcuts introduced and their removal ceiling). Skip repos that were only named in
the problem statement but turned out not to need changes — don't
pad the notes with "nothing here."

## OKF pattern (Organized Knowledge Format)

Each known repo gets its own folder under `knowledge/<repo-name>/`, not a single
flat file — one repo can have several unrelated features, and cramming them into
one file makes it useless to grep for just one.

```
knowledge/<repo-name>/
  _overview.md              # repo-level frontmatter + summary + feature index
  <feature-slug>.md         # one file per feature/subsystem, own frontmatter
  <feature-slug>.md
```

`_overview.md` frontmatter:

```yaml
---
service: <repo-name>
role: library | service
depends_on: [<repo-name>, ...]      # repos this one calls into / builds against
consumed_by: [<repo-name>, ...]     # repos that depend on this one
features: [<feature-slug>, ...]     # every <feature-slug>.md in this folder
integrations: [<service-a>--<service-b>, ...]   # integration docs involving this repo
---
```

Body: one paragraph on what the repo is/does, then a short feature index (one line
per `<feature-slug>.md`, what it covers), then anything genuinely repo-wide that
doesn't belong to one feature (environment/build quirks, cross-cutting gotchas).

Each `<feature-slug>.md` frontmatter:

```yaml
---
service: <repo-name>
feature: <feature-slug>
status: active | deprecated | experimental
related_files: [path/to/File.java, ...]     # entry points, not an exhaustive list
integrations: [<service-a>--<service-b>, ...]   # only if this feature is part of one
---
```

Body: same "short, dated, file/symbol-specific" notes discipline as before — one
`## <YYYY-MM-DD> — <short title>` section per finding, nothing that belongs in
`design.md` or a changelog.

`services.json` carries the same shape as `_overview.md`'s frontmatter per entry
(`purpose`, `role`, `depends_on`, `consumed_by`) - fill in a new repo's entry there
when you first touch it deeply enough to know its actual role and dependencies.
Don't backfill an existing empty entry, or split an existing flat
`knowledge/<repo-name>.md` into features, from a guess; only from something you've
actually verified by reading the repo.

## Integration docs

When Plan+Design (below) identifies that two repos share a real interface contract -
a message schema one produces and another consumes, a shared library API, a
version-pin relationship - write or update
`integrations/<service-a>--<service-b>.md` (alphabetical order in the filename,
`--` separator). Same frontmatter idea, smaller:

```yaml
---
integration: <service-a> <-> <service-b>
protocol: grpc | kafka | java-library-api | rest | ...
direction: <which one depends on which, or "bidirectional">
---
```

Body: what actually crosses the boundary, the contract that must stay in sync
between both sides, and where to look first when something breaks across it. This is
what Phase 4's integration check (below) should read before re-deriving the contract
from source each time.

**Read it first:** in Phase 1 step 2, alongside each candidate repo's
`knowledge/<repo-name>/`, check `integrations/` for any doc naming two of the
affected repos - it's a faster start than re-deriving the shared contract from source.

**Update it last:** same timing as the repo knowledge notes below - after
`status.md` is marked complete, for any integration this feature actually touched or
newly discovered.

## Phase 1 — Plan + Design (shared, run once per feature/fix)

1. Get the problem statement from the user. Ask clarifying questions one at
   a time (same style as `superpowers:brainstorming`) until scope is clear.
2. Cross-reference `services.json`, each candidate repo's own CLAUDE.md, and
   its `knowledge/<repo-name>/_overview.md` (if present) to propose which
   repos are affected. Confirm the list with the user before writing anything.
3. Write `e5-feature-specs/<date>-<slug>/design.md` with:
   - a shared section: why, cross-repo architecture, any interface
     contracts repos must agree on (e.g. a message schema one repo produces
     and another consumes)
   - one subsection per affected repo: its specific change, the interface
     contract it must honor, and its acceptance criteria
4. Write `e5-feature-specs/<date>-<slug>/status.md`: a table of
   repo → phase reached, all rows starting at "not started".
5. Do not start any repo's Implement/Test/Review until this step is done —
   if asked to jump ahead, point to the missing `design.md` and stop.

## Phase 2 — Implement + Test (independent, per repo)

Run one repo at a time, in a session/agent scoped to just that repo.

1. Brief the session with only: that repo's subsection from `design.md`,
   plus a link to the shared section for cross-repo context. Do not
   re-explain the whole spec.
2. Follow the repo's own conventions. TDD comes from the bundled
   `superpowers:test-driven-development` skill; minimal-diff discipline
   comes from the bundled `ponytail` skill — both are already active in
   this marketplace, don't re-implement their behavior here.
3. When the repo's implementation is done and its tests pass (or, per
   `status.md` convention, are marked `n/a` with the reason — e.g. this
   environment can't build/test that repo), update that repo's Implement
   and Test columns in `status.md`, then move to Phase 3 for that repo.

## Phase 3 — Review (independent, per repo)

This is a distinct, mandatory phase — not a checkbox folded into Phase 2.
It exists to catch what the implementer missed before Validation ever
looks at the code.

1. Dispatch review using the bundled `superpowers:requesting-code-review`
   skill (its `code-reviewer.md` template, as a `general-purpose` subagent)
   against that repo's diff for this feature (`BASE_SHA` = the branch point
   before this feature's commits, `HEAD_SHA` = the repo's current HEAD).
   Give the reviewer that repo's `design.md` subsection as
   `{PLAN_OR_REQUIREMENTS}` — it must check the diff against the actual
   acceptance criteria, not just general code quality.
2. If the review comes back clean (no Critical/Important findings), mark
   that repo's Review column `done` in `status.md` and move on.
3. If it finds Critical or Important issues: apply `superpowers:receiving-
   code-review` to weigh the feedback (push back on anything wrong, with
   reasoning, rather than applying it blindly), fix what holds up, then
   repeat step 1 (re-review) — loop until a review pass comes back clean.
   Minor/nitpick findings don't block the loop; note them in `status.md`
   and move on.
4. Never skip straight to Phase 4 for a repo whose Review column isn't
   `done`.

## Phase 4 — Validation (shared, once all affected repos reach Review)

1. Acceptance check: for each repo, confirm its subsection's acceptance
   criteria in `design.md` are actually met — read its code, don't just
   trust the status table.
2. Integration check: confirm the interface contracts declared in the
   shared section of `design.md` actually match across repos (e.g. a
   producer's schema vs. a consumer's expectation) by reading the relevant
   code/config in each repo. Check `integrations/<service-a>--<service-b>.md`
   for the affected repo pair first if one exists - it may already document
   the exact contract to verify.
3. If a mismatch is found, treat it as a normal bug: identify which repo(s)
   need a follow-up change and send just those back through Phase 2 — don't
   restart the whole feature.
4. Mark `status.md` complete once both checks pass for every affected repo.
5. **Last step:** update `knowledge/<repo-name>/<feature-slug>.md` for every
   repo actually touched or investigated this feature, per "Repo knowledge
   notes" above,
   and `integrations/<service-a>--<service-b>.md` for any repo pair whose
   shared contract this feature touched or newly discovered, per
   "Integration docs" above. Do this after `status.md` is marked complete,
   not before — notes should reflect what was actually true of the finished
   change.
