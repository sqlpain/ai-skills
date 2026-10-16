---
id: PLAN-014
title: Role Factory rollout — gated generation pipeline
status: draft
created: 2026-08-20
updated: 2026-08-20
implements: [SPEC-009]
tags: [role-factory, migration]
---

<!--
This is the "known-good" counterpart to ./known-bad-lossy-plan.md. It is
adapted from a real migration case: a 406-line design document (21 headings —
architecture, a G0–G7 gate state machine, an 11-row rules table, a worked
YAML example, an options appraisal with pros/cons tables, effort estimates
with justification, a management summary) was handed to the tracker skill
with a request for a phased implementation plan. The first pass silently
dropped ~45% of it. This file shows what "copy, don't summarise" actually
produces: every source section has a home, the state machine and rules table
survive verbatim, the options appraisal keeps its tradeoffs, and nothing is
compressed into a checkbox. Compare its length and section count to the lossy
version — the difference is not padding, it's the content that was lost.
-->

## Objective

Ship the Role Factory generation pipeline: given a role definition, run it
through a seven-gate validation sequence (G0–G7) and emit a reviewed,
versioned role package plus `docs/role-factory/README.md` documenting the
gate contract for downstream teams. This plan covers the pipeline
implementation; SPEC-009 items 1–11 track the corresponding requirements.

## Approach

The pipeline is a linear state machine over gates G0–G7. Each gate is a pure
function `(RoleDraft, GateContext) -> GateResult` where `GateResult` is one of
`pass`, `fail(reason)`, or `needs_human(reason)`. A draft only advances to
gate N+1 after gate N returns `pass`. `needs_human` suspends the draft in a
review queue; a human resolution re-enters the same gate rather than
restarting the sequence.

### Gate sequence

```
G0 Intake        — draft is well-formed YAML, required fields present
G1 Schema        — draft validates against the role schema (jsonschema)
G2 Naming        — role name is unique, matches naming convention
G3 Permissions   — requested permissions are a subset of the requesting team's ceiling
G4 Conflict scan — no overlapping permission grant with an existing active role
G5 Risk score    — computed risk score below the auto-approve threshold, else -> needs_human
G6 Peer review   — a second engineer (not the author) approves
G7 Publish       — role package versioned, written to the registry, README regenerated
```

Gates G0–G4 are deterministic and side-effect-free — safe to re-run. G5's risk
score is a pure function of the diff but its *threshold* is configurable per
team, so it is re-evaluated (not cached) on every re-entry after a
`needs_human` resolution. G6 and G7 have side effects (registry writes,
notifications) and are not re-run once passed.

### Worked example — role draft entering the pipeline

The following is the exact input shape gate G0 validates against; preserved
verbatim from the source design because the field names and nesting are
what G1's schema check enforces:

```yaml
role:
  name: payments-read-only
  team: payments-platform
  requested_by: j.example@payroc.com
  permissions:
    - resource: payments.transactions
      access: read
    - resource: payments.refunds
      access: read
  justification: >
    Read access for the new reconciliation dashboard; no write path needed.
  risk_hints:
    handles_pii: false
    handles_funds_movement: false
gate_context:
  team_permission_ceiling: [payments.*:read, payments.*:write]
  existing_roles_in_team: [payments-write, payments-admin]
```

Given this input: G0 passes (well-formed), G1 passes (matches schema), G2
passes (name unique), G3 passes (both requested permissions are within the
team's `read`/`write` ceiling on `payments.*`), G4 passes (no existing role
grants `payments.transactions:read` or `payments.refunds:read`), G5 computes a
low risk score (no PII, no funds movement) and auto-approves, so G6/G7 run
without a human review cycle.

## Architecture

```
role-factory/
  drafts/            # incoming YAML, one file per pending draft
  pipeline/
    gates/           # g0_intake.ts ... g7_publish.ts, one module per gate
    engine.ts         # advances a draft through gates, persists GateResult history
    review_queue.ts   # needs_human suspension + resolution
  registry/           # published role packages, versioned by content hash
docs/role-factory/README.md   # generated from gate metadata on every G7 publish
```

The gate engine persists a `GateResult` history per draft so a suspended
draft's audit trail (which gates passed, when, and why G5 flagged it) survives
process restarts — this was the source document's rationale for not using an
in-memory-only state machine, and it is the reason G7 regenerates the README
rather than hand-maintaining it: the gate metadata is the single source of
truth for the documented contract.

## Alternatives considered

| Option | Description | Pros | Cons |
|---|---|---|---|
| A: Linear gate engine (chosen) | Gates run strictly in order G0→G7, one persisted state machine per draft | Simple to reason about; audit trail is a straight line; matches how reviewers already think about the process | No way to run independent gates concurrently — a slow G5 risk score blocks G6/G7 even when they don't depend on it |
| B: DAG-based gate graph | Gates declare dependencies; independent gates (e.g. G2 naming, G3 permissions) run in parallel | Faster wall-clock for drafts with few dependencies | Audit trail is a DAG, not a line — harder for reviewers to follow; no gate in the current set is actually slow enough to justify the complexity |
| C: Single big-bang validation function | One function checks everything and returns a combined pass/fail | Least code to write initially | No partial progress — a draft that fails at "gate 7 equivalent" re-runs everything; no place to hook `needs_human` suspension per-check; audit trail would have to be reconstructed after the fact |

Option A was chosen: the source design's estimate is that fewer than 5% of
drafts will ever have more than one gate in a `needs_human` state
simultaneously, so B's concurrency benefit doesn't justify its audit-trail
cost. C was rejected outright because the review-queue requirement (G5/G6
suspension) doesn't compose with a single validation function.

## Assumptions

- `Assumption: team permission ceilings (used by G3) are already maintained
  elsewhere and this pipeline only reads them, it does not manage them.`
- `Assumption: the risk-scoring model behind G5 exists and is stable; this
  plan implements the gate that calls it, not the model itself.`

## Risks

- G5's per-team configurable threshold means two teams can see different
  auto-approve behavior for structurally identical drafts — flagged in the
  source design as a fairness concern; accepted for v1, revisit if raised by
  a team.
- Gate engine persistence (SQLite, per `docs/api.md` conventions) becomes a
  write bottleneck if draft volume exceeds ~50/day; no mitigation planned
  for v1, noted as a scaling open question below.

## Effort estimate

| Component | Estimate | Justification |
|---|---|---|
| Gates G0–G4 (deterministic) | 3 days | Each is a small pure function against an existing schema; G2/G3/G4 share a lookup pattern already used elsewhere in the codebase |
| G5 risk score integration | 2 days | Model already exists; work is wiring + per-team threshold config, not model development |
| G6 review queue + notification | 3 days | New: suspension/resumption state, plus a notification hook — no existing pattern to reuse |
| G7 publish + README generation | 2 days | Registry write is straightforward; README generation from gate metadata needs a small template layer |
| Engine + persisted history | 2 days | State machine driver plus the audit-trail schema |
| **Total** | **12 days** | Sequential estimate; G0–G4 and the engine can be built in parallel by two engineers, compressing wall-clock to ~7 days |

## Management summary

Role Factory replaces ad hoc, email-approved role grants with a seven-gate
pipeline that is auditable end to end and requires human review only for
higher-risk requests (G5/G6). Estimated at 12 engineer-days sequential, ~7
days with two engineers in parallel. Primary risk is the per-team configurable
auto-approve threshold producing inconsistent outcomes across teams; accepted
for v1. No changes to who owns permission ceilings — this pipeline consumes
that data, it does not become the new source of truth for it.

## Verification

- Unit tests per gate module (pure functions — table-driven against the
  worked example above and its negative variants).
- Integration test: full draft through G0→G7 with no `needs_human` detour
  (happy path, matches the worked example), and one full run that hits a
  `needs_human` suspension at G5 and resumes correctly.
- Manual check: generated `docs/role-factory/README.md` after a real G7
  publish matches the gate contract described in this plan.

## Open questions

- Does the review-queue notification (G6) need to integrate with the existing
  Slack bot, or is email sufficient for v1? Source design flagged this as
  undecided — needs a human answer before TASK for G6 is started.
- What happens to a draft that sits in `needs_human` for more than N days —
  auto-expire, or indefinite? Not addressed in the source design.
- Should the per-team G5 threshold be visible to the requesting engineer
  before submission, so they can anticipate a `needs_human` detour? Raised
  informally during design review but not resolved in the source document.

## What still needs a human

Gate logic, effort estimates and the options appraisal above are ready to
build from as-is. Three things are explicitly out of this plan's scope and
need a human decision before the corresponding task can start:

- The Slack-vs-email notification question (see Open questions).
- Sign-off on the per-team G5 threshold values themselves — this plan
  implements the mechanism, not the thresholds.
- Legal/compliance review of the audit-trail retention period for the
  persisted `GateResult` history — not addressed anywhere in the source
  design and not assumed here.

## Task breakdown

- TASK for gates G0–G4 (deterministic checks)
- TASK for G5 risk-score integration
- TASK for G6 review queue + notification (blocked on the open question above)
- TASK for G7 publish + `docs/role-factory/README.md` generation
- TASK for the persisted gate-result history / audit trail
