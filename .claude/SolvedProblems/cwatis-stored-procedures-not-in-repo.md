# CWATIS stored procedures live only in the live DB, never in the repo

**Date:** 2026-08-10 · **Project:** CWATIS · **Component:** any feature that calls `CallService.CallProcedureDT(...)`/`CallService.GetTable(...)`

## Symptom

Asked to wire a C# repository method to a stored procedure whose exact schema prefix is unclear — e.g. the user pastes a bare `CREATE PROCEDURE Foo(...)` snippet with no schema, or just mentions a procedure name in chat. There is no file in the repo to grep for it, and guessing wrong compiles fine but fails only at runtime with an opaque "procedure does not exist" RPC error.

## Root cause

This repo has **zero `.sql` files** (`glob **/*.sql` returns nothing). Every stored procedure across every schema (`cwatis.*`, `global_db.*`, `bookkeep.*`) is authored and managed directly against the live MySQL instance, entirely outside version control — there is no migrations folder, no DDL history, nothing `grep`/`glob` can ever find. A past-conversation memory or a code comment can tell you a procedure *exists* and roughly what it does, but never which schema it lives in with certainty.

The `definer = someuser@\`%\`` line on a pasted `CREATE PROCEDURE` is **not evidence of schema** — this codebase reuses a single definer account (seen as both `bookkeep@%`) across procedures that live in `cwatis`, `global_db`, and `bookkeep` schemas alike. Likewise, don't trust naming-convention pattern-matching alone (e.g. "other `Refresh..._LookUp`-style procedures happen to live in `global_db`, so this one probably does too") — it's a reasonable prior, not confirmation.

The dev sandbox also has no `mysql` CLI preinstalled, and `dotnet build`/`dotnet run` can't be improvised into a quick `information_schema.routines` check without adding a new package reference — not a productive path for a one-off lookup.

## Fix / rule

- Don't spend time trying to `grep`/`glob` the repo for a procedure's DDL — it isn't there. Confirm this quickly (empty `**/*.sql` glob) and move on rather than searching harder.
- When a procedure's schema is ambiguous or unconfirmed, **ask the user** for the exact `schema.procedure_name` before hardcoding it into a `CallService.CallProcedureDT(...)` call, rather than picking the most plausible-sounding schema. The cost of asking is one round-trip; the cost of guessing wrong is a silent runtime failure on a path that has no compile-time check.
- The dev DB connection string (reference only, `Cwatis.Legacy.Server/appsettings.json` → `ConnectionStrings.DefaultConnection`) points at `mars.micros.uz` (the same Mars dev environment named in the project's `CLAUDE.md`) — useful context if a future session has an actual DB client available and wants to check `information_schema.routines` directly instead of asking, but don't treat its absence here as a blocker to just asking the user.

Precedent: `CWATIS/Forms/Subdivisions/Repository/SubdivisionsRepository.cs` (`GetSubdivisionsUsedInContractsAsync`, calling `global_db.RefreshRfrSubDivisions_LookUp`) — schema confirmed by asking the user rather than guessed from the `bookkeep`-definer line or the sibling `cwatis.subdivisions_get` naming convention, either of which would have pointed at the wrong schema.
