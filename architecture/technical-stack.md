# Technical Stack

This page summarizes the technologies used by PersonaOS and points to the code that owns each responsibility. Package versions and workspace scripts are maintained in the root and app `package.json` files.

## Architecture

PersonaOS is a Bun workspace monorepo coordinated by Turborepo. The public site and dashboard, backend API, and standalone 3D experience build and run as separate applications.

```text
Next.js frontend ── REST / tRPC ── Express API on Bun
                                         ├── Supabase Auth
                                         ├── PostgreSQL + Drizzle
                                         ├── Supabase Storage
                                         └── external providers

Backend world export ── static world.json ──> Vite 3D experience
```

## Workspace

| Workspace            | Main technologies                                   | Responsibility                                                                        |
| -------------------- | --------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `apps/frontend`      | Next.js App Router, React, TypeScript, Tailwind CSS | Public pages, dashboard, Markdown editor and server-side data loading                 |
| `apps/backend`       | Express, Bun, tRPC, Drizzle, Zod                    | Authentication, API contracts, domain services, provider integrations and persistence |
| `apps/experience-3d` | Vite, React Three Fiber, Three.js                   | Snapshot-driven stadium portfolio and accessible content fallback                     |
| `packages/3d-system` | React Three Fiber, Three.js                         | Shared 3D components, tokens and fixtures                                             |

The root `package.json` declares Bun workspaces and delegates multi-package tasks to Turborepo. Each workspace's package scripts define its actual dev, test, lint and build commands. The repository pins Bun in `package.json` and uses the same version in Docker and CI.

## Frontend

The frontend uses the App Router under `apps/frontend/src/app`. Route files compose feature components; shared UI is grouped under `src/components`. Content loaders and schemas live in `src/content`, and server/client API helpers live in `src/lib`.

Auth state, content reads and writes, and provider data flow through the PersonaOS backend. Server-only modules read the internal API URL and secrets; browser code uses same-origin routes and must never receive backend credentials. Markdown is validated and parsed by the shared content pipeline before rendering.

## Backend

`apps/backend/src/server.ts` mounts Express REST routers and authenticated tRPC at `/api/trpc`. REST/OpenAPI is used for authentication, public contracts, health and docs, and binary downloads. tRPC is used for typed dashboard workflows. Both should delegate to the same service when they perform the same operation.

The backend keeps thin adapters in `src/routes` and `src/trpc`, domain rules and provider adapters in `src/services`, and Drizzle persistence in `src/db`. Runtime environment parsing and required-variable checks live in `src/config.ts`. One-off imports, exports and bucket setup live in `src/scripts`.

## Data and identity

Supabase Auth owns user identities. PostgreSQL stores profiles, ownership metadata, provider settings, caches and normalized statistics. Drizzle schema and tracked SQL migrations are the source of truth for backend-owned database changes. Supabase Storage holds Markdown documents and larger assets. The backend's service-role key is server-only.

See [Content storage](content-storage-v1.md) for the Markdown contract and [Authentication flow](auth-flow.md) for the cookie and OAuth boundaries.

## 3D delivery

The 3D app is a separate Vite build so its scene dependencies do not enter the main frontend bundle. The backend assembles a published profile snapshot; the 3D app loads that static JSON once and does not make runtime API calls. `packages/3d-system` contains shared scene primitives.

See [3D experience](3d-experience.md) for the current renderer and payload flow.

## Quality and operations

Frontend, backend, 3D app, and shared package use TypeScript and Vitest. ESLint and Prettier are configured at the workspace level. CI runs lint, tests, builds, dependency auditing and Semgrep. Development and production application commands run in Docker; the operational procedures are in the repository's development and deployment guides.

## Where to change things

- Public route or page composition: `apps/frontend/src/app`
- Shared feature UI: `apps/frontend/src/components`
- Content loading, validation or serialization: `apps/frontend/src/content`
- HTTP endpoint: `apps/backend/src/routes`
- Dashboard procedure: `apps/backend/src/trpc/router.ts`
- Provider or domain behavior: `apps/backend/src/services`
- Database schema or query: `apps/backend/src/db` and `apps/backend/drizzle`
- Stadium renderer: `apps/experience-3d/src`
- Shared 3D primitive: `packages/3d-system/src`

For the full subsystem map and extension boundaries, see the [Codebase Guide](codebase-guide.md).
