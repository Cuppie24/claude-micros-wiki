# DataRow.ItemArray filled positionally breaks when a stored proc's column layout changes

_Found in SubdivFinances, 2026-08-28._

## Symptom

- A WinForms form that adds a synthetic "Все" ("All") row to a filter combo throws on load.
- Exception comes from `System.Data` (not the app code): `System.FormatException: "Input string was not in a correct format."` or `System.ArgumentException` ("type of value has a mismatch with column type" / "Input array is longer than the number of columns in this table"), thrown from a line like:

  ```csharp
  DataRow dr = dt.NewRow();
  dr.ItemArray = new object[] { 0, 0, "Все", "Все", "Все" };   // positional
  dt.Rows.InsertAt(dr, 0);
  ```

- Only privileged users hit it (the block is usually guarded by `if (UserRight.Supervisor || UserRight.Manager || ...)`), so it looks intermittent / user-specific.
- `git blame` on the failing line points at a commit years old — the line itself was never touched.

Concrete instance: `SubdivFinances/Main/CashListsForm.cs` `CashListsForm_Load`, `dt` = `MainClass.AccessedCashSubDivDT.Copy()`, whose schema comes from the MySQL procedure `subdivfinances.GetSubDivisions4User`. Fixed in commit `574fc6b` by removing the 5th `"Все"`.

## Root cause

`DataRow.ItemArray`'s setter assigns `value[i]` into **column ordinal `i`**, converting to that column's `DataType`. It is a **positional** contract with the DataTable schema:

- array longer than `Columns.Count` -> `ArgumentException`;
- a value that can't convert to its positional column's type (e.g. the string `"Все"` into an `Int32`/`Boolean` column) -> `FormatException`/`ArgumentException`.

The DataTable schema here is defined entirely by the `SELECT` list of a **stored procedure that lives only on the DB server, never in the repo** (see `../micros-libraries/cwatis-stored-procedures-not-in-repo.md`). Someone altered `GetSubDivisions4User`'s column list (a leading string column such as `MCode` was removed/reordered), so the position that used to be a 3rd trailing `string` is now a numeric flag column. The hard-coded `{ 0, 0, "Все", "Все", "Все" }` (two ints + three strings) no longer matches `int, int, string, string, <numeric>` -> the 5th `"Все"` fails to convert.

The error message names the now-numeric column (e.g. an `IsAccess`-style flag) which appears **nowhere near** the failing code, so it misdirects.

Corroborating tell in git history: a sibling commit changed a combo's display member from `MCode` to `Name` ("Change SDIdsCombo to display 'Name' instead of 'MCode'") and another converted an empty `catch {}` into one that shows `ex.ToString()` — both are fingerprints of the same result-set change and the hunt for the resulting invisible failures.

## Why it stays hidden for a long time

1. The breaking change is server-side (stored proc `ALTER`), invisible to git; `git log`/`blame` on the C# line points away from the cause.
2. The write is **positional** — no column name in the code to grep; the thrown column name isn't referenced anywhere nearby.
3. Guarded by a privilege check — only Supervisor/Manager/"view all" users ever execute it.
4. Unhandled exceptions route through a global catch handler (here `Refers.CatchHelper.HandleException`, external) and surface as a generic "form won't open".
5. The pattern is copy-pasted across ~20 forms, so the first sighting looks like a one-off and gets patched in just that file.

## Fix

Don't populate the row positionally. Set only the columns the binding needs, **by name**:

```csharp
DataRow dr = dt.NewRow();
dr["LocId"] = 0;          // combo ValueMember
dr["Name"]  = "Все";      // combo DisplayMember
dt.Rows.InsertAt(dr, 0);
```

Or a schema-agnostic helper:

```csharp
foreach (DataColumn c in dt.Columns)
    dr[c] = c.DataType == typeof(string) ? (object)"Все" : DBNull.Value; // or 0 for numerics
dr["LocId"] = 0;
```

To confirm the DB-side layout: `SHOW CREATE PROCEDURE subdivfinances.GetSubDivisions4User;` and compare the `SELECT` order.

## Checklist / rule

- Treat `dr.ItemArray = new object[] { ... }` and `dt.Rows.Add(v1, v2, ...)` as **schema-coupled** code. If the table is filled from a stored proc that isn't in the repo, this is a latent break waiting for the next `ALTER`.
- When you see one such positional "add All row" site fail, grep the whole solution for the siblings before calling it fixed:
  `ItemArray = new object\[\]`, `Rows.Add(0, 0, "` , `"Все", "Все"`.
- A `FormatException`/`ArgumentException` from `System.Data` on a form's `Load`, naming a column not mentioned in that file, = positional row write vs. a changed result set.
- Prefer by-name column writes for synthetic rows; they survive column additions/removals/reordering.

## SubdivFinances call sites (all against a `.Copy()` of `SubDivDT` / `Accessed*SubDivDT`, i.e. `subdivfinances.GetSubDivisions4User`)

Migrated 2026-08-28 to the by-name form (`dr["LocId"]=0; dr["Name"]="Все";` — the combos bind
`SetDataBinding(dt, "LocId", "Name")`, so those two columns are all that matters; `LocId=0` also keeps
the row through the `"...>0 or LocId=0"` `Binding.Filter` on several combos):

`Main/BalanceBegNewForm.cs`, `Main/BalancePeriodForm.cs`, `Main/Balance_NewForm.cs`,
`Main/CashLines4AddForm.cs`, `Main/CashListRepForm.cs`, `Main/CurrentBalancesForm.cs`,
`Main/CurrentBalancesNewForm.cs`, `Main/DocConvForm.cs` (note: its `if (UserRight...)` guard is
commented out -> block runs for every user), `Main/DocSDForm.cs`, `Main/DocVehicleForm.cs`,
`Main/OrdersForm.cs`, `Main/ProjectForm.cs`, `Main/SWorkerRefInpForm.cs`,
`Reports/CashReportForm1.cs`, `Reports/InpOutItemsUZSRepForm.cs`, `Reports/SummaryBalanceRepForm.cs`,
`Reports/SummaryBalanceRepNewForm.cs`, `Users/UserRightsForm_New.cs` (was `SDIds.Rows.Add("0","0",...)`
appending at the end -> now `NewRow` + `InsertAt(0)`). `Main/CashListsForm.cs` was the original report,
patched earlier in the same session by commenting the 5th element (still positional — could be tidied
to by-name too).

Still using a 5-element positional `ItemArray` against `AccessedSubDivDT`, **not yet migrated** —
different string payloads (`"Без отдела"/"Б/О"`, `"Все"`), check each against the current proc columns:
`RWSMain/BankAccountsBegSumForm.cs:45,49`, `RWSMain/CashesBegSumForm.cs:45,49`,
`RWSMain/DayCashStatementF.cs:47`, `RWSRefers/BankAccountsForm.cs:46,50`, `RWSRefers/BankAccountsF.cs:48`,
`RWSRefers/CashesForm.cs:46,50`, `RWSRefers/CashesF.cs:46`. Also `HelpClasses/Helper.cs:258,331,349,367`
(different tables — `GetBalances` etc., verify separately).
