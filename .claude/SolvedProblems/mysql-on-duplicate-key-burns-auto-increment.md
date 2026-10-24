# MySQL get-or-create via `INSERT ... ON DUPLICATE KEY` burns an AUTO_INCREMENT value on every call

Found 2026-10-24 in CWATIS (`global_db.StockGoods_Ensure`, stock cards of warehouse documents). MySQL 9.2, InnoDB.

## Symptom

A get-or-create procedure built as a single statement

```sql
INSERT INTO t (a, b) VALUES (...)
ON DUPLICATE KEY UPDATE id = LAST_INSERT_ID(id);
SELECT LAST_INSERT_ID();
```

returns the right id every time, but the ids of **newly created** rows have big gaps (table had ids 1–8, the next new row got 17).
No error anywhere; only visible by looking at the ids.

## Root cause

InnoDB reserves the AUTO_INCREMENT value **before** it detects the duplicate key. When the row already exists, the statement turns into an
update, but the reserved value is not given back. So every get of an existing row consumes one id. With a get-or-create that mostly
finds an existing row (the normal case), the counter grows with the number of calls, not with the number of rows.

`INSERT ... SELECT` (multi-row / unknown row count) additionally reserves ids in batches, so a one-off backfill also leaves a gap after it. This part is harmless and one-off.

## Fix

Look the row up first and run the `INSERT ... ON DUPLICATE KEY` only when it was not found. The `ON DUPLICATE KEY` stays there to handle
two sessions creating the same key at once:

```sql
SELECT id INTO _Id FROM t WHERE a = _A AND b = _B LIMIT 1;

IF _Id IS NULL THEN
    INSERT INTO t (a, b) VALUES (_A, _B)
    ON DUPLICATE KEY UPDATE id = LAST_INSERT_ID(id);
    SET _Id = LAST_INSERT_ID();
END IF;

SELECT _Id AS id;
```

Note `ON DUPLICATE KEY UPDATE id = LAST_INSERT_ID(id)` sets the column to its own value, so the row is not actually changed and
`ON UPDATE CURRENT_TIMESTAMP` columns are not touched.

## Checklist

- Get-or-create on a hot path: SELECT first, insert only when missing. Don't rely on the single-statement upsert just because it is shorter.
- With `INSERT ... SELECT`, a source that returns no rows leaves `LAST_INSERT_ID()` at the **previous** value (someone else's id). Check
  that the source row exists before reading `LAST_INSERT_ID()`.
- Gaps in AUTO_INCREMENT are never a correctness bug. Only fix this when ids are expected to stay dense or the call volume is high.
