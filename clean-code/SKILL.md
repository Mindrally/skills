---
name: clean-code
description: "Clean, maintainable, human-readable code principles combined with anti-over-engineering discipline: naming, single responsibility, DRY, and scoping changes to exactly what was requested. Use when writing new code, refactoring existing code, reviewing code for quality, or deciding how much abstraction a change actually needs."
---

# Clean Code

This skill covers writing code that is easy to read and change, and — just as important — avoiding the over-engineering that makes code harder to read and change in the name of "best practices." Both halves matter together: clean code is simple code, not merely well-decorated code.

## Workflow for Writing or Reviewing Code

1. **Scope the change** — Identify exactly what was asked for. Note what's out of scope before writing anything.
2. **Reach for the simplest solution first** — Prefer the direct, obvious implementation over a general or configurable one, unless a concrete current need justifies more.
3. **Name things for their purpose** — Choose names that reveal intent before writing the body of a function or the shape of a type.
4. **Keep functions single-purpose** — If a function needs a comment to explain what it does, split it.
5. **Remove duplication deliberately** — Extract shared logic only once it's actually duplicated (see Rule of Three below), not preemptively.
6. **Write or update tests** — Cover the new behavior and the edge cases it introduces.
7. **Verify scope before delivery** — Confirm only the requested code changed, check for a simpler approach you might have missed, and confirm no unrequested files were touched.

## Meaningful Names

- Variables, functions, and classes should reveal their purpose from the name alone.
- Names should explain why something exists and how it's used, not just its type or contents (`activeUserIds`, not `list1`).
- Avoid abbreviations unless they're universally understood in the domain (`id`, `url` — fine; `usrCfgTmp` — not fine).

## Constants Over Magic Numbers

- Replace hard-coded values with named constants (`MAX_RETRY_COUNT = 3`, not a bare `3` three call sites later).
- Use descriptive constant names that explain the value's purpose, not just its value.
- Keep constants at the top of the file or in a dedicated constants module when shared across files.

## Smart Comments

- Don't comment on what the code does — make the code self-documenting through naming and structure instead.
- Use comments to explain *why* something is done a certain way, especially when the reason isn't visible in the code (a workaround for a library bug, a non-obvious ordering requirement).
- Document public APIs, genuinely complex algorithms, and non-obvious side effects.

## Single Responsibility

- Each function should do exactly one thing.
- Functions should be small and focused enough to be understood without scrolling.
- If a function needs a comment to explain what it does, that's a signal to split it into named sub-functions instead.

## DRY — Don't Repeat Yourself

- Extract repeated code into reusable functions once the repetition is real, not anticipated.
- Share common logic through a proper abstraction — a shared function or module, not copy-paste with tweaks.
- Maintain a single source of truth for any given piece of business logic or configuration value.

## Encapsulation

- Hide implementation details behind a clear interface; callers shouldn't need to know how a thing works to use it.
- Move nested conditionals into well-named functions or guard clauses instead of deep `if`/`else` trees.

```js
// Before
function canCheckout(cart) {
  if (cart.items.length > 0) {
    if (cart.user.isVerified) {
      if (cart.total <= cart.user.creditLimit) {
        return true;
      }
    }
  }
  return false;
}

// After
function canCheckout(cart) {
  const hasItems = cart.items.length > 0;
  const isWithinCreditLimit = cart.total <= cart.user.creditLimit;
  return hasItems && cart.user.isVerified && isWithinCreditLimit;
}
```

## Clean Structure

- Keep related code together (a feature's components, hooks, and styles in one directory, not scattered by file type).
- Organize code in a logical hierarchy that mirrors how the domain is understood.
- Use consistent file and folder naming conventions across the codebase.

## Avoiding Over-Engineering

- Only change what was asked. The simplest solution that satisfies the request comes first.
- When the right level of abstraction is unclear, ask rather than guessing toward the more elaborate option.
- Do not modify unrequested code, even if it looks improvable — a drive-by refactor in an unrelated function expands the review surface and the risk of the change.
- Do not add abstractions (interfaces, factories, plugin systems, config layers) without a concrete, current need. A single implementation doesn't need an interface "in case" a second one shows up later — that's speculative generality (YAGNI: "You Aren't Gonna Need It").
- Do not import a new dependency to solve a problem a few lines of existing code already solve.
- Do not rewrite entire files for small changes — a targeted diff is easier to review and safer to ship than a full-file rewrite.
- Do not add error handling for scenarios that cannot occur given the surrounding code's guarantees — defensive code for impossible states adds reading cost without adding safety.

### Rule of Three

- Tolerate duplication the first two times a pattern appears.
- Extract an abstraction on the third occurrence, once the actual shape of the shared logic is clear — extracting after one or two instances often guesses wrong about what's actually shared.

### Signs of Over-Engineering

- A configuration option that has only ever been set to one value.
- An interface with exactly one implementation and no test double that needs a second.
- A generic `options` object accreting fields for hypothetical future callers.
- A plugin/strategy pattern introduced before there are two strategies to switch between.

## Code Quality Maintenance

- Refactor continuously in small steps rather than deferring cleanup to a dedicated "refactor sprint."
- Fix technical debt early, while the context for why the code looks the way it does is still fresh.
- Leave code cleaner than you found it, scoped to the area you're already touching — not as license to refactor unrelated files.

## Testing

- Write a failing test before fixing a bug, so the fix is verified and the bug can't silently regress.
- Keep tests readable and maintainable — a test that's harder to understand than the code it tests has failed at its job.
- Test edge cases and error conditions explicitly, not just the happy path.

## Version Control

- Write clear, specific commit messages that explain why a change was made.
- Make small, focused commits — one logical change per commit.
- Use meaningful branch names that describe the work, not the author or the date.

## Before Delivery Checklist

- Only the requested code changed — no unrelated files touched.
- No abstraction was added without a concrete need that exists today.
- No dependency was added that duplicates something already available.
- A simpler approach was considered and ruled out, not just skipped.
- New behavior has test coverage, including at least one edge case.
