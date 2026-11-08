# Culture page: accounts, credentials, and setup

The culture page supports StoryGraph, Spotify, Last.fm, and Trakt. Netflix history reaches the page after you import it into Trakt; PersonaOS does not connect to Netflix directly.

The public page has Books, Music, Films, and Series tabs. Repeated Trakt episodes appear as one series card per collection, with a recent episode count in viewing history. Each populated tab charts available genre tags; Books and any provider response without genre tags show the collection's category mix. The shared public integration filter starts with “ASMR” blocked. These charts describe the displayed items, not a complete lifetime history.

## Agreed direction

- Books: StoryGraph through an Integuru shelf action, synced manually and stored in the culture cache.
- Music: Spotify Web API using your existing Premium account, with optional Last.fm scrobble stats.
- Films/series: Trakt for tracked viewing history, ratings, and watchlist. Trakt can import Netflix history through its streaming connection. SIMKL remains an alternative that PersonaOS has not integrated. TMDB metadata is optional.
- PersonaOS's backend makes provider data requests, stores credentials, and serves cached, explicitly selected public data to the frontend.
- Initial assumption: your personal culture page. Connecting accounts for every PersonaOS user would require additional product and provider-access work.

Connect Netflix to Trakt in Trakt's app or streaming settings. PersonaOS reads the resulting Trakt history through its backend proxy. Spotify and Trakt sign-in use browser redirects, while token exchanges and subsequent data calls belong on the backend.

## What to provide

| Item                                                                 | Required for                                     | Where to provide it                                                                    |
| -------------------------------------------------------------------- | ------------------------------------------------ | -------------------------------------------------------------------------------------- |
| StoryGraph username, Integuru account ID, integration ID and API key | Books                                            | Integrations dashboard; keep the API key secret                                        |
| Spotify developer app Client ID and Client Secret                    | Music                                            | Backend-only secret configuration when implemented                                     |
| Authorization of your Spotify account                                | Personal listening data                          | Future PersonaOS Connect Spotify flow; no password handoff                             |
| Trakt profile, developer app Client ID, and account authorization    | Trakt viewing history, ratings, and watchlist    | Profile URL in the handoff; app configuration on backend and future Connect Trakt flow |
| Verified GitHub account connected to Trakt's developer portal        | Creating a Trakt API app                         | Connect it yourself in Trakt's developer settings                                      |
| SIMKL account and successful Netflix import                          | Viewing history                                  | Set up in SIMKL and your browser; report whether history appears correctly             |
| SIMKL developer app credentials and account authorization            | Backend viewing sync                             | Backend-only configuration and future Connect SIMKL flow                               |
| TMDB API Read Access Token                                           | Optional translated details or alternate artwork | Backend-only secret configuration                                                      |
| Production public origin and backend callback origin                 | OAuth setup                                      | Implementation handoff; domains are not secrets                                        |
| Public visibility choices, language, region                          | Culture page content                             | Use the checklist at the end                                                           |

Provider app credentials belong in the authenticated **Integrations dashboard**, backed by the existing owner-scoped `provider_credentials` database table. OAuth access and refresh tokens are also backend-managed database values. The frontend receives configured field names and connection/publication status, never saved secret values. This follows the existing external-service pattern; there are no Spotify or Trakt API-key environment variables to populate.

## Public integration word filter

Apply migration `0013_public_integration_filter.sql` after `0012_lastfm_stats.sql`. In **Dashboard → Integrations → Public integration filter**, enter one blocked word or phrase per line (commas also separate entries) and save. The list accepts up to 50 entries of 60 characters each and uses case-insensitive substring matching. It defaults to `ASMR`; saving an empty list disables word filtering. It is owner-scoped and stored in the backend database, so no `.env` setting is needed. Use the same dashboard in local and production, which have separate databases.

The backend filters public StoryGraph, Spotify, Trakt, and Last.fm culture items, the legacy Goodreads reading list, GitHub README responses, football fixtures and details, and coding stats. Matches are case-insensitive and include titles, names, artists, authors, genres, and other displayed text. Matching list entries are removed; matching standalone text in structured details is redacted. Cached provider data remains intact, so changes apply to the next public response without another provider request. This is a display filter; it does not delete or change data at the source.

Local Caddy serves `https://127.0.0.1` and redirects HTTP to HTTPS. Set `PUBLIC_APP_ORIGIN=https://127.0.0.1` in `.env.backend` and include that origin in `ALLOWED_ORIGINS`. Open `https://127.0.0.1/dashboard?section=integrations` and sign in there before connecting. Register `https://127.0.0.1/api/integrations/providers/spotify/callback` in the Spotify app. Spotify rejects `localhost` as a redirect URI. Caddy uses its internal CA in Docker, so the browser must trust its root certificate to complete local authorization. The local Trakt dashboard connection uses a device activation code instead of a callback; save your Trakt Client ID there. Open the Trakt activation link, enter the shown code, then click Check connection. Production continues to use the HTTPS browser callback. Provider Client IDs and Secrets are saved in the dashboard, not `.env`.

On macOS, after starting Caddy, you can inspect its root certificate and add it to the system trust store:

```sh
docker cp personaos_caddy_dev:/data/caddy/pki/authorities/local/root.crt /private/tmp/personaos-caddy-root.crt
openssl x509 -in /private/tmp/personaos-caddy-root.crt -noout -subject -fingerprint -sha256
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain /private/tmp/personaos-caddy-root.crt
```

Trust only the certificate copied from your own local Caddy container. Browsers with a separate certificate store may also need the certificate imported there. [Caddy's Docker HTTPS instructions](https://caddyserver.com/docs/running#local-https-with-docker) describe this step.

The existing credential table stores text values, not application-encrypted ciphertext. Database access controls and restricted backups remain important. Do not paste secrets, refresh tokens, Netflix cookies, or passwords into chat, documentation, commits, or frontend environment variables. No Netflix, Spotify, or Goodreads account password is needed by PersonaOS. `PUBLIC_APP_ORIGIN` is deployment configuration used to construct callbacks, not an API credential.

## 1. StoryGraph

Apply migration `0014_storygraph.sql`. In **Dashboard → Integrations → StoryGraph**, enter the StoryGraph username, Integuru shelf integration ID, account ID and API key, then save. Set public display as desired and select **Sync books**. The backend requests all pages of the currently-reading, read and to-read shelves; an unsuccessful or incomplete sync leaves the last successful data intact. Public page views read the stored cache and never trigger Integuru requests. The dashboard displays the last successful sync time. The API key stays on the backend. Run the local migration with `docker-compose -f docker-compose.dev.yml run --rm backend bun run db:migrate:local` before configuring the integration.

## 2. Spotify

### Steps for you

1. Sign in to the [Spotify developer dashboard](https://developer.spotify.com/dashboard) with the account that will own the app. You already have Premium; keep that subscription active for development-mode access.
2. Create an app for PersonaOS and enable Web API use. Supply your real website and application description as requested.
3. Obtain the Client ID and Client Secret. Keep the secret in backend configuration only.
4. Register the callback URL for your environment: `https://127.0.0.1/api/integrations/providers/spotify/callback` locally, or your HTTPS origin with the same path in production.
5. Add your account to the app's authorized development users as required by the dashboard.
6. Click **Connect Spotify** in PersonaOS and approve the selected read permissions. The backend obtains and manages tokens; you should not manually generate refresh tokens.

Suggested description: “PersonaOS displays my selected listening activity and favorite artists on my personal culture page.”

### Permissions and display choices

| Display                 | OAuth scope                   |
| ----------------------- | ----------------------------- |
| Currently playing track | `user-read-currently-playing` |
| Recently played tracks  | `user-read-recently-played`   |
| Top tracks and artists  | `user-top-read`               |

Choose which of these should be public. Account authorization alone must not publish all fetched listening data. The first version does not need playback control, playlist modification, private playlists, or streaming audio.

Spotify requires HTTPS callbacks except for permitted loopback IP development redirects; `localhost` is not accepted. Registered and requested redirects must match. Development mode currently permits up to five authorized users and requires the app owner to have Premium. That user limit concerns connected accounts, not visitors reading a cached public page. Recent playback is not an unlimited lifetime listening archive.

Sources: [redirect rules](https://developer.spotify.com/documentation/web-api/concepts/redirect_uri), [quota modes](https://developer.spotify.com/documentation/web-api/concepts/quota-modes), [scopes](https://developer.spotify.com/documentation/web-api/concepts/scopes).

### Optional Last.fm listening stats

Your Last.fm account is already connected to Spotify for scrobbling. Apply database migration `0012_lastfm_stats.sql` in each environment, then create a [Last.fm API account](https://www.last.fm/api/account/create). Enter your **Last.fm username** and **API key** in **Dashboard → Integrations → Last.fm**. Save, and set **Public display** to **Public** to show last-month top artists and tracks with play counts on `/culture`. The API key is saved in the backend credential database; it does not go in `.env` or the browser. This read-only integration needs no Last.fm client secret, OAuth redirect, or new Spotify permission. Use the same dashboard steps locally and in production; each environment has its own database configuration.

Leave Last.fm's API-account **Callback URL** blank if the form permits it. If the form requires a URL, use `https://hamzahasbi.me/` as the application URL. PersonaOS does not invoke Last.fm authorization, so that URL receives no callback in this integration. You do not need a localhost callback or separate local API account.

The backend requests [top artists](https://www.last.fm/api/show/user.getTopArtists) and [top tracks](https://www.last.fm/api/show/user.getTopTracks) using Last.fm's `1month` period and caches the normalized result for one hour. The public integration word filter applies to these stats. Counts cover every service scrobbled to that Last.fm account, and Spotify plays appear only after Last.fm receives their scrobbles. Last.fm [requires attribution](https://www.last.fm/api/tos), which the Music panel displays.

## 3. Netflix history through Trakt

For the integration currently implemented in PersonaOS, open [Trakt streaming settings](https://trakt.tv/settings/scrobblers) and connect Netflix. Select the intended Netflix profile and ask Trakt to import existing history. Confirm a known film or episode appears in Trakt, then connect Trakt in PersonaOS's Integrations dashboard and enable Public display. PersonaOS's Films and Series tabs read the resulting Trakt history. The Trakt API data does not reliably identify which title came from Netflix, so the page labels it as Trakt viewing history.

Trakt's [account settings guide](https://forums.trakt.tv/t/account-settings/19102) says free accounts can connect one streaming service; check your account's current eligibility in Trakt. Its [streaming sync announcement](https://forums.trakt.tv/t/automatically-sync-your-streaming-services/34947) describes Netflix support and an initial import after connection. Netflix Premium alone does not configure this Trakt connection.

### Alternative: Netflix history through SIMKL

### Steps for you

1. Create or sign in to a [SIMKL account](https://simkl.com/).
2. Follow the [official Netflix import instructions](https://docs.simkl.org/how-to-use-simkl/advanced-usage/import-export-data/importing-to-simkl/supported-platforms/netflix) and install the extension linked by SIMKL.
3. Sign in to Netflix in that browser and select your own Netflix profile. Enable Netflix syncing in the extension and authorize SIMKL as instructed.
4. Wait for the initial import. Check a known film and series in SIMKL to confirm the correct profile was imported.
5. Register a PersonaOS developer application using the [SIMKL API setup](https://api.simkl.org/). Record its Client ID and any secret issued for the selected server-side authentication flow. Use the current authentication version supported by SIMKL.
6. After PersonaOS's OAuth flow is implemented, register its exact callback and connect your SIMKL account. Credentials and access tokens stay on the backend.

The extension imports account history, including activity from other viewing devices, according to SIMKL's guide. Ongoing collection depends on the browser extension being able to run and access the correct Netflix session. PersonaOS cannot refresh history that has not reached SIMKL; this is not guaranteed real-time Netflix playback tracking.

If selecting SIMKL, provide your SIMKL profile URL and confirmation that the import succeeded. Netflix credentials and cookies never go to PersonaOS. SIMKL is not required merely to connect Trakt; these are separate viewing-data options.

SIMKL may contain history from other sources too. Decide whether the page should show all your SIMKL viewing or strictly Netflix activity. Strict Netflix-only filtering is a requirement to validate against returned provenance data, not something to infer from a title being available on Netflix today.

Use the [current rate-limit rules](https://api.simkl.org/resources/rate-limits) for the chosen authentication version; free access has quotas. Check activity timestamps before downloading changed history, following the [sync guide](https://api.simkl.org/guides/sync). Follow [SIMKL's attribution and API rules](https://api.simkl.org/api-rules).

## 4. Trakt

Trakt supplies the viewing activity already recorded in your Trakt account: watch history, ratings, and watchlist. Its standard API access is free; creating an API app does not require Trakt VIP. Some endpoints and account limits depend on the access tier. [API use policy](https://developer.trakt.tv/docs/api-use-policy)

### Steps for you

1. Create or sign in to your Trakt account and provide its profile URL.
2. Open Trakt's developer/My Apps area through the [Create an App guide](https://developer.trakt.tv/docs/create-an-app). Connect a verified GitHub account; Trakt currently requires this for app creation.
3. Register a PersonaOS app with its real website and description. Review the branding and developer requirements during registration.
4. Save the **Client ID** in the Integrations dashboard. Local device authorization and production browser authorization do not require the Client Secret. Production browser authorization uses Authorization Code with PKCE.
5. Register your production HTTPS app origin with `/api/integrations/providers/trakt/callback` in Trakt. It must match the authorization request and token exchange, including case. The local device-code flow does not use a redirect URI.
6. Use **Connect Trakt** and authorize your account. Locally, follow the activation link and code shown in the dashboard, then click **Check connection**. PersonaOS's backend will exchange the code, store tokens, and handle refreshes. No account password or manually copied refresh token is needed.
7. Choose the public sections: recently watched, watching/progress where available, watchlist, and your ratings. Provide one known film and one episode already recorded in Trakt for validation.

Suggested app description: “PersonaOS displays my selected Trakt watch history, ratings, and watchlist on my personal culture page.”

The Client ID identifies the app; account tokens authorize access to your personal data. Keep tokens server-side. Use the same per-owner cache, visibility controls, error handling, and disconnect behavior described below. No write-back to Trakt is needed for the first display-only version. [Authentication documentation](https://developer.trakt.tv/docs/authentication-oauth)

PersonaOS now gets film and series posters from Trakt's summary API during the hourly culture refresh. The backend caches image bytes and serves them through a restricted same-origin proxy, as required by [Trakt's image guidance](https://developer.trakt.tv/docs/images). No TMDB key is needed for these covers; titles without a poster retain the text cover.

### Trakt versus Netflix versus SIMKL

| Choice | What PersonaOS reads                                | What you must arrange                                                                         |
| ------ | --------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| Trakt  | Your existing Trakt history, ratings, and watchlist | Trakt account, developer app, OAuth authorization, and a way to record your watching in Trakt |
| SIMKL  | Your SIMKL viewing library                          | SIMKL account/app and authorization; its extension if importing Netflix history               |
| Both   | Two independently connected libraries               | Both setups plus a defined source-of-truth and deduplication policy; defer unless needed      |

**Free Trakt API access does not mean free automatic Netflix syncing.** Netflix Premium does not include Trakt VIP or developer Premium. Current eligibility and pricing for Trakt's Netflix/streaming sync must be checked in your Trakt account before relying on it; a free Netflix-to-Trakt path has not been verified here. Without that connection, PersonaOS can still display activity you already track in Trakt.

Start with Trakt if that is where you want to maintain your viewing history. Choose SIMKL if its Netflix import better meets your needs. Do not assume SIMKL automatically forwards its imported history to Trakt: that bridge is not part of this proposal.

## 5. Optional TMDB metadata

TMDB adds film/series descriptions, posters, and translated metadata; it does not supply your Netflix viewing history.

1. Create a TMDB account and open its API settings on desktop.
2. Register the PersonaOS application and review/accept the provider's terms yourself.
3. Store the **API Read Access Token** in backend secret configuration when that integration exists.
4. Provide the preferred metadata language and streaming region. Suggested starting values are Arabic or English, and Morocco (`MA`), but confirm your preference.

Noncommercial API use is free with required attribution; commercial use has separate terms. Streaming availability, if added, also requires the watch-provider attribution. Translations and regional availability can be missing, so preserve original titles and provide a sensible language fallback.

Sources: [registration](https://developer.themoviedb.org/docs/getting-started), [licensing FAQ](https://developer.themoviedb.org/docs/faq), [watch providers](https://developer.themoviedb.org/reference/tv-series-watch-providers).

## Backend behavior to implement

This section is a proposed implementation contract, not deployed behavior.

- Frontend → PersonaOS backend → cache/database → provider. Expose fixed provider operations, never an arbitrary URL proxy.
- Authenticate account connection and manual refresh actions. Keep OAuth state bound to the signed-in owner and store tokens securely on the server.
- Publish only the fields and categories you explicitly enable. Keep raw feeds, credentials, and private provider responses out of public endpoints and logs.
- Reuse cached results across page visits and deduplicate concurrent refreshes. Apply per-provider limits, bounded retries, and `Retry-After` handling.
- Preserve the last successful snapshot on failure; label stale data and expose the last successful sync time. Never erase a library because a provider timed out or pagination was incomplete.
- Proposed starting refresh intervals: books every 6 hours, Trakt or SIMKL activity every hour, Spotify top items every 24 hours, recent tracks every 15 minutes, and current playback at most once per minute while the page is active. These are tuning defaults, subject to provider policies and actual quotas, not guaranteed freshness.
- Expire the “currently playing” indication when its snapshot is too old; do not keep presenting a cached track as live after the provider becomes unavailable.
- Honor each provider's storage, branding, attribution, and deletion requirements. Disconnecting an account stops refreshes and removes authorization; public cached data must also be removable.
- Provider API calls belong on the backend. Cover/poster images and outbound provider links may still contact provider CDNs from visitors' browsers; decide separately if you also require media proxying, subject to provider terms.

## Copyable handoff checklist

Fill in non-secret values; for credentials, report only “configured” or “not created yet.”

```text
PersonaOS profile handle:
Public site origin:
Backend public origin (or same-origin routing):
Audience: my personal page / accounts for multiple users
Page language(s):
Film/series region:

Goodreads profile URL:
Bookshelf RSS URL: saved in dashboard / not saved yet
Show book ratings and reviews publicly: yes / no
Three Arabic test books (links or ISBNs):

Spotify developer app: created / not created
Spotify backend credentials: configured / pending implementation
Public music sections: current track / recent tracks / top tracks / top artists
Last.fm username and API key: saved in dashboard / not saved yet
Last.fm public display: Public / Private
Public integration blocked words: ASMR / custom list / disabled

Primary viewing source: Trakt / SIMKL
Trakt profile URL:
Verified GitHub connected to Trakt developer portal: yes / no
Trakt developer app and Client ID: created / not created
Trakt authorization: pending implementation / connected
Trakt public sections: history / watching / watchlist / ratings
Known Trakt test film and episode:
Netflix-to-Trakt sync: already working / not configured / not needed

SIMKL profile URL (if selected):
Netflix extension installed and correct profile imported: yes / no
SIMKL developer app: created / not created
Viewing scope: all selected provider history / Netflix only (provenance must be validated)
Public viewing sections: recently watched / watching / watchlist / ratings

TMDB metadata: include / defer
TMDB backend credential: configured / not created
Anything that must stay private:
```

## Readiness checks before publishing

- Goodreads feed loads from the backend without interactive login; representative Arabic titles and editions render correctly.
- A populated shelf, empty shelf, upstream error, and incomplete pagination are distinguishable.
- Spotify authorization and token refresh work; no-active-playback is a normal state.
- Last.fm top artists and tracks reflect the intended scrobbling account; terms in the saved public filter stay hidden.
- Netflix history first matches SIMKL, then matches the backend snapshot; the selected profile is correct.
- If Trakt is selected, a known film, episode, rating, and watchlist item match the backend snapshot; authorization refresh and revocation work. Netflix syncing is checked separately from Trakt API access.
- Secrets never appear in frontend bundles or public responses. Public output matches the selected visibility settings.
- Repeated visits use the cache; provider failure and quota exhaustion preserve a clearly marked last successful snapshot.
- Disconnect/reconnect and cached-data removal work. Required provider credits and links are present.

This guide does not create provider accounts, subscribe to APIs, or deploy the culture page.

## Dashboard cache status

The dashboard overview lists owner-scoped culture, football and competitive-programming cache bins. Badges show the last successful update and whether data is present; culture TTLs and football detail expiry also distinguish fresh, stale and partially fresh bins. Football fixture freshness depends on match timing, so those bins are labeled cached rather than fresh. Contest and poster caches are process-local and are not included in the persistent counts. Cache contents and credential-derived keys are not shown.

In **Dashboard → Cache**, the clear action removes your culture and football caches. StoryGraph books need a manual sync to reappear; other services may refill automatically. Competitive-programming history and process-local caches are not deleted.
