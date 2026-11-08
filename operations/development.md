# Development Guide

PersonaOS currently has a Next.js frontend, an Express/Bun backend, and a separate Vite-based 3D experience app.

## Primary App

- `apps/frontend`: Next.js 16 app
- `apps/backend`: Express/Bun API
- `apps/experience-3d`: Vite 3D experience app

For a complete map of the technologies in this repository and how they fit together, see [technical-stack.md](../architecture/technical-stack.md).

## Recommended Workflows

### Docker + Caddy

```bash
make up
make logs
make down
```

Make prints each command while it runs, including Docker output during the initial image pulls.

Local endpoints:

- `http://localhost`: frontend through Caddy
- `http://localhost/api`: backend through Caddy
- `http://localhost:3000`: direct frontend
- `http://localhost:5173`: direct 3D experience
- `http://localhost:4000/api`: direct backend

### Direct Bun Development

```bash
cd apps/frontend
bun install
bun run dev
```

```bash
cd apps/backend
bun install
bun run dev
```

```bash
cd apps/experience-3d
bun install
bun run dev
```

The current 3D flow is backend-shaped and snapshot-driven: one protected world endpoint, one exported `world.json`, then no follow-up runtime API calls in the 3D app. The renderer itself is the rebuilt scroll-driven stadium experience documented in [3D experience current state](../architecture/3d-experience.md).

## Database Workflow

Use local Supabase for local development so test data does not pollute hosted projects. Supabase is managed by its CLI, which runs the local Supabase services in Docker. Drizzle owns backend Postgres migrations and records applied migrations in the `drizzle.__drizzle_migrations` table.

```bash
make init-local
```

Copy the local `SUPABASE_URL` and `service_role` key from `make supabase-status` into `.env.backend`. Frontend-only values such as `NEXT_PUBLIC_DOMAIN` live in `.env.frontend`. Keep browser API calls same-origin with `NEXT_PUBLIC_API_URL=/api`; direct `localhost:3000` development is proxied through the Next rewrite. Docker-based backend commands use `host.docker.internal` to reach local Supabase, while `SUPABASE_PUBLIC_URL=http://localhost:54321` keeps OAuth redirects browser-visible.

For local Google OAuth, the Google client must allow this redirect URI exactly:

```text
http://localhost:54321/auth/v1/callback
```

Local Supabase Google auth also keeps `skip_nonce_check = true` in `supabase/config.toml`.

The app auth flow is backend-owned. The frontend redirects to `/api/auth/oauth/google`, the backend stores the Supabase PKCE verifier in an HTTP-only callback cookie, and `/api/auth/callback` exchanges the code, creates a missing profile row, and sets the app session cookie. See [Auth flow](../architecture/auth-flow.md).

To inspect the local database with a lightweight container:

```bash
make db-ui
```

Open `http://localhost:8080` and connect with PostgreSQL server `host.docker.internal:54322`, database `postgres`, user `postgres`, and password `postgres`.

Use reset only when you intentionally want to wipe local Supabase data:

```bash
make supabase-reset
```

Create a local user either through Supabase Studio or with `POST /api/account/signup`. Then import the starter Markdown into Supabase Storage and `content_documents` using that account email:

```bash
EMAIL=<your-account-email> make init-content
```

Use `docker-compose -f docker-compose.dev.yml run --rm -e SUPABASE_URL=http://host.docker.internal:54321 backend bun run import:content --email=<your-account-email> --dry-run` first when checking path mappings. The importer is idempotent: it overwrites the generated Markdown object in the `content-md` bucket and upserts the `content_documents` row.

For production, provide production environment variables in the shell and run:

```bash
DATABASE_URL=<production-postgres-url> bun run init:production
```

Production init only applies Drizzle migrations. It does not start Supabase, reset the database, or create users. Markdown starter content uses the import script because it must upload file bytes into Supabase Storage as well as populate database metadata. Do not edit hosted database schema directly from the dashboard once migrations exist.

Drizzle schema lives in `apps/backend/src/db/schema.ts`. Drizzle SQL migrations live in `apps/backend/drizzle`. Supabase client still handles Auth and Storage.

To replace existing project documents with formatted README snapshots and copied repository images, first provision the private asset bucket, then run a dry run:

```bash
docker-compose -f docker-compose.dev.yml run --rm backend bun run ensure:project-assets-bucket
docker-compose -f docker-compose.dev.yml run --rm backend bun run import:project-readmes --email=<your-account-email> --dry-run
```

Run the same command without `--dry-run` to save drafts. Add `--preserve-status` only when deliberately republishing existing projects.

## Verification

Tests should be grouped under each workspace's `src/__tests__/` folder. Mirror the source area inside that folder when it helps scanning, for example `src/__tests__/routes` or `src/__tests__/components`.

```bash
bun run test
bun run lint
bun run build
```

For backend-only checks:

```bash
bun run test:backend
bun run build:backend
bun run lint -- --filter=@personaos/backend
```

For focused 3D checks:

```bash
bun run test -- --filter=@personaos/experience-3d
bun run build -- --filter=@personaos/experience-3d
bun run lint -- --filter=@personaos/experience-3d
```

## Progress Notes

After meaningful architecture or feature changes, add a short dated entry to the [progress log](../progress.md). Keep it factual: what changed, why it matters, and what remains.
