# `CallService.CallProcedureDT` silently drops dict keys the live procedure doesn't declare

## Symptom

A new filter/argument is added to a stored-procedure call by putting an extra key in the
`Dictionary<string, object>` passed to `CallService.CallProcedureDT(procName, dict)` (or
`CallProcedureNonQuery`). The C# compiles and runs, **no exception, no error string**, the
procedure returns rows as before — but the new argument has no effect at all. The feature behaves
exactly as if the parameter were never passed (e.g. "the picker still lists every row, ignoring the
type filter I added").

Found in CWATIS 2026-08-27, adding `_ContractTypeId` / `_ContractSubTypeId` to
`global_db.RefreshRfrSubDivisions_LookUp` (subdivision picker for the contracts filter): the C#
repository, the control, and the modal were all wired correctly, but the procedure on the DB still
had its old 5-parameter signature, so the two new dict entries did nothing.

## Root cause

`CallProcedureDT` is a thin WCF client wrapper (`GRP-Client.V2` → `Core.Client.CallService`) — it
ships `procName` + the dict to the GRP DB service, which builds the actual `CALL` **server-side**,
binding parameters **by name** against the procedure's real signature (`CommandType.StoredProcedure`
semantics). Consequences:

- A dict key with **no matching `IN` parameter on the live procedure is silently ignored** — not an
  error, unlike passing too many positional args to `CALL` directly (which *would* throw "Incorrect
  number of arguments").
- Because binding is by name, dict **insertion order doesn't matter** for correctness — but keeping
  the dict order aligned with the procedure signature is still worth doing as cheap insurance
  against any positional fallback.
- The procedure body is **not in the repo** (see
  [cwatis-stored-procedures-not-in-repo.md](cwatis-stored-procedures-not-in-repo.md)), so a C#-only
  diff looks complete and correct while the other half of the change is missing.

Distinguishing tell: other, pre-existing parameters in the same call (search text, id) keep working
normally — only the newly-added one is inert. If parameter *order* were the problem instead, the
old parameters would break too.

## Fix

Adding a parameter to a `CallProcedureDT`/`CallProcedureNonQuery` call is a **two-part change** that
must land together:

1. C#: add the key to the dict.
2. DB: `ALTER`/recreate the procedure to declare the new `IN` parameter (get the current body via
   `SHOW CREATE PROCEDURE schema.name` first; hand the user the full updated `DROP` + `create`
   script — schema-prefixed per the "Writing SQL Procedures" rules — since procedures live only in
   the live DB).

To confirm which half is missing when the argument seems ignored: temporarily surface the value the
C# side is actually sending (window title, log line, `MessageBox`), and run the procedure directly
(`CALL schema.name(...)`) with the new arg set vs. NULL. Same row count both ways, or "Incorrect
number of arguments" on the direct call → the procedure wasn't updated (or was created in the wrong
schema — watch for a `definer = other_db@'%'` procedure being recreated as `other_db.name` instead
of the schema it's actually called from). C# title shows `0`/blank → the C# provider/plumbing is
the problem instead.

## Checklist

- Adding a proc parameter? Change the DB **and** the C# in the same unit of work; neither half
  errors on its own when only one is done.
- "New proc argument has no effect, no exception" → assume the live procedure still has the old
  signature until `SHOW CREATE PROCEDURE` proves otherwise.
- Only the *new* argument is inert (old params still filter) → missing proc update, not a
  param-ordering bug.
- After updating the procedure, if it still misbehaves, the running app may be pointed at a
  different DB than the one patched, or the GRP DB service may have cached the proc's parameter
  metadata (restart it).
