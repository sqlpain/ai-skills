# Tracker — Artifact File Format

All Tracker data lives as plain files inside each project at `<project-root>/.tracker/`. Files are markdown with YAML frontmatter (except `project.yaml`). This document is the normative format spec; the `tracker` skill and the viewer (`index.html`) must both conform to it. The files are the only source of truth — there is no service or database.

## 1. Directory layout

```
<project-root>/.tracker/
  index.html                        # read-only viewer, copied in by the skill
  project.yaml                      # project metadata (required)
  specs/
    SPEC-001-authentication.md
    SPEC-002-reporting.md
  plans/
    PLAN-001-auth-mvp.md
  tasks/
    TASK-0001-login-endpoint.md
    TASK-0002-session-storage.md
  decisions/
    D-001-session-store.md
  outcomes/                         # LEGACY: read-only, no new files
    OUT-0001-2026-08-12-auth-session.md
  archive/                          # optional; same layout, excluded from the viewer
```

### File naming

`<ID>-<kebab-slug>.md`. The **ID prefix is authoritative**; the slug is cosmetic and may be edited freely. Files are matched by the leading ID, so renaming the slug is safe. Two files with the same ID in one project are flagged as a collision by the viewer.

## 2. Common frontmatter rules

- YAML frontmatter delimited by `---` lines; body is free-form GitHub-flavored markdown.
- Timestamps are ISO 8601 dates (`2026-08-12`) or datetimes; local time assumed.
- Unknown keys are preserved and ignored (forward compatibility).
- Link fields reference IDs (`SPEC-001`) or spec items (`SPEC-001.3`). Broken links are warnings in the viewer, not errors.
- `tags` is a free vocabulary of kebab-case strings, filterable in the viewer.
- A file whose frontmatter is invalid, whose `id` does not match its filename, or whose status is outside its vocabulary is still shown by the viewer, flagged as a parse error. Nothing is silently skipped.

## 3. `project.yaml`

```yaml
id: sql-review-engine          # stable slug
name: SQL Review Engine
description: Automated review pipeline for database changes.
created: 2026-08-12
tags: [database, tooling]
```

Older files may carry a `settings:` block; it is ignored.

## 4. Spec — `specs/SPEC-NNN-<slug>.md`

*What should exist.* Specs contain enumerated **spec items** — the checkable units that tasks cover and the viewer audits.

```markdown
---
id: SPEC-001
title: Authentication
status: approved            # draft | review | approved | superseded
created: 2026-08-12
updated: 2026-08-12
superseded_by: null         # SPEC id when status = superseded
tags: [security]
---

## Summary

Users authenticate with username/password; sessions persist across restarts.

## Requirements

<!-- Spec items: "### <n>. <title>" headings. Item ref = SPEC-001.<n> -->

### 1. Login endpoint

`POST /login` accepts credentials, returns a session token. Rate-limited.

### 2. Session persistence

Sessions survive service restart; expiry 24h sliding.

### 3. Logout

`POST /logout` invalidates the token immediately.

## Out of scope

SSO, MFA (future spec).
```

**Spec item convention:** each `### <n>. <title>` heading under `## Requirements` defines item `SPEC-NNN.<n>`. Item numbers are stable — never renumber; retire an item by appending `(withdrawn)` to its heading. The viewer parses these headings to power coverage tracking.

## 5. Plan — `plans/PLAN-NNN-<slug>.md`

*How it will be built.* Section order and depth follow `examples/known-good-plan.md`; `templates/plan.md` lists every section.

```markdown
---
id: PLAN-001
title: Auth MVP
status: in-progress         # draft | approved | in-progress | done | abandoned
created: 2026-08-12
updated: 2026-08-12
implements: [SPEC-001]      # specs this plan addresses
tags: []
---

## Objective
## Approach
## Architecture
## Alternatives considered
## Assumptions
## Risks
## Effort estimate
## Management summary
## Verification
## Open questions
## What still needs a human
## Task breakdown
```

Plan **progress is derived**, not stored: the viewer computes it from tasks whose `plan` equals this plan's ID. `status` is the human/Claude-declared lifecycle state; the viewer highlights mismatches (e.g. all tasks done but plan still `in-progress`).

## 6. Task — `tasks/TASK-NNNN-<slug>.md`

*A unit of work.* Small enough to complete in one session. It carries its own mini-plan and, when finished, its own outcome.

```markdown
---
id: TASK-0001
title: Add rate limiting to POST /login
status: done                # todo | in-progress | blocked | done | dropped
created: 2026-08-12
updated: 2026-08-12
plan: PLAN-001              # optional
covers: [SPEC-001.1]        # spec items this task addresses (optional)
blocked_by: []              # TASK ids or free text "waiting on X"
decisions: [D-001]          # decisions that shaped this task (optional)
priority: normal            # low | normal | high
tags: []
---

## Summary

One plain-English sentence.

## Deliverable

One sentence: what exists when this is done.

## Mini-plan

1. Short imperative step.
2. Another step.
3. Another step.

## Description

Reasoning, constraints, worked examples.

## Acceptance

- [ ] Valid credentials return 200 + token cookie
- [ ] 6th attempt within a minute returns 429

## Notes

(running notes during work — optional)

## Outcome

**Done:** …
**Deviations:** …
**Verification:** …
**Follow-ups:** …
```

- Checkbox lines under `## Acceptance` are shown as sub-progress by the viewer.
- `## Mini-plan` holds 3–6 numbered steps; more than 6 means the task should be split.
- `## Outcome` stays empty until the work is done, then records what happened. Tasks that end blocked or partial keep their true `status` and record progress here.
- Older tasks without `Summary`, `Deliverable`, `Mini-plan` or `Outcome` remain valid; the viewer shows whichever sections exist.

## 7. Decision — `decisions/D-NNN-<slug>.md`

*A choice between real alternatives, and why.* One file per decision.

```markdown
---
id: D-001
title: Sessions are stored in SQLite
status: accepted            # proposed | accepted | superseded | rejected
date: 2026-08-12
superseded_by: null         # D id when status = superseded
relates: [PLAN-001, TASK-0002]   # SPEC / PLAN / TASK ids this affects
tags: []
---

## Context
## Options considered
## Decision
## Consequences
```

Decisions are not rewritten. To change one, write a new decision and set the old one to `superseded` with `superseded_by`.

## 8. Legacy — `outcomes/OUT-NNNN-<date>-<slug>.md`

Outcomes are no longer created; the outcome of a task is recorded in the task's `## Outcome`. Existing outcome files remain on disk and are shown read-only by the viewer. Their frontmatter fields are `id`, `title`, `date`, `tasks`, `session`, `result` (`completed | partial | blocked | abandoned`), `follow_ups`, `corrects`, `tags`.

## 9. ID allocation

- Format: `SPEC-NNN` / `PLAN-NNN` / `D-NNN` (3 digits), `TASK-NNNN` (4 digits), zero-padded, per-project scope.
- Allocation rule: list the artifact directory **and its `archive/` counterpart**, take the highest existing ID across both, add 1.
- Two files with the same ID (for example from merged branches) are shown as a collision; fix by renaming one file and updating links that point at it.
