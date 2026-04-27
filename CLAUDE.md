# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo actually is

This repo contains **no application source code**. There is no `src/`, no `tests/`, no `.csproj`, no `global.json`. Do not look for build, lint, or test commands — there are none to run. The artifacts here are pure markdown.

What this repo *is*: a context-engineering toolkit — the instructions, prompts, skills, and agents that condition GitHub Copilot, Claude Code, Codex, Cursor, and other AI agents to behave like a senior engineer on an opinionated **ASP.NET Core 9 / Clean Architecture** stack. When you edit files here, you are editing the rules that govern *another* codebase's AI behavior, not the codebase itself.

The full intent and provider mapping is in [README.md](README.md) and [AGENTS.md](AGENTS.md).

## Where each artifact type lives

| Artifact | Location | Loaded when | File convention |
|---|---|---|---|
| Always-loaded constitution | [.github/copilot-instructions.md](.github/copilot-instructions.md) | Every interaction | Single file. Keep load-bearing content under 4,000 chars (Copilot Code Review reads only the first 4,000). |
| Path-scoped rules | [.github/instructions/](.github/instructions/) | When matching files are in context | `*.instructions.md` with frontmatter `applyTo: '<glob>'` |
| Reusable prompts | [.github/prompts/](.github/prompts/) | On demand via `/<name>` | `*.prompt.md` with `description`, `agent`, `tools`, `argument-hint` |
| Skill packs | [.github/skills/](.github/skills/) | Auto-discovered by description | `<name>/SKILL.md` with `name` + `description` |
| Custom agents | [.github/agents/](.github/agents/) | Selected per chat | `*.agent.md` with `name`, `description`, `tools`, `model` |
| Cross-tool north star | [AGENTS.md](AGENTS.md) | Read by any agents.md-aware tool | Single file at repo root |

`AGENTS.md` is the **provider-neutral** layer; `.github/copilot-instructions.md` is the **provider-specific deep config**. They must not contradict.

## Precedence (when rules conflict)

1. Path-scoped instructions (`.github/instructions/*.instructions.md` matched via `applyTo`)
2. Repo-wide [.github/copilot-instructions.md](.github/copilot-instructions.md)
3. [AGENTS.md](AGENTS.md) and provider-specific agent files
4. Organization instructions

If two repo-scoped rules conflict, the **more specific** rule wins. If still ambiguous, raise it in the PR and codify the resolution in the relevant `.instructions.md` so the next contributor inherits the answer.

## The rules these files enforce on the target codebase

Editing the artifacts here without internalizing the rules they teach produces incoherent guidance. The hard architectural rules (full text in [.github/copilot-instructions.md](.github/copilot-instructions.md) §"Hard architectural rules" and [AGENTS.md](AGENTS.md) §3):

- **Dependency direction is inward only.** Domain → Application → Infrastructure → Web. Domain has zero framework dependencies.
- **CQRS via MediatR is mandatory.** Commands return `Result`/`Result<T>`, queries return `Result<TDto>`. Never share a handler.
- **Result pattern, not exceptions, for expected failures.** Validation/not-found/conflict/unauthorized → `Result.Failure`.
- **FluentValidation runs in a MediatR `ValidationBehavior`** before handlers. Handlers assume valid input.
- **Aggregates own invariants.** Public setters on entities are forbidden — use methods.
- **`CancellationToken`** through every async chain. No `.Result`/`.Wait()`/`.GetAwaiter().GetResult()`.
- **`ProblemDetails` (RFC 7807)** at the API boundary. No raw exception text leaks.
- **`TimeProvider`** injected; never `DateTime.UtcNow` directly. **Guid v7** for primary keys.
- **xUnit + FluentAssertions + NSubstitute + Testcontainers.** Coverage gate 80% line / 70% branch on Application + Domain.

When the user asks for guidance on the target stack (handlers, endpoints, EF mappings, etc.), apply these rules; when they ask you to *edit the rules*, keep all artifacts (instructions, prompts, skills, agents, AGENTS.md, copilot-instructions.md) consistent with each other.

## Conventions when adding or editing files here

- **Frontmatter is load-bearing.** Without correct `applyTo` / `description` / `name`, the file may never be loaded. Match the shape used by sibling files in the same directory.
- **Cross-references use markdown links** so they render in chat and IDE. Keep them relative.
- **Internal links must resolve.** Adding or renaming a file means updating every reference to it across `README.md`, `AGENTS.md`, `CLAUDE.md`, `.github/copilot-instructions.md`, and any sibling instructions/prompts/skills/agents that link to it.
- **The 4,000-char budget on [.github/copilot-instructions.md](.github/copilot-instructions.md)** is real (Copilot Code Review truncation). Push detail down into `.github/instructions/*.instructions.md` rather than expanding the constitution.
- **Don't duplicate.** A rule belongs in exactly one place; everywhere else cross-links to it. The anti-patterns file ([.github/instructions/anti-patterns.instructions.md](.github/instructions/anti-patterns.instructions.md)) is the canonical home for "don't do X".
- **Agent files use `*.agent.md`** (the `.agent.md` suffix is the current convention — see [.github/agents/csharp-dev.agent.md](.github/agents/csharp-dev.agent.md)).

## Useful slash-prompts (run these instead of reinventing)

The prompts in [.github/prompts/](.github/prompts/) are how target-codebase work gets done:
`/api-generation`, `/review-architecture`, `/bug-investigation`, `/review-code`, `/review-performance`, `/refactor`, `/review-security`, `/testing`.

Specialist personas live in [.github/agents/](.github/agents/) and are selected per chat session — currently only [`csharp-dev`](.github/agents/csharp-dev.agent.md).
