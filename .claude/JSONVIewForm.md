# JSONViewForm (CWATIS)

`CWATIS.Forms.JSONForm.View.JSONViewForm` — a reusable dialog that shows an arbitrary JSON string two ways: as an expandable tree (key/value columns) or as pretty-printed raw text. Model in `Forms\JSONForm\Models\JsonTreeNodeModel.cs`. See `TreeFlexGridConventions.md` for the tree-grid mechanics it relies on.

**Naming note:** the class is `JSONViewForm`, but the files on disk are still named `JSONVIewForm.cs` / `.designer.cs` / `.resx` (old casing, capital "I") — a rename-in-place that didn't rename the files. Don't be thrown by the mismatch when searching.

## How it works

- **Tree view**: the JSON string is parsed with `Newtonsoft.Json.Linq.JToken.Parse` and walked recursively (`AddNode`) into a flat `List<JsonTreeNodeModel>` (`Id`, `ParentId`, `Name`, `Value`). The root token is **not** wrapped in a synthetic node — its top-level properties (object) or items (array) become root nodes directly (`ParentId = 0`), same as any deeper level; a bare scalar root becomes a single node named `"Value"`. Below the root, `JObject`/`JArray` become parent nodes (their properties/items become children), scalars become leaf nodes with `Value` set. That list is bound via `jsonTreeGrid.BuildTreeNew(nodes)`. Parsing through `JToken` (rather than manual `{`/`}` scanning) is what makes nested-object handling reliable.
- **Raw text view**: the same JSON, reformatted with `JToken.Parse(json).ToString(Formatting.Indented)` (falls back to the original string if parsing fails) and shown in a read-only `TextBox`.
- **Toggle**: a `C1ToolBar` with three commands — `showTreeCommand` ("Показать дерево"), `showRowTextCommand` ("Показать исходный текст"), `copyRowTextCommand` ("Скопировать исходный JSON") — the first two call `ShowTree()`/`ShowRawText()`, which swap `Visible` between `jsonTreeGrid` and `rawTextBox` (both `Dock = Fill` in the same content area, only one visible at a time).
- **Copy**: `copyRowTextCommand` puts the *original* `JSON` string (not the reformatted one) on the clipboard.
- **Columns**: `jsonTreeGrid` uses `AutoGenerateColumns = false` with `ColumnInfo` stored in `JSONVIewForm.resx` — `Name` ("Ключ") and `Value` ("Значение", explicitly `TextAlign:LeftCenter` since TreeFlexGrid right-aligns numeric-looking strings by default) are visible; `Id`/`ParentId` are hidden. Don't flip `AutoGenerateColumns` to `true` — see the gotcha documented in `TreeFlexGridConventions.md`.

## Usage — showing JSON from another form

```csharp
using CWATIS.Forms.JSONForm.View;

using var form = new JSONViewForm { JSON = someJsonString };
form.ShowDialog();
```

- Set the `JSON` property (plain string, any valid JSON — object, array, or scalar) *before* calling `ShowDialog()`; it's read in the form's `Load` handler.
- If `JSON` is empty/null or fails to parse, the tree simply renders empty and the raw text view falls back to showing the unparsed original string — no exception is thrown either way.
- The form is modal (`ShowDialog`) and not tied to any specific caller; it doesn't mutate the JSON it's given.

## Precedent usage

`ContractTypesForm.cs` opens it on double-click of the grid's `HasJson` column (`masterGrid_MouseDoubleClick`), passing that row's `ContractTypeData` — see "Handling Double-Click on a Specific Column" in `SmartGridConventions.md` for the click-handling pattern used there.
