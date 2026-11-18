# Outline (self-hosted, Docker) fails to connect to a plain Postgres image with an SSL error

Date: 2026-09-23. Project: `outline` (self-hosted docker-compose deployment).

## Symptom

`outline` container restart-loops. Logs show, immediately after "injected env":

```
{"error":"The server does not support SSL connections","level":"error","message":"The database does not support SSL connections. Set the `PGSSLMODE` environment variable to `disable` or enable SSL on your database server.","stack":"SequelizeConnectionError: The server does not support SSL connections\n    at Client._connectionCallback ..."}
```

The message itself already names the fix — this one is here because it's a required, easy-to-miss env var, not because the error is misleading.

## Root cause

Outline's Sequelize/pg client attempts an SSL connection to Postgres by default. A stock `postgres:*` Docker image (no TLS cert configured) does not support SSL connections at all, so the handshake fails outright — there is no automatic fallback to plaintext.

## Fix

Set `PGSSLMODE=disable` in the `outline` service's environment (not the `postgres` service — this is a Postgres *client* libpq setting Outline's Node process reads, unrelated to the Postgres server's own config).

```yaml
services:
  outline:
    environment:
      PGSSLMODE: disable
```

This is actually documented as a commented-out line in Outline's own `.env.sample` ("Uncomment this line if you will not use SSL...") — easy to skip past when copying just the required vars.

## Checklist

- Any docker-compose Outline deployment against a plain (non-TLS) Postgres container needs `PGSSLMODE=disable` set for the `outline` service specifically.
- If instead terminating real TLS to Postgres (managed DB, `sslmode=require` upstream), leave `PGSSLMODE` unset.
