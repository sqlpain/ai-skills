---
id: PLAN-014
title: Role Factory rollout
status: draft
created: 2026-08-20
updated: 2026-08-20
implements: [SPEC-009]
tags: [role-factory, migration]
---

<!--
This is the "known-bad" counterpart to ./known-good-plan.md — same source
design document, rendered the way the tracker skill produced it before the
migration guidance existed. It is what the original 4-section template
(Objective / Approach / Risks / Task breakdown) produces when the model
summarises instead of transferring. Annotations in [LOST: ...] mark what the
source document actually contained at each point. Nothing below is "wrong"
on its own terms — it reads as a clean, complete plan. That is exactly the
failure: it looks done while having discarded roughly half the source.
-->

## Objective

Build a pipeline that validates role definitions through several checks
before publishing them.

<!-- [LOST: the source named seven specific gates, G0–G7, each with a name
     and a one-line contract. Collapsed here to "several checks." A reader
     of this plan cannot tell how many gates exist or what any of them do. -->

## Approach

Each role draft goes through a series of validation gates. If a gate fails,
the draft doesn't proceed. Some gates may need a human to review before
continuing. Once all gates pass, the role is published to the registry.

<!-- [LOST: the full G0–G7 state machine table, the note that G0–G4 are
     deterministic and re-runnable while G6/G7 have side effects, the
     46-line worked YAML example showing exactly what a draft looks like and
     tracing it through every gate, and the architecture / file layout
     entirely. "A series of validation gates" is the one-sentence version of
     a state machine that took 90 lines to specify. -->

## Risks

- Some drafts may need manual review, which could slow things down.

<!-- [LOST: the specific fairness risk (per-team configurable G5 threshold
     producing inconsistent outcomes across teams) and the specific scaling
     risk (SQLite write bottleneck above ~50 drafts/day) were both named and
     reasoned about in the source. Both collapsed into one generic sentence
     that could describe almost any review process. -->

## Task breakdown

- TASK: implement validation pipeline
- TASK: implement review workflow
- TASK: implement publishing

<!-- [LOST entirely, not folded into any task:
     - the options appraisal (three approaches — linear engine, DAG-based
       gate graph, single big-bang validation — with pros/cons tables and
       the reasoning for choosing the linear engine)
     - effort estimates and their justification (12 days sequential, ~7
       parallel, broken down per component)
     - the management summary
     - the two open questions (Slack vs. email notification; what happens to
       a draft stuck in needs_human)
     - a task or exclusion for docs/role-factory/README.md — the source
       named this file as a required deliverable of the G7 gate and it has
       no task here at all. -->
