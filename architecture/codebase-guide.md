# PersonaOS Codebase Guide

This guide is the entry point for understanding and changing the current PersonaOS codebase. It describes the runtime boundaries, the main feature flows, where code belongs, and how to validate changes. Use the linked focused documents for detailed setup and operational procedures.

## Product and system shape

PersonaOS is a personal publishing platform with a public portfolio, an authenticated dashboard, provider-backed personal data, and a separately deployable 3D portfolio. It is a Bun workspace monorepo coordinated by Turborepo.

```text
Browser
  ├── Next.js frontend (public pages, auth UI, dashboard)
  ├── Vite + React Three Fiber experience (standalone 3D site)
  └── /api requests ──> Express API running on Bun
                            ├── Supabase Auth (identity)
                            ├── PostgreSQL + Drizzle (metadata, settings, caches)
                            ├── Supabase Storage (Markdown, world and project assets)
                            └── external providers (culture, football, CP, GitHub)

Backend world snapshot ──> apps/experience-3d/public/data/world.json
```

The backend is the trust boundary for credentials, ownership checks, content status, provider requests, and writes. Frontend server components call the backend directly using the internal API URL; browser requests use same-origin `/api` rewrites or explicit app API routes. The public API uses API key and secret headers where configured. Dashboard requests use the authenticated HTTP-only session cookie.

## Workspace map

| Path                            | Responsibility                                                                 | Start here                                                          |
| ------------------------------- | ------------------------------------------------------------------------------ | ------------------------------------------------------------------- |
| `apps/frontend`                 | Next.js App Router site, dashboard, editor and server-side loaders             | `src/app`, `src/components`, `src/content`, `src/lib`               |
| `apps/backend`                  | Express API, business services, persistence adapters, integrations and scripts | `src/server.ts`, `src/routes`, `src/trpc`, `src/services`, `src/db` |
| `apps/experience-3d`            | Independent Vite application for the stadium portfolio                         | `src/app.tsx`, `src/stadium-experience.tsx`, `src/world-fixture.ts` |
| `packages/3d-system`            | Shared Three.js / React Three Fiber components and tokens                      | `src/index.ts`, `src/persona-map.tsx`                               |
| `docker` and root Compose files | Development containers, ingress and production topology                        | `docker/dev`, `docker/caddy`, `docker/edge`, `docker-compose*.yml`  |
| `documentation`                 | Architecture, setup, API and deployment references                             | `documentation/README.md`                                           |

Root scripts are orchestration shortcuts, not separate app implementations. The root package pins Bun, declares workspaces, and routes common tasks through Turbo. Package-level scripts in each app remain the source of truth.

## Frontend

### Routing and presentation

`apps/frontend/src/app` uses the Next.js App Router. Route folders own page entry points and route handlers; shared presentation lives under `src/components` grouped by feature. Public route families include home, projects, blog, resume, professional, personal, culture, football and competitive pages. Dashboard, editor, login, account and password flows are private or account-oriented surfaces.

The root layout (`src/app/layout.tsx`) composes site chrome, theme providers, navigation, portfolio links, auth state and shared error reporting. `src/app/api` contains Next route handlers used for same-origin provider proxies and logs; these complement the backend API and should not duplicate backend business rules.

### Content flow

`src/content` is the frontend's domain boundary for data loading, parsing and saving. Public and dashboard Markdown reads are implemented in `api-content.ts`, which calls backend content endpoints. Markdown parsing and frontmatter validation are in `markdown.ts` and `schema.ts`; shared render components are in `components/content/markdown-content.tsx`.

The backend stores raw Markdown in Supabase Storage and stores lookup, owner, type, status, title and version metadata in Postgres. Public loaders receive only published documents; dashboard loaders use the session cookie and can read editable states. The editor serializes portfolio blocks and saves through the shared content save workflow. For storage and migration invariants, see [Content storage v1](content-storage-v1.md).

### Client/server boundaries

Prefer server components for public data loading and page composition. `server-only` marks loaders that need secrets or `next/headers`. Client components should receive normalized props and call typed frontend wrappers for browser interactions. `src/lib/api.ts` distinguishes the browser API base from the server-to-server API base. Do not expose backend secrets through `NEXT_PUBLIC_*` variables.

## Backend

### Request flow

`src/server.ts` creates the Express app and installs security headers, request logging, CORS, JSON parsing and error handling. It mounts REST routers under `/api` and the authenticated tRPC adapter at `/api/trpc`. Public and private route groups are separated explicitly. Public integration filters run before public routers, and protected public reads use the API key middleware. The server handles SIGINT/SIGTERM by closing the listener and flushing logs.

Use REST for stable/public API contracts, authentication and OAuth callbacks, health/docs, binary downloads, and public resource routes. Use tRPC for authenticated dashboard workflows that benefit from shared frontend/backend types. Both should call the same domain service when they represent the same operation; keep route/procedure handlers thin.

### Backend layers

| Layer                           | Location        | Owns                                                                          |
| ------------------------------- | --------------- | ----------------------------------------------------------------------------- |
| HTTP adapters                   | `src/routes`    | URL parsing, auth middleware, status codes and response shape                 |
| Typed dashboard adapter         | `src/trpc`      | tRPC context, authentication gate and procedure input/output contracts        |
| Domain and integration services | `src/services`  | Validation, provider normalization, business rules, caching and orchestration |
| Persistence                     | `src/db`        | Drizzle queries, database stores and domain persistence interfaces            |
| Runtime config                  | `src/config.ts` | Required env checks, parsing, defaults and startup validation                 |
| Scripts                         | `src/scripts`   | One-off imports, exports and bucket setup                                     |

When adding provider logic, put provider-specific transport and normalization in an adapter, then keep policy and orchestration in the feature service. Inject stores/adapters into services where practical; this keeps domain tests independent of live services.

### Main service areas

- **Content and profiles:** Markdown documents, publishing states, profile lookup, README import and self-hosted site export.
- **Culture:** Goodreads/Storygraph, Spotify, Trakt and Last.fm connections, normalized data and cache.
- **Football:** match and fixture reads, provider integrations, detail caching and publication settings.
- **Competitive programming:** provider registry, rating/stat sync and public profile stats.
- **Daily CP and contests:** daily challenge lifecycle and contest schedule adapter/service.
- **World and assets:** 3D world snapshot storage, project assets, uploads and signed Cloudinary uploads.
- **Security/support:** API-key access, Turnstile, public integration filtering, CORS and request/error logging.

## Data and persistence

Supabase Auth owns user identities. `profiles.id` references the Supabase auth user. Drizzle schema is declared in `apps/backend/src/db/schema.ts`; tracked SQL migrations are under `apps/backend/drizzle`. Treat migrations as the change history for hosted schema and do not apply ad hoc dashboard schema edits.

The database holds relational ownership and lookup metadata, integration credentials/settings, provider caches and normalized statistics. Markdown documents and larger exported assets live in Supabase Storage. The backend uses the service-role credential for server operations; it must never reach browser code. Storage paths and signed URL behavior are part of the content boundary, not a frontend concern.

Schema constraints and Row Level Security policies enforce ownership and write boundaries for data paths that may be exposed through Supabase's Data API. For every new table, decide explicitly whether access is backend-only or user-scoped and encode that in SQL policies and constraints as well as application checks.

## API and authentication boundaries

- Auth and account routes live in `src/routes/auth.ts` and `account.ts`. Supabase verifies identity; the backend issues/clears the application session cookie and resolves the user for private requests.
- tRPC middleware authenticates the request before constructing the dashboard context. Procedures receive `userId` and feature services/stores; they should scope every read/write to that ID.
- Public read routes can require `x-personaos-api-key` and `x-personaos-api-secret`. Never put those values in client bundles.
- CORS is controlled by `ALLOWED_ORIGINS`; production auth callbacks use `PUBLIC_APP_ORIGIN` rather than trusting request Host headers.
- OAuth/provider secrets are stored and used server-side. Public visibility settings determine which connected data can be returned.
- The frontend's Next rewrites proxy `/api/*` to the backend for browser calls. App Router API handlers take precedence and are used for app-specific proxy behavior.

See [Authentication flow](auth-flow.md), [API guide](../operations/api.md) and the generated OpenAPI route for exact contracts.

## Standalone 3D experience

`apps/experience-3d` is a separate Vite build so Three.js and scene assets stay out of the main Next bundle. It renders the published world snapshot from `public/data/world.json`, with scene components and content-only fallback behavior in its own source tree. `packages/3d-system` holds reusable primitives shared with the standalone experience. The backend export script produces the world snapshot; regenerate it through the documented backend command rather than editing derived JSON by hand.

The production edge image serves the built 3D site alongside frontend/backend routing. Development runs it as its own Compose service. See [3D experience](3d-experience.md) and [Development](../operations/development.md).

## Configuration and runtime

Use `.env.frontend.example`, `.env.backend.example` and `.env.compose.example` as templates. The backend validates required credentials at startup in `src/config.ts`; optional integrations can be absent and report an unconfigured state. Environment values differ between local Supabase and production, so follow the relevant operations guide rather than copying values between environments.

Development runs application commands in Docker Compose. Caddy routes local hostnames and TLS; frontend, backend and 3D app are separate services. Production uses prebuilt images and the edge proxy; frontend build-time public values are baked into the image while secrets remain runtime-only. See [Development](../operations/development.md) and [Deployment](../operations/deployment.md).

## Change workflow

1. Find the owning feature under the app/package map above and read adjacent code/tests.
2. Keep contracts at the boundary: validate REST/tRPC inputs, normalize provider payloads, and scope private data to the authenticated owner.
3. Add or update a migration for schema changes; update Drizzle schema and persistence stores together.
4. Add focused tests in the owning app's `src/__tests__` tree or beside the relevant route handler where that app already does so.
5. Run relevant Dockerized checks from the [development guide](../operations/development.md). Do not run host Bun commands for app tests, lint, builds, migrations or dev servers.
6. Update this guide or a focused architecture doc when behavior/boundaries changed; add a dated entry to the [progress log](../progress.md) for meaningful architectural or feature changes.

## Further reading

- [Technical stack](technical-stack.md)
- [Architecture decisions](architecture-decision-records.md)
- [Authentication flow](auth-flow.md)
- [Content storage](content-storage-v1.md)
- [Dashboard tRPC migration](dashboard-trpc-migration.md)
- [3D experience](3d-experience.md)
- [Development and database setup](../operations/development.md)
- [API contracts and testing](../operations/api.md)
- [Deployment](../operations/deployment.md)
- [Culture integration setup](../operations/culture-integrations-setup.md)
- [Dependency security](../operations/dependency-security.md)
