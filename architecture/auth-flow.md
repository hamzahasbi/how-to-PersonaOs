# Auth Flow

PersonaOS keeps authentication behind the backend boundary. The frontend never calls Supabase Auth directly for login, OAuth, logout, or session checks.

## Boundary

```text
Browser -> PersonaOS backend -> Supabase Auth/Postgres
```

- Frontend starts auth by calling PersonaOS API routes under `/api/auth`.
- Backend owns Supabase provider URLs, code exchange, cookies, and profile bootstrap.
- Supabase Auth owns identities in `auth.users`.
- PersonaOS owns app profile rows in `public.profiles`.

## Password Login

1. Frontend posts email and password to `POST /api/auth/login`.
2. Backend calls Supabase password auth.
3. Backend sets `personaos-access-token` and `personaos-refresh-token` HTTP-only session cookies.
4. Frontend reads the current user through `GET /api/auth/me`.

## OAuth Login

1. Frontend redirects the browser to `GET /api/auth/oauth/google?next=/dashboard`.
2. Backend creates a request-scoped Supabase PKCE auth client.
3. Backend asks Supabase for the provider authorization URL.
4. Backend stores Supabase's PKCE verifier in a short-lived HTTP-only `personaos-oauth-code-verifier` cookie scoped to `/api/auth/callback`.
5. Browser follows the redirect to local Supabase Auth, then Google.
6. Google returns to Supabase Auth.
7. Supabase redirects back to `/api/auth/callback?code=...&next=...`.
8. Backend exchanges the code with the stored PKCE verifier.
9. Backend creates `public.profiles` for the OAuth user if it does not already exist.
10. Backend sets the `personaos-access-token` HTTP-only cookie and redirects to `next`.

The fallback frontend route `/auth/callback` only forwards query parameters to `/api/auth/callback`. The backend callback is the canonical callback.

## Cookies

- `personaos-access-token`: session cookie used by protected API routes.
- `personaos-refresh-token`: session cookie used by the backend to renew an expired access token while the browser session remains open.
- `personaos-oauth-code-verifier`: short-lived PKCE verifier used only during OAuth callback exchange.

Both cookies are HTTP-only session cookies. When `/api/auth/me` detects an expired access token, it uses the refresh token and replaces both cookies. The frontend cannot and should not set them manually. Closing the browser normally clears the session cookies.

## Profile Bootstrap

OAuth creates an identity in `auth.users`, but PersonaOS content ownership depends on `public.profiles`.

After a successful OAuth callback, the backend inserts a missing profile row:

- `id`: Supabase user id.
- `display_name`: email prefix when no app profile exists yet.
- `handle`: email prefix plus the first 8 characters of the user id.

The insert is skipped when the profile already exists, so later user profile edits are not overwritten on login.

## Local Google OAuth

Google Cloud must allow this redirect URI exactly:

```text
http://localhost:54321/auth/v1/callback
```

Local Supabase then redirects back to the PersonaOS backend callback configured by `/api/auth/oauth/google`.

Required local environment shape:

```text
SUPABASE_URL=http://host.docker.internal:54321
SUPABASE_PUBLIC_URL=http://localhost:54321
NEXT_PUBLIC_API_URL=/api
```

`SUPABASE_URL` is what backend containers use. `SUPABASE_PUBLIC_URL` is what the browser can open.
