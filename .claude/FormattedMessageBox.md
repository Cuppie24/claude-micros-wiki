# FormattedMessageBox (CWATIS)

`CWATIS.HelperClasses.FormattedMessageBox.Show(...)` — a drop-in alternative to `System.Windows.Forms.MessageBox.Show` for messages that need part of the text **bold**. The native WinForms `MessageBox` only renders plain text in one system font — no bold, no mixed formatting within the string — so this exists specifically to cover that gap.

## Usage

```csharp
using CWATIS.HelperClasses;

var result = FormattedMessageBox.Show(
    $"Текущий основной тип контракта:\n**{previousDefault.DisplayName}**\n\nСделать основным типом:\n**{model.DisplayName}**?",
    "Смена основного типа контракта",
    MessageBoxButtons.YesNo,
    MessageBoxIcon.Question);

if (result == DialogResult.Yes) { ... }
```

- Signature mirrors `MessageBox.Show(text, caption, buttons, icon)` exactly — same parameter order, same `MessageBoxButtons`/`MessageBoxIcon`/`DialogResult` enums — so it's a near-literal find/replace at call sites that need bold text.
- Mark bold spans with `**double asterisks**`, same convention as Markdown. Everything outside `**...**` renders at regular weight.
- Use `\n` for line breaks same as with the native `MessageBox` — the dialog auto-sizes its height to the number of `\n`-separated lines in the text (not real word-wrap measurement, since call sites here are always a handful of short explicit lines, not paragraphs).
- Supports `MessageBoxButtons.OK` / `OKCancel` / `YesNo` / `YesNoCancel` and `MessageBoxIcon.None` / `Information` / `Question` / `Warning` / `Error` (mapped to `SystemIcons`).

## How it works

- **Static facade**: `HelperClasses/FormattedMessageBox.cs` — a thin static class that constructs `Forms.FormattedMessageBoxDialog.View.FormattedMessageBoxForm`, calls `Configure(...)`, and returns `ShowDialog()`'s result. Same "HelperClasses static method opens a Form dialog" shape as `Main.RecInfoView(...)` opening `RecInfoForm`.
- **The dialog itself**: `Forms/FormattedMessageBoxDialog/View/FormattedMessageBoxForm.cs` (+ `.Designer.cs`, no `.resx` — no embedded resources needed). All controls (`pictureBoxIcon`, `richTextBoxMessage`, `buttonYes`/`buttonNo`/`buttonOK`/`buttonCancel`) are declared in the Designer file per the project's "controls live in `.Designer.cs`" rule; the code-behind only toggles their `Visible`/position/content at runtime, never creates new controls.
- **Bold rendering**: `SetMessage` splits the text on `**`, and for each segment sets `richTextBoxMessage.SelectionFont` to bold or regular before `AppendText`-ing it — the standard WinForms technique for mixed-formatting text (a `RichTextBox` is the only stock control that supports per-run formatting; it's styled to look like a plain label — `BorderStyle.None`, `ReadOnly`, `BackColor = SystemColors.Control`, no scrollbars).
- **Buttons**: all four button controls exist in the Designer at all times; `SetButtons` hides the ones not requested by the `MessageBoxButtons` value and right-aligns the visible ones against the message box's right edge, matching the native `MessageBox`'s button layout. `AcceptButton`/`CancelButton` are wired to whichever of Yes/OK and No/Cancel are actually visible.
- **Icon**: `SetIcon` maps `MessageBoxIcon` to `SystemIcons.*` and shows it in `pictureBoxIcon`; for `MessageBoxIcon.None` the icon is hidden and the message area is widened/shifted left to fill the gap.

## Naming gotcha — why the namespace isn't `CWATIS.Forms.FormattedMessageBox`

The View lives in `CWATIS.Forms.FormattedMessageBoxDialog.View`, not `CWATIS.Forms.FormattedMessageBox.View` as you might expect to mirror the static class name. This is deliberate: a call site inside any other `CWATIS.Forms.*` namespace (e.g. `CWATIS.Forms.ContractTemplate.Types.View`) resolves a bare `FormattedMessageBox.Show(...)` reference by first checking sibling namespaces under the shared `CWATIS.Forms` prefix — if a namespace segment literally named `FormattedMessageBox` existed there, the compiler prefers it over the `using CWATIS.HelperClasses;`-imported static class, and fails with `CS0234` ("Show" does not exist in that namespace). Confirmed by hitting exactly this error while building this feature. Keep the View namespace segment named differently from the static facade class to avoid it.

## Precedent usage

`ContractTypesForm.cs` (`masterGrid_StartEdit`) — the "change default contract type" swap-confirmation dialog, bolding the previous and new default type names each on their own line.
