---
id: D-007
title: Gate requests may not use wildcard permissions
status: accepted
date: 2026-08-21
superseded_by: null
relates: [PLAN-014, TASK-0031, SPEC-009.3]
tags: [role-factory, permissions]
---

<!--
Companion to ./known-good-plan.md and ./known-good-task.md. It shows the
decision template filled in well: the situation that forced the choice, the real
alternatives with their trade-offs, the choice with the reason the others lost,
and the consequences that other work now has to live with. A one-line "we chose
X" with no alternatives or reasons is not a decision record.
-->

## Context

Gate G3 checks a draft's requested permissions against the team's permission
ceiling. Ceilings use wildcards (`payments.*:read`). While designing G3 we had
to decide whether a *request* may also use a wildcard. Roles are reviewed by
people who read the `permissions` list, and G5 scores risk from the specific
resources requested.

## Options considered

| Option | Pros | Cons |
|---|---|---|
| A: Requests must name exact resources (chosen) | Reviewers see precisely what is granted; G5 risk score is computed on concrete resources; G4 conflict scan is a simple set comparison | Drafts for broad roles are longer; needs a small expansion helper for authors |
| B: Requests may use wildcards | Short drafts | A wildcard hides what is really granted; G5 would need to expand it to score it; G4 overlap logic becomes pattern-vs-pattern |
| C: Wildcards allowed only for read access | Shorter read-only drafts | Two rules to explain and test; "read" wildcards still hide new resources added later |

## Decision

Option A. A request naming a wildcard resource fails G3 with a reason telling
the author to list the resources explicitly. Reviewers and the risk score both
depend on seeing exact resources, and the cost is only longer drafts.

## Consequences

- G3 and G4 stay simple string/set comparisons, which keeps them deterministic and cheap to re-run.
- The drafting docs (`docs/role-factory/README.md`) must say wildcards are ceiling-only.
- Authors of broad roles need an expansion helper; raised as a follow-up on TASK-0031.
- If a team later needs wildcard requests, supersede this decision rather than editing it.
