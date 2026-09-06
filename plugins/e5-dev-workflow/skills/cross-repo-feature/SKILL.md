---
name: cross-repo-feature
description: Use for any feature or fix that touches one or more e5 service repos. Runs Plan+Design once (shared across affected repos), then Implement/Test/Review independently per repo, then a shared Validation pass. Use when the user describes a problem statement that needs to be turned into work across e5 services, or says "cross-repo feature", "new feature/fix across services", or names multiple e5-* repos.
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

## Phase 1 — Plan + Design (shared, run once per feature/fix)

1. Get the problem statement from the user. Ask clarifying questions one at
   a time (same style as `superpowers:brainstorming`) until scope is clear.
2. Cross-reference `services.json` and each candidate repo's own CLAUDE.md
   to propose which repos are affected. Confirm the list with the user
   before writing anything.
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

## Phase 2 — Implement → Test → Review (independent, per repo)

Run one repo at a time, in a session/agent scoped to just that repo.

1. Brief the session with only: that repo's subsection from `design.md`,
   plus a link to the shared section for cross-repo context. Do not
   re-explain the whole spec.
2. Follow the repo's own conventions. TDD and code review come from the
   bundled `superpowers` skills; minimal-diff discipline comes from the
   bundled `ponytail` skill — both are already active in this marketplace,
   don't re-implement their behavior here.
3. When the repo finishes Implement, Test, and Review, update that repo's
   row in `status.md` to reflect the phase reached.

## Phase 3 — Validation (shared, once all affected repos reach Review)

1. Acceptance check: for each repo, confirm its subsection's acceptance
   criteria in `design.md` are actually met — read its code, don't just
   trust the status table.
2. Integration check: confirm the interface contracts declared in the
   shared section of `design.md` actually match across repos (e.g. a
   producer's schema vs. a consumer's expectation) by reading the relevant
   code/config in each repo.
3. If a mismatch is found, treat it as a normal bug: identify which repo(s)
   need a follow-up change and send just those back through Phase 2 — don't
   restart the whole feature.
4. Mark `status.md` complete once both checks pass for every affected repo.
