# Copilot Engineering Operating System

A production-grade GitHub Copilot context-engineering repository for **ASP.NET Core Web API** development on **Clean Architecture**.

This repo is not a sample app. It is the **AI operating system** that surrounds the app: the instructions, prompts, skills, agents, anti-patterns, and automation that make Copilot behave like a senior engineer who already knows the codebase, the patterns, the conventions, and the tradeoffs.

> Inspired by GitHub's customization model. See: <https://docs.github.com/en/copilot/reference/customization-cheat-sheet>

---

## What's in the box

Everything Copilot consumes lives under `.github/`. There is intentionally no separate `.ai/` or `docs/` directory — GitHub-native locations are the single source of truth.

| Layer | Path | Loaded by Copilot | Purpose |
|---|---|---|---|
| Repo-wide instructions | [.github/copilot-instructions.md](.github/copilot-instructions.md) | Always | The constitution. Architecture, dependency rules, defaults. |
| Path-scoped instructions | [.github/instructions/](.github/instructions/) | When matching files are in context (`applyTo` glob) | Layer-specific and concern-specific rules. |
| Reusable prompts | [.github/prompts/](.github/prompts/) | On demand (`/<prompt-name>`) | Repeatable engineering tasks — endpoint generation, reviews, refactors, migrations, incident response. |
| Agent skills | [.github/skills/](.github/skills/) | Auto-discovered by description, or `/<skill-name>` | Procedural skill packs the agent loads when relevant. |
| Custom agents | [.github/agents/](.github/agents/) | Selected per chat | Specialized personas (architect, reviewer, refactorer). |
| Cross-tool agent file | [AGENTS.md](AGENTS.md) | Always (Copilot, Claude, Codex, etc.) | Provider-neutral north star for any AI agent. |

---

## Tech stack this repo is opinionated about

- **.NET 9 / C# 13** ASP.NET Core Web API (Minimal APIs preferred for new endpoints)
- **Clean Architecture** with strict dependency direction (Domain ← Application ← Infrastructure ← Web)
- **DDD tactical patterns** — aggregates, value objects, domain events
- **CQRS** with **MediatR** (commands return `Result<T>`, queries return `Result<TDto>`)
- **FluentValidation** for input validation; **Result pattern** for error flow
- **EF Core 9** with **PostgreSQL** primary, SQL Server compatible
- **Serilog** + **OpenTelemetry** (traces, metrics, logs)
- **Redis** for caching and distributed locks
- **xUnit** + **FluentAssertions** + **Testcontainers** for tests
- **Docker** + **GitHub Actions** for CI/CD
- **Hangfire** or **Quartz.NET** for background jobs

Each choice is justified in the relevant file under [.github/instructions/](.github/instructions/).

---

## How to use this repo

### As a new contributor
1. Read [.github/copilot-instructions.md](.github/copilot-instructions.md) — the constitution (always-loaded).
2. Skim [AGENTS.md](AGENTS.md) — the cross-tool north star.
3. When you write code, Copilot will automatically load the relevant `.instructions.md` based on the file you are in.

### When you start a non-trivial task
- Need a new endpoint? Run [/api-generation](.github/prompts/api-generation.prompt.md).
- Generating tests? Run [/testing](.github/prompts/testing.prompt.md).
- Refactoring? Run [/refactor](.github/prompts/refactor.prompt.md).
- Investigating a bug? Run [/bug-investigation](.github/prompts/bug-investigation.prompt.md).
- Reviewing a PR? Run a scoped review: [/review-code](.github/prompts/review-code.prompt.md), [/review-security](.github/prompts/review-security.prompt.md), [/review-performance](.github/prompts/review-performance.prompt.md), [/review-architecture](.github/prompts/review-architecture.prompt.md).

### When you want a specialist
Switch the chat to one of the custom agents in [.github/agents/](.github/agents/):
- [`csharp-developer`](.github/agents/csharp-developer.agent.md) — implements C# / .NET code across all four layers under the project conventions.

Additional personas (architect, reviewer, security, performance, etc.) are planned — for now, use the relevant prompt above (e.g. `/review-architecture`, `/review-security`, `/review-performance`) which encodes the same expectations.

---

## Precedence (when rules conflict)

GitHub Copilot resolves customization in this order (highest first):

1. Personal instructions (your settings)
2. Path-scoped instructions (`.github/instructions/*.instructions.md` matched via `applyTo`)
3. Repo-wide (`.github/copilot-instructions.md`)
4. `AGENTS.md` and provider-specific agent files
5. Organization instructions

If two repo-scoped rules conflict, the **more specific** rule wins. If still ambiguous, raise it in the PR — the resolution belongs in the relevant `.instructions.md` so the next contributor inherits the answer.

---

## Contributing to the OS itself

Changes to anything under `.github/` are reviewed against the rules in [.github/instructions/anti-patterns.instructions.md](.github/instructions/anti-patterns.instructions.md). New conventions belong in the relevant `.instructions.md` file with a brief rationale in the PR description.

---

## License

Internal engineering asset. Adapt freely for your team.
