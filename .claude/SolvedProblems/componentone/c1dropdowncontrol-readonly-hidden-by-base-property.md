# C1DropDownControl / C1DateEdit / C1ComboBox: `ReadOnly = false` through a `C1TextBox` variable silently does nothing — modal and drop-down buttons stay dead

Found in CWATIS, 2026-10-19 (inventory report header tab, `WhHeaderControls.SetReadOnly`).

## Symptom

- In "edit mode" a header field built from a `C1DropDownControl` (a reference box with a modal button), a `C1DateEdit` or a `C1ComboBox` stays locked:
  clicking its **modal button / drop-down button does nothing** (no `ModalButtonClick`, no calendar, no list) and the value can't be changed.
  Plain `C1TextBox` fields on the same tab unlock fine.
- No exception, nothing in the error log. The code that unlocks the fields looks correct and *does* run.
- A trace taken right after the unlock shows the split:
  `ProducerRB.ReadOnly=True, dtDate.ReadOnly=True, ReportWarehouseDD.ReadOnly=True, tbNumber.ReadOnly=False`.

## Root cause

`C1DropDownControl` declares **its own non-virtual `ReadOnly` property**, which hides `TextBoxBase.ReadOnly` (`C1DateEdit` and `C1ComboBox` inherit that one).
Confirmed from the assembly metadata of `C1.Win.C1Input` 4.8.20231.596 (`typeof(C1DropDownControl).GetProperty("ReadOnly", DeclaredOnly)` → declared there,
`GetGetMethod().IsVirtual == false`; `C1TextBox` itself does **not** declare it).

A helper such as

```csharp
public static void SetReadOnly(C1TextBox box, bool readOnly) => box.ReadOnly = readOnly;   // WRONG for dropdown-based controls
```

compiles `box.ReadOnly` against `TextBoxBase.ReadOnly` (static type `C1TextBox`), so for a drop-down control it writes the *base* flag. The buttons and the
editing behaviour of the drop-down control look at its **own** `ReadOnly`, which was never touched. The `ReadOnlyChanged` event still fires (it is a virtual
`OnReadOnlyChanged` chain), which makes the call look like it worked.

## Fix

Branch on the real type, and use it both for writing and for the "remember the old value, toggle, restore" pattern:

```csharp
private static bool GetReadOnly(C1TextBox box) => box is C1DropDownControl d ? d.ReadOnly : box.ReadOnly;

private static void WriteReadOnly(C1TextBox box, bool readOnly)
{
    if (box is C1DropDownControl d) d.ReadOnly = readOnly;
    else box.ReadOnly = readOnly;
}
```

(or just assign on the concrete variable — `ProducerRB.ReadOnly = x;` — never through a `C1TextBox`-typed parameter/array element).

## Checklist / rule

- Never set or read `ReadOnly` of a `C1DropDownControl`-derived control (`C1DateEdit`, `C1ComboBox`, reference boxes) through a `C1TextBox`/`TextBoxBase` reference.
  Watch for helpers taking `C1TextBox` and for arrays like `new C1TextBox[] { dateEdit, combo, ... }` iterated with a setter.
- Fingerprint: "works for text fields, not for date/combo/reference boxes, in the same loop", buttons inert, no errors.
- Diagnose by reading the property back right after setting it, and by hooking `ReadOnlyChanged` with `Environment.StackTrace` written to a file
  (a throwaway `owner_trace.txt` next to the exe). A bare-harness test of C1 controls from PowerShell hangs on the ComponentOne license dialog — don't bother.
- Same family as `c1input-text-assign-hidden-parent.md` (C1Input `Text`/`ReadOnly` semantics are not the plain WinForms ones).
