# Project README Snapshot Import Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Import GitHub README content and its repository-local images into PersonaOS-owned project Markdown and Supabase Storage.

**Architecture:** The GitHub importer formats a README snapshot, uploads safe repository-local image assets into the private `project-assets` bucket, and replaces their Markdown URLs with public PersonaOS asset routes. The route checks that the parent project is published before returning an asset. Project pages render saved Markdown and a repository banner, never a live README.

**Tech Stack:** Bun, TypeScript, Express, tRPC, Supabase Storage, Drizzle, Next.js, React Markdown, Vitest.

## Global Constraints

- All GitHub and Supabase credentials remain backend-only.
- Imported projects are drafts by default.
- Only repository-local raster image paths are downloaded; external URLs remain external.
- `project-assets` is private and public routes must verify the project is published.
- Docker commands are required for tests, lint, and builds.

---

### Task 1: Provision private project asset storage

**Files:**

- Create: `supabase/migrations/20260715000000_project_assets_bucket.sql`
- Create: `apps/backend/src/scripts/ensure-project-assets-bucket.ts`
- Modify: `apps/backend/package.json`
- Test: `apps/backend/src/__tests__/scripts/ensure-project-assets-bucket.test.ts`

**Interfaces:**

- Produces a private `project-assets` bucket and `bun run ensure:project-assets-bucket` command.

- [ ] **Step 1: Write the failing bucket-script test**

Assert that the script creates `project-assets` with `{ public: false }` when it does not already exist.

- [ ] **Step 2: Run the focused test and verify it fails**

Run: `docker-compose -f docker-compose.dev.yml run --rm backend bun run test src/__tests__/scripts/ensure-project-assets-bucket.test.ts`

- [ ] **Step 3: Add the SQL migration, idempotent ensure script, and package command**

The migration inserts `('project-assets', 'project-assets', false)` with `on conflict do nothing`. The script mirrors the existing world-bucket script but calls `createBucket('project-assets', { public: false })`.

- [ ] **Step 4: Run the focused test and verify it passes**

Run: `docker-compose -f docker-compose.dev.yml run --rm backend bun run test src/__tests__/scripts/ensure-project-assets-bucket.test.ts`

### Task 2: Format README snapshots and import repository-local images

**Files:**

- Modify: `apps/backend/src/services/github-readme.ts`
- Modify: `apps/backend/src/services/github-project-import.ts`
- Test: `apps/backend/src/services/__tests__/github-readme.test.ts`
- Test: `apps/backend/src/services/__tests__/github-project-import.test.ts`

**Interfaces:**

- `formatGitHubReadmeMarkdown(markdown, repo, assets): string`
- `ProjectAssetUploader.upload({ content, contentType, ownerId, slug, sourcePath }): Promise<{ assetId: string }>`
- `buildGitHubProjectImport(repoUrl, ownerId, dependencies): Promise<{ markdown; slug; title }>`

- [ ] **Step 1: Write failing formatter/importer tests**

Cover removal of a leading README H1, conversion of an image-following hyphen separator into a thematic break, rewriting copied local image URLs to `/api/public/project-assets/<handle>/<slug>/<assetId>`, preserving absolute URLs, and rejecting an oversized or non-raster GitHub response.

- [ ] **Step 2: Run the focused tests and verify they fail**

Run: `docker-compose -f docker-compose.dev.yml run --rm backend bun run test src/services/__tests__/github-readme.test.ts src/services/__tests__/github-project-import.test.ts`

- [ ] **Step 3: Implement the minimal importer**

Read the GitHub token and profile handle server-side, fetch each relative README image with the GitHub Contents API, limit images to PNG/JPEG/GIF/WebP and 10 MiB, upload them to `project-assets/<ownerId>/project/<slug>/<sha256>.<extension>`, and embed the formatted README below project frontmatter. Keep relative non-image links pointing at the repository blob URL.

- [ ] **Step 4: Run the focused tests and verify they pass**

Run: `docker-compose -f docker-compose.dev.yml run --rm backend bun run test src/services/__tests__/github-readme.test.ts src/services/__tests__/github-project-import.test.ts`

### Task 3: Serve imported images only for published projects

**Files:**

- Create: `apps/backend/src/services/project-assets.ts`
- Modify: `apps/backend/src/routes/content.ts`
- Modify: `apps/backend/src/server.ts`
- Test: `apps/backend/src/routes/__tests__/content.test.ts`

**Interfaces:**

- `ProjectAssetStore.download(ownerId, slug, assetId): Promise<{ body: Uint8Array; contentType: string } | null>`
- `GET /api/public/project-assets/:handle/:slug/:assetId`

- [ ] **Step 1: Write failing route tests**

Assert a published project returns its image with the stored content type and cache header, while draft, archived, missing, malformed asset IDs, and another owner’s path return 404.

- [ ] **Step 2: Run the content route tests and verify they fail**

Run: `docker-compose -f docker-compose.dev.yml run --rm backend bun run test src/routes/__tests__/content.test.ts`

- [ ] **Step 3: Add the private Storage downloader and public route**

Require a valid handle, slug, and hashed asset id. Resolve the owner by handle, read the `project` document, require `published`, then download only `ownerId/project/slug/assetId` from the private bucket. Send `Cache-Control: public, max-age=3600` and the validated media type.

- [ ] **Step 4: Run the content route tests and verify they pass**

Run: `docker-compose -f docker-compose.dev.yml run --rm backend bun run test src/routes/__tests__/content.test.ts`

### Task 4: Connect imports and backfill existing projects

**Files:**

- Modify: `apps/backend/src/trpc/middleware.ts`
- Modify: `apps/backend/src/trpc/router.ts`
- Create: `apps/backend/src/scripts/import-project-readmes.ts`
- Modify: `apps/backend/package.json`
- Test: `apps/backend/src/trpc/__tests__/router.test.ts`
- Test: `apps/backend/src/__tests__/scripts/import-project-readmes.test.ts`

**Interfaces:**

- `bun run import:project-readmes --email=<email> [--dry-run] [--preserve-status]`

- [ ] **Step 1: Write failing tRPC and backfill tests**

Assert dashboard imports save formatted README Markdown as a draft. Assert the backfill reads each existing project’s `link` frontmatter, does not save in dry-run mode, and defaults saved projects to draft unless `--preserve-status` is set.

- [ ] **Step 2: Run the focused tests and verify they fail**

Run: `docker-compose -f docker-compose.dev.yml run --rm backend bun run test src/trpc/__tests__/router.test.ts src/__tests__/scripts/import-project-readmes.test.ts`

- [ ] **Step 3: Wire dependencies and implement the command**

Use the same `buildGitHubProjectImport` service and normal `saveContentDocument`/content store path for both tRPC and backfill. Resolve owner identity exactly like the starter importer; report each skipped project without a valid GitHub URL.

- [ ] **Step 4: Run the focused tests and verify they pass**

Run: `docker-compose -f docker-compose.dev.yml run --rm backend bun run test src/trpc/__tests__/router.test.ts src/__tests__/scripts/import-project-readmes.test.ts`

### Task 5: Replace live README rendering with a repository banner

**Files:**

- Modify: `apps/frontend/src/app/projects/[slug]/page.tsx`
- Delete: `apps/frontend/src/content/integrations.ts`
- Modify: `apps/backend/src/routes/integrations.ts`
- Modify: `apps/backend/src/__tests__/routes/integrations.test.ts`
- Test: `apps/frontend/src/__tests__/app/project-page.test.tsx`

**Interfaces:**

- Project pages render stored project Markdown plus a direct repository link.

- [ ] **Step 1: Write failing page and route tests**

Assert a project page renders the imported Markdown once, has a prominent repository link, and does not request a live README. Remove the now-unused public GitHub README endpoint test.

- [ ] **Step 2: Run the focused tests and verify they fail**

Run: `docker-compose -f docker-compose.dev.yml run --rm frontend bun run test src/__tests__/app/project-page.test.tsx`

- [ ] **Step 3: Implement the banner-only page**

Remove the Suspense live README section and use a full-width bordered repository callout beneath the saved Markdown. Keep the existing top-level Open Project link.

- [ ] **Step 4: Run focused tests and verify they pass**

Run: `docker-compose -f docker-compose.dev.yml run --rm frontend bun run test src/__tests__/app/project-page.test.tsx`

### Task 6: Update documentation and verify end to end

**Files:**

- Modify: `documentation/operations/api.md`
- Modify: `documentation/operations/development.md`
- Modify: `documentation/architecture/dashboard-trpc-migration.md`
- Modify: `documentation/architecture/technical-stack.md`

- [ ] **Step 1: Document bucket provisioning, backfill command, import limits, and public-asset publication semantics**

- [ ] **Step 2: Run full validation**

Run:
`docker-compose -f docker-compose.dev.yml run --rm backend bun run test`
`docker-compose -f docker-compose.dev.yml run --rm backend bun run lint`
`docker-compose -f docker-compose.dev.yml run --rm backend bun run build`
`docker-compose -f docker-compose.dev.yml run --rm frontend bun run test`
`docker-compose -f docker-compose.dev.yml run --rm frontend bun run lint`

- [ ] **Step 3: Browser-check `/projects/sys-check`**

Confirm the project has one stored README rendering, a repository callout, and no live README request.
