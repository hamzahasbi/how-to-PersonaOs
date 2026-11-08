# PersonaOS Technical Stack

This document explains the tools used in PersonaOS, why they are here, how this repository uses them, and where to go next when you need official documentation.

## Current Architecture

PersonaOS is a Bun workspace monorepo orchestrated by Turborepo. The product has a markdown-driven Next.js frontend, an Express/Bun backend foundation, a separate Vite app for the standalone 3D portfolio experience, and a reusable 3D component package.

```text
PersonaOS
├── apps/frontend        # Main Next.js portfolio app
├── apps/experience-3d   # Vite app for the standalone 3D portfolio
├── apps/backend         # Express/Bun API for product logic and Supabase access
├── packages/3d-system   # Shared React Three Fiber / Three.js primitives
├── docker/caddy         # Caddy reverse proxy config
├── docker-compose*.yml  # Local container stack
└── documentation        # Repo-owned Markdown documentation
```

Authored content remains Markdown-first, while the current runtime reads it through the backend content API. Raw Markdown is stored in Supabase Storage; Postgres indexes ownership, type, publication status, lookup metadata and versions. The frontend parses and renders Markdown through shared loaders and components. See [Content storage v1](content-storage-v1.md) for the storage model.

## Runtime And Package Management

### Bun

Reference: [Bun documentation](https://bun.sh/docs)

Bun is the JavaScript runtime and package manager for this repo. The root `package.json` pins the package manager to `bun@1.4.4` and defines workspaces for `apps/*` and `packages/*`.

How we use it:

- Install dependencies with `bun install`.
- Run app-local scripts such as `bun run dev`, `bun run test`, `bun run lint`, and `bun run build`.
- Keep dependency state in the root `bun.lock`.
- Build the frontend Docker image from the pinned `oven/bun:1.4.4` image.
- Run the backend Express app with `bun --watch src/server.ts`.

Where to look:

- `package.json`
- `bun.lock`
- `apps/frontend/package.json`
- `apps/backend/package.json`
- `docker/dev/Dockerfile`

## Monorepo Orchestration

### Turborepo

Reference: [Turborepo documentation](https://turborepo.com/docs)

Turborepo coordinates package scripts across the Bun workspaces. Package-level `package.json` scripts remain the source of truth; `turbo.json` only defines task dependencies, caching, persistence, and outputs.

How we use it:

- `bun run dev` maps to `turbo run dev --filter=personaos-frontend`.
- `bun run dev:backend` maps to `turbo run dev --filter=@personaos/backend`.
- `bun run build` maps to `turbo run build`.
- `bun run test` maps to `turbo run test`.
- `bun run lint` maps to `turbo run lint`.
- `bun run dev:3d` maps to `turbo run dev --filter=@personaos/experience-3d`.

Where to look:

- `package.json`
- `turbo.json`
- `apps/*/package.json`
- `packages/*/package.json`

## Main Frontend

### Next.js

Reference: [Next.js documentation](https://nextjs.org/docs)

Next.js is the framework for the main portfolio app. This project uses the App Router, so route files live under `apps/frontend/src/app`.

How we use it:

- Public route files are defined as `page.tsx` files under `src/app`.
- The editor lives at `apps/frontend/src/app/editor/page.tsx`.
- Portfolio variants live at `src/app/portfolio/classic/page.tsx` and `src/app/portfolio/immersive/page.tsx`.
- Blog/project detail routes use dynamic segments such as `src/app/blog/[slug]/page.tsx`.
- Server-only content loading uses the `server-only` package to prevent filesystem markdown loaders from being imported into client bundles.
- Development uses Turbopack through `next dev --turbopack`.

Where to look:

- `apps/frontend/src/app`
- `apps/frontend/next.config.js`
- `apps/frontend/src/content`
- `apps/frontend/src/components`

## Backend API

### Express

Reference: [Express documentation](https://expressjs.com/)

Express is the HTTP API layer for backend product logic.

How we use it:

- The backend runs under Bun from `apps/backend/src/server.ts`.
- Routes are mounted under `/api`.
- `/api/health` is the backend smoke-test endpoint.
- `/api/public/cp/{handle}/stats` remains the API-key-protected public profile
  statistics route; it is distinct from Daily CP.
- Daily CP uses the authenticated `dailyCp` tRPC router rather than private REST.

Where to look:

- `apps/backend/src/server.ts`
- `apps/backend/src/routes/health.ts`
- `apps/backend/src/routes/auth.ts`
- `apps/backend/src/routes/cp.ts`
- `apps/backend/src/trpc/router.ts`
- `apps/backend/src/trpc/middleware.ts`

### Daily CP

Daily CP is a typed dashboard workflow with `dailyCp.state`, `.generate`,
`.resolve`, and `.regenerate` procedures. The service is authoritative for the
`Africa/Casablanca` calendar day and next reset timestamp. Prior-day pending
rows are history, not active work, and are excluded from daily outcome counts.

Live providers are Codeforces (official API), AtCoder (the community-backed
AtCoderProblems data), Kattis (public directory parsing with pagination), and
CodeChef (an experimental first-party practice-listing endpoint). Adapters use
fixed backend origins and validate their responses; clients cannot supply an
upstream problem URL or origin.

Generation records an immutable normalized problem snapshot. Statuses are
`pending`, `solved`, `failed`, and `regenerated`. Drizzle migrations `0006` and
`0007` plus the `cp_daily_challenges` schema enforce authenticated owner reads,
backend-only writes, one pending challenge per owner/Casablanca day, and no
repeated external ID for an owner/provider/day. The `anon` and `authenticated`
Data API roles cannot insert, rewrite, or delete challenge history; the backend
`service_role` owns those transitions.
Resolution conditionally updates only an owned pending row. Regeneration fetches
a replacement before a transaction marks the old row regenerated and inserts
the replacement, which preserves the active problem if the provider fails.

State statistics are server-derived for today, all time, and each provider.
`totalGenerated` includes pending rows; regenerated is separate from failed;
solve rate is `solved / (solved + failed)` and is zero when no terminal solve or
failure exists.

### Site Export

Reference: [`documentation/operations/api.md`](../operations/api.md#site-export)

The self-hosted site export snapshots a user's published site into a static
zip they can download and host anywhere. It is a REST route, not tRPC,
because a binary download does not fit tRPC's JSON envelope.

How we use it:

- `GET /api/export/site.zip` is session-cookie authenticated and backs the
  "Export my site" dashboard card (personal tab).
- The service fetches the owner's own published routes from the frontend over
  the internal network as an anonymous visitor, so drafts are excluded by
  construction rather than filtered after the fact.
- HTML/CSS same-origin links get rewritten to relative paths; skill icons are
  bundled into the archive instead of pointing back at PersonaOS.
- Any non-200 page, cross-path redirect, leftover `/api/` reference, or `..`
  path segment aborts the export rather than shipping a partial site.

Where to look:

- `apps/backend/src/services/site-export/index.ts`
- `apps/backend/src/services/site-export/routes.ts`
- `apps/backend/src/services/site-export/rewrite.ts`
- `apps/backend/src/services/site-export/snapshot.ts`
- `apps/backend/src/services/site-export/templates.ts`
- `apps/backend/src/services/site-export/package.ts`
- `apps/backend/src/services/site-export/documents.ts`
- `apps/backend/src/routes/export.ts`
- `docs/superpowers/specs/2026-07-18-self-hosted-site-export-design.md`

### tRPC

Reference: [tRPC documentation](https://trpc.io/docs)

tRPC is the internal typed API layer for dashboard workflows. It is mounted
inside the existing Express backend rather than replacing Express.

How we use it:

- The backend mounts tRPC under `/api/trpc`.
- Dashboard-only commands and queries can move to tRPC when frontend/backend
  type drift is more important than public API stability.
- `content.save` shares the same backend save service as
  `PUT /api/content/:type/:slug`.
- `github.importProject` lets the dashboard import a GitHub repository README
  as a draft project while keeping the GitHub token on the backend.
- REST/OpenAPI remains the stable contract for public reads, auth/OAuth,
  docs, and the 3D world snapshot endpoints.

Where to look:

- `apps/backend/src/trpc/router.ts`
- `apps/backend/src/trpc/middleware.ts`
- `apps/backend/src/services/content-save.ts`
- `apps/backend/src/services/github-project-import.ts`
- `apps/frontend/src/content/save-content.ts`
- `documentation/architecture/dashboard-trpc-migration.md`

### Supabase

References:

- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [Supabase database documentation](https://supabase.com/docs/guides/database/overview)
- [Supabase local development](https://supabase.com/docs/guides/local-development)
- [Supabase database migrations](https://supabase.com/docs/guides/deployment/database-migrations)

Supabase provides Auth and Postgres. The backend uses server-side Supabase credentials; frontend code should not receive the service-role key.

How we use it:

- Supabase Auth owns user identity.
- `profiles.id` references `auth.users(id)`.
- Postgres stores profile details, portfolios, blog posts, skills, resume data, and competitive programming tracking.
- Flexible personal details live in JSONB columns in `profile_details`.
- Row Level Security policies live with the schema SQL.
- Local development should use a local Supabase stack so test data does not pollute hosted projects.
- Local Supabase is managed through Docker by the Supabase CLI and wrapped by Make targets.

Where to look:

- `apps/backend/src/supabase.ts`
- `apps/backend/src/config.ts`
- `apps/backend/src/db/schema.v0.sql`
- `apps/backend/src/db/schema.sql`
- `supabase/migrations/20260625000000_content_storage_v0.sql`
- `documentation/architecture/content-storage-v1.md`
- `.env.backend.example`
- `.env.frontend.example`

### Content Storage V1

Reference: [content-storage-v1.md](content-storage-v1.md)

Markdown remains canonical for v1. The dashboard editor should save raw `.md` files to Supabase Storage through the backend. Postgres should store only the metadata needed to find, authorize, and publish those files.

How we use it:

- Supabase Storage stores raw Markdown objects.
- Postgres stores `content_documents` metadata: owner, type, slug, storage path, status, and version.
- `profiles` and `content_documents` for the v0 Markdown storage path live in `apps/backend/src/db/schema.ts`.
- `apps/backend/src/db/schema.sql` is not the v0 deployment source; keep it as a broader future data-model draft for now.
- The backend validates Markdown/frontmatter before saving.
- Next.js currently loads Markdown through the backend and keeps the existing parser and renderer pipeline.
- The current 3D direction uses one backend-shaped world payload built from resume + personal content, then exports a static `world.json` snapshot for the 3D app.
- RustFS is deferred until S3-compatible local/prod parity is needed.

### SQL Migrations

Drizzle owns backend Postgres migrations. Keep database structure and RLS policies in tracked migrations rather than dashboard-only changes.

Current state:

- `apps/backend/src/db/schema.ts` is the Drizzle schema for local Supabase content storage.
- `apps/backend/drizzle/0000_content_storage_v0.sql` applies the v0 schema, RLS policies, and `content-md` storage bucket.
- `apps/backend/src/db/schema.sql` is an older broad schema draft and should not be wired into deployment yet.
- Starter Markdown data is imported through `apps/backend/src/scripts/import-starter-content.ts` because it must upload files to storage.

Schema update flow:

```bash
bun run --cwd apps/backend db:generate
make init-local
bun run build:backend
bun run test:backend
```

### Drizzle

Reference: [Drizzle ORM documentation](https://orm.drizzle.team/docs/overview)

Drizzle is the typed Postgres layer for Express backend code.

How we use it:

- Drizzle migrations are the source of truth for backend-owned Postgres schema and RLS.
- Drizzle schema files provide TypeScript-safe queries.
- Express services use Drizzle for app queries instead of ad hoc query strings.
- Supabase client still handles Auth and Storage.

Current state:

- Drizzle is installed in `apps/backend`.
- The first schema covers `profiles` and `content_documents`.
- Existing content storage code still uses `@supabase/supabase-js` for Storage upload/download and Auth admin calls.

## Data Model

The first backend data model is intentionally small:

- `profiles`: stable user-owned identity fields for account bootstrap.
- `content_documents`: metadata index for raw Markdown objects in Supabase Storage.
- `cp_daily_challenges`: Drizzle-managed immutable Daily CP snapshots,
  lifecycle status, and provider/date uniqueness invariants.

The richer normalized tables for profile details, portfolio sections, blog posts,
skills, and resume entities remain in the archived `schema.sql` draft until the
Markdown-first v0 path is replaced. Daily CP is not part of that draft: it is
defined by the Drizzle schema and migration `0006_daily_cp.sql`.

### React

Reference: [React documentation](https://react.dev/learn)

React is the UI component model used by Next.js, the 3D Vite app, and the shared 3D package.

How we use it:

- Components are PascalCase functions.
- Server-rendered app pages compose content and page shell components.
- Interactive editor components live under `apps/frontend/src/components/editor`.
- Shared renderers live under `apps/frontend/src/components/portfolio` and `apps/frontend/src/components/content`.
- 3D UI primitives live in `packages/3d-system/src`.

Where to look:

- `apps/frontend/src/components`
- `apps/experience-3d/src`
- `packages/3d-system/src`

### TypeScript

Reference: [TypeScript documentation](https://www.typescriptlang.org/docs/)

TypeScript is used across all active apps and packages.

How we use it:

- Content schemas export inferred types from Zod.
- Components and loaders use typed props and return values.
- The Vite 3D app and 3D package run `tsc --noEmit` during build to typecheck without writing compiled files.

Where to look:

- `apps/frontend/tsconfig.json`
- `apps/experience-3d/tsconfig.json`
- `packages/3d-system/tsconfig.json`
- `apps/frontend/src/types`

## Styling And UI Development

### Tailwind CSS

Reference: [Tailwind CSS with Next.js](https://tailwindcss.com/docs/installation/framework-guides/nextjs)

Tailwind CSS provides utility classes for the main frontend.

How we use it:

- Tailwind scans `apps/frontend/src/app` and `apps/frontend/src/components`.
- Global styles live in `apps/frontend/src/app/globals.css`.
- The Tailwind config currently extends only a small set of background utilities.

Where to look:

- `apps/frontend/tailwind.config.js`
- `apps/frontend/postcss.config.js`
- `apps/frontend/src/app/globals.css`

### Storybook

Reference: [Storybook documentation](https://storybook.js.org/docs)

Storybook is used to inspect UI components outside the full application.

How we use it:

- The main frontend exposes `bun run storybook` on port `6006`.
- The 3D system exposes `bun run storybook` on port `6007` through the root `storybook:3d` script.
- Storybook telemetry is disabled in scripts with `STORYBOOK_DISABLE_TELEMETRY=1`.

Where to look:

- `apps/frontend/src/components/design-system/resume-system.stories.tsx`
- `packages/3d-system/src/3d-system.stories.tsx`
- `apps/frontend/package.json`
- `packages/3d-system/package.json`

### FullCalendar

Reference: [FullCalendar documentation](https://fullcalendar.io/docs/react)

FullCalendar renders the interactive calendar view in the dashboard Contests section.

How we use it:

- The Contests section at `/dashboard?section=contests` uses FullCalendar v6 to display a
  month grid of competitive-programming contests.
- Dependencies are pinned to version 6.1.21: `@fullcalendar/react`, `@fullcalendar/core`,
  `@fullcalendar/daygrid`, and `@fullcalendar/interaction`.
- The calendar is loaded via `next/dynamic` with `ssr: false` so it stays out of the initial
  bundle.

Where to look:

- `apps/frontend/src/components/dashboard/contests-panel.tsx`
- `apps/frontend/package.json`

## Content System

### Markdown

Reference: [Markdown Guide basic syntax](https://www.markdownguide.org/basic-syntax/)

Markdown is the current content source for PersonaOS. This is the central product model for v1.

How we use it:

- Portfolio data lives in `apps/frontend/content/portfolio/default.md`.
- Blog posts live in `apps/frontend/content/blog`.
- Projects live in `apps/frontend/content/projects`.
- Professional profile and experience live in `apps/frontend/content/professional`.
- Resume profile frontmatter owns renderable sections such as `skillSections`, `socials`, `tools`, and structured `languages`.
- Personal profile content lives in `apps/frontend/content/personal`.
- Imported raw source content lives under `apps/frontend/content/imports`.

Where to look:

- `apps/frontend/content`
- `apps/frontend/src/content/load-portfolio.ts`
- `apps/frontend/src/content/collections.ts`

### gray-matter

Reference: [gray-matter repository](https://github.com/jonschlinkert/gray-matter)

`gray-matter` parses markdown frontmatter.

How we use it:

- Markdown files contain YAML metadata at the top.
- Loaders parse the metadata and body separately.
- Zod validates the parsed metadata before the app renders it.

Where to look:

- `apps/frontend/src/content/load-portfolio.ts`
- `apps/frontend/src/content/collections.ts`

### Zod

Reference: [Zod documentation](https://zod.dev/)

Zod validates structured data at runtime and provides TypeScript types from schemas.

How we use it:

- `portfolioDocumentSchema` validates portfolio frontmatter sections.
- Collection schemas validate blog, project, personal, resume, and experience frontmatter.
- Invalid content fails early in loaders rather than producing broken pages later.

Where to look:

- `apps/frontend/src/content/schema.ts`
- `apps/frontend/src/content/collections.ts`

### react-markdown and remark-gfm

References:

- [react-markdown repository](https://github.com/remarkjs/react-markdown)
- [remark-gfm repository](https://github.com/remarkjs/remark-gfm)

`react-markdown` renders markdown into React components. `remark-gfm` adds GitHub Flavored Markdown support such as tables, strikethrough, and task list syntax.

How we use it:

- Markdown body content is rendered through shared content components.
- Blog and project pages use the same markdown renderer so formatting remains consistent.

Where to look:

- `apps/frontend/src/components/content/markdown-content.tsx`
- `apps/frontend/src/components/content/page-shell.tsx`

### Resume Icons

Resume icons are content-driven. The frontend accepts either a full icon URL/path or a normalized icon slug.

How we use it:

- Resume tools and skills provide icon keys from Markdown frontmatter.
- Full URLs and `/` paths render directly through `<img>`.
- Slugs render through the frontend-owned `/skill-icons/[icon]` route.
- That route checks for a matching local asset in `apps/frontend/public/assets/skills` first, then redirects to `go-skill-icons.vercel.app`.
- Failed or text-only icons fall back to initials.
- `/api/*` stays backend-owned, so icon resolution must not live under the Next.js `/api` namespace.

Where to look:

- `apps/frontend/src/components/resume/resume-icon.tsx`
- `apps/frontend/src/app/skill-icons/[icon]/route.ts`
- `apps/frontend/content/professional/profile.md`

### Brand Assets

PersonaOS keeps app-owned brand assets in the frontend package so the logo, favicon, and public navigation stay deployable with the Next.js app.

How we use it:

- `apps/frontend/src/app/icon.svg` is the App Router favicon source. Next.js serves it as `/icon.svg`.
- `apps/frontend/public/assets/brand/personaos-mark.svg` and `.png` are the compact standalone mark.
- `apps/frontend/public/assets/brand/personaos-logo.svg` and `.png` are the horizontal logo used in the public header.
- `apps/frontend/public/assets/brand/personaos-favicon.png` is the PNG favicon/app icon export.
- `apps/frontend/src/components/content/public-header.tsx` renders the visible header logo with an accessible text fallback.

Where to look:

- `apps/frontend/src/app/icon.svg`
- `apps/frontend/public/assets/brand/personaos-mark.svg`
- `apps/frontend/public/assets/brand/personaos-mark.png`
- `apps/frontend/public/assets/brand/personaos-logo.svg`
- `apps/frontend/public/assets/brand/personaos-logo.png`
- `apps/frontend/public/assets/brand/personaos-favicon.png`
- `apps/frontend/src/components/content/public-header.tsx`

## 3D Experience Stack

### Vite

Reference: [Vite documentation](https://vite.dev/guide/)

Vite powers the standalone `experience-3d` app. This keeps the 3D portfolio isolated from the main Next.js bundle while still allowing the shared 3D package to evolve in one place.

How we use it:

- `bun run dev:3d` runs `turbo run dev --filter=@personaos/experience-3d`.
- `apps/experience-3d` imports shared primitives from `@personaos/3d-system`.
- Build runs TypeScript checking first, then `vite build`.
- The current renderer is a cinematic, scroll-driven stadium experience loaded from one pre-generated `public/data/world.json` snapshot instead of live per-zone API reads.

Where to look:

- `apps/experience-3d/vite.config.ts`
- `apps/experience-3d/src/app.tsx`
- `apps/experience-3d/package.json`

### Three.js

Reference: [Three.js documentation](https://threejs.org/docs/)

Three.js is the core 3D rendering library.

How we use it:

- It is a dependency of the 3D experience and a peer dependency of the shared 3D package.
- Low-level 3D behavior should stay in `packages/3d-system` when it is reusable.

Where to look:

- `packages/3d-system/src`
- `apps/experience-3d/src`

### React Three Fiber and Drei

References:

- [React Three Fiber documentation](https://r3f.docs.pmnd.rs/getting-started/introduction)
- [Drei documentation](https://drei.docs.pmnd.rs/)

React Three Fiber lets us build Three.js scenes with React components. Drei provides useful helpers on top of React Three Fiber.

How we use it:

- `packages/3d-system` provides reusable 3D scene primitives and fallback renderers.
- `apps/experience-3d` consumes those components for the standalone stadium-world portfolio.
- The shipped experience is not a free-walk game. The stadium GLB is a WebGL backdrop while scroll progress drives camera vantages and content panels.
- The main frontend can later reuse the same package for immersive portfolio views.

Where to look:

- `packages/3d-system/src/persona-map.tsx`
- `packages/3d-system/src/primitives.tsx`
- `packages/3d-system/src/fallback-map.tsx`

## Forms And Editor

### React Hook Form

Reference: [React Hook Form documentation](https://react-hook-form.com/get-started)

React Hook Form manages form state in React.

How we use it:

- It supports the portfolio editor UI.
- Editor components are separated into shell, tabs, block list, and block form panel components.

Where to look:

- `apps/frontend/src/components/editor`
- `apps/frontend/src/app/editor/page.tsx`

## Testing And Quality

### Vitest

Reference: [Vitest documentation](https://vitest.dev/guide/)

Vitest is the unit and component test runner.

How we use it:

- Each app/package has a `test` script that runs `vitest run`.
- Frontend tests cover content loading, schema validation, route behavior, editor behavior, and rendering.
- Backend tests cover Daily CP provider normalization, Casablanca-day behavior,
  state transitions, persistence invariants, and tRPC procedures.
- 3D package tests cover reusable 3D behavior.

Where to look:

- `apps/frontend/vitest.config.ts`
- `apps/backend/vitest.config.ts`
- `apps/frontend/vitest.setup.ts`
- `apps/experience-3d/vitest.config.ts`
- `packages/3d-system/vitest.config.ts`
- `*.test.ts` and `*.test.tsx` files

Recommended command:

```bash
bun run test
```

For the main frontend only:

```bash
cd apps/frontend
bun run test
```

For the backend only:

```bash
bun run test:backend
```

### Testing Library

Reference: [Testing Library documentation](https://testing-library.com/docs/)

Testing Library helps test React components through user-visible behavior.

How we use it:

- Component tests render UI and assert visible output.
- `@testing-library/jest-dom` adds DOM-specific assertions.
- `@testing-library/user-event` supports interaction tests.

Where to look:

- `apps/frontend/src/components/**/*.test.tsx`
- `apps/experience-3d/src/app.test.tsx`
- `packages/3d-system/src/persona-map.test.tsx`

### ESLint

Reference: [ESLint documentation](https://eslint.org/docs/latest/)

ESLint catches code quality and correctness issues.

How we use it:

- The frontend uses `eslint-config-next`.
- Vite-based workspaces use flat ESLint config with TypeScript support.
- The backend uses flat ESLint config with TypeScript support.
- Root Turborepo scripts can run lint across all configured workspaces.

Where to look:

- `apps/frontend/eslint.config.mjs`
- `apps/backend/eslint.config.mjs`
- `apps/experience-3d/eslint.config.mjs`
- `packages/3d-system/eslint.config.mjs`

Recommended command:

```bash
bun run lint
```

### Prettier

Reference: [Prettier documentation](https://prettier.io/docs/)

Prettier formats code consistently.

How we use it:

- Root scripts expose `bun run format` and `bun run format:check`.
- Use formatting for mechanical cleanup, not as a substitute for reviewing behavior.

Where to look:

- `package.json`

## Local Runtime And Deployment Shape

### Docker Compose

Reference: [Docker Compose documentation](https://docs.docker.com/compose/)

Docker Compose runs the local container stack.

How we use it:

- `docker-compose.dev.yml` runs the frontend, backend, and Caddy.
- All three app containers build from the single `docker/dev/Dockerfile` and
  share one `personaos-dev` image; they differ only by `working_dir` and
  `command`. The image carries the dependency tree only — the source arrives
  via the `.:/app` bind mount.
- The app source is mounted into `/app` for local development.
- Port `3000` exposes the direct Next.js dev server.
- Port `4000` exposes the direct Express API.
- Caddy exposes ports `80` and `443`.

Where to look:

- `docker-compose.dev.yml`
- `docker-compose.yml`
- `docker/dev/Dockerfile`
- `Makefile`

Recommended commands:

```bash
make up
make logs
make down
```

### Caddy

Reference: [Caddy reverse_proxy directive](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy)

Caddy is the local reverse proxy in front of the Next.js dev server.

How we use it:

- `localhost` is served by Caddy.
- `/health` returns `OK`.
- `/api/*` is reverse proxied to `backend:4000`.
- All other requests are reverse proxied to `frontend:3000`.
- Forwarded headers preserve host, protocol, and port information.

Where to look:

- `docker/caddy/Caddyfile`
- `docker-compose.dev.yml`

## Make Workflow

The root `Makefile` wraps Docker stack shortcuts. App checks stay in Turborepo/Bun scripts.

How we use it:

- `make up` starts the Docker development stack and creates `.env.frontend`/`.env.backend` from their examples if needed.
- `make down` stops it.
- `make logs`, `make logs-frontend`, `make logs-backend`, and `make logs-caddy` inspect logs.
- `make build` and `make rebuild` build containers.
- `make ps`, `make clean`, and `make restart` cover local stack maintenance.
- `make supabase-start`, `make supabase-stop`, `make supabase-status`, and `make supabase-reset` manage the local Supabase Docker services through the Supabase CLI.

Where to look:

- `Makefile`

## Important Workflows

### Start The Full Local Stack

```bash
make up
```

Then open:

- `http://localhost` through Caddy
- `http://localhost:3000` directly against Next.js
- `http://localhost/api/health` through Caddy
- `http://localhost:4000/api/health` directly against Express

### Start The Main Frontend Without Docker

```bash
cd apps/frontend
bun install
bun run dev
```

### Start The Backend Without Docker

```bash
cd apps/backend
bun install
bun run dev
```

### Start The 3D Experience

```bash
bun run dev:3d
```

### Run Verification

```bash
bun run test
bun run lint
bun run build
```

For narrow frontend verification:

```bash
cd apps/frontend
bun run test
bun run lint
bun run build
```

For narrow backend verification:

```bash
bun run test:backend
bun run build:backend
bun run lint -- --filter=@personaos/backend
```

## Progress Documentation

Use the [progress log](../progress.md) for short dated notes about shipped milestones and durable decisions. Keep details here in `technical-stack.md`; keep the progress log chronological and concise.

## How To Add New Work

### Add A Blog Post

1. Add a markdown file to `apps/frontend/content/blog`.
2. Include frontmatter matching the blog schema in `apps/frontend/src/content/collections.ts`.
3. Run frontend tests and lint.

When the backend editor owns blog content, write to `blog_posts` and keep markdown import/export available.

### Add A Project Page

1. Add a markdown file to `apps/frontend/content/projects`.
2. Include `title`, `description`, `date`, `link`, and `tags`.
3. The route `apps/frontend/src/app/projects/[slug]/page.tsx` will load it by slug.

### Add A New Portfolio Block Type

1. Add the block schema to `apps/frontend/src/content/schema.ts`.
2. Add rendering support in `apps/frontend/src/components/portfolio/block-registry.tsx`.
3. Add or update tests near the schema and renderer.

### Add Reusable 3D UI

1. Put reusable primitives in `packages/3d-system/src`.
2. Add tests and Storybook stories there.
3. Consume the primitive from `apps/experience-3d` first.
4. Integrate into the main frontend only after the primitive is stable.

### Add A Database Change

1. Update `apps/backend/src/db/schema.ts`.
2. Run `bun run --cwd apps/backend db:generate`.
3. Review the generated SQL in `apps/backend/drizzle`.
4. Run backend build and tests.

## Backend Status

The backend foundation exists in `apps/backend`. Drizzle now owns backend Postgres schema and migrations; Supabase client remains for Auth and Storage.
