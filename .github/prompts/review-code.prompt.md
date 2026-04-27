---
description: 'Perform a senior-engineer code review on a file, diff, or PR using the project guardrails.'
agent: 'agent'
tools: ['search/codebase', 'search/usages']
argument-hint: '<file path | PR url | diff>'
---

# Code review

Perform a senior-engineer review against this repo's standards. Be **specific, actionable, and prioritized**. Do not produce a feel-good summary.

## Scope and inputs

Identify what you are reviewing:
- A single file → review the file.
- A diff or commit range → review the change.
- A PR URL → fetch with `gh pr diff` and review the change.

## Order of checks (most important first)

1. **Architecture rules** — [anti-patterns](../instructions/anti-patterns.instructions.md).
   - Did Domain pick up a framework dep?
   - Did Application reference Infrastructure?
   - Did Web bypass Application and call EF directly?
   - Did a handler call another handler via `IMediator`?
2. **Correctness** — does the code do what its name says? Edge cases? Off-by-one? Cancellation? Concurrency?
3. **Security** — [secure-coding skill](../skills/secure-coding/SKILL.md). Injection, authn/z, secrets, log hygiene.
4. **Data and persistence** — N+1, missing `AsNoTracking`, missing index, mistaken `Include`, raw SQL with interpolation, migration safety.
5. **Performance** — sync-over-async, allocations on hot paths, missing `LoggerMessage`, sequential awaits that could be parallel.
6. **API contract** — status codes, `ProblemDetails`, OpenAPI metadata, idempotency, versioning.
7. **Tests** — does each new behavior have a test? Cancellation? Failure modes?
8. **Style** — file-scoped namespaces, primary constructors, modern pattern matching, no XML doc on internal members.

## Severity levels

- **Blocker** — must fix before merge. Architecture violation, security bug, data loss risk.
- **Major** — should fix before merge. Missing tests on new behavior, incorrect error mapping, perf regression.
- **Minor** — nice to fix. Naming, style, comment quality.
- **Nit** — purely subjective; mention but do not block.

## Output format

```
## Review of <scope>

### Blockers (n)
- <file>:<line> — <one-line problem>. <Concrete fix>.

### Major (n)
- <file>:<line> — <problem>. <Fix>.

### Minor (n)
- <file>:<line> — <comment>.

### Nit (n)
- <file>:<line> — <comment>.

### Out of scope but noted
- <observation about something the change touches but isn't responsible for>

### Verdict
<Approve | Request changes | Block> — <one-sentence reason>
```

## Hard rules

- Quote file paths with line numbers (`src/Application/Features/Orders/PlaceOrder/PlaceOrderHandler.cs:42`).
- Cite the specific rule violated by reference (e.g. "violates [.github/skills/cqrs-implementation/SKILL.md](../skills/cqrs-implementation/SKILL.md) — handlers must not call SaveChanges").
- No "looks good" without evidence. If you have nothing to flag, say so explicitly and list what you checked.
- No restating what the code does. Only flag what is wrong, missing, or risky.
