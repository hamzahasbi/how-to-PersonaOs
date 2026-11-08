# PersonaOS 3D Backend Subagent Spec

This spec is for backend work that feeds the current 3D app, not the rejected
RPG / free-exploration prototype.

## Goal

Support the live standalone 3D experience with one backend-shaped payload and
one static export:

- one protected read endpoint
- one frontend-ready `world.json` payload
- one export path that writes the snapshot the 3D app loads at startup
- no follow-up runtime API calls from the 3D frontend

## Current Frontend Contract To Match

The live app is documented in `documentation/architecture/3d-experience.md`.

The important part is the data contract, not the old interaction model:

- the frontend loads `public/data/world.json` once via `loadStadiumWorld()`
- it renders a scroll-driven stadium presentation
- it builds five content sections from the payload:
  - hero
  - experience
  - skills
  - personal
  - contact

The payload still carries stadium-world semantics because the content is framed
through stadium copy and landmarks, but the frontend is now a renderer only.

## Required Endpoint

Add or maintain:

```text
GET /api/public/world/:handle/3d
```

This endpoint must use the existing API-key + API-secret read guard pattern
already used by public content routes.

## Data Sources

Use only:

- published `professional` content
- published `personal` content

Do not pull:

- blog
- projects outside what is already expressed through professional/resume content
- competitive programming

Keep the payload narrow.

## Response Shape

Return one JSON document with this shape:

```json
{
  "meta": {
    "version": "v1",
    "generatedAt": "2026-07-06T00:00:00.000Z",
    "theme": "cinematic-night-match",
    "atmosphere": "night-match-floodlights"
  },
  "player": {
    "name": "Hamza Hasbi",
    "title": "Full-stack systems engineer",
    "shirtNumber": 8,
    "style": "blaugrana-inspired"
  },
  "intro": {
    "headline": "Start in the tunnel, emerge into the bowl, then move freely.",
    "cta": "Enter the stadium"
  },
  "views": {
    "spawnAnchor": "spawn_tunnel",
    "revealAnchor": "camera_reveal"
  },
  "zones": [
    {
      "id": "arrival-gate",
      "anchorName": "zone_arrival_gate",
      "title": "Arrival Gate",
      "eyebrow": "Player Card",
      "accent": "#8b5cf6",
      "summary": "Identity, role, and short intro.",
      "bullets": ["..."],
      "stats": ["..."]
    }
  ],
  "landmarks": [
    {
      "id": "trophy-landmark",
      "anchorName": "landmark_trophy",
      "title": "Champions League Trophy"
    }
  ],
  "quests": [
    {
      "id": "quest-1",
      "zoneId": "quest-fields",
      "title": "Lead FullStack Developer — Example",
      "detail": "2024-now"
    }
  ],
  "stats": {
    "attributes": [{ "label": "TypeScript", "value": "Proficient" }],
    "badges": ["..."],
    "counters": [{ "label": "Career Quests", "value": 3 }]
  },
  "ui": {
    "mainSiteLabel": "Main site",
    "mainSiteUrl": "https://example.com"
  }
}
```

## Backend Behavior Rules

- The backend owns the 3D world shaping.
- The frontend should not reinterpret raw resume/personal content into stadium
  semantics.
- The payload must be deterministic for the same underlying content.
- Missing sections should degrade with empty arrays or fallback values, not
  missing top-level keys.
- `ui.mainSiteUrl` should come from the professional `website` frontmatter and
  fall back to `/`.

## Static Export Requirement

The export target is:

```text
apps/experience-3d/public/data/world.json
```

The frontend contract should stay stable whether the data came from the API
route or the exported snapshot.

## Files Expected

Create or modify only what is needed:

- `apps/backend/src/services/world.ts`
- `apps/backend/src/routes/world.ts`
- `apps/backend/src/scripts/export-3d-world.ts`
- `apps/backend/src/openapi.ts`
- `apps/backend/src/server.ts`
- focused tests near those files

## Acceptance Criteria

- `GET /api/public/world/:handle/3d` returns `200` with the full shape above
- route uses existing API-key/API-secret protection
- payload contains only professional/resume + personal driven content
- payload includes the anchor names the frontend expects
- export script writes a valid `world.json`
- the 3D app can boot from the exported file without renderer rewrites

## Non-Goals

- no live per-zone API
- no GLB parsing on the backend
- no blog or competitive-programming expansion
- no gameplay logic on the frontend

## Verification

Minimum checks:

- focused backend tests for world service and route
- focused test for export script
- backend lint
- backend build
