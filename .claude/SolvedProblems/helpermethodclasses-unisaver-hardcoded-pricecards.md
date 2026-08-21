# Gotcha: `HelperMethodClasses.UniSaver` looks generic, is hardcoded to `cwatis.pricecards`

**Where:** `CWATIS\HelperClasses\HelperMethodClasses.cs`, static methods `UniSaver(int ID, int UserId, string colname, object value)` and `UniSaver(List<int> ID, int UserId, string colname, object value)`.

## Symptom

The signature reads like a general-purpose "update one column on one row via `global_db.UniSaver`" helper — `ID`/`UserId`/`colname`/`value`, no schema or table parameter. Calling it for any table other than `cwatis.pricecards` compiles fine and runs with no error, but silently updates (or no-ops against) the wrong table, since the target schema/table is baked into the method body:

```csharp
var blocks = new Dictionary<string, (string, int, List<object>)>
{
    ["cwatis"] = ("pricecards", ID, Items)   // <- hardcoded, ignores the caller's actual entity
};
```

As of 2026-08-13, no call sites exist anywhere in the codebase (`grep -r "HelperMethodClasses.UniSaver"` — zero matches) — it's dead code left over from a single prior use case, not a maintained shared utility.

## Root cause

The method was written for one specific caller (price cards) and never parameterized when the schema/table turned out to be a hardcoded literal instead of an argument. Nothing about the public signature signals this — `colname`/`value` genuinely are generic, only the schema/table tuple isn't.

## Fix / convention to follow instead

`global_db.UniSaver` itself *is* generic (schema/table/id/column/value, driven entirely by the JSON `_Pars` blob — see `MC_GlobalForms.HelperClasses.JsonHelper.BuildJson`). Build the call inline in the owning repository instead of reusing this helper:

```csharp
private static async Task SaveFieldAsync(string schema, string table, int id, int userId, string colname, object value)
{
    dynamic item = new ExpandoObject();
    ((IDictionary<string, object>)item)[colname] = value;

    var blocks = new Dictionary<string, (string, int, List<object>)>
    {
        [schema] = (table, id, new List<object> { item }),
    };

    var dic = new Dictionary<string, object>
    {
        ["_Pars"] = JsonHelper.BuildJson(blocks),
        ["_UserId"] = userId,
    };

    await Task.Run(() => CallService.CallProcedureDT("global_db.UniSaver", dic));
}
```

Precedent: `CWATIS\Forms\Subdivisions\Repository\SubdivisionsRepository.cs` — `SaveFieldAsync(schema, table, id, userId, colname, value)`, used by `RemoveFromLocalAsync` (→ `"cwatis", "subdivisions"`). Note: this repository's *other* table (`refers.rfr_subdivisions`) turned out not to be safely editable via `UniSaver` at all — see `SolvedProblems/unisaver-schema-must-be-physical-not-procedure-namespace.md` for why, and what was used instead.

**Whatever `schema` string you pass must be the table's real physical schema** — see `SolvedProblems/unisaver-schema-must-be-physical-not-procedure-namespace.md` for a case where copying the schema from a sibling procedure call (`global_db.SomeProc`) instead of the table's actual schema (`refers`) produced a MySQL syntax error deep inside UniSaver's generated SQL.

## Checklist

- Before calling `HelperMethodClasses.UniSaver`, open it and check the hardcoded `"cwatis"`/`"pricecards"` literals — do not assume the signature's genericity extends to the schema/table.
- For a new repository needing a single-column `UniSaver` update, write a small private `SaveFieldAsync(schema, table, id, userId, colname, value)` in that repository (or generalize an existing one) rather than reusing the HelperClasses version.
- Don't infer the `schema` argument from a sibling procedure's name (e.g. `global_db.InsUpd_X` does **not** imply table `X` lives in `global_db`) — see the linked note above.
