# Backlog

Deferred or scoped-out work identified during development. Not a sprint
board — just a durable place to park ideas so they survive past the
conversation/PR that surfaced them. Move an entry to the [progress log](progress.md)
once it's actually being worked.

## Self-hosted portfolio code export

**Status:** not started.

**Motivation:** people should be able to leave PersonaOS with an owned,
deployable portfolio artifact rather than only a hosted public URL.

**Scope:** generate a documented final-code export for a selected published
profile, including portfolio assets and deployment configuration required to
run it on the person's own server. This is distinct from the current database
content migration export and excludes backend credentials and Supabase
service-role keys.

**Proposed approach:** define the static/runtime boundary, generate a
profile-specific artifact from published content, include a minimal deployment
guide, and prove the exported artifact runs without the PersonaOS control
plane.

**Risks:** secret exclusion, third-party asset licensing, update ownership,
and the contract between generated output and future PersonaOS versions need
explicit design before implementation.

## Per-profile API clients

**Status:** not started.

**Motivation:** the deployment-wide `PERSONAOS_API_KEY` and
`PERSONAOS_API_SECRET` authenticate the frontend server to the backend; they
are not a profile lookup mechanism and must not be exposed to browsers. A
future public/programmatic API may need credentials that are owned by one
profile, can be independently rotated, and identify that profile without
relying on a deployment-level `PERSONAOS_PUBLIC_HANDLE`.

**Scope:** this is for programmatic API access, not for rendering the current
single public site. Continue using `PERSONAOS_PUBLIC_HANDLE` for that purpose
until the public-site selection flow is deliberately redesigned.

**Proposed approach:**

1. Add a dedicated `profile_api_clients` (or similarly named) table, separate
   from `provider_credentials`, with `owner_id`, a public key/id, a hashed
   secret, scopes, timestamps, expiry/revocation state, and last-used metadata.
2. Generate and display the plaintext secret once in the dashboard; store only
   its hash, use constant-time verification, and support rotation/revocation.
3. Add a narrowly scoped authentication middleware and audit/rate-limit policy
   before permitting any profile-scoped API routes.
4. Never store PersonaOS API secrets as raw values in `provider_credentials`;
   that table is reserved for third-party provider settings such as GitHub and
   Goodreads.

**Risks:** credential lifecycle, rate limiting, scope design, and auditing are
security requirements, not optional polish. Do not use an API secret as a
public tenant-routing mechanism.

## Profile lookup by email

**Status:** not started.

**Motivation:** today the only ways to resolve a profile are by `handle`
([`findProfileIdByHandle`](../apps/backend/src/db/profiles.ts)) or by
`ownerId`/auth user id ([`findHandleByProfileId`](../apps/backend/src/db/profiles.ts)).
Email-based resolution exists, but only as a **CLI-only helper duplicated in
two places** — `resolveOwnerIdByEmail()` in both
[`scripts/export-content.ts`](../apps/backend/src/scripts/export-content.ts)
and [`scripts/import-starter-content.ts`](../apps/backend/src/scripts/import-starter-content.ts).
It works by paginating through `supabaseAdmin.auth.admin.listUsers()` and
scanning for a matching (normalized) email — no HTTP route exposes this, and
it doesn't scale past a few thousand users since it's a linear scan.

**Use cases this would unlock:**

- Admin/support tooling — look up a user's handle/profile from their email
  without shelling into the DB.
- Account-recovery UX — "what's my handle?" for a signed-in-by-email user who
  forgot it.
- A building block toward the multi-tenant handle-resolution roadmap already
  noted in [deployment.md §6](operations/deployment.md) (today
  `PERSONAOS_PUBLIC_HANDLE` is a single fixed env var; serving multiple
  tenants from one deployment will eventually need to resolve "which handle"
  from something other than a hardcoded constant).

**Proposed approach:**

1. Extract `resolveOwnerIdByEmail` out of the two duplicated script copies
   into `services/profiles.ts` (e.g. `findProfileIdByEmail`), so scripts and
   any future route share one implementation instead of copy-pasted logic.
2. Investigate whether the installed `@supabase/supabase-js` version
   (`^2.108.2` at time of writing) supports a native filter on
   `auth.admin.listUsers()` to replace the full paginated scan — if so, use
   it; if not, keep the scan but note it as a scaling limit.
3. Add direct unit tests for the extracted function — it's currently only
   exercised indirectly through the import/export script tests.
4. **Decide the access surface deliberately.** Do not expose bare
   email → handle/profile resolution on a public or `requireApiKey`-only
   route — that's an account-enumeration vector (anyone with an email could
   discover whether it has a PersonaOS account and what handle it maps to).
   If an HTTP route is needed at all, gate it behind authenticated
   self-lookup (the signed-in user's own email only) or genuine admin auth —
   never an open lookup.

**Risks:** treat this as sensitive by default. The CLI scripts get a pass
today because they require server/operator access (Supabase service-role
key), not because email lookup is inherently safe to expose.

## Friendly 500 page for API/backend errors

**Status:** not started.

**Motivation:** when the backend errors out (e.g. a failed Supabase call, an
unhandled exception in a route handler), the frontend currently surfaces
Next.js's default framework error page instead of anything on-brand. There's
already a `not-found.tsx` using a reusable
[`StatusPage`](../apps/frontend/src/components/status/status-page.tsx)
component for 404s — no equivalent `error.tsx` exists for 5xx/thrown errors.

**Proposed approach:**

1. Add `apps/frontend/src/app/error.tsx` (and consider `global-error.tsx` for
   errors thrown from the root layout itself, which `error.tsx` alone can't
   catch) using the same `StatusPage` component for visual consistency with
   the existing 404 page.
2. Decide what's safe to show: Next.js passes the thrown `Error` into
   `error.tsx`, but production builds strip most detail from it by design
   (only `message` and a redacted `digest` survive) — don't try to surface
   raw backend error bodies to the browser regardless.
3. Cover the common failure shapes distinctly if useful (e.g. "API
   unreachable" vs. a generic "something went wrong"), but a single friendly
   fallback is a reasonable v1.

## No-content-yet state per page

**Status:** not started.

**Motivation:** a freshly signed-up account (or the current production
deploy, pre-import) has an empty database — no portfolio/blog/project/resume
documents yet. Right now each page presumably either crashes, renders blank,
or shows some inconsistent fallback when its content query returns nothing;
there's no deliberate "you don't have any content here yet" UX.

**Proposed approach:**

1. Audit each public content page (`portfolio`, `blog`, `projects`, `resume`,
   `professional`, `personal`, `competitive` under `apps/frontend/src/app/`)
   for what currently happens when its backing content query returns empty —
   likely inconsistent today.
2. Design one reusable empty-state component (parallel to `StatusPage`) that
   each page renders with page-specific copy — e.g. "No projects yet" vs. "No
   blog posts yet" — rather than one generic message everywhere.
3. Distinguish "no content because nothing's published" (normal for a new
   account) from a genuine fetch error — this overlaps with the 500-page item
   above; both need the same underlying signal (did the request succeed with
   an empty result, vs. did it fail).

## Default not-found page for the 3D experience

**Status:** not started.

**Motivation:** the 3D app ([apps/experience-3d](../apps/experience-3d)) has
no equivalent of the frontend's `not-found.tsx` — an unknown route or a
failed world-load currently has no dedicated visual treatment matching the
experience's own aesthetic (night-match stadium, cinematic camera — see
[3d-experience.md](architecture/3d-experience.md)).

**Proposed approach:**

1. Design a 404/no-data screen that fits the existing stadium/night-match
   visual language rather than a generic web 404 — worth pulling inspiration
   from other 3D/WebGL portfolio sites for how they handle this
   (dimmed/fog-out scene, a single spotlit message, etc.) rather than
   defaulting to a flat HTML error page that breaks immersion.
2. Cover both cases: an actual unknown route (if/when the app gains real
   client-side routing beyond the single scrollytelling page) and a failed
   world snapshot load — today `loadStadiumWorld()` in
   [world-fixture.ts](../apps/experience-3d/src/world-fixture.ts) silently
   falls back to `FALLBACK_WORLD` on any fetch failure, which may be worth
   keeping as the ultimate safety net regardless of what UI is added on top.
