# Fullscreen Homepage Carousel Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the PersonaOS homepage lead with a looping, image-led, full-viewport carousel and record self-hosted portfolio export as future work.

**Architecture:** `PurposeSlider` owns client-side carousel state, timed movement, pause behavior, and accessible controls. `page.tsx` owns the four slide definitions and profile content. Static generated artwork lives in the frontend public directory. The export slide describes planned work only; the matching roadmap entry is created in the backlog and Project 4.

**Tech Stack:** Next.js 16, React, TypeScript, Tailwind CSS, Vitest, Testing Library, Docker Compose, GitHub CLI.

## Global Constraints

- Keep the shared public header and render the carousel before all homepage introductory copy.
- Use `min-h-[calc(100svh-6.5rem)]` for the carousel on desktop and mobile.
- Create four text-free wide images in `apps/frontend/public/assets/home/`.
- Retain explicit previous/next buttons and the accessible thumbnail tablist.
- Auto-advance every 7 seconds, pausing on hover, focus, and reduced-motion preference.
- Describe self-hosted export as coming soon, never as a shipped capability.
- Run frontend commands in Docker only.

---

## File Structure

- `apps/frontend/public/assets/home/profile-foundation.webp`: source-of-truth artwork.
- `apps/frontend/public/assets/home/publishing-surfaces.webp`: multi-surface artwork.
- `apps/frontend/public/assets/home/stadium-experience.webp`: stadium-world artwork.
- `apps/frontend/public/assets/home/self-hosted-export.webp`: planned export artwork.
- `apps/frontend/src/components/home/purpose-slider.tsx`: carousel state, controls, and slide rendering.
- `apps/frontend/src/app/page.tsx`: slide data and homepage order.
- `apps/frontend/src/__tests__/app/hitl-feedback.test.tsx`: carousel user-visible behavior.
- `documentation/backlog.md`: durable export roadmap record.

### Task 1: Test And Implement Fullscreen Carousel Behavior

**Files:**

- Modify: `apps/frontend/src/__tests__/app/hitl-feedback.test.tsx:266-299`
- Modify: `apps/frontend/src/components/home/purpose-slider.tsx`

**Interfaces:**

- Consumes: the existing `PurposeSlide[]` with `body`, `label`, `thumbnail`, and `title` fields.
- Produces: `PurposeSlider({ slides }: PurposeSliderProps)` with wrapping arrow controls, thumbnail tabs, and a 7-second loop.

- [ ] **Step 1: Write a failing test**

```tsx
it('loops the fullscreen purpose carousel through all product surfaces', async () => {
  const user = userEvent.setup({ advanceTimers: vi.advanceTimersByTime });
  vi.useFakeTimers();
  render(await Home());
  expect(screen.getByRole('region', { name: 'PersonaOS purpose slider' })).toHaveClass(
    'min-h-[calc(100svh-6.5rem)]',
  );
  expect(screen.getByRole('group', { name: /Collect the person/i })).toBeInTheDocument();
  await user.click(screen.getByRole('button', { name: 'Next PersonaOS purpose slide' }));
  expect(screen.getByRole('group', { name: /Publish every surface/i })).toBeInTheDocument();
  await user.click(screen.getByRole('button', { name: 'Next PersonaOS purpose slide' }));
  expect(screen.getByRole('group', { name: /Keep it alive/i })).toBeInTheDocument();
  await user.click(screen.getByRole('button', { name: 'Next PersonaOS purpose slide' }));
  expect(screen.getByRole('group', { name: /Collect the person/i })).toBeInTheDocument();
  vi.advanceTimersByTime(7000);
  expect(screen.getByRole('group', { name: /Publish every surface/i })).toBeInTheDocument();
  vi.useRealTimers();
});
```

- [ ] **Step 2: Verify the test is red**

Run `docker-compose -f docker-compose.dev.yml run --rm frontend bun run test src/__tests__/app/hitl-feedback.test.tsx`.

Expected: failure because the component has no full-viewport class or timed interval.

- [ ] **Step 3: Write the minimal component implementation**

```tsx
const AUTO_ADVANCE_MS = 7000;

useEffect(() => {
  if (isPaused || prefersReducedMotion || slides.length < 2) return;
  const interval = window.setInterval(() => {
    setActiveIndex((current) => (current + 1) % slides.length);
  }, AUTO_ADVANCE_MS);
  return () => window.clearInterval(interval);
}, [isPaused, prefersReducedMotion, slides.length]);
```

Import `useEffect` from React and `useReducedMotion` from `motion/react`. Add `isPaused` state. Put `min-h-[calc(100svh-6.5rem)]` on the region, retain the existing contrast background, and set `isPaused` with mouse and focus enter/leave handlers. Preserve `aria-selected` and current arrow wrapping.

- [ ] **Step 4: Verify green**

Run `docker-compose -f docker-compose.dev.yml run --rm frontend bun run test src/__tests__/app/hitl-feedback.test.tsx`.

Expected: the focused frontend suite passes.

- [ ] **Step 5: Commit the task**

Run `git add apps/frontend/src/components/home/purpose-slider.tsx apps/frontend/src/__tests__/app/hitl-feedback.test.tsx`, then run `git commit -m "feat: make homepage carousel loop fullscreen"`.

### Task 2: Add Artwork, Slide Copy, And Homepage Ordering

**Files:**

- Create: the four `.webp` images listed in File Structure.
- Modify: `apps/frontend/src/app/page.tsx:11-42`
- Modify: `apps/frontend/src/__tests__/app/hitl-feedback.test.tsx`

**Interfaces:**

- Consumes: the `PurposeSlide` type expanded with a required `backgroundImage: string` field.
- Produces: four product slides before the existing homepage copy.

- [ ] **Step 1: Extend the failing test**

```tsx
expect(screen.getByRole('tab', { name: /Step into the 3D experience/i })).toBeInTheDocument();
expect(screen.getByRole('tab', { name: /Own the final result/i })).toBeInTheDocument();
expect(screen.getByText(/exported for deployment to the person.s own server/i)).toBeInTheDocument();
```

- [ ] **Step 2: Verify the test is red**

Run `docker-compose -f docker-compose.dev.yml run --rm frontend bun run test src/__tests__/app/hitl-feedback.test.tsx`.

Expected: the last two tabs and export copy are absent.

- [ ] **Step 3: Add the image assets and exact slide data**

Generate four cohesive, text-free images: profile data workspace; portfolio surfaces in use; cinematic football stadium with digital displays; owned server/deployment hardware. Use these exact definitions:

```tsx
const purposeSlides = [
  {
    backgroundImage: '/assets/home/profile-foundation.webp',
    body: 'Bring profile data, projects, writing, and personal context into one structured source of truth.',
    label: '01',
    thumbnail: 'Profile data',
    title: 'Collect the person',
  },
  {
    backgroundImage: '/assets/home/publishing-surfaces.webp',
    body: 'Turn one living profile into a portfolio, focused public routes, visual resume, and every surface that needs the same story.',
    label: '02',
    thumbnail: 'Publishing',
    title: 'Publish every surface',
  },
  {
    backgroundImage: '/assets/home/stadium-experience.webp',
    body: 'Let visitors step into an immersive stadium-world portfolio shaped from the same published profile data.',
    label: '03',
    thumbnail: '3D world',
    title: 'Step into the 3D experience',
  },
  {
    backgroundImage: '/assets/home/self-hosted-export.webp',
    body: 'Soon, a finished portfolio can be exported for deployment to the person’s own server, keeping the final code in their hands.',
    label: '04',
    thumbnail: 'Your server',
    title: 'Own the final result',
  },
] as const;
```

Make `<PurposeSlider slides={purposeSlides} />` the first child of `main`. Place the existing heading, profile section, and route cards after it inside `mx-auto w-full max-w-6xl px-4 pb-10 sm:px-6 lg:px-8`. Preserve profile loading and all existing route URLs.

- [ ] **Step 4: Verify green and commit**

Run `docker-compose -f docker-compose.dev.yml run --rm frontend bun run test src/__tests__/app/hitl-feedback.test.tsx`, then run `git add apps/frontend/public/assets/home apps/frontend/src/app/page.tsx apps/frontend/src/__tests__/app/hitl-feedback.test.tsx` and `git commit -m "feat: add homepage product carousel artwork"`.

Expected: all four tabs and planned-export copy render.

### Task 3: Record The Self-Hosted Export Roadmap Item

**Files:**

- Modify: `documentation/backlog.md`
- Create: one GitHub issue in `hamzahasbi/PersonaOS`, placed in Project 4 `Todo`.

**Interfaces:**

- Consumes: the approved future-export scope.
- Produces: a matching documented backlog item and open tracker item.

- [ ] **Step 1: Verify the record is missing**

Run `rg -n "Self-hosted portfolio code export" documentation/backlog.md`.

Expected: no output.

- [ ] **Step 2: Add the backlog entry**

```md
## Self-hosted portfolio code export

**Status:** not started.

**Motivation:** people should be able to leave PersonaOS with an owned, deployable portfolio artifact rather than only a hosted public URL.

**Scope:** generate a documented final-code export for a selected published profile, including portfolio assets and deployment configuration required to run it on the person's own server. This is distinct from the current database content migration export and excludes backend credentials and Supabase service-role keys.

**Proposed approach:** define the static/runtime boundary, generate a profile-specific artifact from published content, include a minimal deployment guide, and prove the exported artifact runs without the PersonaOS control plane.

**Risks:** secret exclusion, third-party asset licensing, update ownership, and the contract between generated output and future PersonaOS versions need explicit design before implementation.
```

- [ ] **Step 3: Create and verify the Project issue**

Write the same four paragraphs to `/tmp/personaos-self-hosted-export-issue.md`. Run `gh issue create --repo hamzahasbi/PersonaOS --title "Backlog: Export a self-hosted portfolio code artifact" --body-file /tmp/personaos-self-hosted-export-issue.md`. Add the new URL with `gh project item-add 4 --owner hamzahasbi --url <new-issue-url>`, then set the new item status to `Todo` using field `PVTSSF_lAHOAPg5Qc4BHhM1zg4QCho` and option `f75ad846`. Confirm it through `gh project item-list 4 --owner hamzahasbi --limit 100`.

- [ ] **Step 4: Commit the documentation**

Run `git add documentation/backlog.md`, then run `git commit -m "docs: track self-hosted portfolio export"`.

### Task 4: Validate The Responsive Experience

**Files:**

- Modify only if inspection finds a defect in Task 1 or Task 2 files.

**Interfaces:**

- Consumes: the carousel and artwork from Tasks 1-2.
- Produces: desktop and mobile screenshots with readable copy and unobstructed controls.

- [ ] **Step 1: Run static validation**

Run `docker-compose -f docker-compose.dev.yml run --rm frontend bun run lint` and `docker-compose -f docker-compose.dev.yml run --rm frontend bun run test src/__tests__/app/hitl-feedback.test.tsx`.

Expected: both commands pass.

- [ ] **Step 2: Inspect the browser**

Verify `npx` is installed, start the Docker frontend, then use `/Users/hamzahasbi/.codex/skills/playwright/scripts/playwright_cli.sh` to inspect the homepage and write desktop/mobile screenshots to `output/playwright/`. Activate every thumbnail and verify its image, title, and selected state change. Fix only observed overlap, clipping, or contrast issues.

- [ ] **Step 3: Commit an inspection fix only when needed**

Run `git add apps/frontend/src/components/home/purpose-slider.tsx apps/frontend/src/app/page.tsx apps/frontend/src/__tests__/app/hitl-feedback.test.tsx`, then run `git commit -m "fix: refine homepage carousel responsiveness"`. Skip this when inspection needs no source change.
