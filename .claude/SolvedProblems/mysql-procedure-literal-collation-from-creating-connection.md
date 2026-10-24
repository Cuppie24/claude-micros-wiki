# MySQL procedure: string literals take the collation of the connection that *created* the procedure

Found 2026-10-24 in CWATIS (`global_db.StockGoodsGroups_Get` / `StockGoods_Get`, search filter). MySQL 9.2 server, created via `mysql.exe` 8.4.

## Symptom

The procedure is created without errors, but every call that reaches the search condition fails at runtime:

```
ERROR 1267 (HY000): Illegal mix of collations (utf8mb4_unicode_ci,IMPLICIT) and (utf8mb4_0900_ai_ci,IMPLICIT) for operation 'like'
```

The failing condition:

```sql
IN _Search varchar(200)            -- no charset / collation declared
...
WHERE s.stockgoods_name LIKE CONCAT('%', _Search, '%')   -- stockgoods_name is utf8mb4_unicode_ci
```

None of the tables involved uses `utf8mb4_0900_ai_ci`. That collation comes from nowhere visible in the procedure text.

## Root cause

A stored routine saves the `character_set_client` / `collation_connection` of the session that ran `CREATE PROCEDURE` (visible in
`SHOW CREATE PROCEDURE`: `collation_connection: utf8mb4_0900_ai_ci`). String **literals** inside the body (`'%'` here) get that collation on every
call, whoever calls it. `mysql --default-character-set=utf8mb4` sets the connection collation to the server default for utf8mb4, which on
MySQL 8+ is `utf8mb4_0900_ai_ci`.

A parameter with no declared charset gets the **database** default (`global_db`: `utf8mb3_general_ci`). So `CONCAT('%', _Search, '%')` mixes the
literal (`utf8mb4_0900_ai_ci`) and the parameter (`utf8mb3`). The result is `utf8mb4_0900_ai_ci`, which then can't be compared with a
`utf8mb4_unicode_ci` column. The same body created from a client with a different connection collation (e.g. a GUI tool set to
`utf8mb4_unicode_ci`) works. So the bug depends on *who created* the procedure, not on the SQL.

## Fix

Declare the string parameter's charset and collation explicitly, matching the column it's compared with. An explicit parameter collation
outranks the literal's, so `CONCAT` and `LIKE` resolve to it:

```sql
IN _Search varchar(200) CHARSET utf8mb4 COLLATE utf8mb4_unicode_ci
```

Comparing it with `utf8mb3_general_ci` columns (e.g. `rfr_goods_tree.MCode`) still works: MySQL converts the utf8mb3 side up to its utf8mb4 superset.

## Checklist

- A string parameter that ends up in `LIKE` / `=` / `CONCAT` with literals: declare `CHARSET ... COLLATE ...` matching the target column.
- Runtime "Illegal mix of collations" naming a collation no table uses: check `SHOW CREATE PROCEDURE` → `collation_connection`, i.e. the creating
  session, and look for literals in the failing expression.
- Test a new procedure's text-filter branch right after creating it. The non-filter branch does not touch the literal and passes.
