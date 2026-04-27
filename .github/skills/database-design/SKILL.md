---
name: database-design
description: Design PostgreSQL schemas for this codebase — tables, columns, indexes, constraints, EF Core configurations, and safe migrations. Use when modeling a new aggregate's persistence, optimizing a slow query, or planning a schema change.
---

# Database design

Activates when working on the persistence layer or making schema decisions. See [.github/instructions/database.instructions.md](../../instructions/database.instructions.md) and [.github/instructions/migrations.instructions.md](../../instructions/migrations.instructions.md).

## Engine and conventions

- **PostgreSQL 16+** primary. Snake_case via `EFCore.NamingConventions`.
- Default schema `public`; bounded contexts get their own schema as the system splits.
- Tables plural (`orders`, `order_items`); indexes `ix_<table>_<cols>`; FKs `fk_<table>_<ref>_<col>`; unique `ux_<table>_<cols>`.

## Modeling rules

- Strongly-typed IDs as `uuid` columns via `ValueConverter`.
- Money: `<name>_amount numeric(19,4)` + `<name>_currency char(3)`. Owned via `OwnsOne`.
- Enums: stored as `text` with `CHECK` constraint, mapped via `HasConversion<string>()`.
- Timestamps: `timestamptz`. Never `timestamp without time zone`.
- Optimistic concurrency: `xmin` mapped as a row-version `uint`.
- Audit: `created_at`, `created_by`, `updated_at`, `updated_by` shadow properties applied in `SaveChanges`.
- Soft delete (opt-in): `IsDeleted` shadow + global query filter.

## EF configuration template

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

        b.HasMany(o => o.Items)
            .WithOne()
            .HasForeignKey("order_id")
            .OnDelete(DeleteBehavior.Cascade);

        b.HasIndex(o => o.CustomerId);
        b.HasIndex(o => o.Status).HasFilter("status = 'pending'");
        b.Property<uint>("xmin").IsRowVersion();
    }
}
```

## Indexing decisions

- Every FK gets an index.
- Every column used in a hot `WHERE` / `ORDER BY` gets an index. Composite follows query column order.
- Partial indexes for low-selectivity columns: `WHERE status = 'pending'`.
- BRIN for monotonic columns (timestamps) on huge tables — measured.
- GIN for `jsonb` and full-text columns — measured.
- Don't over-index — every index is a write penalty. Quarterly review with `pg_stat_user_indexes`.

## Migration safety on Postgres

| Change | Risk | Strategy |
|---|---|---|
| Add nullable column | Low | Single migration. |
| Add NOT NULL with non-volatile default | Low (PG 11+) | Single migration. |
| Add NOT NULL with no default | High | Add nullable → backfill → set NOT NULL (3 migrations). |
| Drop column | Medium | Two stages across releases. |
| Rename column | High | Add new → backfill → switch reads → switch writes → drop old. |
| Add index on big table | Medium | `CREATE INDEX CONCURRENTLY` (raw SQL). |
| Add FK | Medium | Add `NOT VALID` then `VALIDATE CONSTRAINT`. |

EF generates blocking `CREATE INDEX` by default. For hot tables, hand-edit the migration.

## Outbox table (mandatory)

```
outbox_messages (
  id          uuid primary key,
  type        text        not null,
  payload     jsonb       not null,
  occurred_at timestamptz not null default now(),
  processed_at timestamptz null,
  attempts    int         not null default 0,
  error       text        null
);
create index ix_outbox_messages_unprocessed on outbox_messages (occurred_at) where processed_at is null;
```

`OutboxProcessor` background service polls every 1s, batches of 100, exponential backoff on failure, dead-letters after 10 attempts.

## Query rules

- `AsNoTracking()` always on reads.
- `Select(...)` to DTO — never load entities then map.
- Parameterize. Never interpolate into `FromSqlRaw`.
- `Include` sparingly; `AsSplitQuery()` for collections.
- For hot read paths: Dapper against `NpgsqlDataSource`, queries in `Infrastructure/Queries/`.

## Connection pooling

- `NpgsqlDataSource` registered as singleton.
- Pool size tuned via `Maximum Pool Size`; default 100, lower for small instances.
- Connection string via `IOptions<DatabaseOptions>` with validation.

## Output when designing

```
Aggregate: <Name>
Tables: <list>
Columns:
  - <table>.<col> <type> <constraints>
Indexes:
  - <ix_name> on <cols> <filter?>
FK / unique:
  - <name> on <cols>
Migration plan: <single | multi-stage with steps>
Lock duration estimate (1M-row table): <ms>
```
