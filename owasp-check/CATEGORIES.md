# OWASP checklist

Reference for step 3 of [`owasp-check`](SKILL.md). Each category lists what to look for; apply every check that can touch the code in scope.

## OWASP Top 10:2025

### A01 Broken Access Control

- Every route and action enforces authentication and the right role, on the server: hiding a button or a page is not access control.
- Object-level checks: an ID from the client (path, query, body) is scoped to the caller (`WHERE owner_id = current_user`), so changing the ID cannot reach another user's or tenant's record. Same for update and delete.
- Field-level checks: the client cannot set fields it should not own (`role`, `is_active`, `owner_id`, `price`) through mass assignment or a permissive schema.
- Admin and internal endpoints are not reachable by ordinary users; default deny for new routes.
- CORS allows only known origins; credentials are not combined with a wildcard origin.
- Server-side request forgery: a user-supplied URL fetched by the server is checked against an allowlist, and internal addresses (`localhost`, `169.254.169.254`, private ranges) are refused.
- Path traversal: user input never builds a file path without normalizing and confining it.
- Tokens and links that grant access (invitations, password resets, file links) are unguessable, scoped, expiring, and single-use where the flow implies it.

### A02 Security Misconfiguration

- Debug mode, stack traces, and verbose error pages are off in production; error responses do not leak internals.
- Default credentials, sample accounts, and seed data are absent from production paths.
- Security headers where relevant: `Content-Security-Policy`, `Strict-Transport-Security`, `X-Content-Type-Options`, `Referrer-Policy`, frame protection.
- Cookies carrying sessions are `HttpOnly`, `Secure` outside local development, and `SameSite`.
- Containers run as non-root; only needed ports are exposed; admin consoles and databases are not public.
- Cloud and storage permissions are least privilege; buckets are private unless meant to be public.
- Unused features, routes, and dependencies are removed.

### A03 Software Supply Chain Failures

- Dependencies have no known vulnerabilities (dependency audit) and are pinned with a lockfile that CI installs from (`--frozen`, `npm ci`).
- New dependencies are maintained, widely used, and spelled correctly (typosquatting).
- CI pipelines pin third-party actions, do not expose secrets to untrusted pull requests, and grant tokens minimal permissions.
- Base images are pinned and current; build steps do not pipe remote scripts into a shell.
- Install-time scripts of dependencies are reviewed or disabled where the package manager allows.

### A04 Cryptographic Failures

- Passwords are hashed with a slow, salted algorithm (argon2id, scrypt, bcrypt); never stored in plain text or with a fast hash.
- Secrets (keys, API tokens, database passwords) come from the environment or a secret store, are absent from source control and logs, and are long and random enough.
- Sensitive data is encrypted in transit (TLS) and, where warranted, at rest; long-lived third-party tokens stored in the database are encrypted.
- Token signing uses a vetted library, a fixed algorithm (refuse `none` and algorithm confusion), and verifies expiry.
- Randomness for security uses a cryptographic generator (`secrets`, `crypto.randomBytes`), not `random` or `Math.random`.
- Comparisons of secrets, MACs, and tokens are constant-time.

### A05 Injection

- SQL: parameterized queries or an ORM's bound parameters everywhere; any raw SQL with string formatting or concatenation is a lead.
- Command execution: no shell built from user input; argument lists instead of shell strings.
- Cross-site scripting: output is escaped by the template or framework; look for bypasses (`dangerouslySetInnerHTML`, `|safe`, `v-html`, `innerHTML`), user input in `href`/`src` (`javascript:` URLs), and user content rendered in emails or PDFs.
- Template, LDAP, XPath, NoSQL, and header injection: user input never becomes part of a query language or a header without escaping (watch for CRLF in headers and redirect targets).
- Open redirects: redirect targets from user input are allowlisted or relative.

### A06 Insecure Design

- Business rules are enforced on the server, not just in the UI: limits, ownership, state transitions, one-time actions.
- Abuse cases are handled: brute force (login, codes, tokens), enumeration (different responses for existing and missing accounts or records), replay, and races in multi-step flows (double-spend, double-booking).
- Sensitive flows have rate limits and quotas.
- Concurrency-sensitive operations are atomic in the database (unique constraints, conditional updates, transactions), not check-then-act in application code.

### A07 Authentication Failures

- Login, password reset, and token endpoints resist brute force (rate limits, lockout, or backoff) and do not reveal whether an account exists.
- Password rules set a minimum length and allow long passphrases; breached-password checks where feasible.
- Sessions and tokens expire, are invalidated on logout, deactivation, and password change where the design requires it, and cannot be fixed by the attacker (rotate on login).
- Password reset tokens are random, single-use, short-lived, and bound to the account.
- Multi-factor authentication for privileged accounts where the product needs it.

### A08 Software or Data Integrity Failures

- No deserialization of untrusted data with unsafe formats (`pickle`, unsafe YAML loaders, Java/PHP native serialization).
- Webhooks and callbacks verify signatures; client-side state the server trusts (cookies, hidden fields, JWT claims) is signed and verified.
- Updates, plugins, and downloaded artifacts are verified (checksums, signatures).
- Database migrations and data changes keep integrity constraints in the database.

### A09 Security Logging and Alerting Failures

- Security-relevant events are logged: logins (success and failure), permission denials, account and role changes, admin actions.
- Logs never contain passwords, tokens, full session cookies, or unnecessary personal data.
- Log entries cannot be forged through user input (newlines, control characters).
- Someone is alerted on suspicious patterns (repeated failures, privilege changes) where the product warrants it.

### A10 Mishandling of Exceptional Conditions

- Errors fail closed: an exception in an authorization or validation path denies, never allows.
- Error responses are generic to clients and detailed only in server logs.
- Partial failures roll back (transactions), leaving no half-applied state.
- Resource exhaustion is bounded: request size limits, pagination, timeouts on outbound calls, upload size and type limits.
- Unexpected input (nulls, huge values, wrong types, empty collections) is validated at the boundary rather than crashing deeper in.

## OWASP API Security Top 10:2023

Apply to any HTTP API. Several overlap with the list above; check them from the API angle.

- **API1 Broken Object Level Authorization**: see A01 object-level checks, for every endpoint taking an ID.
- **API2 Broken Authentication**: see A07; also API keys and tokens in URLs, missing expiry, tokens accepted from multiple places without consistent checks.
- **API3 Broken Object Property Level Authorization**: responses expose only the fields the caller may see (no password hashes, internal flags, other users' contact data); requests accept only fields the caller may set.
- **API4 Unrestricted Resource Consumption**: pagination limits, request and upload size limits, rate limits, and caps on expensive operations (emails, SMS, exports).
- **API5 Broken Function Level Authorization**: see A01 role checks, especially admin functions reachable by changing the method or path.
- **API6 Unrestricted Access to Sensitive Business Flows**: flows that can be automated to harm the business (bookings, sign-ups, purchases) have limits and abuse detection.
- **API7 Server Side Request Forgery**: see A01 SSRF.
- **API8 Security Misconfiguration**: see A02; plus CORS, verbose errors, and unneeded HTTP methods.
- **API9 Improper Inventory Management**: old API versions and debug or test endpoints are retired; documentation endpoints are intentionally public or protected.
- **API10 Unsafe Consumption of APIs**: data from third-party APIs is validated like user input; outbound calls use TLS and timeouts; redirects from them are not followed blindly.
