# Content Storage

Markdown is the canonical format for authored PersonaOS content. The backend stores document bytes in Supabase Storage and indexes ownership, type, slug and publication state in PostgreSQL.

## Data model

The `content_documents` table is defined in `apps/backend/src/db/schema.ts`. Its unique owner/type/slug key identifies a document, and its storage bucket/path points to the Markdown object. Status is `draft`, `published`, or `archived`; versions and update timestamps are maintained with the metadata.

Supported types are `portfolio`, `blog`, `project`, `personal`, `professional`, and `competitive`. The current Storage bucket for Markdown is `content-md`. Drizzle migrations in `apps/backend/drizzle` are the database change history; do not use archived schema drafts as deployment instructions.

## Read and write paths

- Dashboard REST reads and writes use `/api/content/:type` and `/api/content/:type/:slug` and are scoped to the authenticated owner.
- Public reads use `/api/public/content/:handle/:type` and `/api/public/content/:handle/:type/:slug`. Only published documents are returned.
- Dashboard `content.save` uses tRPC and shares the backend `saveContentDocument` validation and persistence service with the REST save route.
- Public responses include the Markdown content and a temporary signed URL for clients that need it. The browser-facing site currently consumes the inlined Markdown through its server-side loader.

The backend validates frontmatter before saving and generates storage paths from the owner, type and slug. The browser does not receive the Supabase service-role credential or write directly to the content bucket.

## Authoring and migration

The dashboard editor saves Markdown documents through the backend. Portfolio block data is serialized to Markdown before it crosses the persistence boundary. Imported starter content and GitHub README projects use the same backend storage and metadata path as normal edits.

`apps/frontend/content` may be used as local import input where the importer expects it; it is not the runtime source of published pages. Avoid editing or adding starter files as a replacement for a dashboard/API content change.

## Access and consistency

Application authorization scopes every private operation to the authenticated profile. Database ownership and RLS policies protect tables exposed through the Supabase Data API; the backend owns Storage access and publication checks. Keep Markdown and its metadata consistent by using the content store/service rather than writing either side directly.

For the frontend parsing/rendering path, see the [Codebase Guide](codebase-guide.md). For endpoint schemas, use the API Reference.
