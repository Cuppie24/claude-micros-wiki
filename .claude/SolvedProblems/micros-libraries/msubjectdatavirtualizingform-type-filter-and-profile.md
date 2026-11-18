# `MSubjectDataVirtualizingForm`: forcing the "Тип" filter overwrites the user's remembered choice

Found 2026-11-10 in CWATIS (warehouse goods owners form). Verified against MC_GlobalForms source 1.1.7.16
(`GlobalForms/Forms/Subjects/MSubjectDataVirtualization/MSubjectDataVirtualizingForm.cs`) and by disassembling the 1.1.7.24 DLL
(same `Load` / `SetSelectedSubjType` / `TYPE_SUBJCombo_SelectedValueChanged` sequence).

## Symptom

No error. After opening the counterparty picker from a screen that forces a type (`ModeEdit = false; SubjType = 0`), the same
picker opened from *other* screens starts on that forced type instead of the user's own last choice. Opening it with
`ShowSelect(text)` likewise resets the user's remembered type to "Все".

## Root cause

The profile manager passed to the constructor (`new MSubjectDataVirtualizingForm(Main.ProfileKeysValueManager)`) is used for
exactly one thing: remembering the "Тип" filter (profile key `_SubjTypeFilter`, value = combo `SelectedIndex`).

`Load` (named `MSubjectDataVirtualizinfgForm_Load`, typo in the library):
1. if the name filter is empty: `TYPE_SUBJCombo.SelectedIndex = <profile value>`, then `SetSelectedSubjType()` — which, when
   `!ModeEdit && SubjType > -1`, sets `SelectedValue = SubjType` and **disables** the combo;
   if the name filter has text (`ShowSelect(text)`): `SelectedIndex = 0` ("Все"), no forcing.
2. Every value change runs `TYPE_SUBJCombo_SelectedValueChanged` → refresh → `AddOrReplaceProfileKey("_SubjTypeFilter", SelectedIndex)`.

So forcing the type (or the "Все" from `ShowSelect`) is saved into the user's profile as if they had picked it.

## Fix

When the caller fixes the type itself (or searches by text), don't give the form the profile manager. With `null`,
`GetSelectedProfileValue` returns `""` and `AddOrReplaceProfileKey` is a no-op; the forcing still works (`SubjType` + `ModeEdit = false`).

```csharp
using (var form = new MSubjectDataVirtualizingForm((Profile.IKeysValueManager)null)) // cast required, see below
{
    form.ModeEdit = false;
    form.SubjType = 0;          // ЮЛ; the combo is set and disabled in Load
    form.MultiSelect = true;
    form.Owner = this;
    if (form.ShowDialog() != DialogResult.OK) return;
    ...
}
```

`new MSubjectDataVirtualizingForm()` does **not** compile: `CS0121` ambiguous call between the `(IKeysValueManager = null)` and the
`(ISubjMediator, IMapper<Subject, SubjDTO>, bool)` constructors (both all-optional). Pass the typed `null`.

## Checklist

- Picker type must be fixed? `ModeEdit = false` + `SubjType = <id>` (-1 = not forced; 0 ЮЛ, 1 ФЛ, 2 Ю/Ф, 3 SPC) **and** no profile manager.
- `ShowSelect(text)` always shows "Все" (no way to force a type there without changing the library).
- Precedent: `CWATIS/Features/Warehouse/GoodOwners/View/WhGoodOwnersForm.PickSubjects` / `ShowSubject`.
