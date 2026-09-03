---
name: pr-review
description: "Focused pull request review practices with severity-ranked, file-and-line-cited findings across four angles: security, performance, tests, and architecture. Use when asked to review a pull request, review a diff or set of changes, review \"this PR,\" or provide a code review before merge."
---

# PR Review

This skill covers focused, angle-specific pull request review: picking the right lens (security, performance, tests, or architecture), producing specific and severity-ranked findings, and closing with a clear merge verdict.

## Workflow for Reviewing a PR

1. **Pick the angle** — Determine emphasis from the user's request ("security", "perf", "tests", "arch"). If unspecified, ask which angle to use, or default to security as the highest-risk default.
2. **Get full file context** — A diff alone routinely misses bugs that live just outside the changed lines. If only a diff is available, request the surrounding file(s) before making confident claims.
3. **Review against the angle's checklist** — Work through the relevant checklist below in priority order; do not mix angles unless the user asked for a full review.
4. **Write specific, cited findings** — Every finding names a file and line number and states the concrete defect, not a vague impression.
5. **Rank by severity** — Group findings as blocker, important, or nit.
6. **State uncertainty explicitly** — If the diff doesn't give enough context to be sure, say so and ask for more code rather than guessing.
7. **End with a verdict on its own line** — `Safe to merge | needs changes | reject` (or the architecture-specific verdict below).

## Output Discipline

Applies to every review angle:

- Cite file path and line number for each finding.
- Rank findings by severity: blocker, important, nit.
- Be specific. "This looks risky" is not a finding; "`src/auth.ts:42` — JWT secret read from request body, see line 41" is a finding.
- If the diff doesn't give enough context to be sure, say so explicitly and ask for the surrounding file.
- End with a verdict on its own line: `Safe to merge | needs changes | reject`.

## Angle 1: Security

Review the PR for security defects, in order of priority:

1. **Auth/authz** — new endpoints or branches missing auth checks, role assumptions, IDOR (insecure direct object reference).
2. **Input validation** — untrusted input flowing into queries, shell commands, file paths, deserialization, or `eval`.
3. **Injection** — SQL, NoSQL, command, prompt injection, template injection.
4. **Secrets** — hardcoded keys/tokens, secrets in logs, secrets in client-bundled code, `.env` committed to the repo.
5. **Output encoding** — XSS via unescaped templating, raw HTML in user content, JSONP-style leaks.
6. **Crypto/randomness** — `Math.random()` used for tokens, MD5/SHA1 for password or integrity purposes, missing IVs, custom-rolled crypto.
7. **Data exposure** — PII in logs, overshared API responses, missing redaction.

Skip nice-to-haves; stick to defects.

## Angle 2: Performance

Review for performance regressions:

1. **N+1 patterns** — loops doing a DB or network call per item instead of batching.
2. **Hot-path allocations** — new objects/arrays/maps created inside loops, regexes recompiled per call.
3. **Unbounded work** — missing pagination, unconstrained result sets, recursion without a depth cap.
4. **Bad async** — sequential `await`s where `Promise.all` is correct, missing concurrency limits.
5. **Cache misuse** — cache keys that omit a relevant variable, absent or pathological TTLs.
6. **Algorithm complexity** — hidden O(n²) (e.g. `.some()` inside `.map()`), sorting inside a loop.

Quote the specific line, name the complexity class or bad pattern, and suggest the fix.

## Angle 3: Tests

Review the test coverage on the PR:

1. **Tests for new code paths** — every new branch should have at least one test.
2. **Edge cases** — empty input, null/undefined, boundary values, errors thrown by dependencies.
3. **Assertion strength** — assertions that would pass with the wrong value, snapshot-only tests, tests that only check the happy path.
4. **Mocking discipline** — mocks that don't fail when the real interface changes, over-mocking that hides real behavior.
5. **Determinism** — date/time/random/network not stubbed, leading to flaky tests.
6. **Test names** — names that don't describe the behavior under test.

A test that exists is not the same as a test that catches regressions — read the assertions, not the test name.

## Angle 4: Architecture

Pull back from line-level concerns to review the shape of the change:

1. **Boundary drift** — where did the seam between layers move? Did UI start reaching into the DB? Did domain types start importing transport types?
2. **Premature abstraction** — interfaces, factories, or config layers with only one implementation; flag these as debt.
3. **Coupling** — utilities now importing from feature modules, shared mutable state being introduced.
4. **Scalability** — if this code path goes 10x, what breaks first?
5. **Reversibility** — if this turns out wrong in a month, how hard is the rollback? Call out one-way doors explicitly.
6. **Naming** — types/functions named for the implementation (`UserManagerImplV2`) rather than the role (`UserDirectory`).

End with: `Architecturally sound | needs trim | re-think before merging`.

## Example Finding Format

```
src/api/orders.ts:118 (blocker, security)
  Order lookup uses `req.params.orderId` directly in a WHERE clause
  built via string concatenation — SQL injection. Use a parameterized
  query or the existing `db.orders.findById()` helper instead.

src/api/orders.ts:142 (nit, tests)
  `createOrder` has no test for the case where `items` is an empty
  array; current tests only cover the happy path with 1-3 items.

Verdict: needs changes
```

## Reviewing Multiple Angles at Once

- When the user asks for a "full review" rather than one specific angle, run all four checklists in sequence rather than blending them into one undifferentiated pass — each angle catches different classes of defect, and keeping them separate keeps findings traceable to a specific concern.
- Present each angle under its own heading with its own verdict line; do not merge four verdicts into one summary sentence.
- If time or context is limited, prioritize security first, then tests, then performance, then architecture — a security defect that ships is categorically worse than an architectural wart.

## Common Pitfalls in Review

- Approving based on the diff's intent ("looks like it adds validation") rather than reading what the code actually does.
- Treating a passing test suite as proof of correctness — a test suite only proves the tests that exist pass, not that the right tests exist.
- Letting scope creep into the review: flagging pre-existing issues untouched by the diff is fine to mention separately, but shouldn't block the PR under review.
- Giving a verdict without having seen enough of the surrounding file to justify it — ask for more context instead of guessing.
