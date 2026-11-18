# CallService — how to talk to the database from CWATIS

`CallService` is a static facade. Every DB access in the WinForms client goes through it — there
is no direct MySQL connection on the client. Its methods have no logic of their own: each one
packs its arguments into a gRPC request and sends it to a remote service (see "Which assembly
actually runs" below — this matters if you go looking for the implementation).

Written/verified 2026-09-28 from the CWATIS codebase (306 `CallProcedureDT`, 65 `GetTable`,
56 `CallProcedureNonQuery`, 32 `ExecuteString`, 15 `GetValueFromFunc`, 13 `DeleteLines`,
12 `UpdateCell` call sites), by decompiling the real assembly with `ilspycmd`, by reading the
client source in `D:\projects\projects\webserviceclient` (repo `microsdvl/webserviceclient`), and —
for `DeleteLines` specifically — by reading the actual **server** source, which is not checked out
under `D:\projects\micros` at all but exists as a bare mirror at
`D:\git mirrors\websvc.legacy.server.git` (repo `microsdvl/websvc.legacy.server`, real code on its
`master` branch — `main` is a near-empty placeholder). Read a bare mirror with
`git --git-dir="D:\git mirrors\<repo>.git" show <branch>:<path>`.

## Which assembly actually runs — a naming trap

CWATIS references **two** different compiled implementations of a class called `CallService`,
and it is not obvious which one your code calls:

1. `packages\GRP-Client.V2.1.0.3.4\lib\net48\GRP-Client.V2.dll` — an older, WCF-based
   (`System.ServiceModel`, `NetTcpBinding`) implementation, namespace `GRP_Client`. Its source is
   not in any local repo.
2. `packages\WebServiceClient.1.0.0.30\lib\net48\WebServiceClient.dll` — a newer, gRPC-based
   implementation, namespace `Core.Client`. Its source **is** local, in
   `D:\projects\projects\webserviceclient\source\Core\Core.Client\CallService.cs`.

`CWATIS\GlobalUsings.cs` has:

```csharp
global using GRP_Client = Core.Client;              // alias: "GRP_Client" now MEANS Core.Client
global using Core.Client;
global using static GRP_Client.CallService;
```

`GRP_Client` here is a **using-alias for the `Core.Client` namespace** — chosen, presumably, to
keep old code reading the same after the migration. A using-alias shadows a real namespace of the
same name for qualified lookups in C#, so both of these forms in CWATIS source:

```csharp
CallService.DeleteLines(...);            // via the plain `using Core.Client;`
GRP_Client.CallService.DeleteLines(...);  // via the alias — NOT the compiled GRP_Client namespace
```

resolve to `Core.Client.CallService` in `WebServiceClient.dll` (#2), never to the WCF class in
`GRP-Client.V2.dll` (#1). Assembly #1 is still referenced by the `.csproj` but is effectively dead
code for everything documented here. Decompile #2, not #1, if you need to check something.

## What Core.Client.CallService actually does

It is a thin static forwarder with no logic — every method is one line, delegating to
`Legacy.Services.Services.LegacyDbService` (also in the `webserviceclient` repo):

```csharp
public static DataTable DeleteLines(int[] DelIds, string ICN, string dBName, string DBTableName,
    string DbTabExclude, ref int[] TrueIds, string FKWhere, ref string res)
    => _dbService.DeleteLines(DelIds, ICN, dBName, DBTableName, DbTabExclude, ref TrueIds, FKWhere, ref res);
```

`LegacyDbService` packs the same arguments into a protobuf message (`legacy_db.proto`, service
`legacy.v1.LegacyDbService`) and sends it over `Grpc.Net.Client` to a remote server. That server's
implementation — the code that turns `db_name`/`db_table_name` into an actual SQL statement — is
**not** in `webserviceclient`, and **not** in `Cwatis.Legacy.Server` (which hosts a different,
unrelated gRPC service, `ContractsGrpcService`). It is the repo `microsdvl/websvc.legacy.server`,
which is not checked out as a working copy anywhere under `D:\projects\micros` — it only exists as
a bare mirror at `D:\git mirrors\websvc.legacy.server.git`, and only on its `master` branch (`main`
is a near-empty placeholder). Its `DeleteLines` handler is read in full in the `DeleteLines`
section below — including one confirmed bug in it. For every other method (`CallProcedureDT`,
`GetTable`, `UniSaver`'s procedure, etc.) the same repo almost certainly has the handler too, under
`WebSvc.Legacy.Server\Application\Features\LegacyDb\`, just not yet read for this document.

## Setup you never do yourself

- `CWATIS\GlobalUsings.cs` imports it globally (see above). Write `CallService.GetTable(...)` — no
  `using` needed. A few old files alias it (`using CallService = Core.Client.CallService;`) —
  harmless, don't add new ones.
- `Connect`/`Disconnect` are called by `Program.cs` at login. Never call them from a form.
- `CallService.UserInfo` (id, name) is filled at login. `CallProcedureDT`/`CallProcedureNonQuery`/
  `ExecuteScalarProc` automatically add `USER` and `USERNM` keys to your dictionary from it.

## Ready-to-use methods

| Method | Use it for | Returns / on error |
|---|---|---|
| `CallProcedureDT(proc, dict)` | **Default choice.** Call a stored procedure that `SELECT`s a result set (lists, single rows, "get" procs). | `DataTable`. On a DB error the library shows a message box and returns **`null`** — always null-check. |
| `CallProcedureNonQuery(proc, dict, out string error)` | Procedures that write (insert/update/delete/status change) when you need to **see the error** (FK violation, `SIGNAL SQLSTATE '45000'`). | affected rows; `error` is non-empty on failure. |
| `ExecuteScalarProc(proc, dict, out error)` | Procedure that returns one value. (Unused in CWATIS so far — `CallProcedureDT` + `Rows[0][0]` is the common way.) | `object` / `null`. |
| `GetValueFromFunc(func, dict)` | Call a MySQL **function** (`bookkeep.GetMaxDocState`, `global_db.GET_CURRENCY_CODES`). | `object` — wrap in `Numeric.Int(...)` / `Convert.ToString(...)`. |
| `GetTable(table, fields="*", where="1=1", leftjoin="", colOrder="")` | Read a plain table/view without a procedure. | `DataTable` / `null`. |
| `ExecuteString(sql)` | Ad-hoc `SELECT` returning one cell as text. | `string` / `null`. |
| `UpdateCell(schema, table, col, value, idAsString)` | Legacy single-cell update. **Prefer `UniSaver`** (below) — it logs who changed what. | `string` (error text or empty). |
| `DeleteLines(ids, idCol, schema, table, excludeArg, ref trueIds, fkWhere, ref res)` | **Standard delete** of one or many rows by id, with a server-side generic FK-reference scan before each delete. No procedure needed. `excludeArg` is currently inert (server bug) and a scan-blocked id sets no error — see below. | `DataTable`; `trueIds` the ids actually deleted (check this, not just `res`); `res` only set on a raw SQL exception. |
| `GetMaxColValue(table, col, filter)` | `MAX(col)` — unused in CWATIS. | `int`. |

Not used in CWATIS and not recommended for new code: `CallProcedureNonQueryEx`, `Save1Execute`,
`MultiSave1Execute`, `EditExecute`, `RWSDataToBookkeep`, `BookkeepToRWS_OtherPayments`,
`GetProfileKeysValue`/`AddProfileKeyValue` (use `ProfileKeysValueManager` instead — see
`../../projects/CWATIS/.claude/docs/ProfileKeysValueManager.md`), `GetPermission`.

### CallProcedureDT — the standard read

```csharp
public async Task<IEnumerable<ContractsModel>> ListAsync(ContractsFormFilterDTO dto)
{
    var dic = new Dictionary<string, object>
    {
        ["_SubdivId"]   = dto.SDId == 0 ? null : (object)dto.SDId,   // null → procedure sees NULL
        ["_StartDate"]  = dto.DateStart,
        ["_CntrTypeId"] = dto.CntrTypeId
    };
    var dt = await Task.Run(() => CallService.CallProcedureDT("cwatis.Contracts_Get", dic));
    if (dt == null) return Enumerable.Empty<ContractsModel>();
    return dt.AsEnumerable().Select(_mapper.Map);
}
```

Rules:

1. **Always schema-qualify** the procedure: `cwatis.X`, `global_db.X`, `bookkeep.X`, `refers.X`.
2. **Dictionary keys must equal the procedure's `IN` parameter names**, including the leading
   underscore (`_UserId`, `_CalcPrice_Ids`). Binding is by name.
3. **Unknown keys are silently ignored** — no error. If a new filter "does nothing", the live
   procedure doesn't have that parameter yet (see `../SolvedProblems/micros-libraries/callproceduredt-unknown-params-silently-dropped.md`).
   Procedures live only on the DB server, not in the repo — check with `SHOW CREATE PROCEDURE`.
4. **Lists of ids go as `string.Join(",", ids)`**, never `JsonConvert.SerializeObject(ids)` —
   `FIND_IN_SET` on the server chokes on the brackets (see `claude-global/SolvedProblems/mysql/find-in-set-json-array-drops-first-last-element.md`).
5. **Every call blocks the calling thread** (synchronous WCF). In repositories wrap it in
   `await Task.Run(() => ...)`; forms `await` the repository.
6. Check the `DataTable` for `null` before `.Rows` — errors come back as `null`, not exceptions.

### CallProcedureNonQuery — the standard write

```csharp
string error = await Task.Run(() =>
{
    CallService.CallProcedureNonQuery("cwatis.Contracts_Delete", dic, out string e);
    return e;
});
if (!string.IsNullOrEmpty(error)) throw new InvalidOperationException(error); // presenter shows it
```

Use this (not `CallProcedureDT`) for deletes and anything that may fail on a constraint —
`CallProcedureDT` hides DB errors from the caller. Precedent: `ContractsRepository.DeleteAsync`.

### GetTable — read a table without writing a procedure

```csharp
var dt  = CallService.GetTable("cwatis.pricecards", where: $"PriceCards_Id = {priceCardsId}");
var dt2 = CallService.GetTable("global_db.m_subject", where: $"MCODE_SUBJ LIKE '%{text}%'");
var dt3 = CallService.GetTable("global_db.rfr_subdivisions");            // whole table
var dt4 = CallService.GetTable("cwatis.calcprices", fields: "CalcPrice_Id, Editor_Id",
                               where: "IsEditing IS NOT NULL", colOrder: "CalcPrice_Id DESC");
```

The arguments are pasted into `SELECT {fields} FROM {table} {leftjoin} WHERE {where} ORDER BY {colOrder}`
on the server. So:

- **Only interpolate trusted values** (numeric ids, values you generated). User-typed text in
  `where` is a SQL-injection hole — escape it or use a procedure.
- Good for lookups of one row by id, reference tables, and "does it exist" checks.
- Not for anything with business logic, joins across schemas, or paging — write a procedure.

### ExecuteString / GetValueFromFunc — one value

```csharp
string json = CallService.ExecuteString($"select ContractType_Data from cwatis.contracttypes where ContractType_Id = {id}");

var dic = new Dictionary<string, object> { ["_DocId"] = docId };
int state = Numeric.Int(CallService.GetValueFromFunc("bookkeep.GetMaxDocState", dic));
```

Same injection warning as `GetTable`. `ExecuteString` returns the first column of the first row as text.

### DeleteLines — the standard delete

Deletes rows by primary key without writing a procedure. Before deleting each row it runs a
**generic** foreign-key scan on the server (not a check against one table you name), and only
deletes the row if that scan finds nothing referencing it.

```csharp
public async Task<(string error, int[] deleted)> DeleteAsync(IEnumerable<int> selectedIds)
{
    int[] ids = selectedIds.ToArray();
    int[] deleted = ids;          // out: ids the server actually deleted
    string res = "";

    await Task.Run(() => CallService.DeleteLines(
        ids,                      // 1. ids to delete
        "PreContrId",             // 2. primary-key column of the table
        "cwatis",                 // 3. schema (ignored if argument 4 already has a schema prefix)
        "cwatis.pre_contracts",   // 4. table to delete from — always pass it schema-qualified
        "cwatis.rep_act",         // 5. "exclude" argument — see warning below, currently has no effect
        ref deleted,              // 6. out: ids that were really deleted
        "",                       // 7. extra filter for the FK scan ("" in every CWATIS call)
        ref res));                // 8. out: raw SQL error text — only set on a genuine SQL exception

    return (res, deleted);
}
```

Verified 2026-09-28 by reading the real server source (repo `microsdvl/websvc.legacy.server`,
branch `master` — bare-mirrored at `D:\git mirrors\websvc.legacy.server.git`, not checked out
under `D:\projects\micros`; its `main` branch is a near-empty placeholder). Full write-up:
`../SolvedProblems/micros-libraries/deletelines-dbtabexclude-silently-ignored-and-fk-block-is-silent.md`.

**Argument 3 vs 4 (schema).** The handler does:
```csharp
if (tableinfo.Contains(".")) { databaseName = tableinfo.Split('.')[0]; databaseTableName = tableinfo.Split('.')[1]; }
else { databaseName = request.dBName; databaseTableName = request.DBTableName; }
```
A dot in argument 4 makes its own prefix win and argument 3 is ignored entirely; no dot falls back
to argument 3. Always pass argument 4 schema-qualified (the CWATIS convention) so this never
depends on argument 3 matching.

**⚠ Argument 5 ("exclude") currently does nothing.** The gRPC controller
(`WebSvc.Legacy.Server/Web/Controller/Grpc/LegacyDbService.cs:32`, unchanged since 2025-10-02)
passes the table-name argument **twice** by mistake instead of forwarding the real exclude
argument:
```csharp
await _legacyDbService.DeleteLines(request.DelIds, request.Icn, request.DbName,
    request.DbTableName, request.DbTableName /* bug: should be request.DbTabExclude */, request.FkWhere, ...);
```
Whatever you pass as argument 5 (CWATIS callers pass things like `"cwatis.rep_act"`) never reaches
the server logic. Keep passing it — it documents intent and costs nothing — but don't rely on it
to actually exclude anything.

**⚠ A row blocked by the FK scan looks like success.** Per id, the handler first calls a MySQL
stored procedure (`FKCheckW`, source not in any repo) that scans for references to that row. If it
finds any, the id is skipped — not deleted, not added to the `deleted`/`TrueIds` output — **but
`res` is left empty**, exactly as on success. `res` only gets text from the *other* failure path: a
raw `MySqlException` thrown by the actual `DELETE FROM {schema}.{table} WHERE {icn} = {id}`
statement (a real DB-level constraint the scan missed, a bad column, a timeout). **Never treat an
empty `res` as proof the delete fully succeeded** — compare the returned `deleted` ids against the
ids you asked to delete:

```csharp
var (error, deleted) = await _repo.DeleteAsync(ids);
if (!string.IsNullOrEmpty(error))
{
    MyMessageBox.Show(error, "Удаление невозможно");   // a real SQL error
    return;
}
if (deleted.Length < ids.Count())
{
    // some ids were silently blocked by the FK scan — res says nothing about this
    MyMessageBox.Show("Часть записей не удалена: на них есть ссылки.", "Удаление невозможно");
}
await ReloadAsync();
```

`PreContrRepository`'s existing `res.ToLower().Contains("rep_act")` check (matching text into a
user-friendly message) only fires on the raw-SQL-exception path — it happens to work there because
a real DB-level FK constraint between `rep_act` and `pre_contracts` throws a MySQL error whose text
names `rep_act`. It says nothing about the generic FK-scan path above; don't copy it as a general
pattern for detecting a blocked delete.

**No transaction across ids.** Each id gets its own connection and its own `DELETE` statement. If
id 3 of 5 fails, ids 1–2 are already committed — there is no rollback of the whole batch.

When to write an `X_Delete` procedure instead: you need real control over the failure/skip
behavior above — a custom message when blocked, atomicity across the whole batch, a soft delete,
or a cascade. Then call it with `CallProcedureNonQuery` and surface `error`
(precedent: `ContractsRepository.DeleteAsync`).

## UniSaver — universal edit of a row

`global_db.UniSaver(IN _UserId int, IN _Pars text)` is a stored procedure that writes any set of
columns into a row of any table in any schema, described by JSON. It builds the `UPDATE` (or
`INSERT`) dynamically from `information_schema`, runs it in a transaction and writes a change-log
entry with your user id. Use it for **inline grid edits** and **saving a form's fields into one
row** — instead of writing an `X_Update` procedure per table. Facts below are verified against
the procedure body (2026-09-28).

### JSON format

```json
{
  "cwatis":    { "contracts":      { "Id": 123, "data": { "Comment": "text", "Editor_Id": 7 } } },
  "global_db": { "rfr_goods_tree": { "Id": 555, "data": { "PLName": "Название для ПЛ" } } }
}
```

- Top level: **physical schema name**. Inside: **table name**. Inside: one `Id` and one `data` object.
- Several schemas and several tables per call are allowed; **one row per table** per call.
- `Id > 0` → `UPDATE table SET ... WHERE <primary key> = Id`.
  `Id = 0` → `INSERT INTO table (...) VALUES (...)`.
- Every key in `data` that matches a column name (case-sensitive) becomes `column = value`.
  Keys matching no column are **silently skipped**.
- Values are pasted into SQL exactly as JSON renders them: `"text"` (quoted string), `12.5`,
  `true`/`false` (become 1/0), `null` (writes NULL), `"2026-09-28T10:00:00"` (MySQL parses
  ISO-8601 into DATETIME). Do not put nested objects or arrays in `data` — they produce invalid SQL.

You never write this JSON by hand: `JsonHelper.BuildJson` (MC_GlobalForms) builds it.

### Step by step in C#

```csharp
using System.Dynamic;
using MC_GlobalForms.HelperClasses;   // JsonHelper, Numeric

public async Task SaveContractCommentAsync(int contractId, string comment)
{
    // 1. Columns to write. An ExpandoObject used as a dictionary: key = column name, value = new value.
    //    null writes NULL. Column names exactly as in the DB, case included.
    dynamic item = new ExpandoObject();
    var cols = (IDictionary<string, object>)item;
    cols["Comment"]     = comment;
    cols["Editor_Id"]   = User.ID;
    cols["EditDate"]    = DateTime.Now;

    // 2. Which row. Key = physical schema, value = (table, id of the row, list of column sets).
    var blocks = new Dictionary<string, (string, int, List<object>)>
    {
        ["cwatis"] = ("contracts", contractId, new List<object> { item })
    };

    // 3. Parameters: _Pars = JSON, _UserId = who is saving (goes into the change log).
    var dic = new Dictionary<string, object>
    {
        ["_Pars"]   = JsonHelper.BuildJson(blocks),
        ["_UserId"] = User.ID
    };

    // 4. Call. It is a procedure, so CallProcedureDT. Returns SELECT 1 — nothing to read.
    await Task.Run(() => CallService.CallProcedureDT("global_db.UniSaver", dic));
}
```

Instead of an `ExpandoObject` you may pass a **model object**: every public property becomes a
column. Use this only when the model's property names equal the column names and you really want
all of them written (precedent: `CalcPriceLinesRepository` builds an `ExpandoObject` explicitly
to control the list).

### Variants

**Save one cell from a grid** — a reusable repository method:

```csharp
public Task SaveFieldAsync(string schema, string table, int id, string column, object value)
{
    dynamic item = new ExpandoObject();
    ((IDictionary<string, object>)item)[column] = value;

    var blocks = new Dictionary<string, (string, int, List<object>)>
    {
        [schema] = (table, id, new List<object> { item })
    };
    var dic = new Dictionary<string, object> { ["_Pars"] = JsonHelper.BuildJson(blocks), ["_UserId"] = User.ID };
    return Task.Run(() => CallService.CallProcedureDT("global_db.UniSaver", dic));
}

// usage from AfterEdit:
await _repo.SaveFieldAsync("cwatis", "pricelistrows", rowId, "RentRetail_Prc", newValue);
```

**Two tables in one transaction** — two schema keys, one table each:

```csharp
var blocks = new Dictionary<string, (string, int, List<object>)>
{
    ["cwatis"]    = ("pricecards",     priceCardId, new List<object> { cardCols }),
    ["global_db"] = ("rfr_goods_tree", goodId,      new List<object> { goodCols })
};
```

Both statements run inside the same `START TRANSACTION … COMMIT`.

**Many rows of the same table** — `BuildJson` can hold only one table per schema, so either loop
over `SaveFieldAsync` (one call per row, one transaction each) or use a bulk procedure such as
`cwatis.MassEdit` (precedent: `CalcPriceLinesRepository.MassEditColumnAsync`, one atomic
`UPDATE … CASE WHEN`). Never `blocks.Add("cwatis", …)` twice — duplicate key exception.

**Clear a value** — pass `null`: `cols["Editor_Id"] = null;` writes `NULL`
(precedent: `CalcPriceRepository.SetLockAsync` releasing a lock).

### Inline grid edit, end to end

1. `SmartGrid.AfterEdit` fires; the view passes column name, new value and the row's id to the presenter.
2. Presenter validates and calls `repository.SaveFieldAsync(schema, table, id, column, value)`.
3. Repository calls `UniSaver` as above.
4. Repository re-reads that row with the feature's `_Get` procedure and refreshes the grid row
   (so computed columns and triggers are reflected).
Precedent: `CalcPriceRepository.UpdateCalcPriceAsync` → `UpdateCalcPriceModelRow`.

### What the procedure does and does not do

| Does | Does not |
|---|---|
| Finds the table and its primary key via `information_schema` | Validate that `Id` exists — a missing row updates 0 rows silently |
| Writes only keys that match real columns | Report unmatched keys — they are skipped without error |
| Runs all statements in one transaction | Return anything useful — always `SELECT 1` |
| Logs `EDIT` via `<schema>.add_rec_info(id, table, 'EDIT', _UserId)` for `global_db`, `refers`, `bookkeep`, `cwatis`, `manufactures`, `salary`, `subdivfinances` | Log inserts, or log for other schemas |
| `INSERT` when `Id = 0` | Return the new id — it is computed as `AUTO_INCREMENT - 1` only for logging and is wrong under concurrent inserts |

Therefore: **use `UniSaver` for updates only**. Create rows with the feature's `X_Add`/`X_Save`
procedure that returns the id.

### Errors

The procedure has no exception handler: a bad value or a wrong column type raises the MySQL error,
which `CallProcedureDT` shows in a message box and turns into a `null` return. If the caller needs a
status, wrap the call in `try/catch` and map to a result object
(precedent: `CalcPriceLinesRepository.TransferPLNameToPriceListAsync` → `MassEditResult`).

### Pitfalls

1. **Wrong schema** (`refers.rfr_subdivisions` when the table lives in `global_db`) → the table is
   never found or a syntax error appears deep inside UniSaver
   (`../SolvedProblems/micros-libraries/unisaver-schema-must-be-physical-not-procedure-namespace.md`). Check the real
   schema per table; don't infer it from a procedure's namespace.
2. **Column name typo or wrong case** → silently not written. If a save "does nothing", compare the
   key against `SHOW COLUMNS`.
3. **`HelperMethodClasses.UniSaver(...)`** looks generic but is hardcoded to `cwatis.pricecards`,
   and its `List<int>` overload throws on the second id
   (`../../projects/CWATIS/.claude/docs/SolvedProblems/helpermethodclasses-unisaver-hardcoded-pricecards.md`). Use your own `SaveFieldAsync`.
4. **Passing a model object** writes every public property, including ones you did not intend
   (ids, computed fields). Prefer an explicit `ExpandoObject`.
5. **Strings with quotes/backslashes** are JSON-escaped by Newtonsoft and MySQL understands the
   same escapes, so they are safe. Nested objects are not.

## Checklist for a new data-access method

- [ ] Lives in the feature's `Repositories/` class, not in the form.
- [ ] Schema-qualified procedure/table name.
- [ ] Dictionary keys = procedure `IN` parameter names; id lists joined with `,`.
- [ ] `await Task.Run(() => CallService....)`.
- [ ] Reads: `CallProcedureDT` + null check. Writes that can fail: `CallProcedureNonQuery` + `error`.
- [ ] Deletes: `DeleteLines` with argument 4 schema-qualified; compare returned `deleted` ids against requested ids (don't trust empty `res` alone); `X_Delete` procedure only when you need real control over failure/skip behavior.
- [ ] Single-row field edits: `UniSaver` with the physical schema, exact column names, `Id > 0`.
- [ ] No user-typed text interpolated into `GetTable`/`ExecuteString`.
