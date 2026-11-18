# Git text merge silently corrupts a form's `.resx` (duplicate + missing `<data name=...>` keys)

Found 2026-10-07 in CWATIS (`WhInventoryActsForm.resx`), after a `pull merge` of two branches that both touched the same form.

## Symptom

- Build is green, merge had no conflict markers — but at runtime/designer a SmartGrid lost its columns/headers/styling (cells empty, default columns), and/or my earlier `ColumnInfo` edit (e.g. a column's `AllowEditing`) was gone.
- In the merged `.resx`: the same key appears twice (`gridLines.ColumnInfo`, `gridLines.StyleInfo`) and `gridActs.ColumnInfo`/`gridActs.StyleInfo` are missing. The VS designer then "heals" the file by regenerating the missing grid's resources with defaults (a ~300-char `ColumnInfo`), which hides the damage further.

## Root cause

The designer re-orders `<data>` blocks (grid resource order flips between saves), so a line-based 3-way merge matches blocks by position and pairs one grid's entries with the other grid's. Git sees no conflict.

## Fix

Don't merge the `.resx` by hand-picking hunks. Take the other side's whole file (`git show <rev>:path`), then re-apply only your own edits (e.g. regex-replace `AllowEditing:False;` for the specific columns). Check keys: every `<data name="grid*">` appears exactly once (`grep -c`), and compare per-key content against both parents with a small script (parse `<data name=...><value>...`).

## Checklist

After any merge touching a `.Designer.cs`/`.resx` pair: (1) `grep '<data name=' file.resx | sort | uniq -d` must be empty; (2) each grid has its `ColumnInfo`/`StyleInfo`/`SelectedRows`; (3) re-check hand edits made to `ColumnInfo` (see `../../Topics/SmartGridConventions.md`) survived.
