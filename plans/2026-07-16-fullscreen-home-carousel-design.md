# Fullscreen Homepage Carousel Design

## Goal

Make the PersonaOS homepage lead with a commercial, full-viewport carousel
before the existing introductory copy. The carousel should explain the product
through four distinct portfolio outcomes, with thumbnail navigation inspired by
the accepted Smart Slider testimonial pattern.

## Experience

The carousel occupies the first viewport below the shared public header. Each
slide uses a full-bleed generated background image, a dark contrast overlay,
and concise product copy. Arrow controls and the thumbnail strip remain
available at all viewport sizes. The thumbnails use a crop of each slide's
background rather than abstract text tiles.

Slides are:

1. **Collect the person** - a structured source of truth for profile,
   projects, writing, and personal context.
2. **Publish every surface** - one profile powering the classic portfolio,
   public content routes, resume views, and related presentations.
3. **Step into the 3D experience** - the existing stadium-world portfolio as
   an immersive surface generated from the same profile data.
4. **Own the final result** - a future export that packages a person's final
   portfolio code for deployment to their own server. This slide describes the
   product direction without implying that the export feature already exists.

The existing homepage heading and supporting text move below the carousel,
followed by the existing profile-specific "Built for {name}" section and the
public route cards.

## Behavior And Accessibility

- The active slide advances on a fixed interval and wraps from the last slide
  to the first.
- Previous, next, and thumbnail controls remain explicit and also wrap at the
  ends.
- Autoplay pauses while the carousel is hovered or keyboard-focused and when
  the user requests reduced motion.
- Each active slide has an accessible group label; thumbnails remain a tablist
  with the active state exposed by `aria-selected`.
- Background media is decorative. The slide title and body remain real text
  with sufficient contrast independent of the image content.

## Assets

Generate four original wide images for the homepage instead of adding stock
images or external image dependencies. They should share a restrained
PersonaOS visual language: high-contrast, modern, editorial product imagery;
one stadium image for the 3D slide; no embedded text. Store the optimized
images under `apps/frontend/public/assets/home/`.

## Backlog

Track the self-hosted export as a separate GitHub Project `Todo` issue and add
the same idea to `documentation/backlog.md`. The future scope is an exportable
portfolio artifact plus deployment instructions, not raw database backup or
the current content migration exporter.

## Validation

- Extend the focused homepage test for all four slides, wraparound controls,
  and the autoplay lifecycle.
- Run the focused frontend test and frontend lint in Docker.
- Run the frontend locally through the Docker dev stack and inspect desktop and
  mobile screenshots with Playwright to confirm the full-viewport composition,
  background imagery, thumbnail rail, and non-overlapping text.
