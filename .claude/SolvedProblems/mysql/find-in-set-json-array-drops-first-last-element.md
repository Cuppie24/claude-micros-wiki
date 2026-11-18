# MySQL `FIND_IN_SET` silently drops the first and last element when fed a JSON array string

**Symptom**: a bulk delete/operate-on-many-ids feature where **only the first and last row of the
selected set fail** — every row in between succeeds. No exception, no error message, no non-zero
`out error` — the affected rows just reappear after a refresh as if nothing happened. Easy to miss
initially because a 2-3 row test selection can look like "sometimes it just doesn't work" rather than
a first/last pattern; the pattern only becomes obvious once someone selects 4+ rows and compares which
specific rows survived across repeated attempts (that's how it was actually found — same two boundary
rows failed every time, regardless of which ids they were).

**Root cause**: the C# side built the `_Ids` procedure parameter with `JsonConvert.SerializeObject(ids)`,
producing a JSON array string like `"[400,401,402,475]"` (**with square brackets**). The stored
procedure, however, does not parse JSON at all — it does a plain SQL-side membership test:

```sql
create
    definer = bookkeep@`%` procedure CategorizedSubdivisions_Delete(IN _Ids varchar(4000))
BEGIN
    DELETE FROM global_db.categorized_subdivisions
    WHERE FIND_IN_SET(categorized_subdivision_id, _Ids) > 0;
END;
```

`FIND_IN_SET(str, strlist)` splits `strlist` on commas and does a **string** comparison against the
string form of `str` for each token — it does not know or care about JSON syntax, and it does not
numeric-parse the tokens. Splitting `'[400,401,402,475]'` on commas yields the tokens `'[400'`,
`'401'`, `'402'`, `'475]'`. Comparing `categorized_subdivision_id`'s string form against each:

- `'400'` vs `'[400'` → **not equal** (stray leading `[`) → row 400 never matches, never deleted.
- `'401'` vs `'401'` → equal → deletes normally.
- `'402'` vs `'402'` → equal → deletes normally.
- `'475'` vs `'475]'` → **not equal** (stray trailing `]`) → row 475 never matches, never deleted.

Only the first and last elements of the JSON array carry a stray bracket character, so only they fail
the string comparison — every interior element happens to be a clean numeric substring and matches
fine. This is why the bug reads as "first and last row silently don't delete, everything else works":
that's not a coincidence or an off-by-one in application code, it's a direct, mechanical consequence of
where the `[` and `]` land in the comma-split.

The pre-existing code comment on the C# method (`"_Ids - JSON-массив, процедура разбирает его через
JSON_TABLE"`) was itself wrong/stale — it asserted the procedure parses JSON via `JSON_TABLE`, which
is what you'd expect from a JSON-shaped parameter, but the actual procedure never did that. Trusting a
comment like this over reading the actual procedure body is exactly how this kind of bug hides.

**Fix**: build `_Ids` as a bare comma-separated list, not a JSON array — drop `JsonConvert` entirely
for this parameter:

```csharp
// Wrong - produces "[400,401,402,475]", breaks FIND_IN_SET's boundary elements:
{ "_Ids", JsonConvert.SerializeObject(ids) }

// Right - produces "400,401,402,475":
{ "_Ids", string.Join(",", ids) }
```

**Checklist / rule**:
- Before assuming a stored procedure parses JSON just because the C# caller serializes one, **read
  the actual procedure body**. A parameter name like `_Ids` or a comment claiming `JSON_TABLE` proves
  nothing on its own — confirmed here by the real procedure using `FIND_IN_SET` on a plain string,
  not `JSON_TABLE`.
- Whenever a bulk id-list is passed as a single string parameter to a procedure, check whether that
  procedure consumes it via `FIND_IN_SET` (wants bare CSV, e.g. `"1,2,3"`) or via `JSON_TABLE`/
  `JSON_EXTRACT` (wants an actual JSON array, e.g. `"[1,2,3]"`) — these two are **not
  interchangeable**, and mismatching them doesn't error, it silently drops rows at the boundaries
  (`FIND_IN_SET` fed JSON) or fails entirely/differently (`JSON_TABLE` fed bare CSV).
- "No error, but a partial/inconsistent result" on a batch operation is a strong hint to inspect
  *which* items failed for a positional pattern (first/last, every-other, etc.) before looking for a
  per-row data reason — a structural string-building bug produces exactly this kind of positional
  signature, and it's cheap to rule in/out by just comparing string forms.

Found in GlobalForms, 2026-08-26, in `CategorizedSubdivisionsRepository.DeleteAsync` /
`global_db.CategorizedSubdivisions_Delete` (bulk-delete path for the categorized-subdivisions grid,
introduced in the same feature work as the grid's category-Code changes).
