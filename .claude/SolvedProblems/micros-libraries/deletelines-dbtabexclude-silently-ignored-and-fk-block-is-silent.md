# CallService.DeleteLines: the exclude-table argument is silently discarded server-side, and a
# blocked delete leaves the error text empty

**Symptom:** `CallService.DeleteLines(..., dbTabExclude: "schema.some_table", ...)` — the row you
expect to delete is sometimes just not deleted (its id is missing from the `trueIds`/`TrueIds`
output), but the `res`/`error` out-parameter comes back **empty**, exactly as on success. No
exception, no message box, no log line on the client. Found investigating CWATIS 2026-09-28 while
verifying the client-side `DeleteLines` usage instruction against the actual server.

**Root cause — two independent things, both confirmed by reading the real server
(`microsdvl/websvc.legacy.server`, `master` branch, not checked out under `D:\projects\micros` —
only in the bare mirror at `D:\git mirrors\websvc.legacy.server.git`):**

1. **The `DbTabExclude` argument never reaches the FK-exclusion check.** The gRPC controller
   (`WebSvc.Legacy.Server/Web/Controller/Grpc/LegacyDbService.cs`, line 32, single commit
   `11f11be6`, 2025-10-02, unchanged since) calls the internal service with the table-name
   argument passed **twice**:

   ```csharp
   await _legacyDbService.DeleteLines(request.DelIds, request.Icn, request.DbName,
       request.DbTableName, request.DbTableName /* should be request.DbTabExclude */, request.FkWhere, ...);
   ```

   Whatever the client sends as `DbTabExclude` (CWATIS passes things like `"cwatis.rep_act"`) is
   silently thrown away; the "table to exclude from the FK scan" the server actually uses is
   always just the table being deleted from, itself — a no-op exclusion. This is a copy-paste bug
   in the controller, not a documented behavior.

2. **A row that fails the FK check is skipped without ever setting an error.** The handler
   (`WebSvc.Legacy.Server/Application/Features/LegacyDb/DeleteLines/DeleteLinesCommand.cs`) does,
   per id:

   ```csharp
   var tmpRes = await _mediator.Send(new FkCheckWCommand(databaseName, databaseTableName, Id,
       request.DbTabExclude, request.FKWhere));   // calls MySQL proc `FKCheckW` — generic FK scan
   if (tmpRes != null) { resDt.Merge(tmpRes); continue; }   // <- blocked: skip, no error set
   // ... only reached if FKCheckW found nothing: real DELETE runs here
   ```

   `FKCheckW` is a MySQL stored procedure (not in any repo) that appears to scan for rows
   elsewhere referencing this id, filtered by the (effectively always-self) exclusion and by the
   `FKWhere` argument (this one **is** passed through correctly). If it finds a match, the id is
   `continue`d — never deleted, never added to `TrueIds` — but **`response.Result` is not touched
   on this path**. It only gets text on the *other* failure path: a raw `MySqlException` thrown by
   the actual `DELETE FROM {schema}.{table} WHERE {icn} = {id}` statement (e.g. a physical FK
   constraint `FKCheckW` didn't catch, a bad column name, a timeout).

   So `CallService.DeleteLines`'s `res`/`error` out-parameter only ever reports the second failure
   mode. A row silently blocked by the generic FK scan looks exactly like a successful, complete
   delete unless you check something else.

Two more details, same read of `DeleteLinesCommand.cs`:

- **No dot in `DBTableName` → the separate schema argument is used; a dot in `DBTableName` →
  its own prefix wins and the separate schema argument is ignored entirely**:
  ```csharp
  if (tableinfo.Contains(".")) { databaseName = tableinfo.Split('.')[0]; databaseTableName = tableinfo.Split('.')[1]; }
  else { databaseName = request.dBName; databaseTableName = request.DBTableName; }
  ```
  So `DeleteLines(ids, "Id", "cwatis", "cwatis.some_table", ...)` and
  `DeleteLines(ids, "Id", "cwatis", "some_table", ...)` behave identically as long as the separate
  schema argument matches the schema embedded in the table name — but if they ever disagree, the
  schema-qualified table name silently wins and the separate schema argument does nothing.
- **No transaction across ids.** The handler loops the input ids one at a time, opening/closing
  its own connection and running one `DELETE ... WHERE icn = {id}` per id. If id #3 of 5 throws, ids
  #1–2 are already committed. There is no rollback of the whole batch.

**Fix / how to detect a silently-blocked delete from the client:** don't rely on
`res`/`error` alone. After calling `DeleteLines`, compare the `trueIds`/`deleted` array the call
fills in against the ids you passed — any id missing from it was not deleted, FK-blocked or not,
whether or not `res` is empty. If you need to tell the user which rows are blocking it, the
returned `DataTable` (table name `FKCHECKTable`) holds the referencing rows `FKCheckW` found for
the skipped ids — but it comes back only via the raw `PackedDataTable`, not through `res`.

**Checklist / rule:**
- Treat `DeleteLines`'s `DbTabExclude` argument as currently inert. Passing a "known safe"
  related table there (as `PreContrRepository`, `ActRepository`, and several report repos already
  do, e.g. `"cwatis.rep_act"`) has no effect on the server as of 2026-09-28 — it is not excluded
  from anything.
- Never treat an empty `res` as proof a `DeleteLines` call fully succeeded. Check the returned
  `trueIds` against the ids you requested.
- If a `DeleteLines` call needs the exclusion argument to actually work, or needs errors surfaced
  reliably, that is a server change in `microsdvl/websvc.legacy.server`
  (`WebSvc.Legacy.Server/Web/Controller/Grpc/LegacyDbService.cs` line 32 and/or
  `DeleteLinesCommand.cs`'s `continue` branch), not something fixable from CWATIS.
- This repo (`websvc.legacy.server`) is not checked out under `D:\projects\micros`; it exists only
  as a bare mirror at `D:\git mirrors\websvc.legacy.server.git` — read it with
  `git --git-dir="D:\git mirrors\websvc.legacy.server.git" show master:<path>` rather than
  assuming it's absent. Its `main` branch is a near-empty placeholder (README only); the real code
  is on `master`. Precedent for finding a server implementation: CWATIS's `App.config` names the
  runtime endpoint (`mars.micros.uz:9090`); its bare-mirror repo is not always the one whose name
  matches the NuGet package (`GRP-Client.V2` ships from `grp-service.git`, but CWATIS's client
  actually resolves to a *different*, gRPC-based facade whose server is `websvc.legacy.server.git`
  — see `SolvedProblems/callservice-grp-client-alias-shadows-real-namespace.md` [not yet written
  as a separate file; documented inline in `../../Topics/CallServiceUsage.md`'s "Which assembly actually runs"
  section] for why `GRP-Client.V2`/`grp-service` turned out to be the wrong trail).
