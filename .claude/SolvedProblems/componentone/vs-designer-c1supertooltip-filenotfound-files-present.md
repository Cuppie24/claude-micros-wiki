# VS designer: "Could not load file or assembly 'C1.Win.C1SuperTooltip.4.8…'" while the DLL is present and everything is version-consistent (KNOWN ISSUE — root cause unconfirmed)

**Status: known issue, NOT a verified fix.** Recorded so the next session doesn't re-run the same disk checks. Update this file once the real cause is confirmed.

**Symptom** — Opening a form's designer (seen on `CWATIS/Forms/Telegram/TgMembers/View/TgMembersForm.cs`) shows the "To prevent possible data loss before loading the designer, the following errors must be resolved:" page with 1 error:

```
Could not load file or assembly 'C1.Win.C1SuperTooltip.4.8, Version=4.8.20231.596, Culture=neutral, PublicKeyToken=79882d576c6336da' or one of its dependencies. Не удается найти указанный файл.
```

The form's `.Designer.cs` doesn't mention `C1SuperTooltip` at all. The build itself succeeds, so this is designer-only.

**What was ruled out (verified 2026-10-08, CWATIS, right after a fresh Debug build)**
- `packages\C1.Win.C1SuperTooltip.4.8.20231.596\lib\net48\C1.Win.C1SuperTooltip.4.8.dll` exists; so does the global NuGet cache copy.
- `.csproj` `<Reference>` + `packages.config` + `licenses.licx` all name exactly `4.8.20231.596`, token `79882d576c6336da`.
- `bin\Debug\C1.Win.C1SuperTooltip.4.8.dll` exists, same version; `CWATIS.exe` was freshly built.
- The DLL's only C1 dependency (`C1.Win.4.8`, `4.8.20231.596`) is in `bin\Debug`.
- Every assembly in `bin\Debug` that references SuperTooltip (`MC_GlobalForms`, `Refers.V2`, `MicrosSmartGrid.V2`, `DSSFormLib`, `DSSLib`, `C1.Win.FlexReport.CustomFields.4.8`, `CWATIS.exe`) asks for the same `4.8.20231.596` — no version skew, so it is not the [MC_GlobalForms/Refers.V2 mismatch](../build-and-tooling/subdivfinances-refersv2-mcglobalforms-version-mismatch.md) pattern.
- Not a C1 license-coverage issue (see [componentone-license-coverage-blocks-build.md](componentone-license-coverage-blocks-build.md)); that one breaks the *build*, this doesn't.

**Context worth knowing** — the same project had just been through repeated C1 bump/revert cycles (see the license-coverage file). `packages\` still held a leftover `C1.Win.C1Input.4.8.20261.813` folder next to the `20231.596` one. So the leading suspect is **stale Visual Studio designer state** (a designer host process that loaded newer C1 assemblies earlier and cached its resolution), not a real missing file. This is an inference, not confirmed.

**Follow-up findings (same day)**
- Call stack top frame: `at MC_GlobalForms.BaseForm.TmplMDIForm..ctor()` — the base-form constructor is what touches SuperTooltip, and **every** form deriving from `TmplMDIForm` fails the same way (not just the one first noticed).
- Deleting `.vs\` + Rebuild did **not** fix it.
- Also verified: `packages\` DLL and `bin\Debug` DLL have identical hashes; no `Zone.Identifier`; PE is MSIL; `bin\Debug` `MC_GlobalForms.dll` is byte-identical to the package's 1.1.7.9; OutputPath is plain `bin\Debug\`.
- `msbuild -t:ResolveAssemblyReferences -getItem:ReferencePath` resolves `C1.Win.C1SuperTooltip.4.8` to the **global NuGet cache** (`%USERPROFILE%\.nuget\packages\c1.win.c1supertooltip\4.8.20231.596\lib\net48\`), not the `packages\` HintPath. That file exists and is hash-identical, so it isn't the cause by itself, but it shows the resolved path differs from what the csproj HintPath suggests.

**Likely root cause found (build side confirmed; designer fix awaiting user's retest)** — a NuGet "Update" of `MC_GlobalForms` 1.1.7.9→1.1.7.10 left `packages.config` at 1.1.7.10 but **deleted the `<Reference Include="MC_GlobalForms">` block from the `.csproj`** (half-finished update). Simultaneously a stale `obj\project.assets.json` + `obj\*.nuget.g.props/.targets/.dgspec.json/project.nuget.cache` (a leftover of an earlier `msbuild -t:restore` / VS restore on this *packages.config* project) injected ~152 references straight from the global cache `%USERPROFILE%\.nuget\packages\` — including `MC_GlobalForms` **1.1.6.47** (taken from `Refers.V2`'s transitive dependency list), where `UnitsPresenter` is `internal`. Symptoms: build error `CS0122: "UnitsPresenter" недоступен из-за его уровня защиты` even though the freshly published package has it `public`, plus an unrelated-looking `CS1503` tuple-list→Dictionary error (also from the old API); and the designer loading an old `TmplMDIForm`. Tell: `msbuild X.csproj -t:ResolveAssemblyReferences "-getItem:ReferencePath"` (PowerShell, not Bash) lists `FullPath`s under `.nuget\packages\` for a packages.config project — they should all be under the solution's `packages\`.
Fix applied: re-added the `MC_GlobalForms` `<Reference>` (Version=1.1.7.10, HintPath `..\packages\MC_GlobalForms.1.1.7.10\lib\net48\MC_GlobalForms.dll`, `<Private>True</Private>`), and moved the five stale `obj\` restore artifacts out. Result: build exit 0, **0** references from the global cache, SuperTooltip now resolves to `packages\`. Backup of the removed files was left in the session scratchpad.
Rule: after any NuGet update on a packages.config project, `git diff` the `.csproj` for a **removed** `<Reference>` and check no `obj\project.assets.json` exists.

**Open discriminating questions (only if the designer still fails after reopening VS)**: does a `TmplMDIForm`-derived form open in the designer in another solution on the same machine (e.g. bookkeep)? Does the VS ActivityLog (`devenv /log`, then `%APPDATA%\Microsoft\VisualStudio\18.0_*\ActivityLog.xml`) name the exact path it failed to find?

**Next step when it recurs** — click **Show Call Stack** on the error page; the stack names which control/assembly asked for SuperTooltip, which the message alone doesn't. Then try, cheapest first, and record which one actually works:
1. Close all VS instances, delete `.vs\` (solution-level) and the designer's `%TEMP%` shadow-copy folders, reopen, **Rebuild** (not Build), reopen the designer.
2. Delete the stale `packages\C1.Win.C1Input.4.8.20261.813` folder (nothing references it).
3. Clean `bin\` + `obj\`, rebuild.

**Rule** — when a designer reports "can't find file" for an assembly, first check the disk and version agreement (as above) before touching packages; if all agree, suspect designer-host state and get the call stack.

Found in: CWATIS, 2026-10-08.
