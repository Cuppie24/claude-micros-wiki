# Gotcha: `global_db.UniSaver` schema for `rfr_subdivisions` — corrected

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
