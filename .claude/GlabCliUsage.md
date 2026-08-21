# Using the GitLab CLI (glab)

Practical notes on installing, authenticating, and invoking `glab` for the internal GitLab (`gitlab.micros.uz`), gathered during first-time setup on 2026-08-18.

## Content language: issues/work items go in Russian

**When creating a GitLab issue/work item (`glab issue create`, etc.), write the title and description in Russian**, even though this instructions file itself stays in English. The team's own codebase comments, UI text, and prior issues are Russian — match that audience. This file (and other `~/.claude/` knowledge-base files) is written for Claude to read, so it stays English; content that ships to GitLab is for the team, so it goes in Russian.

## Priority labels

`microsdvl/smartgrid` has three plain (non-scoped) priority labels: **`High priority`**, **`Medium priority`**, **`Low priority`** (exact casing/spacing, lowercase "priority", no `::` — see "Scoped labels" below for why not scoped). **When creating a work item there, apply the appropriate one** via `--label "High priority"` (etc.) on `glab issue create`/`glab issue update` — don't leave a new work item unlabeled. Judge priority roughly as: High = blocking or crash-causing, no workaround yet; Medium = confirmed real defect with a working deployed workaround (e.g. the footer/last-row crash issue — the bug is real but CWATIS already ships fine without the affected feature); Low = cosmetic/minor. Other repos may have their own label sets — check with `glab label list --repo <repo>` before assuming these three exist elsewhere.

### Scoped labels (`key::value`) render as a single flat pill here, not split two-tone

GitLab's scoped-label visual styling (the two-tone `scope | value` split, e.g. `devops | agent foundations`) is a **Premium-tier feature** — confirmed via a GitLab forum thread with the identical symptom, and consistent with `enterprise: false` in this instance's `glab api version` output. On Community Edition (which `gitlab.micros.uz` is running), a label named e.g. `Priority::Critical` saves and applies fine, but always displays as one solid-colored pill with the literal `::` in the text — never the split styling, regardless of GitLab version (confirmed current on `19.2.2`, so it's not a "too old" issue). This is why the priority labels above are plain names instead of a `Priority::` scope — scoping them would gain nothing here and just add literal `::` clutter to the label text.

## Installation

`winget install GLab.GLab` — the package ID is `GLab.GLab`, not the "obvious" `GitLab.GLab` (that ID doesn't exist; `winget` returns "No package found matching input criteria"). If unsure of the exact ID, check first: `winget search glab`.

Installs to `%LOCALAPPDATA%\Programs\GLab\glab.exe`.

## PATH doesn't pick it up immediately after install

`winget` does add the folder to the persistent User `PATH` (verify with `[Environment]::GetEnvironmentVariable('Path', 'User')`), but already-running terminal hosts (Windows Terminal, VS Code, etc.) hold a stale environment snapshot and won't see `glab` until the host process itself restarts — a new tab or window inside the same host isn't enough.

Right after install, it's more reliable to invoke by full path rather than rely on `PATH`:

```powershell
& "C:\Users\<user>\AppData\Local\Programs\GLab\glab.exe" ...
```

Quick way to force an environment refresh without logging the user off — restart `explorer.exe`:

```powershell
Stop-Process -Name explorer -Force; Start-Process explorer
```

Won't help if the terminal is spawned from inside another already-running app (VS Code, etc.) — that app itself needs to fully quit and relaunch.

## Authenticating against a self-hosted instance

`glab auth login --hostname gitlab.micros.uz` (not the default `gitlab.com`!) launches an interactive wizard with several prompts that are easy to get wrong:

- **"API hostname"** — only for when the API lives on a different host than GitLab itself. Leave blank (just press Enter) if it's the same host.
- **"SSH hostname"** — same idea, leave blank unless it genuinely differs.
- Choose **Token**, not Web/Device.
- The token needs at minimum the `api` and `write_repository` scopes (generate it on the instance itself: Settings → Access Tokens).

**This wizard is interactive and cannot be driven by the agent.** Claude Code's Bash/PowerShell tools run non-interactively (stdin attached to the null device) — console prompts read EOF immediately. So `glab auth login` must be run by the user themselves, in their own terminal. Never ask the user to paste their token into chat — only directly into their own `glab auth login` session.

Verify after login with `glab auth status --hostname gitlab.micros.uz` — look for "Logged in to ... (keyring)". The wizard's own "You're already logged in" message on a repeat `auth login` run is not by itself reliable confirmation of a successful auth — cross-check with `auth status`.

If the state gets tangled (logged into the wrong host, empty token, etc.), reset and start over:

```powershell
glab auth logout --hostname gitlab.com
glab auth logout --hostname gitlab.micros.uz
```

## Multi-line text (e.g. an issue description) as a command-line argument

A normal PowerShell 5.1 call via the `&` operator — whether a string passed directly (`glab issue create --description $body`) or an array splat (`& $glab @argList`) — mangles arguments containing newlines: the text gets split into pieces, and `glab` returns an error like `Unknown command "..."` with a fragment of the text instead of treating it as one argument. `$psi.ArgumentList.Add(...)` (the modern .NET API for `ProcessStartInfo`) came back `$null` in this environment (Windows PowerShell 5.1) and isn't usable without checking first.

What works: build `ProcessStartInfo.Arguments` manually as a single string, wrapping each value in quotes, and launch via `[System.Diagnostics.Process]::Start($psi)` directly, bypassing the `&` operator entirely:

```powershell
$psi = New-Object System.Diagnostics.ProcessStartInfo
$psi.FileName = "C:\Users\<user>\AppData\Local\Programs\GLab\glab.exe"
$psi.Arguments = "issue create --repo group/repo --title `"$title`" --description `"$body`" --yes"
$psi.RedirectStandardOutput = $true
$psi.RedirectStandardError = $true
$psi.UseShellExecute = $false
$proc = [System.Diagnostics.Process]::Start($psi)
$stdout = $proc.StandardOutput.ReadToEnd()
$proc.WaitForExit()
```

Check beforehand that none of the interpolated variables contain a literal `"` (otherwise the argument terminates early — `\"` escaping is possible too, but simpler to just swap to single quotes in the source text once).

Also avoid smart dashes (—) and similar non-ASCII punctuation in text crossing the PowerShell-to-native-process boundary — codepage mismatches can mangle these. Cyrillic inside a fenced code block (e.g. a stack trace) came through fine — it's specifically decorative punctuation like "—" that gets corrupted, not Cyrillic text itself.

## Distinction from `gh`

`gh` (GitHub CLI) is for GitHub, already configured separately. `glab` is for GitLab instances, including `gitlab.micros.uz`. Don't confuse the two — the commands are similar in spirit (`issue create`, `mr create` vs `pr create`), but they're different binaries with separate authentication.
