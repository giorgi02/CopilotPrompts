---
description: 'Investigate a bug systematically — reproduce, locate root cause, propose minimal fix, write the regression test.'
agent: 'agent'
tools: ['search/codebase', 'search/usages', 'edit/files', 'web/fetch']
argument-hint: '<bug description, error message, or issue link>'
---

# Bug investigation

A good bug fix is small, has a regression test, and improves the system's testability.

## The seven questions (answer each before changing code)

1. **What is the symptom?** Exact error message, status code, stack trace, log lines, screenshot.
2. **What is the trigger?** Minimal reproduction — what input, what state, what sequence?
3. **What did the user expect?** State the contract that was broken.
4. **When did it start?** Recent deploy? Recent migration? Look at `git log` of the suspected file.
5. **Where is the boundary?** Is it Domain, Application, Infrastructure, or Web that's wrong?
6. **What is the root cause?** Not "where it crashes" — *why* the wrong state was possible.
7. **What else could be affected?** If the root cause is a missing invariant, where else does the same primitive flow without that invariant?

## Investigation order

1. **Reproduce locally first.** No debugging without a repro.
2. **Add the failing test.** Red. This is your "definition of fixed".
3. **Trace through the call stack** — handler → domain → repository. Find the layer where the wrong state was first allowed.
4. **Identify the missing guard or wrong assumption.** Quote the file:line.
5. **Propose the minimal fix** at the right layer. If the bug is a missing aggregate invariant, fix the aggregate — not the handler.
6. **Run the test.** Green.
7. **Run the full suite.** No regressions.
8. **Look around.** If you fixed a missing guard, search for the same primitive elsewhere — likely the same bug.

## Anti-patterns to refuse

- Catching the exception higher up to "make the symptom go away".
- Adding a `try/catch` around the bad code instead of fixing it.
- Adding a null-check at the call site instead of fixing the contract.
- Patching a symptom in Web that should be a Domain invariant.
- Skipping the regression test "because the fix is obvious".

## Output

```
## Bug: <one-line>

### Reproduction
<minimal steps>

### Symptom
<observed behavior — error, status, log>

### Root cause
<file>:<line> — <why the wrong state was possible>

### Layer of fix
<Domain | Application | Infrastructure | Web>  (justify briefly)

### Fix
<file>:<line> — <what changed and why>

### Regression test
<test class>::<test method> — covers <which path>

### Related risk
<other places the same root cause might surface, or "none found">

### Follow-up
<refactor proposal, rule update, or doc update — or "none">
```
