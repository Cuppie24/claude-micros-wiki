# VS designer silently reverts edits made to a form's `.resx` from outside while the form is open in Visual Studio

Found in CWATIS, 2026-10-15 (`WhInventoryReportsForm`, grid `gridLines`).

## Symptom

- An edit to `<data name="gridLines.ColumnInfo">` in the form's `.resx` (renaming four column `Name:"…"` values) was verified on disk right after writing,
  and **was gone again later** — the old names were back. It happened **twice** in a row.
- Downstream behaviour that depended on the new names broke with no error and no build failure: the grid columns bound to the old model
  properties, and an `OwnerDrawCell` handler that matches columns by `Cols[e.Col].Name` matched nothing ("zeros show again" / "still zeros").
- `git diff --stat` on the `.resx` showed only the one-line change right after the edit, and a different state later.

## Root cause (inferred, not instrumented)

Visual Studio was running (`devenv`) with the form's designer/code tabs open. The `.resx` and `.Designer.cs` are owned by the designer: when it
saves (build/run saves modified documents, or the user touches the form) it re-serialises `ColumnInfo`, `Headers` etc. from **its in-memory copy**,
overwriting whatever was changed on disk behind its back. Evidence: reverts only while VS had the form open, the names reverted to exactly the
pre-edit values, and the same file was being edited in the designer (header captions) between the attempts. The overwrite itself was not captured —
treat the mechanism as the leading explanation, not a proven one.

The misleading part: the symptom appears far from the cause (a draw handler "doesn't work"), and the code change that looks guilty is fine.

## Fix / how to do it safely

1. Before editing a form's `.resx` / `.Designer.cs` from outside VS, check whether VS has it open (`Get-Process devenv` shows the window title =
   the active document). Ask the user to close that form's designer **and** code tabs (or reload from disk) first.
2. After the edit, re-read the file and verify the change survived the next build/run — don't assume. `git diff --stat <resx>` should show only
   the intended lines.
3. Prefer having the user rename a column in the designer's Properties grid (`ColumnInfo` editor) when the designer is open — it is the owner of that string.
4. If a rename must survive designer re-saves regardless, the durable place is the designer itself, not an external edit.

## Companion gotcha: `sed -i` can strip CRLF from a `.resx`

Editing the `.resx` with `sed -i` in Git Bash turned all 167 `\r\n` into `\n` (confirmed by counting `\r` before/after and by file size) — a whole-file
diff in git for a four-name change. A follow-up `sed 's/$/\r/'` then appended one stray `\r` to the last line (no trailing newline).
Edit such files at byte level instead, and assert the line-ending counts are unchanged:

```python
b = open(p, 'rb').read()
crlf = b.count(b'\r\n')
assert b.count(old) == 1
b = b.replace(old, new)           # old/new are bytes: b'Name:"A"' -> b'Name:"B"'
open(p, 'wb').write(b)
assert crlf == b.count(b'\r\n') and b[:3] == b'\xef\xbb\xbf'   # CRLF count and UTF-8 BOM intact
```

## Checklist / rule

- External edits to designer-owned files are only trustworthy if VS has them closed; verify after the next build.
- "Handler / binding stopped working after I renamed a column" → first re-read the `ColumnInfo` names on disk; don't start by debugging the handler.
- Never `sed -i` a CRLF resx; check `\r` counts before and after.
