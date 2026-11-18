# README image stays broken on GitHub after the corrupted file was already fixed and pushed

*Found 2026-09-02 in `feedbackservice` (github.com/Cuppie24/FeedbackService).*

## Symptom

An image referenced from `Readme.md` with plain relative markdown — `![tables_schema.png](tables_schema.png)` —
renders as a broken-image placeholder on the GitHub repo page. In the rendered DOM the `<img>` tag is
present and its `src` is correct, but the image failed to decode:

```js
{ src: ".../raw/main/tables_schema.png", complete: true, naturalWidth: 0, naturalHeight: 0 }
```

Crucially this persists **after** a commit that replaced the bad file with a good one was already pushed,
so every local check says everything is fine and the problem looks like a markdown/path/case issue.

## Root cause — two separate things, and only the second one is still live

**1. The original corruption: a binary file re-saved as UTF-8 text.**
The committed PNG began with `ef bb bf` (a UTF-8 BOM) before the `89 50 4e 47` PNG signature, and had grown
from 1,365,059 to 2,995,708 bytes (~2.2x). That is the fingerprint of a binary read as text and re-encoded:
under a Windows-1252/OEM decode, bytes `>= 0x80` come back as codepoints that re-encode to 2- *and 3-byte*
UTF-8 sequences (hence expansion well past 2x). Classic causes on Windows:

- `Get-Content img.png | Set-Content out.png` (or `Out-File`, or `>` redirection) — PowerShell text cmdlets
- an editor's "Save with encoding: UTF-8" on a binary file

A BOM before the signature alone makes the file undecodable; the interior bytes are mangled too.
Use `Copy-Item`, or `[IO.File]::WriteAllBytes(...)`, never the text cmdlets, for binary.

**2. What was still breaking it: a stale object on GitHub's raw CDN, on one URL alias only.**
The repo content was already correct, but `raw.githubusercontent.com` kept serving the *old corrupted bytes*
for the short branch-name path. Same file, same branch, three ways of addressing it:

| URL form | bytes | first 4 | verdict |
|---|---|---|---|
| `/raw/main/<file>` — **what rendered markdown uses** | 2,995,708 | `ef bb bf 89` | stale + corrupt |
| `/refs/heads/main/<file>` | 1,365,059 | `89 50 4e 47` | correct |
| `/<full commit sha>/<file>` | 1,365,059 | `89 50 4e 47` | correct |
| API `contents/<file>?ref=main` | 1,365,059 | sha `acca6f6…` | correct |

A cache-busting query string (`?cb=<timestamp>`) does **not** help — raw.githubusercontent drops the query
from its cache key. `Cache-Control: max-age=300` is advertised but the edge held the object far longer.

## Why it misleads

Every local and git-level check passes and points away from the real cause:

- `git ls-tree origin/main -- <file>` and `git cat-file -s` show the good blob and size
- `git status -sb` shows no ahead/behind, so it looks fully pushed (it is)
- the rendered README HTML has a correct `src`, so it isn't markdown, path, or filename case
- the GitHub **API** reports the good blob — only the raw CDN disagrees

Note also that `git fetch` and `raw.githubusercontent.com` are different backends. Git's view being right is
**not** evidence that the raw CDN is serving the same bytes.

## Fix

**Wait — it clears itself.** Confirmed 2026-09-02: the stale object expired on its own and the
`/raw/main/` path started serving the correct bytes with no rename, no new commit and no push. Elapsed
time from the corrected file being pushed to the raw CDN serving it was roughly an hour, i.e. far longer
than the advertised `max-age=300`, but it does resolve. Re-probe before changing anything.

Only if you genuinely cannot wait, change the path to force a new cache key:

```bash
git mv tables_schema.png docs/tables_schema.png
# update the markdown reference to the new path, then commit and push
```

Renaming is a real cost — it breaks every existing link to the old path — so treat it as the fallback,
not the first move. Pointing the README at the absolute `…/refs/heads/<branch>/<file>` URL also works as
an interim workaround (GitHub does **not** camo-rewrite raw.githubusercontent URLs — verified via the
`POST /markdown` API, the `src` is passed through verbatim), but it hardcodes owner/repo/branch into the
README and becomes wrong on a fork or a branch rename.

## Checklist

- Broken image on GitHub but fine locally? Probe all three URL forms (`/raw/<branch>/`, `/refs/heads/<branch>/`,
  `/<sha>/`) and compare **byte lengths and the first 4 bytes**. Differing bytes across forms = stale CDN, not your repo.
- Having confirmed that, **do nothing and re-probe later** — the repo is already correct and the CDN reconciles
  itself. Diagnosing this is worth it precisely so you *don't* rewrite history or rename files over a cache entry.
- Verify a committed binary is actually intact rather than eyeballing the header: walk PNG chunks and check
  every CRC (`zlib.crc32(type + data)`) and that the last chunk ends exactly at EOF. A valid signature and a
  trailing `IEND` do not prove the middle survived.
- A leading `ef bb bf` on any binary, plus a size increase of roughly 1.5x-2.5x, means it went through a text
  re-encode. Re-copy from the source with a binary-safe command; do not try to repair it by stripping the BOM.
- Consider `.gitattributes` (`*.png binary`) to keep such files out of text-transforming paths.
