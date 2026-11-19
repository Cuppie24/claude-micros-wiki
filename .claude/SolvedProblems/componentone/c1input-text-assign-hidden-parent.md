# C1Input: "Cannot assign text to readonly, disabled or hidden C1Input control" when a parent panel is collapsed

Found 2026-10-19 in CWATIS (`Forms/Partners/View/PartnersForm.cs`, `UpdateCardFields`).

## Symptom

Runtime, via the global error dialog:

```
Cannot assign text to readonly, disabled or hidden C1Input control when its Text property is not detached.
```

Only happens in some UI states. In PartnersForm it happened when the user clicked a grid row while the collapsible card panel (`C1SplitterPanel`, `Collapsible = true`) was collapsed. It works fine when the panel is open and the main tab is selected.

## Root cause

With the default `TextDetached = false`, the `.Text` setter on C1Input controls (`C1TextBox`, `C1DateEdit`, `C1DropDownControl` and derived lookups such as `SmartCounterpartyLookUpDropDown`) throws if the control is **ReadOnly, disabled or not Visible**.

- "Hidden" means WinForms `Control.Visible`, which is `false` whenever any parent is hidden. Collapsing a `C1SplitterPanel` sets its `Visible` to false (see [[c1splitterpanel-collapsed-is-visible-not-collapsed]]). A non-selected `TabPage` is also hidden.
- The existing workaround `ctrl.ReadOnly = false; ctrl.Text = x; ctrl.ReadOnly = true;` only handles one of the three conditions. That makes it look correct in normal testing.
- Assigning `.Value` (e.g. `C1DateEdit.Value`) is **not** checked. A permanently disabled `C1DateEdit` whose `Value` was set on every row change never threw. This was observed, not decompiled.

## Fix

Set `TextDetached = true` on the C1Input control (Designer property) and drop any ReadOnly toggling around `.Text`:

```csharp
// Designer
this.partnerMSubjField.TextDetached = true;

// code
partnerMSubjField.Text = partnerModel.MCode;   // no ReadOnly dance
```

## Don't apply it to a C1DateEdit filled through `Value` (2026-11-19)

Found in CWATIS (`WhMovementMainControl.dtPlannedDate`, a new date field that starts hidden). `TextDetached = true` was added "to be
safe" because the field starts hidden. In the running app, picking a date from the calendar did nothing visible ("on click nothing
happens"), and a date assigned through `Value` did not show either. Removing `TextDetached` fixed it. The neighbouring `dtExecDate`
had the same setup without the flag and always worked. Verified by testing in the app. Inferred mechanism: with the text detached,
the displayed text no longer follows `Value`.

Rule: `TextDetached = true` only on C1Input controls whose code assigns `.Text` (text boxes, lookups). A `C1DateEdit` that is set
and read through `Value` needs no protection, because assigning `Value` is not checked (see Root cause), so keep the default
`false` there, even when the field is hidden.

## Checklist

- If you see a `ReadOnly = false / Text = … / ReadOnly = true` sandwich on a C1Input control, it's a latent bug. Replace it with `TextDetached = true`.
- Reproduce by putting the control in a hidden container: a collapsed splitter panel, another tab, or a hidden panel. Making it read-only alone won't reproduce it.
- Plain WinForms `TextBox`/`RichTextBox` accept `Text` in any state, so this applies only to C1Input controls.
