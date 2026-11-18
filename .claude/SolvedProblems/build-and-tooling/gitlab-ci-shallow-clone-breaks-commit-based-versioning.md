# GitLab CI shallow clone breaks commit-history-based versioning (version "downgrades" between pipelines)

Found 2026-10-13 in CWATIS (`ci/Compute-Version.ps1`, pipelines 1317 → 1318 on `development`).

## Symptom

`refers.m_system.version` went **1.0.17 → 1.0.16** between two consecutive successful pipelines, even though the second
pipeline's commit was a descendant of the first and added a `fix:` commit. The job log shows:

```
Fetching changes with git depth set to 20...
Computed version: 1.0.17.0 (from 124 commit(s) since repo start)
...
Computed version: 1.0.16.0 (from 120 commit(s) since repo start)
```

The real version (full history, anchored on tag `v1.2.0`) was 1.2.29 → 1.2.30.

## Root cause

GitLab's project-level default `GIT_DEPTH` is **20** (Settings → CI/CD → General pipelines → "Git shallow clone"), and
`.gitlab-ci.yml` did not override it. With a depth-20 shallow clone:

1. The tag commit (248 commits back) isn't in the clone, so `git describe --tags` fails → the script silently falls back
   to `1.0.0` + "replay the whole repo" ("since repo start").
2. "Whole repo" is only whatever the depth-20 cut contains. On a merge-heavy history, depth is counted per parent path,
   so the set of reachable commits (and how many `fix:` commits are in it) depends on the shape of the graph near HEAD.
   A new merge with a short side branch shifts the cut and can drop *more* old `fix:` commits than it adds → the
   number goes **down**.

So both CI numbers were meaningless, not just the second one. The build job had the same shallow clone, so the
version embedded in the deployed binary was wrong too.

## Fix

Set full clone depth (in every job that computes the version — build and DB-update), e.g. globally:

```yaml
variables:
  GIT_DEPTH: 0   # full history; Compute-Version.ps1 needs the anchor tag and every commit since it
```

and make sure tags are present (`git fetch --tags --force` as a first script step is a cheap safety net).
Optionally make the script fail loudly in CI when `git rev-parse --is-shallow-repository` is `true` instead of
silently replaying a truncated history.

## Checklist

- Any script that derives something from git history (`git describe`, commit counts, conventional-commit replay)
  must not run on a shallow clone. Look for "git depth set to N" in the job log.
- "since repo start" / "no tag found" in CI while a tag exists locally ⇒ shallow clone or tags not fetched.
- Version numbers going backwards or jumping erratically between pipelines on the same branch ⇒ check depth first.
