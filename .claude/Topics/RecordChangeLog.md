# Record Change Log (log_db) — writing and viewing

Every change to a DB entity is logged in `log_db`, and every grid that shows DB entities lets the user open that log for the
current row («Информация о записи»). This file covers both halves. Facts below were verified on mars on 2026-11-10 unless noted.

## Rules (short)

1. **Every procedure that inserts, updates or deletes rows of an entity table logs each changed row** by calling the
   `add_rec_info` procedure of the **table's** schema. Never `INSERT` into `log_db.*` directly.
2. **Every form that shows DB entities in a grid wires «Информация о записи»** into that grid's context menu, opening the log
   of the current row's table.
3. If a write path can't log (`CallService.DeleteLines`, `CallService.UpdateCell`, `ExecuteString` / raw SQL), replace it with a
   procedure that does. `global_db.UniSaver` already logs (`EDIT`), so single-field saves through it are covered.

## How the log works

| Table's schema | Logger | Log table | Open in `RecInfoForm` with `MainDBName` |
|---|---|---|---|
| `cwatis` | `cwatis.add_rec_info` | `log_db.rec_info_cwatis` | `"cwatis"` |
| `global_db` | `global_db.add_rec_info` | `log_db.rec_info_refers` (**not** `rec_info_global`, which is empty) | `""` (the form then opens refers) |
| `refers` | `refers.add_rec_info` | `log_db.rec_info_refers` | `""` |
| `bookkeep`, `salary`, `subdivfinances`, `manufactures` | `<schema>.add_rec_info` | `log_db.rec_info_<schema>` | `"<schema>"` |

- Signature, the same in every schema: `add_rec_info(IN _ID_REC int, IN _TABLE_NAME varchar(100), IN _TYPE_CODE varchar(10), IN _ID_USER int)`.
  `_TABLE_NAME` is the table name **without** the schema. `_ID_USER` is `refers.m_user.UserId`; a negative id is a system user.
- Type codes come from `refers.m_rec_info_type`: `CREATE`, `EDIT`, `DELETE`, plus status codes stored as `EDIT` with a fixed
  description (for example `CONF` / `CONFNO` = «Установлен / Снят статус Утвержден»). Use `CREATE` / `EDIT` / `DELETE` unless a
  status code describes the change better.
- What gets logged: the logger reads the row itself (dynamic `SELECT` over the table's columns) and writes a snapshot of all
  non-empty columns. Columns whose comment starts with `!!! ` are skipped. `cwatis.add_rec_info` and `global_db.add_rec_info`
  don't fill `EditValues`, so an entry is a full snapshot, not a diff of what changed (switching to diffs is a deferred task —
  draft `CWATIS/Features/Warehouse/Sql/rec-info-editvalues.sql`, not applied, reviewed by nobody; deferred by the user
  2026-11-10). The draft adds an index `(TABLE_NAME, ID_REC)` on `log_db.rec_info_cwatis`, resets `@comment`/`@comment1`
  in `global_db.get_rec_info_editvalues_Javlon` (stale session values leak in otherwise), and makes both `add_rec_info`
  write `EditValues` and skip empty EDITs. Facts (mars, 2026-11-10): `EditValues` was filled 2025-04-23..2025-07-10 and
  is NULL since 2025-12-15, reason unknown; `_Javlon` is also called by `subdivfinances.add_rec_info`. It changes logging
  for all of CWATIS + `global_db` (33 cwatis procedures, UniSaver), so it needs the logger owner's agreement; before
  picking it up, re-dump the three procedure bodies and rebase the draft.
- Reading: `Refers.RecInfoForm` calls `refers.r_rec_info(_DBName, _ID_REC, _TABLE_NAME, …)`, which reads `log_db.rec_info_<_DBName>`.

## Writing the log in a procedure

- **Which logger:** the one for the schema of the **table**, not of the procedure. A `cwatis` procedure that writes
  `global_db.wh_list` calls `global_db.add_rec_info`.
- **Who:** every write procedure takes `IN _UserId int`, and the client passes `User.ID`. Adding the parameter to an existing
  procedure is a two-part change (procedure + C# dictionary key) that must ship together (`CallServiceUsage.md`: unknown or missing
  keys are not reported).
- **Every row:** batch procedures log each affected row. A procedure can't be called from a query, so loop over the ids.
  Precedent helper `cwatis.WhRecInfo_LogIds(_TableSchema, _TableName, _TypeCode, _UserId)` logs every id the caller put into the
  temp table `tmp_wh_rec_info_ids (Id int PRIMARY KEY)`; it supports `cwatis` and `global_db` tables and can be reused outside the
  warehouse.
- **`LAST_INSERT_ID()`:** `add_rec_info` inserts a log row and changes it. Save the new row's id in a variable **before** logging,
  and return that variable, not `LAST_INSERT_ID()`.
- **`DELETE` — before the delete, in a transaction.** The logger reads the row, so it must still exist. For a missing row the
  current logger silently records a snapshot left over from the previous call in the same session, because its dynamic
  `SELECT … INTO @comment` doesn't reset the variable. The transaction makes a delete blocked by a foreign key or trigger roll back
  its log entry too (`log_db` tables are InnoDB).
- **`EDIT` — only rows that really change.** Choose the set before the `UPDATE` with the same condition, then update through it.
  For a single row, `IF ROW_COUNT() > 0` after the `UPDATE` works.
- **Handler:** `DECLARE EXIT HANDLER FOR SQLEXCEPTION BEGIN ROLLBACK; DROP TEMPORARY TABLE IF EXISTS …; RESIGNAL; END;`.
  `CREATE TEMPORARY TABLE` / `DROP TEMPORARY TABLE` inside a transaction don't commit it.

```sql
-- single row
START TRANSACTION;
INSERT INTO global_db.wh_list (wh_list_name) VALUES (_Name);
SET _Id = LAST_INSERT_ID();                                     -- before add_rec_info
CALL global_db.add_rec_info(_Id, 'wh_list', 'CREATE', _UserId);
COMMIT;

-- batch delete
DROP TEMPORARY TABLE IF EXISTS tmp_wh_rec_info_ids;
CREATE TEMPORARY TABLE tmp_wh_rec_info_ids (Id int NOT NULL PRIMARY KEY) ENGINE = InnoDB;
START TRANSACTION;
INSERT INTO tmp_wh_rec_info_ids (Id) SELECT wh_doc_line_id FROM cwatis.wh_doc_lines WHERE wh_document_id = _DocumentId;
CALL cwatis.WhRecInfo_LogIds('cwatis', 'wh_doc_lines', 'DELETE', _UserId);  -- before the DELETE
DELETE l FROM cwatis.wh_doc_lines l JOIN tmp_wh_rec_info_ids t ON t.Id = l.wh_doc_line_id;
COMMIT;
DROP TEMPORARY TABLE IF EXISTS tmp_wh_rec_info_ids;
```

DataGrip flags `UPDATE … JOIN tmp …` without `WHERE` as "updates all table rows" — a false positive: the join restricts it.

Precedents: `CWATIS/Features/Warehouse/Sql/wh-rec-info.sql` (14 procedures, 2026-11-10); older single-row ones such as
`cwatis.ContractTypesApprove`. Note that `cwatis.ContractTypesDelete` logs **after** the `DELETE`; don't copy that.

## Viewing the log from a grid

```csharp
// in Load (not the constructor — see ../SolvedProblems/winforms/winforms-child-control-setup-constructor-vs-load.md)
grid.AddItemToContextMenu("Информация о записи", Resources.Info, (_, _) =>
{
    // footer rows («Всего:») and an empty grid: do nothing — reading a model on a footer row throws
    var model = /* current row's model, see SmartGridConventions.md → "Record Info" */;
    if (model == null || model.Id == 0) return;
    Main.RecInfoView(model.Id, "table_name");          // cwatis table
    // Main.RecInfoView(model.Id, "table_name", "");   // global_db / refers table
});
```

- `Main.RecInfoView(id, table)` always opens the `cwatis` log; use the three-argument overload for any other schema (table
  above). Passing `"refers"` explicitly also works, but adds a duplicate item to the form's database list.
- One menu item per grid; a master/detail form wires both grids, each with its own table.
- Warehouse precedent that also handles footer rows: `CWATIS/Features/Warehouse/Shared/View/WhRecordInfo.Attach(grid, schema,
  table, getCurrentId)`.
- **No menu item needed:** pickers and dialogs that only choose a value (status picker, lookup popups), trees of reference data
  owned by another form, and grids whose rows are computed (reports, totals) rather than one table row each.
- The log shows only what the write paths logged. When wiring the menu for a table, check its write procedures too (rule 1);
  otherwise the user sees an empty or partial log.
