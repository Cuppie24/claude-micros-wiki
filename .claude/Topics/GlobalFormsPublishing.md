# Publishing GlobalForms (the `MC_GlobalForms` NuGet package)

How a change to the shared GlobalForms library (`D:\projects\projects\globalforms`, GitLab `microsdvl/globalforms`) reaches the consuming
products (CWATIS, bookkeep, SubdivFinances, ...). Verified 2026-10-15 from the repo's `.gitlab-ci.yml`, `AssemblyInfo.cs` history and an actual
push (`8f1f0fe..18bc1dc`). The pipeline run itself was not watched — see "Not verified".

## The process (what a developer does)

1. Make the change in `GlobalForms\` and build it locally (`MC_GlobalForms.sln` or `GlobalForms\MC_GlobalForms.csproj`, MSBuild — use the
   PowerShell tool, not Git Bash, see `claude-global/SolvedProblems/tooling/git-bash-mangles-msbuild-slash-flags.md`).
2. **Bump the patch (last) version number by 1**, in `GlobalForms\Properties\AssemblyInfo.cs` — both lines, kept equal:
   ```csharp
   [assembly: AssemblyVersion("1.1.7.12")]
   [assembly: AssemblyFileVersion("1.1.7.12")]
   ```
   This is the **only** place the version is stored. `MC_GlobalForms.nuspec` uses the `$id$` / `$version$` tokens, which `nuget pack` fills from the
   built assembly. History shows every release is +1 on the last segment (`1.1.7.9` → `.10` → `.11` → `.12`), usually in the same commit as the change
   (`bump version to 10` was a standalone commit once).
3. Commit and **push to `development`**. Nothing else is done by hand — no tag, no manual `nuget push`.
4. CI/CD does the rest (below). The package appears on the internal feed `https://mars.micros.uz:9109/nuget`.
5. Each consumer then has to bump its own reference (see "After publishing").

Edit `AssemblyInfo.cs` at byte level or with an editor that preserves its CRLF line endings and BOM (a plain `sed -i` can strip them, see
`../SolvedProblems/winforms/vs-designer-reverts-external-resx-edits.md`).

## What CI/CD does (`.gitlab-ci.yml`)

| Job | Stage | Runs on | What |
|---|---|---|---|
| `build_project` | build | every branch | `nuget restore` + MSBuild `Release` `/t:Rebuild` of `MC_GlobalForms.sln`; artifact `GlobalForms\bin\Release\` |
| `pack_nuget` | package | **only `development`** | `nuget pack ./GlobalForms/MC_GlobalForms.csproj -Prop Configuration=Release` → `*.nupkg` |
| `publish_nuget` | deploy | **only `development`** | `nuget push *.nupkg -Source https://mars.micros.uz:9109/nuget` |

So: pushing any other branch only builds; **a push to `development` publishes a package**. Consequence — never push to `development` without bumping the
version: NuGet feeds reject a package whose id+version already exists, so the `publish_nuget` job would fail (standard NuGet behaviour; not observed here).
Work on a feature branch and merge into `development` when the change should be published.

## After publishing — consumers

- The consumer's `packages.config` and `.csproj` `<Reference>` (`HintPath` `..\packages\MC_GlobalForms.<version>\lib\net48\MC_GlobalForms.dll`) must be bumped
  by hand to the new version; nothing updates them. Until then the consumer compiles against the old API (e.g. a new public property is "not found").
- To get the new package into the solution's `packages\` folder without opening Visual Studio (verified 2026-10-15 for CWATIS, 1.1.7.12; PowerShell tool):
  ```powershell
  msbuild CWATIS.csproj /t:restore /p:RestorePackagesConfig=true /p:SolutionDir="D:\projects\projects\CWATIS\"
  ```
  Without `RestorePackagesConfig=true` a plain `/t:restore` does nothing for a `packages.config` project (see the ComponentOne note in the solved-problems index).
  `SolutionDir` (with trailing `\`) makes the package land in the solution-level `packages\` folder that the `HintPath` points to.
  Do **not** use the ancient `NuGet.exe` files that ship inside old packages under `packages\` — they can't talk to a v3 feed.
- **`Refers.V2` pins an exact `MC_GlobalForms` version** in its own nuspec. Bumping `MC_GlobalForms` in a consumer without matching what `Refers.V2` was compiled against
  breaks the designer / runtime with a `FileNotFoundException` — see `../SolvedProblems/build-and-tooling/subdivfinances-refersv2-mcglobalforms-version-mismatch.md`.
  State in CWATIS on 2026-10-15: `Refers.V2` 1.1.3.65 was compiled against `MC_GlobalForms` **1.1.7.7** (read from its nuspec and the dll's referenced-assembly
  list), while CWATIS runs `1.1.7.12` — it has been drifting from the pin for several releases with a working build; the designer-time error is the thing to watch for.
- `GlobalForms` itself must not reference a package named `MC_GlobalForms` (see
  `../SolvedProblems/build-and-tooling/globalforms-refersv2-self-referential-mc-globalforms-package.md`).
- A new public member added to a GlobalForms form is **not usable from the consumer until its package is published and bumped**; write the consumer-side
  line after that, so the consumer keeps compiling in between.

## Not verified

- The CI pipeline run for a given push was not inspected (job duration, a failing job, runner availability). Check the pipeline page of the project after pushing.
- The retention/yanking of old package versions on the feed, and who may push to `development` / branch protection rules, were not looked at.
