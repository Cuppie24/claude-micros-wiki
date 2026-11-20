# User permissions: binding form controls to right objects (`UserPermissionsControl`)

How a WinForms form, its toolbar commands and its grid columns are tied to rights. Library: NuGet
`UserPermissionsControl` (1.1.0.6 at the time of writing; the editor UI is `UserPermissionsUI.V2`). Source: the
`userpermissionscontrol` repo, `master`, `PermissionHandlers/` — not cloned under `projects/`, consumers only have the DLL.

Each control carries a small JSON string with a right-object **name**; the library looks that name up in
`Right_Object_Code` of `right_objects` and applies the current user's right level.

## Tables (one set per schema)

Each application schema (e.g. `cwatis`) and `global_db` has its own copy: `right_objects`, `right_values`,
`right_subjects`, `role_rights` (CWATIS also has a legacy `userrights`).

`right_objects` — the tree of right objects:

| Column | Meaning |
|---|---|
| `Right_Object_Id` | PK |
| `Right_Object_ParentId` / `Right_Object_ParentIds` | parent (0 = root) / id chain `"179,180,182"` |
| `Right_Object_IsGroup` | group flag |
| `Right_Object_Code` | **the name used in the JSON** (unique) |
| `Right_Object_PathCode` | display path `"Склады / Редактирование складских справочников / …"` |
| `Right_Object_Name` | display name (unique) |
| `Right_Object_CheckAndApproveEnable` | 1 = the object can be checked/approved (pairs with `canApprove` below) |

`right_values` — the levels:

| Value | Name | Grid column effect |
|---|---|---|
| 0 | Недоступен | `Visible = false` |
| 10 | Просмотр | `Visible = true`, `AllowEditing = false` |
| 20 | Выбор | `Visible = true`, `AllowEditing = false` |
| 30 | Редактирование | `Visible = true`, `AllowEditing = true` |

The JSON name matches `Right_Object_Code` **case-insensitively** (`"Pl_Lines"` in `GoodsTree4PriceListForm` finds
`PL_Lines`) — most likely the column's `_ci` collation, not the library; don't rely on it, copy the code exactly.

To see a schema's objects (read-only):

```sql
SELECT Right_Object_Id, Right_Object_ParentId, Right_Object_Code, Right_Object_Name,
       Right_Object_CheckAndApproveEnable
FROM cwatis.right_objects ORDER BY Right_Object_ParentIds;
```

New objects are often created on the dev server (Mars) first — check the server you actually connected to
(`SELECT @@hostname`) before concluding an object doesn't exist.

## Startup (once per application)

The client configures the library before any form opens (CWATIS: `HelperClasses/Main.cs`, `InitializeGlobalPermissions`):

```csharp
UserPermissionsUI.GlobalConfig.LocalDataBase  = DBName;        // e.g. "cwatis"
UserPermissionsUI.GlobalConfig.GlobalDataBase = "global_db";

UserPermissionsControl.GlobalConfig.Enabled        = true;
UserPermissionsControl.GlobalConfig.LocalDataBase  = DBName;
UserPermissionsControl.GlobalConfig.GlobalDataBase = "global_db";
UserPermissionsControl.GlobalConfig.LocalSubjectId  = (int)CallService.GetValueFromFunc($"{DBName}.GetSubjectId",   new Dictionary<string, object> { { "_UserId", UInfo.UserId } });
UserPermissionsControl.GlobalConfig.GlobalSubjectId = (int)CallService.GetValueFromFunc("global_db.GetSubjectId", new Dictionary<string, object> { { "_UserId", UInfo.UserId } });
```

How the library decides whether a given name is looked up in the local schema or in `global_db` was not checked.

## Binding controls

### 1. The whole form — form `Tag`

```csharp
this.Tag = "{\"name\":\"wh_list_edit\"}";
```

Checked by `PermissionManager.CanShowForm()`. Open forms with `form.ShowWithPermission()` (`TmplMDIForm` in GlobalForms,
`TemplateFormMDI` in Refers): it shows «Доступ запрещен» and closes the form when the right is missing, otherwise
`Show()`. Outside those base classes: `new PermissionManager(form).CanShowForm()` (CWATIS wraps it as
`Main.CheckPermissionOnForm`).

### 2. Toolbar commands — `C1Command.UserData`

```csharp
this.CreateCommand.UserData  = "{\"name\":\"wh_list_edit\"}";
this.DeleteCommand.UserData  = "{\"name\":\"wh_list_edit\"}";
this.ApproveCommand.UserData = "{\"name\":\"wh_list_edit\",\"canApprove\":true}";
```

`canApprove` marks an approve action; use it with an object whose `Right_Object_CheckAndApproveEnable = 1`. What exactly
the library checks for it was not read in the source.

### 3. Grid columns — grid `Tag`

Works for any `C1FlexGrid`, so SmartGrid too (`C1FlexGridPermissionHandler`). A list of right objects, each naming the
columns it controls by column **`Name`** (not caption):

```csharp
grid.Tag = "{\"Tags\":[{\"Name\":\"PriceEdit\",\"Columns\":[\"Price4Disc\",\"BaseRtlPrice\"]},"
         + "{\"Name\":\"QntEdit\",\"Columns\":[\"FreeDepQNT\",\"BkBlnQnt\"]}]}";
```

Effect per level: see the `right_values` table above. Details and the edit-event traps: [SmartGridConventions.md](SmartGridConventions.md),
section "Per-column edit permissions".

### 4. Apply it — `Initialize()` in `Load`

```csharp
private async void MyForm_Load(object sender, EventArgs e)
{
    new UserPermissionsControl.PermissionManager(this).Initialize();   // commands + grid columns
    ...
}
```

`ShowWithPermission()` checks **only** the form `Tag`. Commands and grid columns are applied only by `Initialize()`,
which the form must call itself — in `Load`, after the grid columns exist, and **once**: it re-queries every right
synchronously, and calling it from `Activated` hung `GoodsTree4PriceListForm`.

## Traps

- **A missing right object behaves differently per control.** Toolbar commands default to Недоступен (disabled for
  everyone); grid columns are skipped and stay as the designer left them (fails open). Keep `AllowEditing = false` in the
  designer on guarded columns so a missing right means read-only.
- **Name typos are silent.** A JSON name with no matching `Right_Object_Code` raises no error. Example (CWATIS, 2026-11):
  the warehouse forms used `wh_list` / `wh_doc_types` while the DB objects are `wh_list_edit` / `wh_doc_types_edit`, so
  Create/Delete/Approve were disabled for every user. After adding a binding, compare every name against the table.
- **Not enforced after `Initialize()`.** Later `AllowEditing = true`, rebuilt columns, cells written from code or
  SmartGrid's Ctrl+V `Paste()` are not checked — re-check the right in the form's own edit gate or save.
- **It can un-cancel your `StartEdit`.** Its handler sets `e.Cancel = false` on Edit-permitted columns after the
  Designer's handlers run; put business-rule cancels in `BeforeEdit`.
- **It un-hides columns.** Every listed Edit/View column gets `Visible = true`.

## How this was verified

- Read in source: `TmplMDIForm.ShowWithPermission` (GlobalForms) only calls `CanShowForm()`.
- Seen in working forms: form `Tag` (`CalcImportPricesForm`, `TgMembersForm`, `GoodsTree4PriceListForm`), command
  `UserData` and grid `Tag` (CWATIS `WhWarehousesForm`, `WhDocTypesForm`), `Initialize()` in `Load`.
- Checked in the DB (Mars, 2026-11-20): table schemas, `right_values`, name ↔ `Right_Object_Code` matching.
- From the `userpermissionscontrol` source and DLL (2026-11-17, see SmartGridConventions): level → column effect,
  command default on a missing right, `StartEdit` override, `Activated` hang.
- **Not verified:** that `CanShowForm()` reads the form `Tag` (only seen working), what `canApprove` checks, how local vs
  global schema is chosen.
