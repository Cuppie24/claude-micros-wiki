# `OurOrgsForm_New.OutId` is `m_oursubject.OurSubId`, not the organization's `m_subject` id

Found 2026-11-09 in CWATIS (warehouse goods owners form).

## Symptom

None at first — wrong-id bugs here are silent. Picking one of our organizations in `OurOrgsForm_New` (the `Refers` picker of
"our own organizations") and storing `form.OutId` into a column that references `global_db.m_subject.ID_M_SUBJ` either:
- fails the FK (no such subject), or — worse —
- silently points at an unrelated counterparty, because `OurSubId` values (1..~40) are also valid `ID_M_SUBJ` values of other subjects.

## Root cause

`refers.m_oursubject` has its own key `OurSubId` plus `ID_M_SUBJ` (the organization's `m_subject` row). On mars the two differ for
all 37 rows (e.g. `OurSubId 32 -> ID_M_SUBJ 25`). `OurOrgsForm_New.OutId` returns `OurSubId`.

**Misleading evidence** that points the other way: `ContractAddForm` / `ContractDSAddForm` assign `ContractModel.ContractSellerId = form.OutId`
in the picker handler, and `contracts.contract_SellerId` (FK -> `m_subject`) only ever holds `ID_M_SUBJ` values of our orgs. That is
because their save path later overwrites it: `ContractSellerId = seller.ID_M_SUBJ` (seller = `SellerLD.ConvertTo<OurOrgsModel>(...)`).
Checking the DB alone "proves" the wrong conclusion — follow the whole save path, not just the picker handler.

## Fix

Convert after picking, as `CalcImportPricesForm.BuyerDDFilter_ModalButtonClick` does:

```csharp
int ourSubId = form.OutId;                       // OurSubId
var dt = CallService.GetTable(DSSFormLib.StaticVariable.AddDBPrefix("m_oursubject"), where: $"OurSubId = {ourSubId}");
int subjectId = dt?.Rows.Count > 0 && dt.Rows[0]["ID_M_SUBJ"] != DBNull.Value ? Convert.ToInt32(dt.Rows[0]["ID_M_SUBJ"]) : 0;
```

`StaticVariable` lives in `DSSFormLib` (not `Refers`) — `CS0103` with only `using Refers;`.

### Newer Refers.V2 (after 1.1.3.78, change made 2026-11-10 in `refers/Refers/OurOrgsForm_New.cs`)

The form exposes the subject ids itself — no lookup needed once a consumer is on that version:
- `OutSubjIds` — `m_subject.ID_M_SUBJ` (grid column `ORGID`) of the picked orgs; `OutIds` — their `OurSubId`, same order.
- Rows are marked with **Insert** (Ctrl+A — all; TDBG's own marking, red rows). On double-click / Enter `OutIds`/`OutSubjIds`
  hold all marked rows, or the current row if none is marked. `OutId`/`OutName` are still the current row only.
  (1.1.3.79 briefly gated this behind a `MultiSelect` flag + Ctrl + click marking; both were removed afterwards — TDBG has no
  mouse marking of its own, Insert is the way rows are marked in this library.)
- Picking still closes with `DialogResult.Cancel` (pre-existing) — check `OutIds.Count > 0`, not `ShowDialog() == OK`.

## Checklist

- Any value from `OurOrgsForm_New.OutId` that goes into an `m_subject`-typed column must go through `m_oursubject.OurSubId -> ID_M_SUBJ`.
- When inferring an id's meaning from stored data, trace every write between the picker and the DB — a later overwrite invalidates the inference.
- Precedent: `CWATIS/Features/Warehouse/GoodOwners/Repositories/WhGoodOwnersRepository.GetOurOrgSubjectIdAsync`.
