# MSubjectShortForm: `SubjType` is the «Тип» list index, not `TYPE_SUBJ`

**Symptom:** `MSubjectShortForm` with `SubjType = 0` (meant: legal entities) opens on «Тип» = «Все»; `SubjType = 2` with the comment
"Физ лицо" shows ФЛ although `TYPE_SUBJ` of a person is 1.

**Cause:** `MSubjectShortForm_Load` (GlobalForms ≤ 1.1.8.6) did
`TYPE_SUBJCombo.SelectedIndex = SubjType != 0 ? SubjType ?? 0 + 1 : 0;` — `+` binds tighter than `??`, so it is
`SubjType ?? 1`: a non-zero `SubjType` is used as the index into `GlobalSource.SubjTypeAll`
(0 «Все», 1 ЮЛ, 2 ФЛ, 3 Ю/Ф, 4 SPC), and 0 maps to index 0 «Все». Callers adapted to it (CWATIS `ContractsForm`,
`LocalMSubjectsForm`: `SubjType = 2` + `IsSubjTypeEditable = false` for persons).

**Fix (1.1.8.7, commit `3e8f4f9`):** only the 0 case changed — `SubjType == 0 ? 1 : SubjType ?? 0`, clamped. Making it
"`SubjType + 1`" would have moved every locked `SubjType = 2` picker from ФЛ to Ю/Ф.

**How to apply:** for the short form pass the **list index** for non-zero types (2 = ФЛ, 4 = SPC); `0` = ЮЛ from 1.1.8.7.
`MSubjectDataVirtualizingForm` has its own `SubjType` handling (`SetSelectedSubjType`) — don't assume the same mapping there.
`CustomComboBox` (`TYPE_SUBJCombo`) has no `Items`; bound indexes by the source list.
