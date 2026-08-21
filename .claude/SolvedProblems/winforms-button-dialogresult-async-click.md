# WinForms Button.DialogResult closes a modal before an async Click handler's awaited work finishes

**Date:** 2026-07-28 · **Project:** CWATIS · **Component:** System.Windows.Forms (Button.DialogResult on a form shown via ShowDialog)

## Symptom

A modal form's "OK" button has `DialogResult = DialogResult.OK` set in the Designer (the normal way to make a button close a dialog). The Click handler is changed to `async void` to do awaited work first (validation, a repository call, etc.) before the form should actually close. At runtime, the dialog closes immediately with `DialogResult.OK` as soon as the button is clicked — the awaited work either hasn't started or is still running, and any validation/error message the handler tries to show afterward either never appears or appears after the caller has already moved on treating the dialog as successfully confirmed.

No exception, no error message — the bug is silent, wrong-looking-correct behavior, which is what makes it easy to ship.

## Root cause

A `Button` with `DialogResult` set to anything other than `None` auto-closes its parent form (when shown via `ShowDialog()`) as soon as the **synchronous portion** of its `Click` event handler returns control to the message loop. For an `async void` handler, "returns" happens at the first `await` of an incomplete task — not when the whole method finishes. The form closes (and `ShowDialog()` returns with that `DialogResult` on the caller's side) right there, before any code after that `await` ever runs.

This is a general WinForms/async interaction, not specific to any one control — it will bite any Designer-configured `Button.DialogResult` paired with an `async void` Click handler that awaits something before deciding whether to actually succeed.

## Fix

1. In the Designer, do **not** set `DialogResult` on the button (leave it `None`, or delete the assignment if the Designer generated one before the handler became async).
2. Only set `this.DialogResult = DialogResult.OK;` (optionally followed by `this.Close();`) programmatically, at the exact point after the awaited work has actually succeeded — e.g. inside a method the presenter calls only on success.

```csharp
// Designer.cs - do NOT do this if the Click handler is async and can fail/validate first:
// this._okButton.DialogResult = System.Windows.Forms.DialogResult.OK;

// code-behind
private async void _okButton_Click(object sender, EventArgs e)
{
    _okButton.Enabled = false;
    if (OnCreate != null)
        await OnCreate.Invoke();   // may call ShowValidationError/ShowError and return without closing
}

public void CloseSuccessfully()   // called by the presenter only after everything succeeded
{
    DialogResult = DialogResult.OK;
    Close();
}
```

Precedent: `CWATIS/Forms/Contracts/BkLink/View/BkCntrCreateForm.cs` + `.Designer.cs` (create-and-link-to-bookkeep flow — validation and two sequential repository calls must all succeed before the dialog is allowed to close).

## Checklist / rule

- Any time a Click handler on a Designer-`DialogResult`-bound button becomes `async void` (or is about to), remove that Designer `DialogResult` assignment in the same change.
- Set `this.DialogResult` in code, manually, only at the success point — never rely on the button's own property once the handler does awaited work.
- This applies equally to `Form.AcceptButton`/`CancelButton` wiring — the button itself is what triggers the premature close, regardless of which property pointed at it.
