# PersonaOS Progress

This file is the durable project log. Add a dated entry after meaningful architecture or feature work. Keep entries factual: what changed, why it matters, verification when known, and remaining follow-ups.

## 2026-10-27

### Culture page redesign

- Rebuilt the public `/culture` page as focused client components under
  `apps/frontend/src/components/culture/`: a full-bleed hero whose photo follows
  the site theme (projector beams in dark mode, a grand library in light mode,
  both free Unsplash photos served from `public/assets/culture`),
  a tab strip with an animated active pill, a crossfading panel switch that
  holds the previous height, a per-section spotlight item, a segmented stats
  band, and cover cards that tilt toward the pointer with a glare sweep, cover
  zoom, and a caption overlay on hover-capable devices. Captions stay below
  covers on touch devices.
- Entrances are cinematic: titles sharpen out of blur with tightening tracking,
  covers fold in from 3D with a capped stagger. Under `prefers-reduced-motion`
  every effect collapses to a short opacity fade; the preference is read after
  mount so server and client markup match, and `MotionConfig` makes transforms
  instant for those visitors. Data selection lives in a pure, unit-tested
  module; the backend and public API are unchanged. The old
  `app/culture/culture-tabs.tsx` is deleted.
- Fixed a pre-existing hydration mismatch on the "Synced" time by formatting it
  deterministically in UTC.

Verification:

```bash
docker-compose -f docker-compose.dev.yml run --rm --no-deps frontend sh -lc 'cd /app && bun install --frozen-lockfile && cd apps/frontend && bun run test && bun run lint && bun run build'
```

Verification result: 69 frontend test files and 367 tests passed, lint
reported 0 errors (3 pre-existing warnings from the generated `coverage/`
folder), and the Next.js production build compiled with `/culture` as a dynamic
route. Checked in the built-in browser at `http://localhost:3000/culture`: dark
and light hero photos and credits, eyebrow contrast in both modes, tab switch
with the held panel height, cover hover (tilt transform, 1.08 image zoom, caption
overlay, badge), phone width with captions below covers and no horizontal
overflow, and no console errors or hydration warnings. Not checked in a browser:
the reduced-motion path, which is covered by component tests only.

## 2026-10-26

### Culture page and provider integrations

- Added a public `/culture` page with Books, Music, and Film & series sections
  sourced from Goodreads RSS shelves, Spotify, and Trakt. Items render with
  `dir="auto"` so Arabic titles and authors keep their direction, and only
  HTTPS links and provider-hosted images are shown.
- Added `GET /api/public/integrations/:handle/culture`, which returns one
  section per provider with a `not-connected`, `private`, `ready`, `stale`, or
  `unavailable` status. Each section is fetched independently; a provider
  failure keeps the last successful snapshot and does not block the others.
- Added Spotify and Trakt as providers on the owner-scoped
  `provider_credentials` table, with backend-managed OAuth: `connect` starts
  authorization (PKCE for Trakt), `callback` binds the result to the signed-in
  owner and a per-connection generation, token refresh is deduplicated, and
  `DELETE /api/integrations/providers/:provider` disconnects and clears cached
  public data. Secret values are never returned to the frontend.
- Migration `0011_culture_integrations.sql` widens the provider check and adds
  the owner-scoped `culture_cache` and `culture_connections` tables with RLS.
- Replaced the dashboard's Goodreads card with a culture settings panel that
  manages shelf feeds, Spotify and Trakt app credentials, the Public/Private
  toggle, connect and disconnect actions, and a confirmation after the OAuth
  redirect. New configurations start private; OAuth alone publishes nothing.
- Football: RapidAPI fixtures now cache for 48 hours; BSD supplies live
  scores and final status merged onto uniquely matched calendar rows, so no
  RapidAPI polling happens at match time.
- Documented the required accounts, credentials, and callbacks in
  [culture-integrations-setup.md](operations/culture-integrations-setup.md)
  and the endpoints in the API guide and OpenAPI spec.

Verification:

```bash
docker-compose -f docker-compose.dev.yml run --rm backend bun run test
docker-compose -f docker-compose.dev.yml run --rm backend bun run lint
docker-compose -f docker-compose.dev.yml run --rm frontend sh -lc 'cd /app && bun install --frozen-lockfile && cd apps/frontend && bun run test && bun run lint && bun run build'
```

Verification result: backend 497 tests passed with lint and `tsc` clean,
including the DB integration suites run with `RUN_DB_INTEGRATION_TESTS=1`
against local Supabase after applying migration `0011` through
`db:migrate:local`; frontend 63 test files and 339 tests passed, lint reported
0 errors, and the Next.js production build compiled with `/culture` as a
dynamic route. Remaining follow-ups: SIMKL and TMDB are documented options only, and the `/me`
current-reading widget now requires the Goodreads provider to be published.

## 2026-10-23

### Patch Brotli decoder runtimes

- Updated Bun to 1.4.4 in the workspace, CI, and every Bun image pin, and
  updated the frontend runtime to Node 24.22.1. These releases include the
  CVE-2026-111497 Brotli decoder fix.
- Kept all container images digest-pinned and updated runtime documentation.
- Refreshed vulnerable dependencies, including Next.js, proxy-addr, sharp,
  Storybook, Vite, and Vitest. Added a scoped audit exception for `braces`, which
  has no patched release and appears only through frontend build/lint tooling.
- Verified the frozen lockfile, dependency audit, test suite, lint, and build in
  Docker. Tests passed with the CI-equivalent `NEXT_PUBLIC_API_URL=/api` setting.

## 2026-09-28

### Replace Bake shortcuts with GNU Make

- Replaced the root `Bakefile` with a GNU Make `Makefile`, preserving the local
  Docker, Supabase, initialization, and content-import commands.
- Updated active documentation, the `init:local` package script, and CI's
  common-change matcher to use Make.
- Checked target expansion with `make -n` and whitespace with `git diff --check`.

## 2026-08-31

### Calm command dashboard workspace

- Made Overview the default `/dashboard` landing view, combining publishing
  readiness, quick actions, and isolated Daily CP and integration snapshots.
- Grouped navigation into Content, Performance, and System while retaining
  stable `section` and supported `action=new` URLs for direct links, browser
  history, and existing editor flows.
- Added a responsive navigation drawer with keyboard focus handling and
  restoration, plus phone-safe provider and statistics cards that preserve the
  desktop information set without a horizontal-scroll dependency.
- Established the token-backed light and dark calm-command dashboard system
  across shared controls, status treatments, cards, editors, and existing
  dashboard panels. This work made no backend, schema, migration, or API
  contract change.

Verification:

```bash
docker-compose -f docker-compose.dev.yml run --rm frontend sh -lc 'cd /app && bun install --frozen-lockfile && cd apps/frontend && bun run test && bun run lint && bun run build'
```

Verification result: 47 frontend test files and 255 tests passed; lint and the
Next.js production build passed. The user visually approved the redesigned
dashboard before merge.

## 2026-08-29

### Daily CP live dashboard

- Replaced the undeployed private REST prototype with the authenticated typed
  `dailyCp.state`, `.generate`, `.resolve`, and `.regenerate` dashboard flow.
  The public `/api/public/cp/{handle}/stats` profile API remains separate.
- Daily CP uses live backend-controlled sources: the official Codeforces API,
  community-maintained AtCoderProblems data, Kattis public-directory parsing,
  and an experimental first-party CodeChef practice-listing endpoint. It stores
  snapshots, never a local provider catalog, and never accepts a user-supplied
  upstream origin or problem URL.
- The service resets the active view at midnight in `Africa/Casablanca` while
  retaining history. Each generation stores a problem snapshot; the application
  and store transition boundary updates only its status and resolution fields.
  The Drizzle migrations enforce valid provider/status/resolution combinations,
  foreign keys, one pending row per owner/day, and no same-provider problem twice
  per owner/day. Forward migration `0007_daily_cp_access_boundary.sql` narrows RLS
  to authenticated owner reads, revokes all direct `anon`/`authenticated` table
  writes, and retains backend `service_role` CRUD. Regeneration fetches before
  replacing a pending row in one transaction, so an upstream failure preserves
  the active problem.
- Daily CP provider page/ordinal selection uses `node:crypto.randomInt`; Kattis
  retains unrated problems and tries up to three distinct pages even when page one
  is selected. The open dashboard retries failed, stale, or superseded midnight
  refreshes with capped backoff and never overlaps them with a busy mutation.
- The provider selector stays available while a challenge is active; Regenerate may switch providers.
- A failed replacement fetch leaves the current challenge unchanged.
- Statistics are server-derived for today, all time, and provider. Pending rows
  count only toward generated totals; regenerated is not a failure; solve rate
  is `solved / (solved + failed)` (or zero without solved/failed rows).
- Removed the old REST routes, OpenAPI contract, unused selector/catalog/client,
  and conflicting executable Daily CP draft DDL. The source of truth is the
  Drizzle schema and migrations `0006_daily_cp.sql` and
  `0007_daily_cp_access_boundary.sql`.

Verification commands:

```bash
docker-compose -f docker-compose.dev.yml run --rm backend sh -lc 'bun install --frozen-lockfile && bun run test'
docker-compose -f docker-compose.dev.yml run --rm backend sh -lc 'bun install --frozen-lockfile && bun run lint'
docker-compose -f docker-compose.dev.yml run --rm backend sh -lc 'bun install --frozen-lockfile && bun run build'
docker-compose -f docker-compose.dev.yml run --rm frontend sh -lc 'bun install --frozen-lockfile && bun run test'
docker-compose -f docker-compose.dev.yml run --rm frontend sh -lc 'bun install --frozen-lockfile && bun run lint'
docker-compose -f docker-compose.dev.yml run --rm frontend sh -lc 'bun install --frozen-lockfile && bun run build'
```

Verification results:

- The regenerated OpenAPI contract test passed; it retains public CP stats and
  rejects the retired private paths.
- Backend: 322 tests passed (eight opt-in DB tests skipped in the standard
  suite), lint passed, and the TypeScript build passed. The enabled Daily CP
  database-invariant and access-boundary suite then passed all 8 tests against
  local Supabase after applying only the forward `0007` migration.
- Frontend: 219 tests passed, lint passed, and the production build passed.

## 2026-07-18

### Self-Hosted Site Export

- Shipped `GET /api/export/site.zip`, an authenticated (session cookie) REST
  endpoint that snapshots the signed-in owner's published site into a static
  zip, downloadable from a new "Export my site" dashboard card on the
  personal tab.
- Snapshot-of-live-SSR approach: the backend fetches the owner's own
  published routes from the frontend over the internal network
  (`FRONTEND_INTERNAL_URL`, default `http://frontend:3000`) as an anonymous
  visitor, forwarding no cookies — drafts and auth-gated content are excluded
  by construction, not by a post-hoc filter. Exported routes are fixed: `/`,
  `/blog`, `/resume`, `/me`, `/professional`, `/projects`,
  `/portfolio/classic`, `/portfolio/immersive`, plus `/blog/<slug>` and
  `/projects/<slug>` for every published document.
- REST download exception: kept this endpoint on REST/OpenAPI instead of the
  dashboard's tRPC layer, because a binary zip is a poor fit for tRPC's JSON
  envelope and a cookie-authenticated `GET` is just a plain dashboard link.
- Fail-loud guardrails: any non-200 page, a redirect that leaves the exported
  path, a residual `/api/` reference, or a `..` path segment aborts the whole
  export (502 with a `failures` list) rather than shipping a partial site.
  Only one export can run per user at a time (429); 404 if the account has no
  profile handle.
- Snapshot update contract: the zip is a point-in-time export with no
  compatibility promise about its internal layout across versions —
  re-download to pick up new content.

Verification:

- `docker-compose -f docker-compose.dev.yml run --rm backend bun run test src/services/__tests__/ src/routes/__tests__/export.test.ts`
- `docker-compose -f docker-compose.dev.yml run --rm frontend bun run test src/components/dashboard/__tests__/site-export-card.test.tsx`
- Backend site-export suites green (routes 4, rewrite 13, snapshot 7,
  templates 3, package 2, orchestrator 7, proof 1), route tests 4, frontend
  dashboard tests, lint clean in both apps.

Source:

- `docs/superpowers/specs/2026-07-18-self-hosted-site-export-design.md`
- `docs/superpowers/plans/2026-07-18-self-hosted-site-export.md`
- [GitHub issue #58](https://github.com/hamzahasbi/PersonaOS/issues/58)

## 2026-07-15

### Dashboard tRPC Migration Pilot

- Added tRPC as an internal dashboard API layer mounted at `/api/trpc`.
- Migrated dashboard content saves to `content.save` while keeping the existing REST content route in place.
- Extracted shared content save validation into a backend service so REST and tRPC use the same Markdown/frontmatter and owner-id behavior.
- Documented the migration boundary in `documentation/architecture/dashboard-trpc-migration.md` and updated architecture/API docs.

Verification:

- `docker-compose -f docker-compose.dev.yml run --rm backend bun run test src/trpc/__tests__/router.test.ts`
- `docker-compose -f docker-compose.dev.yml run --rm frontend bun run test src/content/save-content.test.ts`

## 2026-07-06

### 3D Experience: backend payload, then a full frontend rebuild

- Implemented the 3D world backend: `GET /api/public/world/:handle/3d` (API-key/secret
  guarded), a `world` service that assembles resume + personal content into one
  deterministic payload (`meta`, `player`, `intro`, `views`, `zones`, `quests`,
  `stats`, `landmarks`, `ui`), an export path, and OpenAPI + tests.
- Built the stadium RPG/guided-tour frontend, but it was rejected. Rebuilt the
  `experience-3d` app from scratch as a restrained, cinematic scroll-driven
  experience: the stadium GLB is a night-match backdrop and scroll glides the
  camera between five vantages, each paired with a content panel; content-only
  fallback for mobile / no-WebGL. Deleted the whole game layer (tour, collect,
  flags, trophy, avatar, keyboard steering, `stadium-scene.tsx`).
- Wired real data: the app loads `public/data/world.json` (a snapshot of the live
  API) at startup. Added `ui.mainSiteUrl` so the main-portfolio link is
  backend-driven from the professional `website` frontmatter.

Verification: backend + frontend test suites, lint, and builds green; browser-verified
the scroll camera, panels, rail, mobile fallback, and live content.

Source:

- `docs/superpowers/specs/2026-07-06-3d-stadium-scrollytelling-design.md`
- `documentation/architecture/3d-experience.md`
- `documentation/operations/api.md` (3D World Snapshot Flow)

## 2026-07-05

### 3D Stadium World Direction

- Approved the first real 3D portfolio direction as a standalone stadium-based RPG experience.
- Scoped the first 3D content set to resume and personal content only.
- Chose one backend-shaped 3D world payload instead of multiple per-zone reads.
- Chose one pre-generated static `world.json` snapshot for the 3D app instead of runtime follow-up API calls.
- Chose a stadium tunnel intro, a free-exploration stadium ring, and a Champions League trophy landmark for stats and achievements.
- Wrote the design spec and implementation plan before code changes.
- Updated repo docs so the 3D direction, API contract intent, and content-storage implications are visible before implementation starts.

Source:

- `docs/superpowers/specs/2026-07-05-3d-rpg-portfolio-design.md`
- `docs/superpowers/plans/2026-07-05-3d-rpg-portfolio.md`

## 2026-07-04

### Dashboard Markdown Editing

- Simplified dashboard editing to raw Markdown textareas for personal, resume, and the first professional entry.
- Changed dashboard saves to send raw `markdown` payloads through the existing authenticated content API.
- Kept competitive programming editing as a coming-soon panel until that content model is stable.

Verification:

- `bun run test -- --filter=personaos-frontend -- src/__tests__/components/dashboard/profile-dashboard.test.tsx`
- `bun run test -- --filter=personaos-frontend -- src/__tests__/app/hitl-feedback.test.tsx src/__tests__/content/collections.test.ts`
- `bun run lint -- --filter=personaos-frontend`
- `bun run build -- --filter=personaos-frontend`

### Supabase Data API Grants

- Added a Drizzle migration granting explicit Data API table privileges for `profiles` and `content_documents`.
- Fixed local Drizzle config so host-side local migrations ignore Docker-only `host.docker.internal` placeholder URLs and use the local Supabase Postgres port.
- Added backend OAuth callback error logging so future callback failures expose the underlying Supabase/Postgres error in backend logs.
- Repaired the existing local OAuth user by creating its missing `public.profiles` row.
- Added `PERSONAOS_DEFAULT_HANDLE` for local OAuth profile bootstrap so the generated profile handle matches the frontend public content handle after resets.

Verification:

- `bun run --cwd apps/backend db:migrate:local`
- `bun run --cwd apps/backend test src/__tests__/routes/auth.test.ts`
- `bun run build:backend`
- `bun run lint -- --filter=@personaos/backend`
- `bun --cwd apps/backend drizzle-kit check --config=drizzle.local.config.ts`

## 2026-07-03

### Backend-Owned Auth Flow

- Fixed local OAuth to keep Supabase PKCE state on the backend with a short-lived HTTP-only verifier cookie.
- Kept the canonical OAuth callback at `/api/auth/callback`; the frontend `/auth/callback` route only forwards compatibility callbacks to the backend.
- Added OAuth profile bootstrap so successful OAuth logins create a missing `public.profiles` row without overwriting later profile edits.
- Added optional local Adminer support through `bake db-ui` for inspecting the local Supabase Postgres database.
- Documented the backend-owned auth flow in `documentation/architecture/auth-flow.md` and linked it from operations docs.

Verification:

- `bun run --cwd apps/backend test src/__tests__/routes/auth.test.ts`
- `bun run build:backend`
- `bun run lint -- --filter=@personaos/backend`
- `docker-compose -f docker-compose.dev.yml config --quiet`

## 2026-07-02

### Frontend API Integration

- Added protected published content reads under `/api/public/content/:handle/:type` and `/api/public/content/:handle/:type/:slug`.
- Protected public reads with `x-personaos-api-key` and `x-personaos-api-secret` so Next server loaders can call the API without exposing secrets to browser components.
- Changed published read responses to return temporary Markdown storage URLs instead of raw Markdown payloads.
- Added backend formatting for structured content saves so dashboard/editor forms send raw data and the backend writes canonical `.md` files.
- Switched frontend content loaders to fetch API-returned Markdown URLs and reuse the existing Markdown/Zod parsing pipeline.
- Wired dashboard/editor saves to authenticated content `PUT` requests with structured `data`.
- Documented the API key headers, Markdown URL response, structured save payloads, and env variables.
- Updated the starter Markdown import so it can populate Supabase Storage and `content_documents` by profile handle, preview mappings with `--dry-run`, and keep only explicit `published: false` files as drafts.
- Added Drizzle ORM/Kit for backend-owned Postgres schema, migrations, and typed DB access.
- Added the first Drizzle migration for `profiles`, `content_documents`, RLS policies, and the private `content-md` storage bucket.
- Added `bun run init:local` and `bun run init:production` wrappers for local Supabase bootstrap and production migration application.
- Moved public profile handle lookup onto Drizzle while keeping Supabase client for Auth and Storage.
- Moved local init/content import entrypoints into Bakefile-backed Docker commands so backend migrations and imports run from the backend container against local Supabase Docker ports.

Verification:

- `bun run --cwd apps/backend test src/__tests__/services/content-formatters.test.ts src/__tests__/routes/content.test.ts`
- `bun run --cwd apps/backend test src/__tests__/scripts/import-starter-content.test.ts src/__tests__/services/content-documents.test.ts`
- `bun run test -- --filter=personaos-frontend -- src/__tests__/content/api-content.test.ts src/__tests__/content/collections.test.ts src/__tests__/content/load-portfolio.test.ts`
- `bun run test -- --filter=personaos-frontend -- src/__tests__/components/dashboard/profile-dashboard.test.tsx src/__tests__/components/editor/editor-shell.test.tsx`
- `bun run test -- --filter=personaos-frontend`
- `bun run lint -- --filter=personaos-frontend`
- `bun run build -- --filter=personaos-frontend`
- `bun run test:backend`
- `bun run build:backend`
- `bun run lint -- --filter=@personaos/backend`
- `bun run --cwd apps/backend import:content --dry-run --owner-id=00000000-0000-0000-0000-000000000000`
- `bun --cwd apps/backend drizzle-kit check --config=drizzle.local.config.ts`
- `bunx tsc --noEmit -p tsconfig.json`
- `bun run test -- --filter=personaos-frontend -- src/__tests__/tooling/monorepo-config.test.ts`
- `docker-compose -f docker-compose.dev.yml config --quiet`

## 2026-06-30

### Brand Assets

- Added a PersonaOS SVG favicon at `apps/frontend/src/app/icon.svg`.
- Added reusable brand assets at `apps/frontend/public/assets/brand/personaos-mark.svg` and `apps/frontend/public/assets/brand/personaos-logo.svg`.
- Added PNG exports for the mark, logo, and favicon under `apps/frontend/public/assets/brand`.
- Updated the public header to render the horizontal logo while preserving an accessible text label.
- Documented the brand asset paths in `documentation/architecture/technical-stack.md` and `apps/frontend/README.md`.

Verification:

- `bun run test src/__tests__/app/hitl-feedback.test.tsx src/__tests__/app/auth-shell.test.tsx`
- `bun run lint`
- `bun run build`

## 2026-06-28

### Human Review Feedback Folder

- Established `docs/hitl` as the folder for Hamza's review feedback during human-in-the-loop review.
- Future review-driven work should check `docs/hitl` before continuing changes.

### HITL UI Feedback Fixes

- Fixed Drupal blog frontmatter so updated blog posts parse through `gray-matter`.
- Changed home portfolio entries so classic and immersive portfolio cards open external destinations in new tabs.
- Moved the personal page surface to `/me` and kept `/personal` as a redirect.
- Replaced the competitive programming stats page with a coming-soon page until the API is ready.
- Added reusable status pages for 404, coming soon, and maintenance states.
- Switched resume icon slugs to `skillicons.dev` while still allowing backend-provided URL/path icons.
- Reworked `/me` into a distinct personal page ready to render preferences, setup details, and gaming build data when profile content/API data provides them.
- Improved blog templates with article navigation, metadata, tag display, and a more responsive index layout.
- Added resume navigation to the personal, blog, and project pages, plus detail links for resume sections.
- Moved the 3D portfolio link into a global top banner and promoted the gaming build into its own personal-page section.
- Replaced the unsupported PCPartPicker embed path with renderer/schema support for a styled gaming parts list.
- Added a global theme switcher using the existing `next-themes` provider, plus light-mode client surface variables.
- Added lightweight CSS motion and hover feedback without adding a new animation dependency.
- Reworked landing-page copy to explain PersonaOS as a living personal profile rather than an implementation detail.
- Fixed the public theme switcher visibility issue by covering hardcoded dark utility classes in light mode.
- Allowed `2hlabs.dd` as a Next.js dev origin so local HMR can connect from the custom hostname.
- Expanded light-mode overrides for existing opacity/hex utility classes so text remains readable across public pages.
- Corrected the local Next HMR dev origin to use a full origin string: `http://2hlabs.dd`.
- Made the Next HMR dev origin env-driven through `NEXT_PUBLIC_DOMAIN`/`NEXT_DEV_ORIGIN`.
- Replaced the text theme switcher with an icon-only control that cycles system, light, and dark modes.
- Split local Docker env files into `.env.frontend` and `.env.backend` so containers only receive the variables they need.

Verification:

- `bun run test -- --filter=personaos-frontend -- src/__tests__/components/design-system/theme-system.test.tsx src/__tests__/tooling/monorepo-config.test.ts`
- `bun run test -- --filter=personaos-frontend -- src/__tests__/components/design-system/theme-system.test.tsx src/__tests__/tooling/monorepo-config.test.ts src/__tests__/app/hitl-feedback.test.tsx`
- `bun run test -- --filter=personaos-frontend -- src/__tests__/app/hitl-feedback.test.tsx src/__tests__/components/design-system/theme-system.test.tsx src/__tests__/content/collections.test.ts`
- `bun run test -- --filter=personaos-frontend -- src/__tests__/content/collections.test.ts src/__tests__/app/hitl-feedback.test.tsx`
- `bun run test -- --filter=personaos-frontend -- src/__tests__/app/hitl-feedback.test.tsx`
- `bun run test -- --filter=personaos-frontend -- src/__tests__/app/hitl-feedback.test.tsx src/__tests__/components/resume/resume-design.test.tsx src/__tests__/content/collections.test.ts`
- `bun run test -- --filter=personaos-frontend`
- `bun run lint -- --filter=personaos-frontend`
- `bun run build -- --filter=personaos-frontend`

### Dynamic Resume Markdown Data

- Removed hardcoded resume socials and tools from `apps/frontend/src/components/resume/resume-design.tsx`.
- Extended the resume profile schema with `skillSections`, `socials`, `tools`, and structured language metadata.
- Added a backend-driven resume icon resolver for icon URLs, local paths, or Simple Icons slugs, with text fallback for failed loads.
- Added tests proving resume content and UI use Markdown-backed sections, socials, and tools.

Verification:

- `bun run test -- --filter=personaos-frontend -- src/__tests__/content/collections.test.ts src/__tests__/components/resume/resume-design.test.tsx`
- `bun run lint -- --filter=personaos-frontend`
- `bun run build -- --filter=personaos-frontend`

Correction:

- Removed the local icon allowlist because backend-owned profile data must be able to control icon values without a frontend deploy.

## 2026-06-27

### Grouped Repo-Owned Markdown Documentation

- Moved durable repo docs into `documentation/`:
  - `documentation/architecture/technical-stack.md`
  - `documentation/architecture/content-storage-v1.md`
  - `documentation/operations/development.md`
  - `documentation/operations/deployment.md`
- Added `documentation/README.md` as the docs index.
- Kept root `README.md`, `AGENTS.md`, and `PROGRESS.md` for discoverability and continuity.
- Updated repo links to point at the grouped documentation paths.

## 2026-06-25

### PersonaOS Journey Blog Drafts

- Added four unpublished blog drafts about the PersonaOS build journey:
  - `From Portfolio To PersonaOS`
  - `Designing The Private Workspace`
  - `Choosing The Development Foundation`
  - `Keeping Markdown While Adding A Backend`
- Kept titles free of AI branding while reflecting on the discussions, tradeoffs, reversals, and implementation choices behind the project.
- Wrote the drafts in first person so they can become public posts from Hamza's perspective after review.
- Kept the drafts as local content because new `apps/frontend/content` files should be imported into storage instead of tracked in Git.

Follow-ups:

- Review tone and details before publishing.
- Add screenshots, diagrams, or commit references if the posts need stronger evidence.
- Import the drafts into Supabase Storage once the local account/content workflow is ready.

### Content Storage V0 Implementation

- Added `apps/backend/src/db/schema.v0.sql` as the active local bootstrap schema for `profiles`, `content_documents`, and the private `content-md` Supabase Storage bucket.
- Added `supabase/migrations/20260625000000_content_storage_v0.sql` so the same v0 schema can be applied by the local Docker-managed Supabase stack.
- Kept `apps/backend/src/db/schema.sql` unchanged as a broader archived draft; it should not be used for v0 deployment.
- Added account routes:
  - `POST /api/account/signup` creates a Supabase Auth user and initial profile.
  - `PUT /api/account/profile` creates or updates the authenticated user's profile.
- Added content document routes:
  - `GET /api/content/:type`
  - `GET /api/content/:type/:slug`
  - `PUT /api/content/:type/:slug`
- Added server-side Markdown frontmatter validation, server-generated storage paths, and Supabase Storage-backed raw `.md` persistence.
- Added `bun run import:content --owner-id=<auth-user-id>` in `apps/backend` to migrate canonical starter Markdown from `apps/frontend/content` into the API storage layer.
- Added Bake targets for local Supabase Docker management: `supabase-start`, `supabase-stop`, `supabase-status`, and `supabase-reset`.
- Updated `bake up` and `bake down` so the local app stack and local Supabase stack are managed together.
- Updated local development docs, stack docs, and `.env.example` for local Supabase content-storage setup.
- Grouped app and package tests under `src/__tests__/` folders to keep production source folders clean.
- Fixed local Docker startup by ignoring app `node_modules` in Docker build contexts and removing an unsupported Caddy `experimental_http3` option.
- Confirmed `bake up -v` starts Supabase, backend, frontend, and Caddy locally.

Verification:

- `cd apps/backend && bun run test`
- `cd apps/backend && bun run build`
- `cd apps/backend && bun run lint`
- `bun run test`
- `bun run lint`
- `bun run build`

Follow-ups:

- Move `schema.v0.sql` into a real Supabase migration before deployment wiring.
- Switch frontend and 3D Markdown loaders to the content API, then remove `apps/frontend/content`.
- Add proper sign-in/session issuance flow in the dashboard UI; signup currently creates the account and profile, while token issuance remains Supabase Auth responsibility.

## 2026-06-24

### Turborepo Migration

- Replaced the Nx-light monorepo orchestration with Turborepo.
- Root scripts now use `turbo run ...` and package-level `package.json` scripts are the source of truth.
- Added `turbo.json` for task dependencies, outputs, persistent dev/storybook tasks, and cache behavior.
- Removed root `nx.json` and workspace `project.json` files from active configuration.
- Kept Bun workspaces and the root `bun.lock`.
- Updated `README.md`, `documentation/operations/development.md`, and `documentation/architecture/technical-stack.md` to describe Turborepo instead of Nx.

Verification:

- `bun run test src/tooling/monorepo-config.test.ts`
- `bunx turbo run build --dry`
- `bun run build`
- `bun run lint`
- `bun run test`

Follow-ups:

- `tsc --noEmit` build tasks for `@personaos/backend` and `@personaos/3d-system` produce no build output, so Turbo warns about missing outputs. Split typecheck from build later if noisy caching becomes a problem.

### Bake Docker Shortcuts

- Replaced the root Makefile with a small `Bakefile`.
- Bake now owns only Docker stack shortcuts: `up`, `down`, `build`, `rebuild`, `logs`, service logs, `ps`, `clean`, and `restart`.
- Kept app checks under Bun/Turborepo instead of duplicating them in Bake.

### Dashboard And Theme Direction

- Added dashboard/client theme separation using `next-themes`.
- Public routes stay under `app/*`; private/user-specific routes start under `app/dashboard/*`.
- Added dashboard-scoped CSS variables and reusable dashboard primitives.
- Added the first mock dashboard page for editing profile, resume, and competitive programming data.
- Kept saves mocked for now so the page can be wired to the future API without blocking UI structure.

Follow-ups:

- Replace mock save state with API-backed create/update calls.
- Continue dashboard components from the Gravity UI direction while keeping public pages on the existing client/resume design system.

### Backend Foundation

- Added an Express/Bun backend workspace at `apps/backend`.
- Added Supabase server-client configuration and bearer-token user validation.
- Added `/api/health`.
- Added initial SQL schema draft for profiles, profile details, portfolios, blog posts, skills, resume data, and RLS policies.
- Added Docker Compose backend service and Caddy `/api/*` reverse proxy.
- Documented that schema changes should move into Supabase migrations before shared use.

Verification:

- `bun run test`
- `bun run lint`
- `bun run build`

Follow-ups:

- Move `apps/backend/src/db/schema.sql` into real Supabase migrations.
- Rotate any real credentials if the previous `.env.example` database password was ever valid.

### Content Storage V1 Decision

- Kept Markdown as the canonical v1 content format.
- Chose Supabase Storage for raw `.md` files and Postgres for a small `content_documents` metadata index.
- Kept the existing Next.js and 3D renderers on the current Markdown parser path.
- Deferred MongoDB and RustFS; Supabase already covers local Auth, Postgres, Storage, and migrations for v1.

Source:

- `documentation/architecture/content-storage-v1.md`

## 2026-06-22

### Competitive Programming Public Page

- Added a `/competitive` page backed by mock markdown data.
- Kept the implementation markdown-first while future Codeforces, AtCoder, and Kattis APIs are not ready.
- Added a typed collection loader shape close to future API output:
  - platform handles and links
  - recent solved problems
  - topic statistics
  - total solved count
- Linked the competitive programming page into the public app.

Verification target from plan:

- `bun run test`
- `bun run lint`
- `bun run build`

Source:

- Commit `2a79767` (`feat: add competitive programming page`)
- `docs/superpowers/plans/2026-06-22-competitive-programming-page.md`

### Backend Architecture Decision

- Decided on `Next.js -> Express/Bun -> Supabase` for backend direction.
- Supabase Auth/Postgres owns user identity and database storage.
- Backend uses server credentials and exposes product API routes.
- Markdown remains available during migration; database storage does not replace markdown in one step.
- Codeforces is the first API-backed competitive programming source.
- LeetCode, AtCoder, and Kattis start as curated/manual problem sources.
- Daily competitive programming challenge generation is lazy on request, not cron-based.
- Drizzle is deferred; start with Supabase client and SQL migrations first.

Source:

- `docs/superpowers/plans/2026-06-22-express-supabase-backend-v1.md`

## 2026-06-21

### 3D System And Experience App

- Added a dedicated Vite 3D experience app at `apps/experience-3d`.
- Added reusable 3D primitives/package at `packages/3d-system`.
- Kept Three.js and React Three Fiber out of the main Next.js app bundle.
- Added the first PersonaOS map direction with 3D scene and 2D fallback.
- Added Storybook support for the 3D system package.
- Added tests for the 3D system and experience app.

Verification target from plan:

- `bun run test`
- `bun run lint`
- `bun run build`
- `bun run build-storybook:3d`
- Playwright canvas/screenshot checks for the 3D experience

Source:

- Commit `3190821` (`feat: add 3d personaos map app`)
- `docs/superpowers/plans/2026-06-21-3d-system-experience-map.md`

### Monorepo Tooling Baseline

- Added the first Bun workspace monorepo baseline.
- Added Nx-light as the first root task orchestrator.
- Added Prettier as a separate formatting tool from ESLint.
- Added tooling tests to guard workspace/package orchestration.
- Preserved existing frontend-local commands while introducing root commands.

Later change:

- This Nx-light decision was reversed on 2026-06-24 in favor of Turborepo.

Source:

- Commit `75c25ba` (`chore: add nx monorepo tooling baseline`)
- `docs/superpowers/plans/2026-06-21-monorepo-tooling-baseline.md`

### 3D Architecture Direction

- Planned a small explicit monorepo shape:
  - `apps/frontend`
  - `apps/experience-3d`
  - `apps/backend`
  - `packages/3d-system`
  - later shared packages only when a second consumer exists
- Chose a dedicated 3D app instead of a route inside the main Next.js app.
- Planned production routing as separate services:
  - main domain to frontend
  - 3D subdomain to experience app
  - API subdomain or `/api` routing to backend
- Kept markdown as the canonical shared content model at that stage.

Source:

- `docs/superpowers/specs/2026-06-21-monorepo-3d-architecture-design.md`

## 2026-06-20

### Resume Design System

- Added a markdown resume design system.
- Fixed responsiveness in the resume design system after the initial implementation.
- Public accessible pages/components use the client/resume-oriented design system.

Source:

- Commit `1b38b2a` (`feat: add markdown resume design system`)
- Commit `ca458de` (`fix: make resume design system responsive`)

## 2026-06-16

### Frontend Portfolio Builder Rewrite

- Rebuilt the frontend into a Next.js 16 App Router portfolio builder.
- Kept Markdown as the single source of truth for v1.
- Added typed content parsing and validation around local markdown.
- Added public portfolio routes:
  - `/portfolio/classic`
  - `/portfolio/immersive`
- Added `/editor` workspace for local portfolio editing and preview.
- Added a fixed portfolio block library:
  - `hero`
  - `about`
  - `rich-text`
  - `timeline`
  - `projects`
  - `gallery`
  - `contact`
  - `fun-facts`
  - `showcase`
- Kept classic rendering usable without 3D dependency.
- Kept immersive rendering as an enhanced output from the same content model.
- Added tests around schema, markdown loading, renderer behavior, and routes.

Source:

- Commit `12d1ff1` (`feat: rewrite frontend portfolio builder`)
- `docs/superpowers/specs/2026-06-16-frontend-portfolio-builder-design.md`
- `docs/superpowers/plans/2026-06-16-frontend-portfolio-builder-plan.md`

## Earlier Baseline

- Initialized the repository.
- Added repository guidelines.
- Added initial infrastructure with Frunken.
- Removed AI metadata docs from tracking.
- Stopped tracking ignored `docs/` content as repo-owned documentation.
- Removed the previous Symfony/PostgreSQL backend stack before the Express/Supabase direction was chosen.

Source:

- Commit `4a44f3e` (`init`)
- Commit `c000c66` (`feat: infra with frunken`)
- Commit `75bd2b6` (`docs: add repository guidelines`)
- Commit `9cd9db0` (`chore: remove AI MD`)
- Commit `60a3a98` (`chore: ignore docs directory`)
- Commit `e8b1e65` (`chore: stop tracking docs`)
- Commit `82acc0b` (`chore: remove legacy backend stack`)

## Current Guardrails

- Public app content remains markdown-first until API-backed persistence is ready.
- User-specific and private surfaces live under `app/dashboard/*`.
- Dashboard/user-specific UI uses the dashboard theme; public/client pages keep the existing public design system.
- Keep root `README.md`, `AGENTS.md`, and `PROGRESS.md` small; group detailed repo docs under `documentation/`.
- Keep generated/cache/local import folders out of commits unless intentionally promoted.
- Prefer small backend changes: Express routes, Supabase SQL migrations, and tests before adding heavier abstractions.

## Open Follow-Ups

- Convert backend schema draft into Supabase migrations and seed files.
- Add `content_documents` migration and Supabase Storage bucket policy for raw Markdown.
- Wire dashboard forms to backend create/update endpoints.
- Add content API endpoints that save and serve raw Markdown through the backend.
- Add API-backed competitive programming data sources for Codeforces first, then curated/adapter paths for AtCoder and Kattis.
- Decide whether `.memsearch/` should stay ignored/local or become an intentional project artifact.
