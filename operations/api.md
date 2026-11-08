# API Testing Guide

The backend listens on `PORT` from `.env.backend`, defaulting to `4000`.

```bash
API_URL=http://localhost:4000/api
TOKEN=<backend-or-supabase-access-token>
PERSONAOS_API_KEY=dev-frontend-key
PERSONAOS_API_SECRET=dev-frontend-secret
```

Behind Caddy, use the app origin plus `/api` instead.

The API documentation redirects to the Mintlify site:

```text
https://docs.2hlabs.com/
```

The raw OpenAPI document is available at:

```text
http://localhost:4000/api/openapi.json
```

## tRPC Dashboard API

Dashboard-only typed calls are mounted at:

```text
http://localhost:4000/api/trpc
```

Current migrated procedures include `content.save` for dashboard Markdown saves
and `github.importProject` for importing a GitHub repository README as a draft
project document. This is an internal TypeScript contract, not the public API
docs surface. Keep generated OpenAPI/Mintlify coverage focused on REST routes
that external callers or operational curl flows need.

## Mintlify

The Mintlify site root is `documentation/`; its navigation and theme are configured in `documentation/docs.json`. The sync workflow publishes this complete directory to the public documentation repository.

Mintlify generates endpoint pages from `documentation/openapi.json`. Regenerate that file from the backend OpenAPI source after endpoint changes:

```bash
docker-compose -f docker-compose.dev.yml run --rm -w /app backend \
  bun run docs:openapi
```

By default this writes the reserved placeholder `https://api.example.com/api`. The generator rejects credentials, query strings, fragments, and non-HTTP(S) URLs. For hosted Mintlify playground requests, set the deployed API base URL while generating:

```bash
docker-compose -f docker-compose.dev.yml run --rm -w /app \
  -e OPENAPI_SERVER_URL=https://api.example.com/api backend \
  bun run docs:openapi
```

The private app repo can sync an explicit allowlist of public Mintlify files into the public `hamzahasbi/how-to-PersonaOs` repo through `.github/workflows/sync-mintlify-docs.yml`. Configure `DOCS_SYNC_TOKEN` as a private repository secret with contents read/write access to the public docs repo. Optionally configure `OPENAPI_SERVER_URL` as a repository variable containing only the intended public API base URL. Never put credentials or signed URLs in it.

The publish allowlist lives in the workflow. Review new pages before adding them; maintainer notes, agent instructions, environment files, and runbooks must stay outside the public artifact. An allowlist limits which files ship but does not scan their contents for secrets. Review generated examples before publishing.

Authenticated endpoints accept the backend session cookie from `POST /api/auth/login`. Curl can also pass a bearer token:

```bash
-H "Authorization: Bearer $TOKEN"
```

## Health

```bash
curl "$API_URL/health"
```

Expected:

```json
{ "ok": true }
```

## Account

## Auth

### Login

Creates a backend-owned HTTP-only session cookie. The frontend uses this endpoint instead of calling Supabase directly.

```bash
curl -i -X POST "$API_URL/auth/login" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "hamza@example.com",
    "password": "password-123"
  }'
```

### Current User

```bash
curl "$API_URL/auth/me" \
  -H "Authorization: Bearer $TOKEN"
```

### Logout

```bash
curl -i -X POST "$API_URL/auth/logout"
```

### OAuth

OAuth starts through the backend, then returns to a backend callback that exchanges the Supabase PKCE code, creates a missing `public.profiles` row, and sets the session cookie.

```bash
open "$API_URL/auth/oauth/google?next=/dashboard"
```

The frontend does not set auth cookies directly. See [Auth flow](../architecture/auth-flow.md) for the full redirect chain.

### Football match centre

Football settings and provider credentials are private and require the backend session cookie. Configure provider keys from Dashboard → Football; keys are stored server-side and are never included in public responses.

```bash
curl "$API_URL/football/settings" -H "Cookie: personaos-access-token=$TOKEN"
curl "$API_URL/football/matches" -H "Cookie: personaos-access-token=$TOKEN"
curl -X PUT "$API_URL/football/settings" \
  -H "Content-Type: application/json" \
  -H "Cookie: personaos-access-token=$TOKEN" \
  -d '{"published":true}'
```

The public read endpoint is available only after publishing and requires the normal PersonaOS public API key pair:

```bash
curl "$API_URL/public/football/hamza/matches" \
  -H "x-personaos-api-key: $PERSONAOS_API_KEY" \
  -H "x-personaos-api-secret: $PERSONAOS_API_SECRET"
```

RapidAPI Football supplies the fixture calendar, cached for 48 hours even during matches. BSD supplies live scores and final status for uniquely matched fixtures; the client refreshes that data every 60 seconds around kickoff. This avoids RapidAPI match-time polling, but each calendar refresh still consumes a RapidAPI request. When that plan's quota is exhausted, pause its subscription or key until the reset to avoid overage charges.

### Create Account

Creates a Supabase Auth user and initial profile. This endpoint is intentionally unauthenticated.

```bash
curl -X POST "$API_URL/account/signup" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "hamza@example.com",
    "password": "password-123",
    "displayName": "Hamza Hasbi",
    "handle": "hamza"
  }'
```

### Update Profile

Updates the authenticated user's profile. The profile id is taken from the backend session or bearer token, not the request body.

```bash
curl -X PUT "$API_URL/account/profile" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "displayName": "Hamza Hasbi",
    "handle": "hamza",
    "avatarUrl": null
  }'
```

## Content Documents

Supported content types:

```text
portfolio, blog, project, personal, professional, competitive
```

Supported statuses:

```text
draft, published, archived
```

All content endpoints are authenticated. The backend validates the session, uses that user id as `ownerId`, and generates the Supabase Storage path server-side.

Published read endpoints are protected with the server-to-server API key and secret. They return temporary Markdown file URLs from storage; frontend server loaders fetch those URLs and parse the Markdown.

### 3D World Snapshot Flow (live)

The 3D experience is a single backend payload, snapshotted to a static file and
loaded once at startup — no per-zone or runtime API calls. The current frontend
uses that snapshot to drive the scroll-based stadium presentation documented in
`documentation/architecture/3d-experience.md`.

Live endpoint (protected with the API key + secret read guard):

```text
GET /api/public/world/:handle/3d
```

Response is one JSON document with these top-level keys:

- `meta` — version, generatedAt, theme, atmosphere
- `player` — name, title, shirtNumber, style, avatarUrl, location, bio
- `intro` — headline, cta
- `views` — spawn/reveal anchor names
- `zones` — labelled content blocks (id, title, eyebrow, accent, summary, bullets, stats)
- `quests` — career items grouped by zone
- `stats` — attributes, badges, counters
- `landmarks` — trophy + major stadium landmarks
- `ui` — interface strings: `mainSiteLabel` and `mainSiteUrl` (the main-portfolio
  link; dashboard refresh can set `mainSiteUrl`, then env/profile website
  values are used as fallbacks before `/`)

Content is drawn from resume (`professional`) + `personal` published documents
only. Missing sections degrade to empty arrays/objects rather than missing keys.

Snapshot to the static file the `experience-3d` app reads at startup
(`apps/experience-3d/public/data/world.json`). With the backend running:

```bash
set -a; . ./.env.backend; set +a
curl -s -H "x-personaos-api-key: $PERSONAOS_API_KEY" \
     -H "x-personaos-api-secret: $PERSONAOS_API_SECRET" \
     "http://localhost:4000/api/public/world/<handle>/3d" \
  | python3 -m json.tool > apps/experience-3d/public/data/world.json
```

The `bun run export:world` script does the same via a direct DB connection, but
requires the database to be reachable from the host; the curl-from-API flow above
is the reliable path in the Docker dev setup.

### List Published Documents

```bash
curl "$API_URL/public/content/hamza/blog" \
  -H "x-personaos-api-key: $PERSONAOS_API_KEY" \
  -H "x-personaos-api-secret: $PERSONAOS_API_SECRET"
```

### Read Published Document

```bash
curl "$API_URL/public/content/hamza/portfolio/default" \
  -H "x-personaos-api-key: $PERSONAOS_API_KEY" \
  -H "x-personaos-api-secret: $PERSONAOS_API_SECRET"
```

The response includes the document's `markdown` inline, plus a signed `markdownUrl` for clients that prefer to stream the `.md` file directly before it expires. Public lists preserve the complete collection and its order, hydrating at most four documents concurrently from their listed storage metadata. Documents whose storage objects disappear during download or signing are omitted; other storage failures still fail the request.

### List Documents

```bash
curl "$API_URL/content/blog" \
  -H "Authorization: Bearer $TOKEN"
```

### Read Document

```bash
curl "$API_URL/content/blog/hello-world" \
  -H "Authorization: Bearer $TOKEN"
```

### Create Or Update Document

Dashboard/editor forms should send structured `data`; the backend formats canonical Markdown before saving the `.md` file:

```bash
curl -X PUT "$API_URL/content/personal/profile" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Hamza Hasbi",
    "status": "draft",
    "data": {
      "name": "Hamza Hasbi",
      "headline": "Full-stack engineer",
      "email": "hamza.hasbi@gmail.com",
      "website": "https://hamzahasbi.2hlabs.com/",
      "body": "I enjoy building practical systems."
    }
  }'
```

Raw Markdown is still accepted for imports and direct migration scripts:

```bash
curl -X PUT "$API_URL/content/blog/hello-world" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Hello World",
    "status": "draft",
    "markdown": "---\ntitle: Hello World\n---\n# Hello World"
  }'
```

### Delete Document

Deletes the Markdown file from Supabase Storage, then deletes the `content_documents` row.

```bash
curl -i -X DELETE "$API_URL/content/blog/hello-world" \
  -H "Authorization: Bearer $TOKEN"
```

Expected status:

```text
204 No Content
```

## Site Export

`GET /api/export/site.zip` downloads a static zip snapshot of the authenticated
owner's published site. It backs the "Export my site" card on the dashboard
personal tab, so self-hosters can take their portfolio off PersonaOS entirely.
Design: [`docs/superpowers/specs/2026-07-18-self-hosted-site-export-design.md`](../../docs/superpowers/specs/2026-07-18-self-hosted-site-export-design.md).

Authenticated with the backend session cookie (or bearer token) like any other
account-scoped route — no API key/secret pair, unlike the `/public/*` read
endpoints.

### Why REST instead of tRPC

The dashboard's typed internal API is tRPC, but this endpoint stays REST: a
binary zip download is a poor fit for tRPC's JSON envelope, and a
cookie-authenticated `GET` is just a plain link the browser can navigate to
directly (`<a href="/api/export/site.zip">`). The OpenAPI entry lives in
`apps/backend/src/openapi.ts`.

### How the export works

The backend fetches the owner's own published routes from the frontend over
the internal Docker network (`FRONTEND_INTERNAL_URL`, default
`http://frontend:3000`) as an anonymous visitor — no cookies are forwarded, so
drafts and auth-gated content are excluded by construction, not by a filter.
Exported routes are fixed: `/`, `/blog`, `/resume`, `/me`, `/professional`,
`/projects`, `/portfolio/classic`, `/portfolio/immersive`, plus `/blog/<slug>`
and `/projects/<slug>` for every published document. HTML/CSS links and
same-origin assets are rewritten to relative paths so the export is portable;
skill icons are bundled locally at `assets/skill-icons/<name>.svg`; external
URLs are left untouched.

The export is fail-loud: any non-200 page, any redirect that leaves the
exported path set, a residual `/api/` reference, or a `..` path segment
aborts the whole export rather than shipping a partial/broken site. Only one
export can run per user at a time.

### Response codes

```text
200  application/zip attachment — the export succeeded
401  missing or invalid session
404  no profile handle on this account
429  a site export is already running for this account
502  export failed for one or more routes; body includes a `failures` list
```

### Snapshot contract

The zip is a point-in-time snapshot, not a live mirror — re-download
(re-export) to pick up new content. There is no compatibility promise about
the internal layout of the archive (`export-manifest.json` fields, directory
names) across PersonaOS versions; treat each export as self-contained.

```bash
curl -b "<session-cookie>" -o site.zip "$API_URL/export/site.zip"
```

## Competitive Programming

### Daily CP dashboard contract

Daily CP is an authenticated dashboard feature, not a REST/OpenAPI surface and
has no curl contract. Dashboard clients use the typed `dailyCp` tRPC router
instead:

- `dailyCp.state` returns the selected-day view, active challenge, server-derived
  statistics, supported providers, and the next reset timestamp.
- `dailyCp.generate({ provider })` creates a live problem snapshot for one of
  `codeforces`, `atcoder`, `kattis`, or `codechef`.
- `dailyCp.resolve({ challengeId, outcome })` accepts only `solved` or `failed`.
- `dailyCp.regenerate({ challengeId })` replaces a pending challenge with another
  live problem from the same provider.

The backend derives the day and `nextResetAt` in `Africa/Casablanca`. A pending
row from a previous Casablanca day remains historical but is not active and
does not enter that day's outcome counters.

Every generated row is an immutable provider problem snapshot. Its status is
`pending`, `solved`, `failed`, or `regenerated`; a partial unique index permits
only one pending row per owner and day, and a unique owner/provider/day/external
ID index prevents a same-day repeat. RLS permits authenticated owners to select
their own rows, while `anon` and `authenticated` have no direct Data API write
privileges; all challenge creation and transitions use the backend `service_role`.
Regeneration fetches the replacement before its transaction changes the active
row, so an upstream failure preserves the existing challenge.

The state response reports today, all-time, and per-provider totals. Pending
rows count toward total generated only; regenerated rows are separate from
failures; solve rate is `solved / (solved + failed)` and is zero when the
denominator is zero.

Problem discovery is live and backend-controlled: Codeforces uses its official
API; AtCoder uses the community-maintained AtCoderProblems dataset; Kattis
parses its public problem directory (including pagination); and CodeChef uses
its experimental first-party practice-listing endpoint. No client input can
choose an upstream origin or problem URL.

## Culture integrations

The culture feature follows the existing external-services boundary: authenticated
dashboard configuration writes owner-scoped `provider_credentials` records;
provider requests and OAuth token management run in the backend. The public
frontend calls the backend with its existing server-side API credentials.

| Method | Path                                             | Purpose                                                                                     |
| ------ | ------------------------------------------------ | ------------------------------------------------------------------------------------------- |
| GET    | `/api/integrations/providers`                    | Configuration field names, connection state, and publication state; no stored secret values |
| PUT    | `/api/integrations/providers/:provider`          | Save allowed configuration fields in `{ "entries": { ... } }`                               |
| GET    | `/api/integrations/providers/:provider/connect`  | Start Spotify or Trakt authorization                                                        |
| GET    | `/api/integrations/providers/:provider/callback` | Complete owner-bound authorization                                                          |
| POST   | `/api/integrations/providers/trakt/device/start` | Start Trakt activation with a code and no redirect URI                                      |
| POST   | `/api/integrations/providers/trakt/device/poll`  | Check approval at Trakt's interval and save the resulting account tokens                    |
| DELETE | `/api/integrations/providers/:provider`          | Disconnect a culture provider and clear its cache, or reset the public filter               |
| GET    | `/api/public/integrations/:handle/culture`       | Read `books`, `music`, `viewing`, and `lastfm` sections through the public API-key gate     |

Culture providers are `goodreads`, `spotify`, `trakt`, and `lastfm`. Last.fm uses a username and API key saved through the dashboard; it has no OAuth connect or callback. New configurations
start private. Set `published` to the string `"true"` only when opting into public
display; use `"false"` to make the provider private again. OAuth connection does
not itself publish activity.

Sections return `provider`, `status`, `updatedAt`, and `items`. Status is one of
`not-connected`, `private`, `ready`, `stale`, or `unavailable`. A failed provider
refresh must preserve its last successful snapshot, and must not block the other
providers. The Culture page is a recent-activity display, not a full library export.

The owner-scoped `public-filter` provider stores a JSON string array in `blockedWords` through the same authenticated configuration endpoint. The provider summary includes the editable `blockedWords` array. When unset, the list defaults to `ASMR`; an explicit `[]` disables the filter. Public integration responses filter matching displayed text after reading provider caches. The public API key gate runs before the filter settings lookup.

For required accounts, database-backed credential configuration, callback URLs,
and the Netflix/SIMKL distinction, see the
[culture setup guide](culture-integrations-setup.md).

## Token For Manual Tests

For browser testing, sign in through the app or `POST /api/auth/login`; the backend sets the session cookie.

For curl-only bearer-token testing, request a password token from Supabase Auth with the local Supabase URL and publishable key:

```bash
SUPABASE_URL=http://127.0.0.1:54321
DATABASE_URL=postgresql://postgres:<password>@host.docker.internal:54322/postgres
SUPABASE_PUBLISHABLE_KEY=<publishable-or-anon-key>

curl -X POST "$SUPABASE_URL/auth/v1/token?grant_type=password" \
  -H "apikey: $SUPABASE_PUBLISHABLE_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "hamza@example.com",
    "password": "password-123"
  }'
```

Copy `access_token` into `TOKEN`.
