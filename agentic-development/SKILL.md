---
name: agentic-development
description: Plan, implement, verify, and review software changes using a specification-driven workflow. Use for features, bug fixes, refactoring, migrations, and other repository changes.
---

# Agentic Development

Follow this workflow for software changes.

## 1. Understand the request

Before editing:

1. Restate the desired outcome.
2. Identify relevant acceptance criteria.
3. Identify ambiguities that could materially affect behavior.
4. Ask questions only when the missing information would change the implementation.

Do not invent product requirements.

## 2. Inspect the repository

Find and read:

- applicable repository instructions;
- architecture documentation;
- relevant implementation files;
- similar existing features;
- tests covering the affected behavior;
- build, lint, type-check, and test commands.

Prefer existing project patterns over introducing new abstractions.

## 3. Produce a plan

For non-trivial changes, describe:

- files or components likely to change;
- intended implementation;
- tests to add or update;
- compatibility, security, or migration risks;
- assumptions requiring confirmation.

Do not implement until the plan is sufficiently concrete.

## 4. Decompose the work

Split large changes into bounded, independently verifiable tasks.

Prefer vertical slices that deliver observable behavior.

Use parallel agents only for independent work such as:

- codebase exploration;
- backend and frontend investigation;
- test-gap analysis;
- documentation research;
- independent review.

Avoid parallel agents editing the same files or shared working tree.

## 5. Implement

For each task:

1. Make the smallest coherent change.
2. Preserve existing behavior unless the specification changes it.
3. Follow established architecture and naming conventions.
4. Handle errors and important edge cases.
5. Avoid unrelated refactoring.
6. Add or update tests with the implementation.

Do not modify generated files manually.

## 6. Verify

Run the narrowest relevant checks first, followed by the broader project checks when practical:

1. Changed-feature tests
2. Unit and integration tests
3. Type checking
4. Linting and formatting
5. Build
6. End-to-end tests, when applicable

Never claim a check passed unless it was actually executed.

Report:

- exact commands executed;
- whether they passed;
- failures or skipped checks;
- remaining uncertainty.

## 7. Review the diff

Before finishing, inspect the complete diff for:

- incorrect behavior;
- missing authorization;
- security vulnerabilities;
- race conditions;
- non-idempotent operations;
- backward-compatibility problems;
- unsafe database migrations;
- missing tests;
- unnecessary complexity;
- accidental unrelated changes.

For significant changes, request an independent review agent when available.

## 8. Present the result

Provide:

1. Outcome
2. Important implementation decisions
3. Files or components changed
4. Verification performed
5. Remaining risks or follow-up work

Do not describe incomplete or unverified work as completed.
