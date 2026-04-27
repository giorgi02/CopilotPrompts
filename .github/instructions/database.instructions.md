---
name: 'Database & EF Core conventions'
description: 'PostgreSQL-first schema design, EF Core configurations, query rules, and migration discipline.'
applyTo: 'src/Infrastructure/Persistence/**/*.cs,src/Infrastructure/**/Migrations/**/*.cs'
---

# Database & EF Core conventions

## Engine

- **PostgreSQL 16+** primary. Schema and queries must work on Postgres without provider-specific hacks.
- SQL Server compatibility is a **best-effort** secondary target. If a feature requires Postgres-only (e.g. `jsonb` operators, `gen_random_uuid()`), call it out in the PR description and isolate via provider-conditional configuration.

## Naming

- Tables: **snake_case**, plural (`orders`, `order_items`).
- Columns: **snake_case** (`created_at`, `customer_id`).
- Indexes: `ix_<table>_<columns>`.
- Foreign keys: `fk_<table>_<referenced_table>_<column>`.
- Unique constraints: `ux_<table>_<columns>`.
- Apply globally via a `NpgsqlDbContextOptionsBuilderExtensions.UseSnakeCaseNamingConvention()` (`EFCore.NamingConventions`).

## Schema

- Default schema: `public`. Bounded contexts get their own schema (`orders`, `billing`) when the codebase splits.
- Strongly-typed IDs map to `uuid` columns via a `ValueConverter`.
- Money: two columns, `<name>_amount numeric(19,4)` + `<name>_currency char(3)`. Owned via `OwnsOne`.
- Enums: stored as `text` with a `CHECK` constraint, mapped via `HasConversion<string>()`. Avoid Postgres native enums (migration pain).
- Timestamps: `timestamptz`, never `timestamp without time zone`.
- Soft delete: opt-in per aggregate via an `IsDeleted` shadow property + global query filter.
- Row version for optimistic concurrency: `xmin` (Postgres) mapped as `[Timestamp]`-equivalent uint.

## Configuration

- One `IEntityTypeConfiguration<T>` per aggregate root in `Persistence/Configurations/`.
- Apply via `modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly)`.
- Never decorate Domain types with EF attributes.

```csharp
internal sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> b)
    {
        b.ToTable("orders");
        b.HasKey(o => o.Id);
        b.Property(o => o.Id).HasConversion(id => id.Value, v => new OrderId(v));
        b.OwnsOne(o => o.Total, m =>
        {
            m.Property(p => p.Amount).HasColumnName("total_amount").HasPrecision(19, 4);
            m.Property(p => p.Currency).HasColumnName("total_currency").HasMaxLength(3);
        });
        b.HasMany(o => o.Items).WithOne().HasForeignKey("order_id").OnDelete(DeleteBehavior.Cascade);
        b.HasIndex(o => o.CustomerId);
        b.Property<uint>("xmin").IsRowVersion();
    }
}
```

## Querying rules

- **Always** `AsNoTracking()` on read queries. Tracking only when you intend to mutate.
- **Always** project to a DTO with `Select(...)` for read endpoints. Never load entities then map.
- **Always** parameterize. EF Core does this for you — never use `FromSqlRaw` with string interpolation.
- Use `Include` sparingly — split queries with `AsSplitQuery()` when including collections.
- Pagination via `Skip/Take` for offset, `Where(x => x.Id > cursor).Take(n)` for cursor.
- For hot read paths: `Dapper` against `NpgsqlDataSource`, queries colocated in `Infrastructure/Queries/`.

## Migrations discipline

- One migration per logical change. Do not bundle unrelated schema changes.
- Migration name describes the change: `AddOutboxTable`, `AddOrderShippedAtColumn`.
- Inspect generated SQL before commit: `dotnet ef migrations script <FromMigration> <ToMigration>`.
- **Forward-only.** Drops/renames require:
  1. Add the new column/table.
  2. Backfill in a separate migration or out-of-band.
  3. Switch reads/writes.
  4. Drop the old in a later migration once all instances are upgraded.
- Never edit a migration after it has been applied to any non-local environment.

## Indexing

- Every foreign key gets an index.
- Every column used in a `WHERE` or `ORDER BY` of a frequent query gets an index.
- Composite indexes follow query column order.
- Partial indexes for filtered queries (`WHERE status = 'pending'`) when selectivity warrants.
- Review indexes quarterly with `pg_stat_user_indexes`.

## Transactions

- The `UnitOfWorkBehavior` opens a transaction per command and commits on success.
- For multi-aggregate operations, the transaction spans all `SaveChangesAsync` calls.
- Read queries do not start transactions.

## Outbox

- `outbox_messages` table: `id uuid pk, type text, payload jsonb, occurred_at timestamptz, processed_at timestamptz null, attempts int default 0, error text null`.
- `IDomainEventDispatcher` collects events from tracked aggregates in `SaveChangesAsync` and writes them to the outbox **in the same transaction**.
- `OutboxProcessor` `BackgroundService` polls every 1s, batches of 100, exponential backoff on failure, dead-letters after 10 attempts.

## Connection pooling

- Use `NpgsqlDataSource` registered as singleton. Pool size tuned via `Maximum Pool Size` (default 100, lower for resource-constrained envs).
- Connection strings via `IOptions<DatabaseOptions>` with validation.

## Forbidden

- `EF.Functions.Like("...%" + input + "%")` without input validation — use parameterized `ILike` and length-cap.
- `ToList()` early in a query chain (forces materialization).
- N+1 queries — if you see a `foreach` calling `.Include` per item, you are doing it wrong.
- Lazy loading. **Disabled** at context level.
- `DbContext` constructed manually. Always via DI.
- Mixing read and write in one `IQueryable` chain.
