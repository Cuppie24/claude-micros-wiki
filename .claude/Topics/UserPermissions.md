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

Checked by `PermissionManager.CanShowForm()` **and** by `Initialize()` (both run `FormInitializer.InitializeForm`): empty
`Tag` or `GlobalConfig.Enabled = false` → allowed; `IsAdmin` → allowed; level Недоступен, the object missing, or any
error (bad JSON, DB failure) → denied. Two ways to enforce it:
- `form.ShowWithPermission()` (`TmplMDIForm` in GlobalForms, `TemplateFormMDI` in Refers): «Доступ запрещен», form not shown.
  Outside those base classes: `new PermissionManager(form).CanShowForm()` (CWATIS: `Main.CheckPermissionOnForm`).
- `Initialize()` in `Load`: «У вас нет доступа на просмотр этой формы!» and `BeginInvoke(Close)` — works even when callers
  use plain `Show()` (the rest of `Load` still runs before the close).

### 2. Toolbar commands — `C1Command.UserData`

```csharp
this.CreateCommand.UserData  = "{\"name\":\"wh_list_edit\"}";
this.DeleteCommand.UserData  = "{\"name\":\"wh_list_edit\"}";
this.ApproveCommand.UserData = "{\"name\":\"wh_list_edit\",\"canApprove\":true}";
```

What decides `Enabled` (`CommandLinkPermissionHandler`; the same for `Button` / `C1Button` with a `Tag`):
- no `canCheck` / `canApprove` → level ≥ Редактирование;
- `canApprove: true` → only the user's approve flag (`Permission.CanApprove`), the level is ignored; `canCheck: true` →
  only `CanCheck`. Use them with an object whose `Right_Object_CheckAndApproveEnable = 1`;
- `IsAdmin` is **not** considered for commands.

A denied command is **locked**: the library hooks `EnabledChanged` and sets `Enabled = false` again, so the form's own
`cmd.Enabled = true` (selection/state refresh, edit-mode toggles) can't re-enable it. A permitted command gets
`Enabled = true` — it overwrites the state the form set earlier, so re-run the form's own command refresh right after
`Initialize()`. Don't bind commands that are disabled because they're not implemented: binding would switch them on.

Other controls with a JSON `Tag` are handled too: `C1DropDownControl` has its own handler; anything else gets
`ReadOnly` (or, if it has none, `Enabled`) = level ≥ Редактирование.

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
synchronously, and calling it from `Activated` hung `GoodsTree4PriceListForm`. A form with `UserData` on commands that only
uses `ShowWithPermission()` has **unrestricted** commands (CWATIS `ContractTypesForm`'s Утвердить, 2026-11-20).

How it walks the form: every control recursively (UserControls included, so commands in a UserControl's toolbar are
reached from the form's `Initialize()`); a `C1ToolBar` only if it is `Enabled` at that moment (a toolbar inside a
disabled panel is skipped), plus nested `C1CommandMenu` links. Each right is fetched once per `Initialize()` call.

When the form's own code sets column `AllowEditing` (in `Load` after `Initialize()`, or by document status), the grid `Tag`
gets overwritten — check the right in code instead and gate the grid there:

```csharp
var permission = new UserPermissionsControl.Services.PermissionService(new Legacy.Services.Interfaces.PermissionSubjects
{
    GlobalSubjectId = UserPermissionsControl.GlobalConfig.GlobalSubjectId,
    LocalSubjectId  = UserPermissionsControl.GlobalConfig.LocalSubjectId
}).GetPermission("wh_documents");                     // null = object missing; throws on a DB error
bool canEdit = permission != null && permission.Value >= Legacy.Services.Interfaces.PermissionValue.Edit;
```

(`Permission` has `Value`, `CanCheck`, `CanApprove`, `IsAdmin`. CWATIS wraps this as `Features/Warehouse/Shared/View/WhRights.cs`.)

## Traps

- **A missing right object behaves differently per control.** Toolbar commands default to Недоступен (disabled for
  everyone); grid columns are skipped and stay as the designer left them (fails open). Keep `AllowEditing = false` in the
  designer on guarded columns so a missing right means read-only.
- **Name typos are silent.** A JSON name with no matching `Right_Object_Code` raises no error. Example (CWATIS, 2026-11):
  the warehouse forms used `wh_list` / `wh_doc_types` while the DB objects are `wh_list_edit` / `wh_doc_types_edit`, so
  Create/Delete/Approve were disabled for every user. After adding a binding, compare every name against the table.
- **A non-JSON `Tag` on any control closes the form.** `Initialize()` parses every control's `Tag`: plain text
  (`JsonReaderException`) falls into its catch-all → «Произошла ошибка при инициализации формы!» and the form closes. A
  JSON value of the wrong shape (e.g. a number) instead disables that control. Before adding `Initialize()` to a form,
  grep it for `.Tag =` used for other purposes.
- **Grid columns are not enforced after `Initialize()`** (commands are, see above). Later `AllowEditing = true`, rebuilt
  columns, cells written from code or SmartGrid's Ctrl+V `Paste()` are not checked — re-check the right in the form's own
  edit gate or save.
- **A new object must exist on every server before release.** A form `Tag` naming an object missing on that server
  denies access to everyone (dev objects are often created on Mars only).
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
- Decompiled `UserPermissionsControl` 1.1.0.6 (`ilspycmd`, 2026-11-20): form `Tag` check, command rules incl.
  `canApprove` / `IsAdmin`, the `EnabledChanged` lock, control walk, non-JSON `Tag` behaviour, `PermissionService` API.
- **Not verified:** how the DB side (`CallService.GetPermission` in `WebServiceClient`) picks the local schema or
  `global_db` for a name.
