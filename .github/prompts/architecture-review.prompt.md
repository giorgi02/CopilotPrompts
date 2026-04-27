---
description: 'Architecture review of a change, file tree, or proposal against Clean Architecture and the project rules in .github/instructions/.'
agent: 'agent'
tools: ['search/codebase', 'search/usages']
argument-hint: '<change, folder, or proposal to evaluate>'
---

# Architecture review

You are reviewing a change for **structural integrity**, not for line-level bugs (use `/code-review` for that).

## What you are checking

1. **Dependency direction** — does the change respect Domain ← Application ← Infrastructure ← Web?
2. **Layer responsibilities** — is logic in the right layer?
3. **Aggregate boundaries** — are aggregates the right size? Are invariants where they belong?
4. **Pattern consistency** — does the change follow CQRS, Result, FluentValidation conventions?
5. **Cross-cutting concerns** — are they in pipeline behaviors, middleware, or the wrong place?
6. **Coupling** — does the change introduce a new dependency that shouldn't exist?
7. **Rule alignment** — does it contradict an existing rule in `.github/instructions/`? Does it warrant a new rule there?

## Architecture smell catalog

For each finding, name the smell from [anti-patterns](../instructions/anti-patterns.instructions.md):

- **Domain bleed** — framework or persistence dep imported into Domain.
- **Layer skip** — Web calling Infrastructure directly (bypassing Application), or Web calling EF/persistence outside `Program.cs` composition.
- **Anemic domain** — entities reduced to data bags, business logic in services.
- **Service-locator** — `IServiceProvider` injected into business code.
- **God handler** — one handler doing many things; should be split or pushed to Domain.
- **Repository abuse** — repository for an entity that isn't an aggregate root, or returning `IQueryable`.
- **Cross-aggregate transaction** — operations spanning aggregates should communicate via events, not direct calls.
- **Reverse dependency** — Application referencing a concrete Infrastructure type.
- **Broken vertical slice** — feature spread across the codebase instead of one folder.
- **Pattern inconsistency** — same need solved two different ways in two places.

## Output

```
## Architecture review of <scope>

### Dependency direction
<OK | violations: list>

### Layer responsibilities
<OK | issues: list with smell names>

### Aggregate / domain boundaries
<observations>

### Pattern consistency
<observations>

### Rule alignment
- `.github/instructions/<file>`: <consistent | violated by <which file>>
- New pattern detected: <propose rule update in <file> or use existing>

### Recommendations
1. <action> — owner: <area>
2. ...

### Verdict
<Approve | Request changes | Block> — <one-line reason>
```

## Hard rules

- Cite the rule (file path + section) for every finding.
- If the change is fine, say so and list what you checked.
- Recommend a rule update in `.github/instructions/` only when the change introduces or changes a **pattern**, not just for new features.
