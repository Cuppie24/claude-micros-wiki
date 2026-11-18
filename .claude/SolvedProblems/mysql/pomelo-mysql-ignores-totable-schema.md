# Pomelo EF Core MySQL silently ignores the schema in `ToTable(name, schema)`

_Found 2026-09-08 in the `feedbackservice` project (micros)._

## Symptom

Runtime, from an EF Core query (not a build error):

```
MySqlConnector.MySqlException (0x80004005): Table 'feedback.m_user' doesn't exist
```

The entity was explicitly mapped to another schema:

```csharp
user.ToTable("m_user", "refers");   // real table is refers.m_user
```

The DbContext looks correct, the table really does exist as `refers.m_user`, yet
EF asks for it in the connection's own database (`feedback`, from
`Database=feedback` in the connection string). The error names the **wrong
schema**, which points away from the real cause.

## Root cause

`Pomelo.EntityFrameworkCore.MySql` does **not support schemas**. MySQL has no
schema concept distinct from *database*, so Pomelo's default
`MySqlSchemaBehavior.Ignore` **silently strips** the schema argument of
`ToTable(name, schema)`. EF Core then emits `... FROM \`m_user\`` with no
qualifier, and MySQL resolves it against the current database. Tell: the
exception names `<connection-db>.<table>`; if the schema had been honored it
would name `refers.<table>`.

`MySqlSchemaBehavior.Translate` does **not** fix it either - Pomelo wraps the
whole `schema.table` string in a single pair of backticks
(`` `refers.m_user` ``), an invalid identifier
(PomeloFoundation/Pomelo.EntityFrameworkCore.MySql#1241, wiki "Translations").

## Fix

MySQL FKs cannot target a view, so keep the split:

1. **Bridge the external table into the app database as an updatable view**, and
   drop the schema argument from the mapping (Pomelo ignores it anyway):

   ```sql
   CREATE OR REPLACE SQL SECURITY INVOKER VIEW `feedback`.`m_user` AS
   SELECT `UserId`, `UserName`, `PW` FROM `refers`.`m_user`;
   ```
   ```csharp
   user.ToTable("m_user");   // no schema -> resolves to the view in `feedback`
   ```
   Requires: same MySQL server instance, and the app's DB user has `SELECT` on
   `refers.*`. A single-table `SELECT col-list` view is updatable/insertable.

2. **Physical FKs still reference the real base table** `refers.m_user`
   (cross-database FK, same instance). EF reads go through the view; integrity is
   checked against the base table. If cross-database FKs aren't acceptable, drop
   those FKs and keep the columns + indexes (app-level integrity).

Alternative if you control the target DB: put everything in one database (no
cross-database anything).

## Secondary trap in the same fix

With `.UseSnakeCaseNamingConvention()`, an unmapped `public int Id` maps to
column **`id`**. Legacy micros tables use `UserId` / `MSysId`. After the schema
is resolved the next error is `Unknown column 'id'`. Map every such column
explicitly, and `Ignore(...)` model properties the legacy table lacks
(e.g. `CreatedAt`/`UpdatedAt` on `refers.m_system`):

```csharp
user.Property(u => u.Id).HasColumnName("UserId");
system.Property(s => s.Id).HasColumnName("MSysId");
system.Property(s => s.Name).HasColumnName("MSysName");
system.Ignore(s => s.CreatedAt);
system.Ignore(s => s.UpdatedAt);
```

## Checklist

- EF error `Table '<db>.<x>' doesn't exist` where `<db>` is your connection DB
  but the mapping says another schema -> Pomelo dropped the schema, not a typo.
- Every `ToTable("x", "someschema")` in a Pomelo context is a latent bug; the
  schema does nothing. Grep for the 2-arg overload.
- Cross-database access needs a view (for reads) + base-table FKs (FK cannot hit
  a view) + same server instance + `SELECT` grant on the other DB.
- Re-check column names against the real legacy table, not the snake_cased
  convention default; `Ignore` properties with no backing column.
- If EF migrations are later adopted, mark these entities
  `ToTable(..., t => t.ExcludeFromMigrations())`.
