---
description: 'Surgical, behavior-preserving refactor of a file or method. Defines small steps and verifies after each.'
agent: 'agent'
tools: ['search/codebase', 'search/usages', 'edit/files']
argument-hint: '<file or method to refactor> — desired outcome'
---

# Refactoring

Refactoring is **changing structure without changing behavior**. If the test suite output changes, you are not refactoring — you are changing behavior.

## Before touching code

1. **Confirm tests exist** for the unit you are about to refactor. If they do not, write them first (characterization tests).
2. **Confirm the goal** in one sentence: "extract X to make Y reusable", "collapse two near-duplicates", "remove a leaky abstraction".
3. **Confirm the smell** you are addressing — match it to one in [anti-patterns](../instructions/anti-patterns.instructions.md). If it does not match a known smell, ask whether the refactor is worth it.

## Allowed refactor moves

- Extract method, extract type, inline method, inline variable.
- Rename (with full project rename, no aliases left behind).
- Replace conditional with polymorphism (rare in this codebase — prefer pattern matching).
- Move type to a different file/folder/namespace.
- Replace primitive with value object.
- Replace exception with `Result.Failure`.
- Replace nested `if` with guard clause + early return.
- Replace `IEnumerable` chain with explicit `for` on hot paths.

## Disallowed under "refactor"

- Adding a feature.
- Changing the public API surface.
- Changing behavior under any input.
- Touching code you did not need to touch (no opportunistic cleanup outside scope).

## Process — one move at a time

For each refactor move:

1. **Run tests** — note the current pass/fail state.
2. **Make the move** — single change.
3. **Run tests** — they must still pass with the same set.
4. **Commit** with a descriptive message: `refactor: extract OrderTotalCalculator from PlaceOrderHandler`.
5. Move on.

## Cross-cutting refactors (rename, move, etc.)

- Use IDE-grade rename. Do not regex.
- Update **every** reference, including XML docs, OpenAPI tags, log messages.
- Do not leave a `// renamed from` comment.
- Delete dead code. No commented-out blocks.

## When to stop

Stop and raise it for team agreement (and update the relevant `.github/instructions/*.instructions.md`) if you find yourself:
- Introducing a new abstraction with one consumer.
- Restructuring layer boundaries.
- Replacing a library with another.
- Changing how errors are represented.

## Output

```
Refactor: <one-line goal>
Smell addressed: <ref to anti-patterns.md>
Moves applied:
  1. <move> in <file>
  2. <move> in <file>
Behavior unchanged: <yes — tests pass before and after>
Rule update triggered: <yes — file | no>
```
