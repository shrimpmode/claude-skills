---
name: owasp-check
description: OWASP security check of code. Walks a diff or a whole codebase through the OWASP Top 10 (2025) and, for HTTP APIs, the API Security Top 10 (2023), confirms each vulnerability with file and line evidence and an attack path, and reports severity and fixes. Use when asked for an OWASP, security, or vulnerability check, audit, or review, or when a workflow skill reaches its security review step.
---

# OWASP check

Audit code against the OWASP Top 10:2025. The result is a **coverage table** in which every category ends as **Finding**, **Clean**, or **N/A**, plus one confirmed write-up per finding. The check is read-only; fixes come after the report, when asked.

## 1. Fix the scope

- Default scope is the change under review: the diff against the merge-base with the default branch, plus uncommitted changes. Audit the whole codebase only when the user asks for an audit.
- Map the **attack surface** the scope touches: every entry point (HTTP route, form or server action, webhook, file upload, CLI command, background job), who can call it (anonymous, signed-in user, admin, another service), the data stores and external services it reaches, and where its secrets and configuration come from.

Done when every entry point in scope is listed with its caller and its authentication and authorization requirement.

## 2. Run the scanners the project already has

Use the scanners already present in the repository or on `PATH`: dependency audit (`pip-audit`, `uv`/`npm`/`pnpm audit`, `osv-scanner`), static analysis (`bandit`, `semgrep`, framework linters with security rules), and secret scanning (`gitleaks`, `trufflehog`). Record each command and its result. A scanner that is missing becomes a recommendation in the report; ask before installing anything.

Scanner output is a **lead**, never a finding: every hit goes through step 4.

## 3. Walk every category

Read [`CATEGORIES.md`](CATEGORIES.md) and apply each category's checks to the code in scope; for an HTTP API, also apply its API Security Top 10 section. Follow data from each entry point to its sink rather than grepping for keywords alone.

Done when **every** category has a status:

- **Finding**: at least one lead to confirm in step 4.
- **Clean**: name what you checked (files, functions, middleware, config) that rules the category out.
- **N/A**: say why the category cannot apply to this scope (for example, no deserialization of untrusted data exists).

## 4. Confirm each finding

A finding is confirmed only with all three:

1. **Location**: `file:line` of the vulnerable code.
2. **Attack path**: who the attacker is, which input they control, and how that input reaches the impact.
3. **Why defenses miss it**: read the real code path one layer up (middleware, dependencies, decorators, framework defaults, database constraints). Many apparent holes are closed there.

A lead that cannot be confirmed stays in the report as **Plausible**, with what would confirm or rule it out.

Severity:

- **Critical**: an unauthenticated attacker reads or changes other users' data, runs code, or takes over accounts.
- **High**: an authenticated attacker crosses a privilege or tenant boundary, or sensitive data is exposed.
- **Medium**: exploitable only with preconditions (user interaction, a specific configuration), or a significant hardening gap.
- **Low**: a defense-in-depth gap with no direct exploit path.

## 5. Report

1. **Coverage table**: category, status, evidence (one line each).
2. **Findings**, most severe first: severity, category, location, attack path, and a concrete code-level fix.
3. **Plausible** leads and what would settle them.
4. **Scanners**: commands run and results; scanners recommended but absent.
5. **Not covered**: anything outside the scope, or that needs a running system (penetration test, infrastructure, cloud configuration).

When fixes are requested: fix Critical and High first, add a regression test that fails before the fix where practical, and re-run step 4 for that finding to show it is closed.
