# Self-hosted Outline: email magic-link sign-in cannot create the FIRST team

Date: 2026-09-23. Project: `outline` (self-hosted docker-compose deployment).

## Symptom

With only `SMTP_HOST`/`SMTP_PORT`/etc. configured (no OAuth/OIDC provider), `POST /api/auth.config` on a fresh instance returns `{"providers":[]}` — the sign-in page shows no options at all, not even email. There is no error, just an empty list; nothing points at "email" being the misconfigured piece, because email *is* correctly registered as a plugin.

Symptom's sibling: configuring a real OAuth provider (e.g. Slack) with a wrong/placeholder `client_id` instead produces a visible provider button, but clicking it fails downstream with the provider's own `Invalid client_id parameter` error — a different, more obvious failure mode than the silent empty-providers case.

## Root cause

`AuthenticationHelper.providersForTeam` (`server/models/helpers/AuthenticationHelper.ts`) filters the "email" and "passkeys" providers specially: they're only included if `team?.emailSigninEnabled` / `team?.passkeysEnabled` respectively — both require an *existing* `Team` row with that preference already turned on. Every other provider type (Slack, Google, Azure, Discord, OIDC) is included whenever `!team` (no team yet at all), which is exactly the state of a fresh install.

So: enabling email sign-in is a **team setting**, toggled from inside the already-authenticated Settings → Security screen. It can never be the *first* login method, only an additional one added after someone has already created the team via a real OAuth/OIDC provider. This isn't stated anywhere in `.env.sample`; the sample groups SMTP config right next to the OAuth blocks with no note that SMTP alone is insufficient to bootstrap.

## Fix / workaround for local or air-gapped testing: throwaway OIDC via Dex

When no real Google/Slack/Azure/Discord app is wanted (e.g. local dev, CI, a demo box with no internet-facing OAuth registration), stand up [dexidp/dex](https://github.com/dexidp/dex) as a same-compose, static-config OIDC provider purely to create the first team, then switch the team to email sign-in afterward if desired.

Key wiring points (manual-endpoint OIDC mode, not discovery — see `plugins/oidc/server/auth/oidc.ts`, `hasManualConfig` branch):

- `OIDC_AUTH_URI` must be **browser-reachable** (Outline redirects the user's actual browser there) — use the host-published address, e.g. `http://localhost:5556/dex/auth`.
- `OIDC_TOKEN_URI` / `OIDC_USERINFO_URI` are called **server-to-server** from inside the `outline` container — use the docker-network service name, e.g. `http://dex:5556/dex/token`, `http://dex:5556/dex/userinfo`. Using `localhost` here resolves to the `outline` container itself, not Dex — a wrong-but-plausible mistake since it works fine for `OIDC_AUTH_URI`.
- OIDC callback path is fixed: `${URL}/auth/oidc.callback` (`plugins/oidc/server/auth/oidcRouter.ts`) — this exact URL must be in Dex's `staticClients[].redirectURIs`.
- The `email` claim is mandatory in the userinfo/id_token response or Outline throws `AuthenticationError`; Dex's static-password users provide `email` natively.
- Dex's `staticPasswords[].hash` must be bcrypt — generate one without installing anything locally via a throwaway container: `docker run --rm httpd:alpine htpasswd -bnBC 10 <user> <password>`, then strip the `user:` prefix.

Minimal config (fields Dex needs):

```yaml
issuer: http://localhost:5556/dex
storage: { type: memory }
web: { http: 0.0.0.0:5556 }
staticClients:
  - id: outline
    secret: devsecret
    redirectURIs: ["http://localhost:3000/auth/oidc.callback"]
enablePasswordDB: true
staticPasswords:
  - email: "admin@example.com"
    hash: "<bcrypt hash>"
    username: "admin"
    userID: "<any unique string>"
```

### Keeping the admin credentials out of a committed file

Dex has no env-var expansion in its own config file — it's a static YAML blob, and the password must already be a bcrypt hash by the time Dex reads it. To let the credentials live in `.env` (edit password, restart, done) instead of being baked into a checked-in file, use a one-shot **init container** that renders the real config into a shared volume before `dex` starts, since the same base image (`dexidp/dex` is Alpine) has no `envsubst`:

```yaml
services:
  dex-config:
    image: httpd:alpine          # ships htpasswd, and it's plain Alpine sh
    entrypoint: ["/bin/sh", "/dex-init.sh"]
    env_file: .env
    volumes:
      - ./dex-init.sh:/dex-init.sh:ro
      - dex-config:/shared
  dex:
    image: dexidp/dex:v2.41.1
    command: dex serve /shared/config.yaml
    volumes: [dex-config:/shared]
    depends_on:
      dex-config: { condition: service_completed_successfully }
volumes:
  dex-config:
```

`dex-init.sh` does the hashing (`htpasswd -bnBC 10 "$DEX_ADMIN_USERNAME" "$DEX_ADMIN_PASSWORD"`) and writes the YAML via a plain shell heredoc (`cat > /shared/config.yaml <<EOF ... EOF`) — a heredoc sidesteps `sed`-delimiter collisions entirely, which matters here because bcrypt hashes can themselves contain `/`. Re-running `docker compose up -d` after editing `.env` reruns the init container (compose detects the env change and recreates it), regenerating the config before `dex` restarts.

## Checklist

- Fresh self-hosted Outline instance, `providers: []` on the login page → you're missing an OAuth/OIDC provider, not (necessarily) SMTP; email alone can never be the first login.
- Need SMTP working too? Set it up regardless — it just won't produce a usable login button until a team exists.
- For local/offline testing, Dex + `htpasswd` (via a throwaway `httpd:alpine` container) is a fast, zero-external-account way to get one authenticated session to create the team.
