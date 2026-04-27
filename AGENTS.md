# AGENTS.md — Cross-tool agent instructions

> This file is consumed by GitHub Copilot, Claude Code, OpenAI Codex, Cursor, Aider, and other AI coding agents that respect the [agents.md convention](https://agents.md). It is the **provider-neutral north star**. The provider-specific deep config lives in [.github/copilot-instructions.md](.github/copilot-instructions.md) and the layered files under [.github/](.github/).

You are working in a **production ASP.NET Core 9 Web API** built on **Clean Architecture**. Treat this as a real engineering codebase, not a sandbox.

---

## 1. Mental model of the system

```
+-------------------------------------------------------+
|  Web (ASP.NET Core, Minimal APIs, ProblemDetails)     |  ← entry point
+----------------------↓--------------------------------+
|  Application (use cases, CQRS via MediatR, Result<T>) |
+----------------------↓--------------------------------+
|  Domain (aggregates, value objects, domain events)    |  ← pure C#, zero deps
+----------------------↑--------------------------------+
|  Infrastructure (EF Core, Redis, Hangfire, etc.)      |  ← implements Application abstractions
+-------------------------------------------------------+
```

**Dependency direction is inward.** Domain depends on nothing. Application depends on Domain. Infrastructure depends on Application + Domain. Web composes everything.

---

## 2. Before writing code, do this

1. **Locate the right layer.** A new use case? Application. A new entity rule? Domain. A new external integration? Infrastructure. A new HTTP surface? Web.
2. **Find a sibling** — the closest existing pattern in the same layer. Match its structure, naming, error handling, and tests. Consistency beats novelty.
3. **Confirm the contract.** What does this return? What can fail? What does the caller do on each failure mode?
4. **Decide if it needs codifying.** If the change introduces a new pattern, dependency, library, or cross-cutting policy → update the relevant `.github/instructions/*.instructions.md` and call out the rationale in the PR description.

---

## 3. The ten rules you must not break

1. Domain has **no framework dependencies**. If you find yourself adding `using Microsoft.*` to a Domain file, stop.
2. **CQRS via MediatR** for every use case. Commands mutate; queries read. They never share a handler.
3. **Result pattern** for expected failures. Exceptions are for programmer errors and truly exceptional conditions only.
4. **FluentValidation** runs in a MediatR `ValidationBehavior` pipeline before handlers. Handlers assume valid input.
5. **Aggregates enforce invariants.** Public setters on entities are forbidden. Mutations go through methods.
6. **`CancellationToken`** flows through every async call chain. Never `.Result`, `.Wait()`, `.GetAwaiter().GetResult()`.
7. **`ProblemDetails`** at the API boundary. No raw exception text leaks.
8. **`TimeProvider`** is injected. Never `DateTime.UtcNow` directly.
9. **Guid v7** for primary keys. Never `int` identity, never sequential `Guid.NewGuid()` for new code.
10. **Tests are part of the change.** PRs without tests for new behavior are incomplete.

---

## 4. When you are tempted to do something clever

Ask yourself, in this order:
1. Is there an existing pattern in this codebase that solves this? → Use it.
2. Is there a reusable prompt in `.github/prompts/` (e.g. `/api-generation`, `/bug-investigation`, `/refactor`)? → Run it.
3. Is there a rule in `.github/instructions/anti-patterns.instructions.md` I might be violating? → Check.
4. Am I about to introduce a new abstraction for one caller? → Don't. Three call sites unlocks an abstraction.
5. Am I about to add a NuGet package? → It needs a justification in the PR description and, if it introduces a pattern, a rule in the relevant `.github/instructions/*.instructions.md`.

---

## 5. Output discipline

- Match the existing file structure, namespace style, and naming **exactly**.
- Do not introduce `// TODO` comments unless paired with a tracked issue.
- Do not add documentation comments that restate the method signature. Add them only when the *why* is non-obvious.
- Do not introduce backward-compatibility shims for hypothetical callers — there are no hypothetical callers.
- When you delete code, delete it. Do not leave commented-out blocks.
- When you rename, rename everywhere — no aliases, no re-exports.

---

## 6. Where to look

| Need | Go to |
|---|---|
| Repo-wide constitution | [.github/copilot-instructions.md](.github/copilot-instructions.md) |
| Layered rules (auto-applied by file path) | [.github/instructions/](.github/instructions/) |
| Reusable prompts (`/api-generation`, `/bug-investigation`, `/refactor`, …) | [.github/prompts/](.github/prompts/) |
| Specialist personas (architect, reviewer, refactorer, …) | [.github/agents/](.github/agents/) |
| Auto-discoverable skill packs | [.github/skills/](.github/skills/) |
| Anti-patterns & guardrails (named smells, build/PR gates) | [.github/instructions/anti-patterns.instructions.md](.github/instructions/anti-patterns.instructions.md) |
| Standards (naming, style, API conventions) | layer-specific files in [.github/instructions/](.github/instructions/) |

---

## 7. Definition of done

A change is done when:
- All ten hard rules above hold.
- The relevant `.instructions.md` files were respected (Copilot loads them automatically based on file paths).
- Tests are added/updated; the coverage gate still passes.
- The PR description references which prompt was followed.

If you are about to ship something that violates a rule, stop and either fix the change or update the rule (with the rationale in the PR description) — do not silently break it.
