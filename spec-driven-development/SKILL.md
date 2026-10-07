---
name: spec-driven-development
description: Spec-Driven Development (SDD) - write persisted spec.md, plan.md, and tasks.md artifacts under specs/<slug>/ with human sign-off between stages, then implement against tasks.md. Use when the user asks for SDD, spec-driven development, a spec-kit style workflow, or explicitly wants a feature tracked through spec/plan/tasks documents in the repo instead of an in-conversation plan.
---

# Spec-Driven Development

Four stages, each persisted as a file under `specs/<slug>/`, each gated on explicit
human sign-off before the next begins. The files are the durable record — write
them so someone opening the repo cold, with no memory of this conversation,
understands what was decided and why.

For a well-understood, single-file, low-ambiguity change, this ceremony is
overkill — use the lighter in-conversation `agentic-development` workflow instead.

## 0. Set up the spec directory

1. Derive a short kebab-case slug from the feature name.
2. List `specs/` and find the highest existing numeric prefix (`NNN-`); the new
   directory is `specs/<NNN+1>-<slug>/`, zero-padded to 3 digits. First spec is `001-`.
3. If `specs/<NNN>-<slug>/` already has partial artifacts (an in-progress SDD),
   resume from the latest incomplete stage — read what exists rather than
   restarting it.

## 1. Specify → spec.md

Write the problem in terms of WHAT and WHY, never HOW: no tech stack, no file
names, no implementation detail — that belongs in the plan.

Sections: Overview, User stories, Functional requirements (numbered, each one
independently testable), Non-goals, Open questions.

Ask the user only about ambiguities that would change scope; mark anything
else assumed as an explicit assumption in the doc rather than blocking on it.

**Done when:** every functional requirement is numbered and testable, and no
open question left in the doc would change scope if answered differently.

**Checkpoint:** show the user spec.md. Get explicit approval before writing
plan.md — a lack of objection is not approval; ask directly.

## 2. Plan → plan.md

Read spec.md plus the repo's actual conventions (architecture docs, existing
similar features, stack, test setup) before writing this.

Sections: Technical approach, Affected components/files, Data/API changes,
Testing strategy, Risks and tradeoffs, and a requirement-coverage table mapping
each spec.md requirement to the plan section that satisfies it.

Testing strategy names the test framework(s) already in the repo (don't
introduce a new one without calling it out as a decision), the unit/
integration/e2e boundary for this change, and confirms tasks.md will be
written test-first (see §4) — note any task where that isn't practical (e.g.
a pure config or doc change) and why.

**Done when:** every requirement in spec.md's coverage table maps to a
concrete part of the plan — no requirement left unaddressed.

**Checkpoint:** show the user plan.md. Get explicit approval before writing
tasks.md.

## 3. Tasks → tasks.md

Decompose plan.md into small, ordered, independently verifiable tasks as a
checklist. Each task: id, one-sentence description, files it touches,
dependencies on other task ids, the concrete check that proves it done, and a
checkbox.

Default the done-condition to a test written before the implementation, per
the testing strategy in plan.md: name the specific test file/case the task
will add or extend, not just "tests pass." Only fall back to a build or
observable-behavior check for tasks that genuinely have no meaningful test
(config, docs, pure refactors covered by existing tests) — say so explicitly
rather than leaving it implicit.

Prefer vertical slices over horizontal layers. Mark tasks with no dependency
on each other as parallelizable.

**Done when:** every plan.md section maps to at least one task, and every
task's done-condition is something you can actually run or observe, not a
vague description.

**Checkpoint:** show the user tasks.md. Get explicit approval before
implementing.

## 4. Implement

Work tasks in dependency order, one at a time:

1. Mark the task in-progress.
2. If the task's done-condition is a test, invoke the `tdd` skill and run its
   red-green-refactor loop for that one task, following the repo's existing
   patterns. Otherwise (config, docs, etc.), make the change directly.
3. Run the task's stated check. Never check a box without having actually run it.
4. Check the box in tasks.md.

If reality diverges from plan.md mid-implementation — a needed file the plan
didn't foresee, a different approach — update plan.md to match and note the
deviation there. Do not let the artifact silently drift out of sync with the
code.

## 5. Close out

Once every task is checked: summarize what changed against spec.md's
requirements, list anything deferred or descoped, report the verification
actually performed (tests/build/lint run and their results), and leave
spec.md, plan.md, and tasks.md in the repo as the record of the change.
