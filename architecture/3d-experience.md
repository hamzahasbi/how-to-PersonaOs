# 3D Experience (experience-3d)

Current-state doc for the standalone 3D app.

## Concept

A restrained, cinematic "scrollytelling" site. The stadium GLB is an
atmospheric night-match backdrop, not a playable map. One input — scroll —
drives a keyframed camera between fixed vantage points, each paired with a clean
content panel. No game: no zones-to-collect or keyboard steering. The content
panels can still show portfolio data such as the player avatar, counters, and
profile details.

## Interaction

- A fixed WebGL backdrop sits behind normal scrolling DOM sections.
- Scroll progress (0→1) is held in a ref; the camera rig reads it each frame and
  interpolates across a vantage keyframe array (`buildVantages` in
  `src/stadium-experience.tsx`). Scrolling never re-renders React.
- Five sections/vantages: hero reveal → experience → skills → personal → contact.
- A right-edge rail shows the active section and jumps on click.
- Fallback (compact viewport, no WebGL, or reduced-motion): a content-only
  stacked-card version with no canvas.

## Data

- Renderer only. Content loads once from `public/data/world.json` via
  `loadStadiumWorld()`; on any failure it falls back to a bundled world and
  backfills missing sections.
- `world.json` is an exported snapshot of the backend `GET /api/public/world/:handle/3d`
  payload. The API Reference documents the public endpoint contract.
- The main-portfolio link (`ui.mainSiteUrl`) is set by the dashboard refresh
  form when regenerating the snapshot, with env/profile website fallbacks for
  scripts and public reads.

## Files

- `src/app.tsx` — scroll layout, section panels, rail, fallback, world loading.
- `src/stadium-experience.tsx` — WebGL backdrop, night lighting, scroll camera
  rig, vantage keyframes.
- `src/world-fixture.ts` — `StadiumWorld` types, bundled fallback, loader.
- `public/models/stadium-world.glb` — the only active model asset.
- `public/data/world.json` — the content snapshot.

## Retuning

Camera framing per section lives in `buildVantages(radius, sizeY, reveal)` —
edit the position/lookAt for a stop to change how it's framed. Lighting is in
`NightMatchLighting`.
