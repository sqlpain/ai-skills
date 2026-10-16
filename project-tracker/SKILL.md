---
name: project-tracker
description: Create and update project tracking artifacts (specs, plans, tasks, decisions) in .tracker/. Use when asked to spec/plan work, when starting or finishing tracked tasks, or when a choice between alternatives needs recording.
---

# Tracker

This project may use Tracker: specs, plans, tasks and decisions as markdown files
under `<project-root>/.tracker/`. Files are the only source of truth; there is no
service or database. A single read-only viewer, `.tracker/index.html`, shows them.

## Setup (first use in a project)

If the user asks for tracking and `.tracker/` doesn't exist, create it directly:

1. Create `.tracker/` with `specs/`, `plans/`, `tasks/`, `decisions/`.
2. Write `.tracker/project.yaml` with `id` (kebab-case slug) and `name`.
3. Copy `@./assets/index.html` to `.tracker/index.html`.

If `.tracker/index.html` already exists, compare the `<!-- tracker-viewer vN -->`
marker on its first line with the one in `@./assets/index.html`; overwrite it
only when the skill's copy has the higher version. Tell the user to open the
file in Edge or Chrome and pick the `.tracker` folder.

## Format (authoritative reference: @./file-format.md)

- specs/SPEC-NNN-<slug>.md    — requirements; items are "### <n>. <title>" headings
- plans/PLAN-NNN-<slug>.md    — approach; frontmatter `implements: [SPEC-...]`
- tasks/TASK-NNNN-<slug>.md   — work units; `status`, `plan`, `covers: [SPEC-NNN.n]`, `decisions`; carries its own mini-plan and outcome
- decisions/D-NNN-<slug>.md   — one file per decision; never rewritten, superseded instead

`outcomes/OUT-NNNN-*.md` is a legacy artifact. Do not create new ones; the viewer
still shows existing ones read-only.

## ID allocation

List the target directory **and its `archive/` counterpart**, take the highest
existing ID across both, add 1, zero-pad (SPEC/PLAN/D: 3 digits; TASK: 4 digits).

## When to write what

- Asked to specify/design → SPEC (or extend one); enumerate requirements as
  numbered items so coverage can be tracked.
- Asked to plan approved spec work → PLAN, then TASK files for the breakdown,
  each with `covers:` pointing at the spec items it addresses. Replace the plan's
  Task breakdown placeholders with the real TASK IDs.
- Starting work on a task → set its `status: in-progress` (single Edit).
- Finishing a task → set `status: done`, tick its `## Acceptance` boxes, and fill
  its `## Outcome` (Done / Deviations / Verification / Follow-ups). Update
  `updated:`. Work that ends blocked or partial keeps its true status; record
  progress so far in `## Outcome`.
- Discovered-but-not-done work → list it under the task's Outcome `Follow-ups`;
  create a new TASK if it should be tracked.
- A choice between real alternatives is made (design, library, scope, trade-off)
  → write a DECISION from `@./templates/decision.md`, list it in the affected
  task's `decisions:` and the affected files' `relates:`/plan text. To change a
  decision, write a new one and mark the old one `superseded`; never edit its
  Decision section.

## Writing plans

A plan is judged against `@./examples/known-good-plan.md`, not against the
template's size. Before saving a plan, check:

- Every section of `@./templates/plan.md` is present with real content, or has
  an explicit `N/A - <reason>` line. None deleted silently.
- Approach contains the actual sequence/state machine and a worked example with
  real input (verbatim from the source where one exists), not a paragraph about it.
- Alternatives lists real options with trade-offs and says why the others lost
  (and links the `D-NNN` that records the choice).
- Effort has a justification column, a total, and a parallel figure if relevant.
- Verification names concrete tests including a failure path.
- Open questions and "What still needs a human" are filled, not left empty.
- Every deliverable named in the Objective has a TASK in Task breakdown.

Design-wide worked examples live in the PLAN `## Approach`; examples that belong
to one unit of work live in that TASK's `## Description`.

## Writing tasks

A task is judged against `@./examples/known-good-task.md`.

- **Title**: a verb phrase naming the deliverable ("Add rate limiting to POST
  /login"). Not "Implement login", not a ticket-number echo.
- **Summary**: one plain-English sentence for someone who hasn't seen the plan.
- **Deliverable**: one sentence naming what will exist when done.
- **Mini-plan**: 3-6 short numbered steps naming the files/areas touched. More
  than 6 steps means the task should be split.
- **Acceptance**: observable done-when checks only.
- Keep the wording plain. If a teammate couldn't tell from the Summary and
  Mini-plan what is being built, rewrite it.

## Migrating existing material

Trigger: the request references an existing document, spec, design, Confluence
page, prior plan, or any prose the user already wrote — not just requests to
author something new from conversation.

1. Re-read the source in full immediately before writing. Do not work from
   memory of it, even if you produced or read it earlier in this same session —
   summarization creep is the specific failure this section exists to prevent.
2. Build a section inventory: enumerate every heading in the source.
3. Map every inventory entry to a destination artifact and section (see
   `## Where content goes`) before writing anything.
4. Copy, don't summarise. Tracker artifacts are the **canonical home** for the
   content, not an index pointing at the original — even when the original also
   lives in Confluence, a wiki, or a prior plan file. If it mattered enough to
   write, it matters enough to carry over.
5. Nothing is dropped silently. Content judged out of scope is recorded as
   such, with a reason — never just omitted.
6. Verbatim-preserve code blocks, YAML/JSON examples, tables, numeric estimates
   and quoted rationale. Paraphrasing a worked example destroys its value;
   paraphrasing a table destroys its comparability.

## Where content goes

| Source content | Destination |
|---|---|
| Requirements, "what must exist" | SPEC numbered items |
| Rule/constraint tables | SPEC `## Appendix A` (keep as a table) |
| Architecture, file layouts | PLAN `## Architecture` |
| State machines, gate/stage sequences | PLAN `## Approach` **and** a SPEC item — the "how" and the "what" |
| Options appraisal, alternatives, pros/cons | PLAN `## Alternatives considered`, plus a DECISION recording the choice |
| Effort estimates **and their justification** | PLAN `## Effort estimate` — the justification is the load-bearing part |
| Executive/management summary | PLAN `## Management summary` |
| Worked examples (plan-wide) | Verbatim in PLAN `## Approach` |
| Worked examples (one unit of work) | Verbatim in the owning TASK `## Description` |
| Risks, assumptions, open questions | PLAN, as their own sections |
| Human-responsibility boundaries | PLAN `## What still needs a human`, or SPEC `## Appendix A` — never only an "out of scope" line |
| A choice that was made between alternatives | DECISION |

## Description carries the design

`## Description` carries the design, the reasoning and the worked examples.
`## Acceptance` carries only what must be verified. Never convert rationale
into a checkbox — a checkbox states *that* something must hold and cannot
express *why* or *how*, so the conversion loses the content. If a task's
design detail is substantial, `## Description` should be substantially longer
than `## Acceptance`.

## Completeness self-audit

Before reporting migration work finished, produce a mapping table: every
source section → the artifact and section that now holds it. Explicitly list
anything not carried over, with a reason. Separately confirm that every
deliverable, file or artifact named in the source has either a task or a
stated exclusion. `covers:` coverage of spec items is **not** evidence of
source fidelity — it can be complete while the spec itself has already lost
content.

## Common failure modes

- **Summarising instead of transferring.** The dominant failure of this skill.
  Tracker artifacts are where the content now lives; producing a tidy précis
  and leaving the substance in the source document is the failure, not the
  goal.
- **Treating templates as the target size.** The templates are a minimum
  structure. A plan for substantial design work will have more sections and
  far more content than the template shows. Uniform artifact length across
  tasks of wildly differing complexity is a symptom of this.
- **Compressing rationale into acceptance criteria.** See
  `## Description carries the design`.
- **Dropping a named deliverable.** If the source names a file or artifact, it
  needs a task or an explicit exclusion.
- **Vague tasks.** A generic title, no Deliverable, no Mini-plan, and two
  boilerplate checkboxes. See `## Writing tasks`.
- **Decisions buried in prose.** A choice made in a Description or a chat reply
  but never recorded as a DECISION.

See `@./examples/known-good-plan.md` vs `@./examples/known-bad-lossy-plan.md`
for a worked contrast, and `@./examples/known-good-task.md` and
`@./examples/known-good-decision.md` for the task and decision shapes.

## Verifying output

There is no separate verifier skill — the plan checklist, the task rules and the
completeness self-audit above are the check.

## Templates

@./templates/spec.md
@./templates/plan.md
@./templates/task.md
@./templates/decision.md
