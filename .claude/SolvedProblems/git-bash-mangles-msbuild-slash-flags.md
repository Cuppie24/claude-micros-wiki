# Git Bash mangles Windows-style `/flag` arguments (MSBuild, and other Windows CLI tools)

**Symptom** — Running MSBuild (or any Windows-native CLI that uses `/flag` style switches, e.g. `/p:Configuration=Release /nologo /v:minimal`) via the Bash tool fails with an MSBuild-side argument-parsing error, e.g.:

```
MSBUILD : error MSB1008: можно указать только один проект.
    Полная командная строка: '"...\MSBuild.exe" MC_GlobalForms.sln p:Configuration=Release t:Build p:GenerateLicenseFile=false "C:/Program Files/Git/nologo" v:minimal'
```

Note the mangled command line: `/p:Configuration=Release` lost its leading slash and became bare `p:Configuration=Release`, and `/nologo` was rewritten to `C:/Program Files/Git/nologo`. The error message (`MSB1008: only one project can be specified`) points at MSBuild's argument parser, not at the shell — nothing in the message suggests the shell rewrote the arguments before MSBuild ever saw them.

**Root cause** — Git Bash (MSYS2/MSYS) auto-converts any argument that looks like a POSIX absolute path (`/xxx`) into a Windows path, because MSYS assumes `/xxx` means "path rooted at the MSYS install dir" and rewrites it (`/nologo` → `<git-install-dir>/nologo` → further normalized to a Windows-style path). Windows-style single-dash-slash CLI switches (`/p:...`, `/t:...`, `/nologo`, `/v:...`) collide with this heuristic because they're syntactically indistinguishable from a POSIX path. This mangling happens in the Bash tool (a real Git Bash / POSIX sh), not in PowerShell — PowerShell has no such path-conversion layer for arguments passed to native executables.

**Fix** — Invoke MSBuild (or any other tool using `/flag` syntax: `robocopy`, older Windows SDK tools, etc.) via the **PowerShell tool**, not the Bash tool:

```powershell
& "C:\Program Files\Microsoft Visual Studio\18\Community\MSBuild\Current\Bin\MSBuild.exe" MC_GlobalForms.sln /p:Configuration=Release /t:Build /p:GenerateLicenseFile=false /nologo /v:minimal
```

This runs unmangled — PowerShell passes the arguments through to the native exe as-is.

If Bash must be used for some reason, set `MSYS_NO_PATHCONV=1` (or `MSYS2_ARG_CONV_EXCL="*"`) in front of the command to disable the auto-conversion, but preferring the PowerShell tool is simpler and less error-prone.

**Checklist / rule**
- Any CLI tool that takes `/flag`-style switches (MSBuild, and most legacy Windows dev tools) → use the PowerShell tool, not Bash.
- Tools that take `--flag` or `-flag` style switches (npm, git, most POSIX-style CLIs) are unaffected and work fine in either shell.
- If a Windows CLI tool's error message looks like it's complaining about a malformed/duplicated argument that you're sure you typed correctly, and you ran it via the Bash tool, suspect MSYS path-mangling before doubting your own command line — check the "Полная командная строка" / "full command line" the tool echoes back (if it does) for suspicious absolute-path substitutions of what should've been a bare flag.
- Separately: don't assume a documented build command's binary path (e.g. a CLAUDE.md-documented `...\2022\Community\...` MSBuild path) is still accurate — VS installs get upgraded (e.g. to `...\18\Community\...`) and the old path silently becomes "No such file or directory". Locate the real path (`find`/`Get-ChildItem` for `MSBuild.exe`) if the documented one 404s, rather than assuming the tool is missing.

Found in: MC_GlobalForms (globalforms) repo, 2026-08-14, while verifying a build after a schema-driven code cleanup (removing dead `MCode` field references from the `MSubDivisions` module).
