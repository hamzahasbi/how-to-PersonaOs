# Content Storage V1

V1 keeps Markdown as the canonical content format. The backend stores raw `.md` files in Supabase Storage and keeps only a small Postgres index for ownership, lookup, and publishing state.

## Decision

- Use local Supabase for development.
- Use Supabase Storage for raw Markdown files.
- Use Postgres for metadata, auth ownership, and RLS.
- Do not add MongoDB.
- Do not add RustFS for v1; revisit it only if storage portability or S3 parity becomes a real constraint.

## Shape

```text
Dashboard editor
  -> Express API validates the user
  -> API validates Markdown/frontmatter
  -> API writes raw .md to Supabase Storage
  -> API upserts metadata in Postgres
  -> Next.js loads Markdown through the API
  -> 3D backend assembler derives a gamified world payload from resume + personal content
  -> 3D export step writes static world.json
  -> existing parsers/renderers stay in use where applicable
```

## Minimal Table

The v0 bootstrap schema lives in `apps/backend/src/db/schema.v0.sql`. Do not use the broader `apps/backend/src/db/schema.sql` for v0 deployment; it is an archived draft for later normalized data work.

```sql
create table public.content_documents (
  id uuid primary key default gen_random_uuid(),
  owner_id uuid not null references public.profiles(id) on delete cascade,
  content_type text not null,
  slug text not null,
  storage_bucket text not null default 'content-md',
  storage_path text not null,
  status text not null default 'draft' check (status in ('draft', 'published', 'archived')),
  version int not null default 1,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (owner_id, content_type, slug),
  unique (storage_bucket, storage_path)
);
```

`content_type` starts with existing content groups: `portfolio`, `blog`, `project`, `personal`, `professional`, `competitive`.

## V1 API

- `POST /api/account/signup`: creates a Supabase Auth user and initial profile.
- `PUT /api/account/profile`: creates or updates the authenticated user's profile row.
- `GET /api/content/:type`: lists the current user's documents for one content type.
- `GET /api/content/:type/:slug`: returns metadata plus raw Markdown for renderers.
- `PUT /api/content/:type/:slug`: validates and saves raw Markdown.
- `DELETE /api/content/:type/:slug`: deletes the raw Markdown object and metadata row.

All content routes and profile updates require a backend session cookie or bearer token. The backend validates the session with Supabase Auth and uses the authenticated user id as the owner id. Account signup is intentionally unauthenticated because it creates the auth user.

## Starter Import

The existing frontend Markdown is starter data for the API. Import it after applying `schema.v0.sql` and creating/signing up the local Supabase user:

```bash
EMAIL=<your-account-email> make init-content
```

Use `--dry-run` to preview the migration, or `--owner-id=<auth-user-id>` when email lookup is not available. The import reads `apps/frontend/content`, skips `apps/frontend/content/imports`, maps `projects` to the API `project` type, and flattens nested paths into URL-safe slugs such as `professional/experience/void-technical-lead.md` -> `professional/experience-void-technical-lead`.

Each imported file is uploaded to the private `content-md` storage bucket and indexed in `content_documents`. Files with `published: false` stay drafts; all other starter Markdown is published so the API-backed frontend matches the previous file-backed routes.

## Guardrails

- Validate frontmatter before saving.
- Keep storage paths server-generated.
- Keep public renderers reading the same Markdown shape they already understand.
- Let the backend own 3D gamification and world shaping instead of teaching the 3D frontend to reinterpret plain portfolio records.
- Add parsed JSON cache later only if repeated parsing becomes a measured problem.
- Treat `apps/frontend/content` as local starter/import data, not the long-term source of truth.
- New frontend content files are ignored by Git and should be imported into Supabase Storage through the backend.
- Keep the currently tracked starter content until the frontend and 3D app load Markdown through the API. Remove the tracked files only after those render paths are API-backed.
