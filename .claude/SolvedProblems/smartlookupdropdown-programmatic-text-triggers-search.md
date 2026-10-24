# SmartLookUpDropDown: setting Text from code starts a search, and SelectedRowData is not "the saved record"

Found 2026-10-24 in CWATIS (warehouse invoices, seller / buyer fields), library `LookUpDropDownControlLib` /
`ProductLookUpDropDownControlLib` 1.0.0.28 (`SmartCounterpartyLookUpDropDown` and every other `SmartLookUpDropDown` subclass).

## Symptom

- Showing a saved name in the field (`lookup.Text = name` or `lookup.Value = name`, e.g. when the selected row of a list changes) pops the
  dropdown open by itself a moment later, or runs a server search on every row change.
- Reading the id back on save via `lookup.ConvertTo<GlobalDbCounterPartyLookUpModel>(lookup.SelectedRowData).ID` gives the id of whatever
  record the search for that *name* returned — not necessarily the id that was stored in the document (names are not unique).

## Root cause (confirmed by scanning the IL of `SmartLookUpDropDown`)

- `InitializeComponent` subscribes `TextChanged` -> `HandleTextChangedAsync`: after a debounce it fetches data, sets `ModelCollection`, raises
  `DataLoaded` and calls `UpdateUI`, which opens the dropdown **unless `closeDropDownFlag` is true**. This happens for programmatic text too
  (only design mode is skipped).
- Nothing in the library ever sets `closeDropDownFlag` back to false — once you raise it, the dropdown stays closed for typing too until you reset it.
- `initializationTextFlag` is not read anywhere in that path — it does not suppress the search.
- `MSubjectShortForm` (the counterparty modal) sets both `RefName` **and** `ReferId` in its grid double-click handler, so the modal does give
  you the id directly (`ContractsForm` only uses `RefName` and lets the lookup re-find the record by name).

## Fix

Keep the id yourself instead of trusting `SelectedRowData`:

```csharp
private void SetTextSilently(string text)
{
    _box.closeDropDownFlag = true;           // search still runs, but the dropdown won't open
    WhHeaderControls.SetText(_box, text);    // lifts ReadOnly for the assignment
}

_box.KeyPress += (s, e) => { if (_isEditing) _box.closeDropDownFlag = false; }; // user typing: dropdown wanted again
_box.AfterOneRecordSelected += (s, e) =>
{
    var record = _box.ConvertTo<GlobalDbCounterPartyLookUpModel>(_box.SelectedRowData);
    if (record?.ID != null) { _id = record.ID; _confirmedText = _box.Text.Trim(); }
};
// modal: _id = form.ReferId; SetTextSilently(form.RefName);
// save: text empty -> null; text == _confirmedText -> _id; otherwise "typed but not picked" -> validation error
```

Precedent: `CWATIS/Features/Warehouse/Shared/View/WhCounterpartyPicker.cs`.

## Also: set `TextDetached = true` on a lookup whose Text is assigned from code

The `ContractsForm` designer block (copied as the template) has **no** `TextDetached`, i.e. `false`. With that, assigning `Text` throws
`System.Exception: Cannot assign text to readonly, disabled or hidden C1Input control when its Text property is not detached.`
(`C1.Win.C1Input.C1TextBox.set_Text`). Lifting `ReadOnly` around the assignment is not enough: a field on a **non-active tab page** is
hidden, and that alone triggers it (hit 2026-10-24 on the warehouse «Реквизиты» tab at form load). Precedent for `TextDetached = true`
on the same control with code-assigned Text: `Forms/Partners/View/PartnersForm` (`partnerMSubjField`).

## Checklist

- Lookup filled from code → `TextDetached = true` in the designer (the `ContractsForm` template lacks it).

- A lookup that shows stored data: store the id from the DB; never derive it from `SelectedRowData` after a programmatic text set.
- Raise `closeDropDownFlag` before any programmatic text, and reset it on user input.
- `LookUpColumnNamesToHide` is serialized to `.resx` by the designer; a control without a `.resx` must set it in code (counterparty lookups
  hide `ID`, `TIN_SUBJ`, `Аббрев`). Don't hand-write a collection initializer into `InitializeComponent` — the designer can't parse it.
