# Deployment Plan

Target: a single VPS running **Coolify** (self-hosted PaaS) for all three apps,
with **Supabase Cloud** as the managed database/auth/storage, and **subdomains**
per surface. Coolify owns the reverse proxy, automatic TLS (Let's Encrypt),
per-app env/secrets, health checks, restart policies, and git-based auto-deploy —
so there is no hand-rolled Caddy/compose/TLS/SSH in production. GitHub Actions is
only a quality gate (test/lint/build); Coolify deploys on push.

The existing `docker-compose.dev.yml` + `docker/caddy/Caddyfile` remain the
**local** stack only.

## Architecture

| Surface | App | Coolify build | Domain | Notes |
| --- | --- | --- | --- | --- |
| Edge (ingress) | nginx (`docker/edge`) | Docker Compose | all three domains | single internet-facing container |
| Portfolio (SSR) | `apps/frontend` (Next.js) | Docker Compose | proxied via edge | internal only, no published port |
| API | `apps/backend` (Express/Bun) | Docker Compose | proxied via edge | internal only, no published port |
| 3D experience (static) | `apps/experience-3d` (Vite) | baked into the edge image | served directly by edge | no separate container |
| Data | Supabase Cloud | — (managed) | project URL | — |

Data flow: the Next.js frontend fetches the backend **server-side** (server
components → `http://backend:4000/api` over the internal docker network, no
browser CORS). The 3D app is fully static, baked into the `edge` nginx image,
and loads a pre-generated `public/data/world.json` (a snapshot of the backend
world endpoint) at runtime — it makes no runtime API calls of its own.

## Prerequisites

- VPS with Coolify installed and reachable, Docker healthy.
- A GitHub App connection in Coolify (Settings → Sources) so pushes auto-deploy.
- DNS control for `<domain>`.
- A Supabase Cloud project.

## 1. Supabase Cloud

1. Create a project; note the connection string, project URL, service-role key.
2. Set the backend env (below) to point at it.
3. **Authentication → URL Configuration**: set **Site URL** to the production
   frontend domain (`https://<frontend-domain>`) and add the pattern below to
   the **Redirect URLs** list (a *different* field from Site URL — both must
   be set):
   ```
   https://<frontend-domain>/**
   ```
   The trailing `/**` is required — Supabase matches Redirect URLs as glob
   patterns, so a bare `https://<frontend-domain>` entry (no wildcard) will
   NOT match `https://<frontend-domain>/api/auth/callback?next=...` and
   Supabase falls back to the Site URL just as if nothing were registered at
   all. Confirmed failure modes seen in practice:
   - Nothing registered → redirects to whatever Site URL was last configured
     (commonly `http://localhost:3000/?code=...` left over from local dev).
   - Site URL fixed but Redirect URLs missing the `/**` suffix → redirects to
     `https://<frontend-domain>/?code=...` (the corrected Site URL's root),
     still never reaching `/api/auth/callback`, still no session cookie set.
   Keep `http://localhost:3000/**` in the list too if local dev OAuth should
   keep working alongside production.
4. Run migrations against it (drizzle):
   ```bash
   # from a machine with the prod env loaded
   bun run --cwd apps/backend db:migrate:production
   ```
   `drizzle.production.config.ts` reads `DATABASE_URL`. Migrations are
   idempotent, so this is also safe as a Coolify **pre-deployment command** on
   the backend app (see step 4).
5. Create the content storage bucket + policy used by the content store
   (`content-md`), matching the local Supabase setup under `supabase/`.
6. If you need to migrate existing content from another PersonaOS Supabase
   project, export it first:
   ```bash
   docker-compose -f docker-compose.dev.yml run --rm backend \
     bun run export:content --handle=<handle> --out-dir=/app/tmp/content-export
   ```
   This writes Markdown files plus `personaos-export-manifest.json`. The
   existing importer consumes that manifest so draft/archive state survives the
   move.
7. Import that exported snapshot into production:
   ```bash
   docker-compose -f docker-compose.dev.yml run --rm backend \
     bun run import:content --email=<owner-email> --content-root=/app/tmp/content-export
   ```
   Use `--owner-id` or `--handle` instead of `--email` when that is the safer
   identifier for the target account.
8. Seed starter content only when you are not importing an existing export:
   ```bash
   DATABASE_URL=... bun scripts/init-production.ts --email=<owner-email>
   ```

## 2. DNS

Point all records at the VPS IP (Coolify terminates TLS for each):

| Type | Name | Value |
| --- | --- | --- |
| A | `<frontend-domain>` (apex or sub) | VPS IP |
| CNAME/A | `www` | VPS IP / apex |
| A | `api` | VPS IP |
| A | `3d` | VPS IP |

(If the portfolio lives at an existing host like `hamzahasbi.2hlabs.com`, use
`api.hamzahasbi.2hlabs.com` and `3d.hamzahasbi.2hlabs.com`.)

## 3. Build & publish (GitHub Actions → GHCR)

Images are built in CI and published to **GitHub Container Registry (GHCR)** —
Coolify pulls the pre-built images instead of building on the VPS. This keeps
VPS load near zero on deploy, gives immutable versioned artifacts, and means the
image that passes CI is literally the image that runs in prod.

`.github/workflows/ci.yml` publishes three images per push to `main` or
`develop`, once the quality gate (lint/test/build) passes:

| App | Image | Tag on `main` | Tag on `develop` |
| --- | --- | --- | --- |
| Backend | `ghcr.io/<owner>/personaos-backend` | `latest` | `dev` |
| Frontend | `ghcr.io/<owner>/personaos-frontend` | `latest` | `dev` |
| Edge (nginx + baked-in 3D build) | `ghcr.io/<owner>/personaos-edge` | `latest` | `dev` |

Auth uses the built-in `GITHUB_TOKEN` (no extra secrets to configure). Package
visibility defaults to private on first publish — for a solo project you can
make each package public via the GitHub package settings so Coolify pulls them
without a registry credential (no runtime secrets in the images — env is
injected by Coolify).

### Build-time env via GitHub Environments

`NEXT_PUBLIC_*` (frontend) and `VITE_WORLD_URL` (3D, baked into the `edge`
image's build stage) are **inlined at build time** by Next.js and Vite
respectively — they can't be runtime-injected by Coolify. Instead the
`publish` job selects a **GitHub Environment** based on branch and passes
that Environment's `vars` as `--build-arg` to Docker:

| Branch | GitHub Environment | Image tag |
| --- | --- | --- |
| `main` | `production` | `:latest` |
| `develop` | `staging` | `:dev` |

**One-time setup:** Repo → Settings → Environments → create `production` and
`staging`. In each, add these `Variables` (not Secrets — the values ship in
the client bundle anyway):

| Var | production value | staging value |
| --- | --- | --- |
| `NEXT_PUBLIC_API_URL` | `/api` (or omit) | `/api` (or omit) |
| `NEXT_PUBLIC_DOMAIN` | `<frontend-domain>` | `dev.<frontend-domain>` |
| `NEXT_PUBLIC_TURNSTILE_SITE_KEY` | production widget site key | staging widget site key |
| `VITE_WORLD_URL` | `https://<ref>.supabase.co/storage/v1/object/public/world/<handle>.json` | staging equivalent |

Changing a value → push (or rerun the workflow) → CI rebuilds with the new
value → Coolify redeploys the new image. Runtime env in Coolify is unaffected
(no rebuild needed for `DATABASE_URL`, `SUPABASE_*`, etc.).

### Cloudflare Turnstile

Create separate Turnstile widgets for `production` and `staging` in the
Cloudflare dashboard. Restrict each widget to the exact hostnames that serve
its anonymous sign-in and sign-up forms; do not allow the production widget on
staging hostnames or vice versa.

Put each widget's **site key** in its GitHub Environment as the
`NEXT_PUBLIC_TURNSTILE_SITE_KEY` Variable. This key is intentionally public and
is baked into the frontend image. Put the matching **secret key** only in the
corresponding Coolify compose resource as `TURNSTILE_SECRET_KEY`; it is consumed
only by the backend at runtime. Never add the secret to GitHub Variables,
`NEXT_PUBLIC_*`, Docker build arguments, or a frontend environment/image.

The example env files use Cloudflare's always-pass dummy keys for local
development. They must not be reused for production or staging.

### Pre-MVP signup gate

Two independent layers keep new accounts from being created before the app
is ready:

- `SIGNUPS_ENABLED=false` on the backend rejects `POST /account/signup`
  outright and stops the OAuth callback from creating a profile for any
  first-time Google/GitHub user — existing users (including the owner) are
  unaffected. Set to `true` when the MVP is ready to accept signups.
- `BASIC_AUTH_USERNAME`/`BASIC_AUTH_PASSWORD` on the frontend put an HTTP
  Basic Auth prompt in front of `/login`, `/account/create`, `/password/*`,
  and `/dashboard`, hiding them from casual visitors entirely. Unset either
  one to disable the wall (e.g. for local development).

Turning the MVP fully public later requires no code change: set
`SIGNUPS_ENABLED=true` and remove the two `BASIC_AUTH_*` variables.

## 4. Coolify: deploy the compose stack

The whole prod topology lives in [`docker-compose.prod.yml`](../../docker-compose.prod.yml)
at the repo root. It pulls three GHCR images — `backend`, `frontend`, and
`edge` — and declares every runtime env var each service needs. No explicit
`networks:` block: compose creates a default network for the file and
attaches every service to it automatically, so `edge` reaches `backend` and
`frontend` by service-name DNS with no extra config. A single compose file
replaces what would otherwise be three-plus separate Coolify app configs.

**Why a single nginx `edge` service instead of pointing Coolify's Traefik
straight at each app:** it puts routing, gzip, and cache-control policy in one
versioned file ([`docker/edge/default.conf.template`](../../docker/edge/default.conf.template))
instead of several separate implicit Traefik label configs, and it means only
one container is ever internet-facing — `backend` and `frontend` have no
published ports at all, only `edge` reaches them over the default network.
The 3D app's static build is baked directly into the `edge` image
([`docker/edge/Dockerfile.prod`](../../docker/edge/Dockerfile.prod)) and
served by the same nginx process — no separate 3D container, no double proxy
hop. Coolify/Traefik still owns TLS termination and forwards plain HTTP to
`edge` with the `Host` header intact; `edge`'s `server_name` blocks do the
actual subdomain → service dispatch.

**Coolify setup:**
1. **New Resource → Docker Compose**. Point at this repo, branch `main`, and
   the compose file path `docker-compose.prod.yml`.
2. In Coolify → **Sources**, add a `ghcr.io` registry entry with a GitHub PAT
   with `read:packages` scope. Skip this if you made the three packages
   public — Coolify can pull anonymously.
3. In the compose resource → **Environment Variables**, set the values from
   [`.env.compose.example`](../../.env.compose.example) (see §5). Coolify
   populates these into the container environment via the same `${VAR}`
   interpolation compose already uses.
4. **Domains**: assign all three domains — `api.<domain>`,
   `<frontend-domain>` (+ `www`), `3d.<domain>` — to the **`edge`** service
   only. Coolify lets one service carry multiple domains; it generates a
   separate Traefik router per domain, all forwarding to the same `edge:80`
   container with the original `Host` header intact. Leave the `backend` and
   `frontend` domain fields empty — they have no published port for Traefik
   to attach to anyway.
5. **Redeploy trigger**: use Coolify's "Redeploy" webhook (Settings → Webhook)
   and either call it from a follow-up step in the `publish` CI job, or let
   Coolify's registry polling pick up new `:latest` / `:dev` tags.

**Verified end-to-end locally:** built all three images, booted the compose
stack, and confirmed with spoofed `Host` headers against `edge`'s single
mapped port — `api.<domain>` → backend `/api/health` 200, `<frontend-domain>`
and `www.<domain>` → frontend 200, `3d.<domain>` → the baked-in static 3D
build 200, an unmatched `Host` → 404 (no default-server catch-all). Confirmed
`docker compose ps` shows only `backend`/`frontend`/`edge` — no separate 3D
container. Confirmed the nginx template's `${...}` placeholders were replaced
with real values at container start, `/_next/static/` and `/assets/` both got
`Cache-Control: public, max-age=31536000, immutable`, and gzip compressed the
response.

### Staging (optional)
Duplicate the compose resource in Coolify with `IMAGE_TAG=dev` and staging
domain values (`API_DOMAIN=api-dev.<domain>`, etc.) on the same `edge`
service. Every push to `develop` republishes the `:dev` images and Coolify
redeploys.

## 5. Environment variables

Full template with descriptions is at [`.env.compose.example`](../../.env.compose.example).
Copy it, fill in real values, and either commit the resulting `.env` (for
non-Coolify hosts — but do NOT commit real secrets) or paste each variable
into Coolify's environment UI on the compose resource.

Summary of what each service consumes at **runtime** (build-time `NEXT_PUBLIC_*`
and `VITE_*` env vars are baked into the GHCR images by CI — see §3):

| Var | Consumer | Notes |
| --- | --- | --- |
| `GHCR_OWNER` | compose | GitHub user/org owning the images (e.g. `hamzahasbi`) |
| `IMAGE_TAG` | compose | `latest` for prod, `dev` for staging |
| `API_DOMAIN`, `FRONTEND_DOMAIN`, `WWW_DOMAIN`, `THREED_DOMAIN` | edge | subdomain routing — must match the domains assigned to `edge` in Coolify |
| `PERSONAOS_API_KEY`, `PERSONAOS_API_SECRET` | backend + frontend | shared read-side auth |
| `SUPABASE_URL`, `SUPABASE_PUBLIC_URL`, `SUPABASE_SERVICE_ROLE_KEY` | backend | from Supabase Cloud Project Settings → API |
| `PUBLIC_APP_ORIGIN` | backend | canonical HTTPS frontend origin used for OAuth and password-reset redirects |
| `DATABASE_URL` | backend | Supabase Cloud Session-mode URI (port 5432) |
| `TURNSTILE_SECRET_KEY` | backend | server-only Cloudflare Turnstile secret; set in Coolify, never in frontend or GitHub Variables |
| `SIGNUPS_ENABLED` | backend | `false` blocks `/account/signup` and new OAuth profiles; set `true` once the MVP is ready |
| `BASIC_AUTH_USERNAME`, `BASIC_AUTH_PASSWORD` | frontend | optional HTTP Basic Auth wall on `/login`, `/account/create`, `/password/*`, `/dashboard`; unset both to disable |
| `ALLOWED_ORIGINS` | backend | CORS allowlist, comma-separated, no trailing slash |
| `CONTENT_URL_EXPIRES_IN_SECONDS` | backend | default `300`; signed-URL lifetime for `content-md` |
| `PERSONAOS_API_URL` | frontend | leave unset → SSR uses internal `http://backend:4000/api` |
| `FRONTEND_INTERNAL_URL` | backend | internal frontend service URL as seen from the backend container, used by the site-export snapshot fetcher; default `http://frontend:3000`, which already matches the `frontend` service name in `docker-compose.prod.yml` |
| `PERSONAOS_PUBLIC_HANDLE` | frontend | which handle the public routes render |
| `PERSONAOS_CLASSIC_PORTFOLIO_URL` | backend + frontend | classic portfolio link on the home page and in the 3D world |
| `PERSONAOS_3D_PORTFOLIO_URL` | frontend | 3D portfolio link on the home page and flash-news banner |
| `BETTER_STACK_ENABLED` | backend + frontend | default `false`; only the literal `true` enables BetterStack log shipping |
| `BETTER_STACK_SOURCE_TOKEN` | backend + frontend | default empty; BetterStack Telemetry source token |
| `BETTER_STACK_INGESTING_URL` | backend + frontend | default empty; BetterStack Telemetry ingesting URL |

### Log shipping (BetterStack)

Set all three `BETTER_STACK_*` vars on both the `backend` and `frontend` services in Coolify —
the flag defaults to off so shipping stays disabled until every value is present. Browser
errors never talk to BetterStack directly: they post to the same-origin `/betterstack/logs`
route, which attaches the server-side token, so the token never reaches the browser. To
disable shipping again, set `BETTER_STACK_ENABLED=false` on both services and redeploy.

## 6. `world.json` for the 3D app

The 3D app is a renderer only and must not call the protected API from the
browser (that would leak the API secret). Instead the payload is produced
server-side and published to a public object the static app fetches at startup.

Primary flow — **dashboard button**:
1. Create a **public** Supabase Storage bucket named `world`.
2. In the dashboard, the **3D world → Regenerate 3D world** button calls
   `POST /api/world/refresh` (authenticated by the signed-in user). The backend
   builds the payload for that user and uploads it to `world/<handle>.json`,
   returning the public URL.
3. Set the 3D app's `VITE_WORLD_URL` build env to that public URL
   (`<SUPABASE_PUBLIC_URL>/storage/v1/object/public/world/<handle>.json`).
   The 3D app loads it at startup; regenerating overwrites the same object, so
   new content is live on the next page load — **no redeploy**.

Fallbacks: if `VITE_WORLD_URL` is unset the app loads the committed
`public/data/world.json`; if that fails it uses the bundled world. Keep the
committed file for local/offline dev — refresh it with the curl below.

```bash
set -a; . ./.env.backend; set +a
curl -s -H "x-personaos-api-key: $PERSONAOS_API_KEY" \
     -H "x-personaos-api-secret: $PERSONAOS_API_SECRET" \
     "http://localhost:4000/api/public/world/<handle>/3d" \
  | python3 -m json.tool > apps/experience-3d/public/data/world.json
```

### Tenancy: single-tenant now, multi-tenant later

Today the platform is **single-owner-per-deployment**: one `edge` deployment
(with the 3D build baked in) serves one portfolio, so `VITE_WORLD_URL` is a
fixed constant
(`<SUPABASE_PUBLIC_URL>/storage/v1/object/public/world/<handle>.json`) and the
snapshot is refreshed manually via the dashboard button. This is the current,
intended setup — nothing to change.

When the platform needs to serve many users from **one** 3D deployment, the
upgrade path is (and it doesn't invalidate the current design):

1. **Derive the URL at runtime instead of baking it.** Read the handle from the
   route (`/[handle]`, `?handle=`, or a subdomain) and build
   `${VITE_WORLD_STORAGE_BASE}/storage/v1/object/public/world/${handle}.json`,
   where `VITE_WORLD_STORAGE_BASE` is a single platform-wide constant. One build
   serves every user; no per-user URL or rebuild. `resolveWorldUrl()` in
   `apps/experience-3d/src/world-fixture.ts` is the one place to change.
2. **Auto-publish on content publish.** Call `publishWorld` from the backend
   content-publish path so each user's `world/<handle>.json` is (re)written
   automatically. The dashboard button then becomes an optional manual refresh,
   and new users are zero-touch. (Requires the frontend to also route per handle
   — a broader change tracked separately.)

See [backlog: profile lookup by email](../backlog.md#profile-lookup-by-email)
for a related deferred item — a reusable, tested email→profile resolver
would help both admin tooling and any future multi-tenant handle resolution.

## 7. CI quality gate (GitHub Actions)

`.github/workflows/ci.yml` on PRs + `main`:
- `bun install`
- `bun run test` / `lint` / `build` across apps (via turbo).
- Does **not** deploy — Coolify's GitHub App handles deploys on push to `main`.
  This keeps a broken build from shipping (protect `main` with the check).

## 8. Go-live checklist

Repo-side (done — see verification notes inline in each PR/commit):

- [x] Production Dockerfiles added: `apps/backend/Dockerfile.prod`,
  `apps/frontend/Dockerfile.prod` (context = repo root), and
  `docker/edge/Dockerfile.prod` (nginx + baked-in 3D static build).
- [x] Backend `start` script runs without `--watch`; frontend `output: 'standalone'`.
- [x] Backend CORS allowlist wired to `ALLOWED_ORIGINS` (`src/cors.ts`, mounted in `server.ts`).
- [x] CI quality gate + registry publish: `.github/workflows/ci.yml` runs lint + test + build on PRs; on push to `main`/`develop` also builds and pushes `backend`/`frontend`/`edge` images to GHCR (`:latest` / `:dev`).
- [x] `world` Storage bucket migration + `ensure:world-bucket` script.

Deploy-time (against the live VPS / Supabase Cloud):

- [ ] Separate production and staging Turnstile widgets created with exact hostname restrictions.
- [ ] GitHub Environments `production` and `staging` created; `NEXT_PUBLIC_DOMAIN`, `NEXT_PUBLIC_TURNSTILE_SITE_KEY`, and `VITE_WORLD_URL` set as `Variables` on each (see §3).
- [ ] Each Coolify compose resource has the matching `TURNSTILE_SECRET_KEY`; confirm it is absent from GitHub Variables and frontend build configuration.
- [ ] Push to `main` (or `develop`) once so GHCR gets the initial images. Verify at `https://github.com/<owner>?tab=packages`.
- [ ] (Optional) Change each GHCR package's visibility to Public if you don't want to configure Coolify registry auth.
- [ ] Supabase project created; migrations applied; `content-md` + public `world` buckets set.
- [ ] Supabase Auth URL Configuration: Site URL + Redirect URLs updated to the production domain (see §1 step 3) — otherwise OAuth login silently redirects to whatever Site URL was last configured (commonly `localhost:3000` from local dev).
- [ ] Backend `ALLOWED_ORIGINS` set for frontend and 3D domains.
- [ ] DNS records created and resolving.
- [ ] Coolify compose resource created from `docker-compose.prod.yml`, env populated from `.env.compose.example`, all three domains assigned to `edge` with TLS green.
- [ ] `/api/health` returns 200 through the proxy; API reachable at `https://api.<domain>`.
- [ ] Dashboard "Regenerate 3D world" writes `world/<handle>.json`; `VITE_WORLD_URL` set.
- [ ] Frontend renders with live API data; 3D app loads the world snapshot.
- [ ] `mainSiteUrl` (professional `website`) points at the deployed frontend.
- [ ] CI green and required on `main` (branch protection).

## 9. Operations

- **Rollback:** Coolify keeps previous deployments — redeploy the prior commit.
- **Backups:** Supabase Cloud manages DB backups; verify the retention tier.
- **Secrets:** live only in Coolify (runtime) and GitHub Actions secrets (CI) —
  never commit real `.env` files; keep `.env.*.example` current.
- **Logs/metrics:** use Coolify's per-app logs; add uptime monitoring on
  `https://api.<domain>/api/health` and the frontend.
