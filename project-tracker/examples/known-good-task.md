---
id: TASK-0031
title: Build the deterministic gates G0-G4
status: done
created: 2026-08-21
updated: 2026-08-26
plan: PLAN-014
covers: [SPEC-009.1, SPEC-009.2, SPEC-009.3]
blocked_by: []
decisions: [D-007]
priority: high
tags: [role-factory]
---

<!--
Companion to ./known-good-plan.md (this is the "TASK for gates G0-G4" line in
its Task breakdown). It shows the task template filled in well: a summary a
newcomer can read, a one-sentence deliverable, a mini-plan of checkable steps,
a Description that carries the reasoning, acceptance that only verifies, and a
completed Outcome. Compare with a vague task such as "Implement gates" with two
generic checkboxes - the reader cannot tell what "done" looks like.
-->

## Summary

Add the five checks that decide, without any human or side effects, whether a
role draft is well-formed, valid, uniquely named, within the team's permission
limit and free of conflicts.

## Deliverable

Five gate modules (`g0_intake.ts` to `g4_conflict_scan.ts`) under
`role-factory/pipeline/gates/`, each a pure function with table-driven tests.

## Mini-plan

1. Define the shared `GateResult` type (`pass`, `fail(reason)`, `needs_human(reason)`) and the `GateContext` shape in `pipeline/gates/types.ts`.
2. Write G0 (intake) and G1 (schema) against the role schema, using the worked example in PLAN-014 as the first test case.
3. Write G2 (naming) and G3 (permissions ceiling) using the existing lookup helper in `registry/lookup.ts`.
4. Write G4 (conflict scan) comparing requested permissions with active roles in the same team.
5. Add a negative-variant test for every gate (one input that must fail, and why).
6. Run the full gate test suite and record the result in Outcome.

## Description

G0-G4 are the gates that PLAN-014 calls deterministic and side-effect-free, so
they are safe to re-run any number of times when a suspended draft re-enters
the pipeline. That property is the reason they must stay pure functions of
`(RoleDraft, GateContext)`: no registry writes, no clock reads, no network.

G3 compares each requested permission against `gate_context.team_permission_ceiling`
using the `resource:access` form (`payments.*:read` allows
`payments.transactions` read). A wildcard on the ceiling side matches any
resource under that prefix; a wildcard on the *request* side is a `fail`
(see D-007 for why requests may not be wildcards).

Worked example (from PLAN-014): the `payments-read-only` draft passes G0-G4
because both requested permissions sit within the `payments.*:read` ceiling and
no existing team role grants them.

## Acceptance

- [x] The `payments-read-only` worked example passes G0, G1, G2, G3 and G4
- [x] A draft missing `role.team` fails G0 with a reason naming the missing field
- [x] A request for `payments.*:write` against a `read`-only ceiling fails G3
- [x] A draft whose permissions duplicate an active role fails G4 and names that role
- [x] Gate unit tests run in under 2 seconds with no I/O

## Notes

G2 and G3 both need the team lookup; shared it through one helper rather than
two copies.

## Outcome

**Done:** All five gates implemented as pure functions with 23 table-driven
tests. The shared `GateResult` / `GateContext` types are in `gates/types.ts`.

**Deviations:** G4 compares against the team's active roles only, not the
whole registry as first written in step 4 - cross-team overlap is not a
conflict under D-007. No other changes.

**Verification:** `npm test -- gates` passes (23 tests, 0.9 s). Manually
confirmed the worked example output matches the gate-by-gate walk-through in
PLAN-014.

**Follow-ups:**
- Decide whether G4 should warn (not fail) on partial permission overlap.
