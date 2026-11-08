# Dashboard tRPC Migration

PersonaOS now uses tRPC as the typed internal API surface for dashboard-owned
workflows. The migration starts with content document saves and GitHub README
project imports while keeping the
existing REST/OpenAPI routes in place for stable public and operational
contracts.

## Boundary

Use tRPC for:

- Authenticated dashboard commands and queries.
- TypeScript-to-TypeScript frontend/backend calls where drift is likely.
- Workflows that are not intended as public API contracts.

Keep REST/OpenAPI for:

- Public content reads under `/api/public/content`.
- Auth and OAuth routes under `/api/auth`.
- Public 3D world snapshot reads and world export flow.
- API documentation generated from `apps/backend/src/openapi.ts`.
- Any route expected to be called by external tools or curl examples.

## Current Implementation

- The backend mounts tRPC at `/api/trpc` from `apps/backend/src/server.ts`.
- The root tRPC router lives in `apps/backend/src/trpc/router.ts`.
- Express auth still runs before tRPC procedures through
  `apps/backend/src/trpc/middleware.ts`.
- The first procedures are:
  - `content.save`, used by dashboard Markdown save flows.
  - `github.importProject`, used by the integrations tab to turn a GitHub
    repository README into a draft project document.
- REST `PUT /api/content/:type/:slug` and tRPC `content.save` share
  `apps/backend/src/services/content-save.ts`.
- GitHub project import uses `apps/backend/src/services/github-project-import.ts`
  and reads the stored GitHub token server-side through the provider credential
  store. The browser sends only the repository URL.
- GitHub project import uses the README for project metadata only. It saves a
  lightweight project Markdown stub with the GitHub repository link, and the
  project page keeps rendering the live README from that link through the shared
  PersonaOS Markdown renderer.

This keeps authorization and Markdown validation backend-owned while allowing
the dashboard client to move one workflow at a time.

## Extension Pattern

When moving another dashboard API call to tRPC:

1. Extract reusable route behavior into a backend service first.
2. Add a focused backend tRPC test that proves the procedure uses
   `response.locals.userId` from the authenticated request.
3. Add or update a frontend test around the dashboard helper/component.
4. Mount the procedure under the existing root router rather than adding a
   second tRPC endpoint.
5. Leave the REST route in place until all consumers are migrated and the route
   is deliberately deprecated.

## Verification

Use Dockerized commands:

```bash
docker-compose -f docker-compose.dev.yml run --rm backend bun run test src/trpc/__tests__/router.test.ts
docker-compose -f docker-compose.dev.yml run --rm frontend bun run test src/content/save-content.test.ts src/components/dashboard/__tests__/profile-dashboard.test.tsx
```

For broader changes, also run the existing app-level backend/frontend lint,
test, and build commands from `AGENTS.md`.
