# Git Bash `sed -i` rewrites the whole file with LF line endings — even when nothing matched

Found 2026-10-24 in CWATIS, Claude Code Bash tool (Git Bash / MSYS `sed`) on Windows.

## Symptom

After `sed -i 's/.../.../' file` on a CRLF file (`.csproj`, `.Designer.cs`, `.md`):

- `git diff` warns `LF will be replaced by CRLF the next time Git touches it` for the file;
- `file` no longer reports "with CRLF line terminators"; counting shows 0 CRLF and N bare LF lines.

It happened to `CWATIS.csproj` with a pattern that **did not match at all** — the content was unchanged, but all 2596 lines lost their CR.

## Root cause

MSYS `sed -i` reads the file in text mode (CR stripped) and writes the result back with LF only, for every line, regardless of
whether any substitution happened. With `core.autocrlf=true` git hides it in the diff, so it goes unnoticed; Visual Studio and tools that
compare raw bytes see a whole-file change.

## Fix

- Prefer the Edit tool (preserves line endings) for source edits; for scripted replacements use PowerShell
  `[IO.File]::ReadAllText` / `WriteAllText` (keep the BOM the file had).
- If it already happened, restore CRLF:

```powershell
$t = [IO.File]::ReadAllText($p); $t = [regex]::Replace($t, "(?<!`r)`n", "`r`n")
[IO.File]::WriteAllText($p, $t, (New-Object Text.UTF8Encoding($hadBom)))
```

## Checklist

- Never `sed -i` a Windows (CRLF) file from Git Bash; if you did, check `CRLF` vs bare-`LF` counts afterwards.
- Related: [git-bash-mangles-msbuild-slash-flags](git-bash-mangles-msbuild-slash-flags.md) — another Git Bash-on-Windows trap.
