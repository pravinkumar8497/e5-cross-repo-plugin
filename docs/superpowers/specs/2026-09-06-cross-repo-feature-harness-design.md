# Cross-Repo Feature Harness — Design

## Problem

Features and fixes at e5 routinely span multiple independent service repos
(e5-deployment-mgmt-service, e5-platform-kb, e5-platform-ss-credential-monitor,
e5-platform-ss-interface, e5-rf-fb-worker-core, e5-workflow-manager-service,
and others). Today, both discovering which repos are affected and re-explaining
the change's context in each repo are manual, repeated by hand every time.
There is also no common phase discipline (plan/design/implement/test/review/
validation) enforced across the team, and no common baseline of dev-assistant
behavior (TDD discipline, anti-over-engineering) that every contributor's
Claude Code session follows.

## Goals

- One place to describe a cross-repo feature/fix once; per-repo work is
  briefed from that single source, not re-explained per repo.
- A shared phase discipline: **Plan → Design → Implement → Test → Review →
  Validation**, with Plan+Design done once across all affected repos, and
  Implement→Review run independently per repo.
- Validation covers both (a) each repo satisfying its own acceptance criteria
  and (b) cross-repo interface/integration correctness.
- Distributable to the whole team as a single installable unit, so "install
  this" is the only prerequisite to follow the harness — no per-teammate
  configuration.

## Non-goals

- Automatic discovery of repos via network/org scanning. The set of known
  services is a maintained manifest, not inferred.
- Fully automated multi-agent execution (e.g. the Workflow tool) across all
  repos in one shot. Each repo's implement/test/review runs as a normal,
  human-supervised Claude Code session — the repos differ too much in
  toolchain (Gradle/Java, Python) for uniform batch automation to pay off yet.
- Enforcing a runtime "are prerequisites installed" check. Bundling is the
  enforcement (see Distribution).

## Architecture

### Repos involved

1. **`e5-dev-workflow`** (new, this repo) — a Claude Code plugin marketplace
   bundling three plugins:
   - `superpowers` — vendored at the version currently in use, providing
     brainstorming/writing-plans/TDD/code-review skills the harness builds on.
   - `ponytail` — vendored at the version currently in use, providing the
     lazy/minimal-implementation discipline.
   - `e5-dev-workflow` (the actual new plugin) — contains the
     `cross-repo-feature` skill and a `services.json` manifest of known repos.
2. **`e5-feature-specs`** (new, separate repo, cloned alongside the service
   repos) — holds one directory per feature/fix, each with a spec doc and a
   status table. Shared state that every teammate/repo can reference by
   feature name, independent of any one service repo's lifecycle.
3. Existing service repos — unchanged in structure; each is worked on by a
   normal Claude Code session briefed from its subsection of the shared spec.

### Why a plugin bundle, not per-repo CLAUDE.md text

CLAUDE.md text can't run a guided, stateful process (asking questions,
writing structured specs, tracking phase status) — it's static instructions.
A skill can. Bundling `superpowers` + `ponytail` + `e5-dev-workflow` into one
marketplace means a teammate's only setup step is adding that one marketplace
and installing its plugins; there is nothing left to separately configure or
forget.

### Why a separate specs repo, not `workshop/specs/` or a service repo

A cross-repo feature's spec doesn't belong to any single service repo (it
would be arbitrary and go stale relative to whichever repo happened to hold
it), and `workshop/` is this user's personal local layout, not something every
teammate has. A small dedicated repo, cloned once alongside the service
repos, is the shared equivalent of `workshop/` for feature specs specifically.

## Components

### `services.json` (in the `e5-dev-workflow` plugin)

A flat manifest, maintained by hand as repos are added/removed:

```json
{
  "e5-deployment-mgmt-service": { "purpose": "..." },
  "e5-platform-kb": { "purpose": "..." },
  "e5-platform-ss-credential-monitor": { "purpose": "..." },
  "e5-platform-ss-interface": { "purpose": "..." },
  "e5-rf-fb-worker-core": { "purpose": "..." },
  "e5-workflow-manager-service": { "purpose": "..." }
}
```

Used by the skill during Plan to narrow down candidate repos instead of
scanning the whole filesystem blindly.

### `cross-repo-feature` skill

Three phases:

**1. Plan + Design (shared, run once per feature/fix)**
- Gather the requirement; ask clarifying questions (same style as
  `superpowers:brainstorming`).
- Cross-reference `services.json` (and, where present, each candidate repo's
  own CLAUDE.md) to propose which repos are affected; user confirms/edits.
- Write `e5-feature-specs/<YYYY-MM-DD>-<feature-slug>/design.md` containing:
  - a shared "why / cross-repo architecture" section (including any
    interface contracts repos must agree on — e.g. a message schema one repo
    produces and another consumes)
  - one subsection per affected repo: its specific change, the interface
    contract it must honor, and its acceptance criteria
- Write `e5-feature-specs/<YYYY-MM-DD>-<feature-slug>/status.md`: a table of
  repo → phase reached (Implement/Test/Review/Validation), initialized to
  "not started" for each affected repo.

**2. Implement → Test → Review (independent per repo)**
- Run as a normal Claude Code session scoped to one repo at a time.
- Briefing = that repo's subsection from `design.md` + a link to the shared
  section for cross-repo context (not the whole spec re-explained).
- Follows the repo's own conventions; TDD and code-review come from the
  bundled `superpowers` skills, minimal-diff discipline from `ponytail` —
  both already active since they're in the same marketplace.
- On completing each phase for a repo, the skill updates that repo's row in
  `status.md`.

**3. Validation (shared, once all affected repos reach Review)**
- Acceptance check: for each repo, confirm its subsection's acceptance
  criteria are met.
- Integration check: confirm the interface contracts declared in the shared
  section actually match across repos (e.g. producer's schema vs. consumer's
  expectation) — read the relevant code/config in each repo to verify, don't
  just re-read the spec.
- Mark `status.md` complete.

## Distribution

Single marketplace repo (`e5-dev-workflow`) containing:

```
.claude-plugin/marketplace.json
plugins/
  superpowers/          # vendored copy, pinned version
  ponytail/              # vendored copy, pinned version
  e5-dev-workflow-extras/
    .claude-plugin/plugin.json
    skills/
      grilling/          # cherry-picked from mattpocock/skills, pinned version
  e5-dev-workflow/
    .claude-plugin/plugin.json
    skills/
      cross-repo-feature/SKILL.md
    services.json
```

Skills are cherry-picked from third-party marketplaces (not the whole
marketplace) to avoid pulling in skills that overlap or conflict with
superpowers (e.g. mattpocock-skills' own `tdd`/`code-review`). Each
cherry-picked skill is copied into a small `e5-dev-workflow-extras` plugin
owned by this repo, so its version is pinned and updates are deliberate
(re-copy on demand), not silently pulled from the upstream marketplace.

Team setup (one time per teammate), documented as a copy-paste block in this
repo's README:
```
/plugin marketplace add <git-url-of-this-repo>
/plugin install superpowers
/plugin install ponytail
/plugin install e5-dev-workflow-extras
/plugin install e5-dev-workflow
```

**Open item to verify during implementation:** whether Claude Code's
`plugin.json` supports a native "requires plugin X" dependency field. If yes,
`e5-dev-workflow`'s plugin.json declares `superpowers`/`ponytail`/
`e5-dev-workflow-extras` as dependencies and install pulls all four
automatically. If no, the manual install block above is what ships — this
doesn't change anything else in the design.

## Error handling / edge cases

- **Repo not in `services.json`**: skill asks the user to confirm the repo
  and add it to the manifest as part of the Plan phase, rather than failing.
- **A repo's Implement/Test/Review session is run out of order** (e.g. before
  Plan+Design finishes): the skill checks for `design.md`'s existence before
  proceeding into per-repo work and stops with a clear message if missing.
- **Interface contract mismatch found at Validation**: treated as a normal
  bug — the skill identifies which repo(s) need a follow-up change and it
  goes through Implement→Test→Review again for just that repo, not a full
  restart.

## Testing

- The `cross-repo-feature` skill's phase-gating logic (won't start per-repo
  work without `design.md`, updates `status.md` correctly) should be
  exercised with a small dry-run feature against 2 dummy/test repos before
  rollout to the team.
- No automated test framework for the skill itself is warranted (it's an
  instruction file, not code) — the "test" is the dry run above plus normal
  skill review.

## Revision — v0.2.0 (2026-09-06)

Prompted by running this harness end-to-end on the credential-release
app-name feature: Review was only a checkbox folded into the old Phase 2,
with no defined mechanism. Split into its own mandatory phase (now Phase 3,
Validation becomes Phase 4), using the bundled `superpowers:requesting-
code-review` / `receiving-code-review` skills, with a fix → re-review loop
until clean before a repo can proceed to Validation. See `SKILL.md` for the
authoritative phase definitions — this doc's original phase list (Plan →
Design → Implement → Test → Review → Validation) still describes the
concepts correctly, just with Review now formalized rather than implied.
