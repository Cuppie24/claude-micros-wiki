# Gotcha: `global_db.UniSaver` schema/table for `rfr_subdivisions` — corrected

## Mechanism confirmed (2026-08-22): wrong table name → silent no-op, no error at all

Second confirmed case, this time in **bookkeep** (not CWATIS), and this time with the actual
procedure source read directly (`SHOW CREATE PROCEDURE global_db.UniSaver`), so the mechanism
below is proven, not inferred from symptoms.

**Symptom:** wiring inline grid-cell editing for `SubdivisionDirectoryTypesForm` (`Name` column) to
`UniSaver`. The cell edit visibly commits in the grid, `CallService.CallProcedureDT("global_db.UniSaver", ...)`
returns without throwing, but the DB row is never actually updated — no exception, no error text,
nothing. This is a *different* failure signature than the `rfr_subdivisions` case below (which threw
a MySQL syntax error / `NotImplementedException`) — a wrong **table** name (as opposed to a wrong
**schema** name) fails silently instead of loudly.

**Root cause, confirmed from `UniSaver`'s own body:** the procedure does *not* validate that the
table named in the `_Pars` JSON exists. Instead it cursors over **every table name actually present**
in the target schema (`SELECT TABLE_NAME FROM information_schema.TABLES WHERE TABLE_SCHEMA = _db`),
and only acts when a JSON key exactly matches one of those real table names:

```sql
DECLARE table_cursor CURSOR FOR
    SELECT TABLE_NAME FROM information_schema.TABLES WHERE TABLE_SCHEMA = _db;
...
FETCH table_cursor INTO _tb;
WHILE done = 0 DO
  IF JSON_EXTRACT(_tbl, CONCAT('$[0].',_tb)) IS NOT NULL THEN -- таблица найдена в json
    ... -- build and EXECUTE the UPDATE/INSERT
  END IF;
  FETCH table_cursor INTO _tb;
END WHILE;
```

If the JSON's table key doesn't match *any* real table in that schema, this loop simply never enters
the `IF` branch for any iteration — no `PREPARE`/`EXECUTE` ever runs, nothing is written, and the
procedure still `COMMIT`s and returns `SELECT 1` as if it succeeded. There is no error path for
"table not found" — it's indistinguishable from "there was nothing to do." Same applies to the
schema/database name (outer `db_cursor` loops over `information_schema.SCHEMATA` the same way).

The specific case: guessed the table as `bookkeep.subdiv_directory_types` (matching the
`bookkeep.SubdivisionDirectoryTypes_Get/Add/Delete` procedure names and the `subdiv_directory_type_id`
FK column) — plausible, and even confirmed by the user when asked directly, but still wrong. The real
table, confirmed via `SHOW CREATE PROCEDURE bookkeep.SubdivisionDirectoryTypes_Add`, is
`bookkeep.subdivision_directory_types` (full word "subdivision", not the abbreviated "subdiv" used
everywhere else in that feature's naming — procedures, FK column, and C# folder/class names all say
"subdiv"/"Subdivision" inconsistently, which is exactly why this was so easy to get wrong by pattern-matching).
The PK column itself is `subdiv_directory_type_id`, not `Id`/`id` — but that part doesn't need to be
guessed correctly on the C# side, since `UniSaver` resolves the real PK column itself via
`information_schema.KEY_COLUMN_USAGE WHERE CONSTRAINT_NAME = 'PRIMARY'` before building the `WHERE`
clause. Only the schema and table name are taken as given from the JSON and never validated.

**How this was actually resolved:** the user declined to hand over DB credentials once, so the first
fix was defensive — wrap the repository's `UpdateAsync` to re-fetch the row immediately after calling
`UniSaver` and throw a visible exception if the value didn't actually change (turns the silent no-op
into a loud, in-app error). On the *next* attempt the user could see a concrete "didn't save" message
instead of nothing, which was enough for them to then share read-only DB credentials so the real table
name could be confirmed directly (`DESCRIBE`/`SHOW CREATE PROCEDURE`) instead of guessed again. Keep
this verify-after-write pattern in place even after fixing the name — `CallProcedureDT` has no
`out error` parameter on this overload (confirmed via reflection on `Core.Client.CallService`, the
compiled package this project's `CallService` actually resolves to — `Bookkeep2/CallService.cs` itself
is 100% dead/commented-out code), so there is still no reliable success/failure signal from the call
site itself for *other* kinds of failure.

**Checklist addition:**
- A wrong **schema** name in a `UniSaver` call tends to fail loudly (MySQL syntax error or similar,
  since the generated SQL still references a schema that doesn't exist the way the procedure expects).
  A wrong **table** name fails *silently* — same `_Pars` JSON structure, but the procedure's own
  matching loop just skips it. Don't treat "no exception was thrown" as "the write happened."
- When a `UniSaver`-backed save might be wrong and DB access isn't available to verify directly,
  add a re-fetch-and-compare check after the write rather than trusting the call's return value (or
  lack of exception) at all — this is what actually turned an unfalsifiable "not working" report into
  a fixable one here.
- Don't pattern-match a table's physical name from sibling procedure names, FK column names, or C#
  class/folder names, even when several of them agree with each other — they can *all* independently
  drift from the real table name (as happened here: procedures, FK column, and C# code all said
  "subdiv", the table itself says "subdivision"). Verify the exact string via a procedure that's
  already confirmed working against the real table (`SHOW CREATE PROCEDURE`), not by consensus among
  guesses.

## Correction (2026-08-18)

An earlier version of this doc claimed `rfr_subdivisions` physically lives in the `refers` schema,
generalized from *other* `rfr_*` tables (`refers.rfr_currency`, `refers.del_m_user_system`/
`add_m_user_system`). That generalization was wrong for this table — confirmed directly by the
user: **`rfr_subdivisions` lives in `global_db`**, matching every other reference to it already in
`SubdivisionsRepository.cs` (`global_db.InsUpd_RfrSubDivisions`,
`CallService.DeleteLines(..., "global_db", "rfr_subdivisions", ...)`). Not every `rfr_*` table
shares the same schema — don't infer one table's schema from a sibling `rfr_*` table's precedent;
check the specific table.

Use `schema: "global_db"` when targeting `rfr_subdivisions` via `UniSaver`'s `_Pars` JSON blocks.

## Original (now-superseded) investigation, kept for context

The original 2026-08-13 session hit a MySQL syntax error using `schema: "global_db"`, concluded
(incorrectly, per the correction above) that the fix was switching to `"refers"`, and after that
switch hit a second failure — a clean `NotImplementedException`-shaped error ("Метод или операция
не реализована") — traced to `refers.rfr_subdivisions`'s table `COMMENT` containing SQL embedded
for the delete-confirmation feature (`CallService.DeleteLines`, tag `--- FOR DELETE ---`), which
apparently collides with a comment-reading step `UniSaver` does internally for its own audit-trail
captioning.

**This second finding is now unverified** — it was diagnosed by inspecting
`refers.rfr_subdivisions` specifically, which per the correction above may not even be the table
the application actually reads/writes. Whether `global_db.rfr_subdivisions`'s own `COMMENT` has
the same problem is unknown until `UniSaver` is actually exercised against it end-to-end.

If a future attempt with `schema: "global_db"` reproduces the same `NotImplementedException`,
check `SHOW CREATE TABLE global_db.rfr_subdivisions` for an embedded-SQL `COMMENT` the same way —
the general lesson (a table `COMMENT` doing double duty for a different `CallService` feature can
break `UniSaver`) may still hold, just needs re-confirming against the real table.

## Checklist

- Don't assume every `rfr_*` reference table lives in the same schema — `rfr_subdivisions` is in
  `global_db`; that doesn't tell you where some other `rfr_*` table lives. Check each one via its
  own call sites.
- A MySQL syntax error from `UniSaver` whose fragment names a schema *different* from what you
  passed is still a real signal something's wrong with the schema string — just verify the correct
  value carefully (cross-check multiple call sites for the *same* table, or ask) rather than
  pattern-matching from a different table's precedent.
- If a memory doc's claim gets contradicted by the user or by fresh evidence, correct it in place
  rather than layering a new doc on top — a stale "root cause" left uncorrected gets re-applied by
  a future session that has no way to know it was wrong.

Originally found in CWATIS, 2026-08-13, while wiring inline grid-cell editing for
`SubdivisionsForm` (`Code`/`Name` columns) to `UniSaver`. Schema claim corrected 2026-08-18 per
direct user confirmation.

## Gotcha: unknown **column** keys are silently ignored too

Confirmed 2026-10-24 by reading the live `global_db.UniSaver` body (`SHOW CREATE PROCEDURE`): the
statement is built by iterating `information_schema.COLUMNS` of the target table and picking only
the JSON keys that match a real column. A key that is not a column of that table (typo, or an audit
column name from a *different* table) is dropped with no error; the rest of the row is saved.

Precedent: CWATIS warehouse — `wh_documents` audit columns are `update_id`/`update_by`, but
`wh_doc_lines` uses `updater_id`/`updated_by`. Sending `update_id`/`update_by` for a line saves the
value but never records the editor. Rule: when the same save helper serves several tables, pass the
audit column names per table and check them against `information_schema.COLUMNS`.
