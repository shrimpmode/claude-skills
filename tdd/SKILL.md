---
name: tdd
description: Test-driven development (red-green-refactor). Use when building a feature or fixing a bug test-first, when the user says "TDD" or "red-green-refactor", or when another skill (like spec-driven-development) needs a test-first loop for one unit of work.
---

# Test-Driven Development

TDD is the red → green → refactor loop, run one small slice at a time. This
skill governs that loop directly, and is also what `spec-driven-development`
delegates to per task when a task's done-condition is a test.

## The loop

1. **Red.** Write one failing test for the smallest next piece of behavior.
   Run it and confirm it fails for the reason you expect — not because of a
   typo, a missing import, or broken test setup. A test that fails for the
   wrong reason, or that passes immediately, tells you nothing.
2. **Green.** Write the smallest amount of code that makes the test pass.
   Don't implement more than the test demands, even if you can see the next
   requirement coming — that's the next cycle's red.
3. **Refactor.** With the test green, clean up duplication or naming in what
   you just touched, re-running the test after each change. Skip this step
   if there's nothing worth cleaning up.

Repeat, one test at a time. Never write a batch of tests before any
implementation ("horizontal slicing") — it locks in assumptions about shape
before you've learned anything, and produces tests that check imagined
behavior instead of responding to what the previous cycle actually taught
you.

## What a good test looks like

Test behavior through the public interface, not internals. A test should
read like a spec — "returns 404 for an unknown id" — and survive a refactor
that doesn't change behavior. If a test breaks because you renamed a private
helper or restructured internals, it was testing the wrong thing.

Prefer the smallest seam that lets you observe real behavior: a function's
return value, an API response, rendered output — not a mock's call count or
a query against internal state.

## Anti-patterns to avoid

- **Tautological assertions** — the expected value is computed the same way
  the code computes it, so the test can't disagree with a wrong
  implementation (`expect(add(2, 3)).toBe(2 + 3)`). Expected values must
  come from an independent source: a hand-worked example, a fixture, the
  spec.
- **Testing implementation, not behavior** — mocking a collaborator that
  isn't a true boundary, asserting on private state, or querying a database
  directly instead of going through the interface under test.
- **Skipping red** — writing the implementation first and adding a test
  after "to be safe." That test confirms what you already believe the code
  does, not what it must do, and won't reliably catch a real regression.
- **Over-implementing in green** — adding validation, edge cases, or config
  that the current test doesn't require. Wait for the test that demands it.

## Applying this to one task

When you're handed a single task with a stated done-condition — a named test
file/case, e.g. from a `tasks.md` — run:

1. Write that test. Run it, confirm it fails for the expected reason.
2. Write the minimal code to make it pass.
3. Refactor if warranted, keeping the test green.
4. Report the passing test run as the completed check. Never mark the task
   done without having actually run it.

Bugs follow the same loop: write a test that reproduces the bug (red — it
should fail by demonstrating the bug is present), then fix the code until it
passes.
