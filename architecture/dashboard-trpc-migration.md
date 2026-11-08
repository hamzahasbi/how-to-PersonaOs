# Dashboard tRPC API

The backend exposes typed, authenticated dashboard workflows at `/api/trpc`. tRPC is an internal TypeScript contract; REST/OpenAPI remains the contract for authentication, public reads, health/docs, OAuth and binary downloads.

## Runtime boundary

`apps/backend/src/trpc/middleware.ts` authenticates the request before creating the tRPC context. Each procedure receives the authenticated `userId` and feature services/stores. Procedures must scope private reads and writes to that ID. The router is defined in `apps/backend/src/trpc/router.ts`.

## Current procedure groups

| Group         | Current responsibility                                                 |
| ------------- | ---------------------------------------------------------------------- |
| `assets`      | Sign Cloudinary uploads for game assets                                |
| `contests`    | List upcoming contests for connected judges                            |
| `competitive` | Manage provider handles/visibility, read stats and trigger sync        |
| `dailyCp`     | Read daily state, generate challenges, resolve outcomes and regenerate |
| `content`     | Save validated Markdown documents                                      |
| `github`      | Check the saved token and import a repository README project           |
| `personal`    | Manage the clist username, games and favorite team                     |

The exact input and output schemas in `router.ts` are authoritative. Keep public and operational REST contracts in `apps/backend/src/openapi.ts` rather than adding them to tRPC.

## Shared service behavior

REST `PUT /api/content/:type/:slug` and `content.save` both call `saveContentDocument`. GitHub import reads a README and supported repository images on the backend, formats them into a Markdown document, and saves the result as a draft. The public project page renders the saved Markdown; it does not fetch the README on each visit. Re-import to refresh the snapshot.

## Adding a procedure

1. Put reusable validation and business rules in the owning backend service.
2. Add a strict input schema and scope persistence operations to `ctx.userId`.
3. Reuse the service from REST and tRPC if both adapters expose the same operation.
4. Add backend procedure tests and frontend tests for the dashboard caller.

Run affected backend and frontend checks in their Docker containers when changing procedures.
