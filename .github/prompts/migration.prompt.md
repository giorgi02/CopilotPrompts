---
description: 'Plan and generate a safe EF Core migration with backfill strategy if needed.'
agent: 'agent'
tools: ['search/codebase', 'edit/files']
argument-hint: '<schema change description>'
---

# Plan and generate a migration

Migrations are forward-only and applied in production by a dedicated CI step. A bad migration is downtime. Plan, then generate.

## Decide the type of change

| Change | Risk | Strategy |
|---|---|---|
| Add nullable column | Low | Single migration. |
| Add NOT NULL column with non-volatile default | Low (PG 11+) | Single migration. |
| Add NOT NULL column with no default or volatile default | High — table rewrite | Add nullable → backfill → set NOT NULL (3 migrations). |
| Drop column | Medium | Two-stage: stop reading/writing → drop. |
| Rename column | High | Add new → backfill → switch reads → switch writes → drop old (4 migrations across releases). |
| Add index on big table | Medium | `CREATE INDEX CONCURRENTLY` (raw SQL in migration). |
| Add FK | Medium | Add column nullable → backfill → add FK constraint `NOT VALID` then `VALIDATE CONSTRAINT`. |
| Drop table | High | Confirm zero references for one release cycle, then drop. |

## Process

1. **Describe the change** — what tables/columns, why.
2. **Pick the strategy** from the table above. State which migrations are needed across which releases.
3. **Generate the EF model change** in the configuration file.
4. **Generate the migration** — `dotnet ef migrations add <Name> --project src/Infrastructure --startup-project src/Web --output-dir Persistence/Migrations`.
5. **Inspect the generated SQL** — `dotnet ef migrations script <From> <To>`.
6. **Hand-edit if needed** for `CONCURRENTLY` indexes or split steps.
7. **Write the backfill** if needed — small in migration, large as a separate Hangfire job.
8. **Update the seed data** if reference values changed.
9. **Test locally** — apply migration to a fresh DB, then to a populated DB.
10. **Document** in PR description: lock duration estimate on prod-sized table, rollback strategy.

## Forbidden

- Editing a migration after merge.
- Combining unrelated schema changes in one migration.
- `DropColumn` and `AddColumn` of the same logical concept in the same migration.
- Migrations that depend on application code.
- `EnsureCreated()` anywhere.

## Output

```
Migration plan: <one-line>
Strategy: <from the table above>
Migrations to generate (in order):
  1. <Name> — <one-line of what>
  2. ...

Generated:
  - <migration file path>
  - <configuration file path>

Verify with:
  dotnet ef migrations script <From> <To>

Backfill required: <none | inline | separate job>
Lock impact estimate (1M-row table): <ms>
Rollback: <strategy or "forward-only — drops require explicit justification per migrations rules">
```
