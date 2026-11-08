# PersonaOS 3D Designer Handoff

This handoff is for the current 3D product direction, not the rejected
free-exploration RPG prototype.

## Goal

Create the visual foundation for the live `experience-3d` app:

- one stadium `.glb` that works as a cinematic night-match backdrop
- a tunnel-to-bowl reveal mood
- clear stadium landmarks that support section framing
- no gameplay assumptions

The app is a scroll-driven presentation. The stadium is not an explorable map.

## Hard Constraints

- The entire world lives inside the stadium shell.
- The content scope is still resume + personal only.
- The mood is `cinematic night match`.
- The tone is playful and football-coded, but restrained.
- Avoid official FC Barcelona marks, logos, and real-player likenesses.
- Prioritize readable framing over dense decorative detail.

## What The Frontend Actually Does

The live app:

- renders the stadium GLB as a fixed WebGL backdrop
- drives the camera through keyframed vantages as the page scrolls
- pairs each vantage with a DOM content panel
- falls back to a content-only stacked layout on compact / no-WebGL paths

The current sections are:

- hero reveal
- experience
- skills
- personal
- contact

## What Must Be Designed

### 1. World Art Direction

Lock:

- palette
- floodlight mood
- material language
- stadium silhouette
- UI-overlay compatibility

Required decisions:

- deep navy / stadium-light base palette
- gold highlight language for achievements
- readable grass, concrete, metal, and light materials
- enough atmosphere to feel special without turning noisy behind text panels

### 2. Reveal-First Stadium Layout

Design for framing, not navigation.

The model should support:

- a tunnel-origin reveal
- a strong bowl-wide hero frame
- distinct visual reads for experience / skills / personal beats
- a central trophy landmark or equivalent visual focus

Deliverables:

- top-down layout image
- 3 to 5 key framed views
- one note describing what each framed area represents

### 3. Stadium Shell

Create the main environment asset with:

- pitch
- seating bowl
- tunnel
- perimeter structure
- floodlights / stadium lighting elements

Keep it practical:

- favor clean silhouette and depth layering over tiny props
- leave negative space where text panels need to read well
- optimize for mid-distance camera views, not first-person traversal

Deliverable:

- export-ready `stadium-world.glb`

### 4. Landmark Language

The current content model still uses these world semantics:

- `Arrival Gate`
- `Quest Fields`
- `Personal Grove`
- `Champions League Trophy`

These do not need to become separate buildings. They need to read clearly in
framed shots and support the section metaphors.

Deliverables:

- one concept frame or paint-over per landmark area
- one short note on what visual cue sells that area

### 5. Camera / Anchor Support

If the source DCC supports named empties/nulls, keep these anchors:

- `spawn_tunnel`
- `camera_reveal`
- `zone_arrival_gate`
- `zone_quest_fields`
- `zone_personal_grove`
- `landmark_trophy`

The current app no longer uses a free-walk player, but these names still matter
because the frontend/back-end contract and framing logic already reference them.

### 6. Export Package

Prepare:

- `stadium-world.glb`
- embedded or adjacent textures
- preview renders
- one top-down labeled map
- one short note with scene scale and axis conventions

## Non-Goals

- no explorable gameplay map
- no collectible system
- no required player avatar deliverable
- no official team branding

## Engineering Handoff Note

Engineering needs the lazy version first:

- one stable stadium asset
- consistent anchors
- readable framed zones

If a detail does not improve the framed presentation, skip it.
