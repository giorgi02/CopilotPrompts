---
name: 'EF Core migration discipline'
description: 'Rules for creating, reviewing, and applying EF Core migrations safely.'
applyTo: 'src/Infrastructure/Persistence/Migrations/**/*.cs,src/Infrastructure/Persistence/Migrations/**/*.Designer.cs'
---

# Migration discipline

Migrations are append-only history. Treat each one as a contract you cannot break.

## Creating

- One migration per logical change. Do not combine "add column" with "rename table" with "seed data".
- Naming: imperative + scope. `AddOutboxTable`, `AddOrderShippedAtColumn`, `BackfillCustomerEmail`.
- Generated via:
  ```
  dotnet ef migrations add <Name> \
    --project src/Infrastructure \
    --startup-project src/Web \
    --output-dir Persistence/Migrations
  ```
- **Always** inspect the generated SQL before commit:
  ```
  dotnet ef migrations script <FromMigration> <ToMigration> --idempotent > /tmp/preview.sql
  ```
- Review for: unintended `DropColumn` (loss of data), unintended `AlterColumn` (table rewrite on Postgres can take a lock), missing indexes on new FKs.

## Forward-only

- **Never delete a migration that has been applied to any non-local environment.**
- Never edit a migration after it has been applied. Add a new migration to fix.
- `Down()` methods are best-effort only. The deployment strategy is roll-forward.

## Renames and drops — multi-step

Renaming a column or dropping a non-empty table is a four-stage process:

1. **Migration A** — add the new column/table; backfill in code or out-of-band SQL.
2. **Deploy A** — application reads from both, writes to both.
3. **Migration B** — switch the application to read/write only from the new.
4. **Deploy B**.
5. **Migration C** — drop the old.

Single-step drops are allowed only when the column has been unused for a full release cycle and its absence is verified in production.

## Locking on Postgres

- `ALTER TABLE ADD COLUMN` with a default is **non-blocking** on Postgres 11+; without a default it is fast.
- `ALTER TABLE ADD COLUMN ... NOT NULL` with a non-volatile default is fast on PG 11+. Backfilling with a volatile default rewrites the table — split into add nullable → backfill → set not null.
- `CREATE INDEX` blocks writes; use `CREATE INDEX CONCURRENTLY`. EF generates blocking by default — for production-impacting indexes, hand-edit the migration to use raw SQL with `CONCURRENTLY` (and remove the corresponding model snapshot reference if needed).

## Data backfills

- Small backfills (< 100k rows) can run in the migration via `migrationBuilder.Sql("UPDATE ...")`.
- Large backfills run as a separate one-shot Hangfire job, idempotent, batched, with progress logging.
- Document any backfill in the migration's XML doc comment with row count estimate and runtime.

## Seeding

- Reference data (currencies, countries, statuses) seeded via `HasData` in the entity configuration.
- User/test data **never** seeded by migrations. Use a separate `DataSeeder` invoked by environment.

## Schema review

- Every PR with a migration tags `@db-reviewer` (CODEOWNERS).
- Review against [anti-patterns](./anti-patterns.instructions.md) database section.

## Application at deploy time

- **Local / CI**: `db.Database.MigrateAsync()` at startup is fine.
- **Staging / Prod**: a separate CI step runs `dotnet ef database update` against the target DB **before** the new application instances start. Application boot does **not** run migrations in production.
- Failure of the migration step blocks the deploy.

## Forbidden

- Editing a migration after merge.
- `DropColumn` or `DropTable` in the same migration that introduces a replacement.
- Migrations that depend on application-level code execution (e.g. calling a static method to compute defaults).
- `EnsureCreated()` anywhere in the codebase.
