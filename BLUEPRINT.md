# Apex Visual Fitness — 3D Website Blueprint

Futuristic, high-energy, premium. Matte black `#121212`, electric neon lime
`#CCFF00`, dark charcoal `#1F1F1F`. Cinematic dark environment with cyber-punk
rim lighting on a chrome hero asset.

## Tech stack

| Layer   | Choice                                                       |
| ------- | ------------------------------------------------------------ |
| Build   | Vite 8 (rolldown) + React 19 + TypeScript                    |
| 3D      | three.js via `@react-three/fiber` 9 + `@react-three/drei` 10 |
| Styling | Tailwind CSS 4 (`@tailwindcss/vite` plugin, `@theme` tokens) |
| Fonts   | Bebas Neue (display) + Inter (body) via Google Fonts         |

---

## 1. Layout architecture — how the HTML overlay connects to the canvas

```
┌────────────────────────────── Browser viewport ──────────────────────────────┐
│                                                                              │
│  <Navbar/>  (fixed, z-40) child of #root, OUTSIDE the Canvas                │
│                                                                              │
│  ┌────────────────────────── Canvas (fixed, inset-0, z-0) ────────────────┐ │
│  │  <Canvas dpr={[1,1.5]} frameloop="always">                             │ │
│  │    <Lights/>            rim spot + neon points + procedural env        │ │
│  │                                                                        │ │
│  │    <ScrollControls pages={9} damping={0.3}>                            │ │
│  │      ── STICKY 3D LAYER (no vertical translate) ──                     │ │
│  │      <ScrollWrapper>           useScroll().offset → x: 0→-3.1, rot π→-π│ │
│  │        <ParallaxTilt>          window pointer tilt, no R3F events      │ │
│  │          <GymModel/>           hero chrome dumbbell (spins, floats)    │ │
│  │          <ContactShadows/>     baked grounding shadow, frames=1        │ │
│  │                                                                        │ │
│  │      ── DOM OVERLAY (translates with the ONE scroller) ──              │ │
│  │      <Scroll html>               React portal to the scroller div      │ │
│  │        <Hero/>          id="top"        100vh, right copy + scroll cue │ │
│  │        <Features/>      id="features"   100vh, 3-card grid + <Reveal/>│ │
│  │        <Classes/>       id="classes"    100vh, weekly schedule         │ │
│  │        <Membership/>    id="membership" 100vh, 3-tier pricing          │ │
│  │        <Gallery/>       id="gallery"    100vh, lazy photo grid         │ │
│  │        <Trainers/>      id="trainers"   100vh, coach cards + ratings   │ │
│  │        <Testimonials/>  id="stories"    100vh, reviews + 5-star        │ │
│  │        <JoinCTA/>       id="join"       60vh, free-pass CTA            │ │
│  │        <Footer/>                     newsletter + link columns         │ │
│  │    </ScrollControls>                                                   │ │
│  │    <OrbitControls/>   locked: zoom/pan/rotate disabled                 │ │
│  │    <AdaptiveDpr/>     drop resolution if frames start dropping         │ │
│  │    <Preload all/>                                                      │ │
│  │  </Canvas>                                                             │ │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│  <LoadingScreen/>  (fixed z-50, `useProgress`)                                │
└──────────────────────────────────────────────────────────────────────────────┘
```

**The one rule that makes everything sync:** *one scroll source of truth.*
Drei's `<ScrollControls>` creates a hidden internal scroll container (height =
`pages * 100vh`). The wheel never scrolls `document.body` — it scrolls that
container, and both the 3D (`useScroll().offset`) and the DOM (portal) read it.

- The 3D model is deliberately **outside** `<Scroll>` so it does not translate
  up/down with the page. It stays pinned in the viewport, and `ScrollWrapper`
  slides it **left** (x: `0 → -3.1`) and spins it (`π → -π`) as `offset` rises.
  The drive is scaled (`* 1.5`) so the model **parks** in its hero pose around
  the middle of the page and holds there through the rest of the journey
  (membership → gallery → reviews).
- The DOM sections are portalled **inside** `<Scroll html>`; with 9 content
  "pages" (`ScrollControls pages={9}`) the single scroller drives them all.
  Fixed chrome (Navbar) and the loader stay outside the Canvas — plain
  `position: fixed` overlays.
- Section jumps (nav links, CTAs) use `src/lib/scroll.ts` (`scrollToId`) because
  native `#hash` navigation can't drive the drei scroller.

### Z-index contract
`Canvas (z-0)` → `Navbar (z-40)` → `LoadingScreen (z-50)`.

### Pointer events contract
The DOM overlay wrapper is `pointer-events-none` so the canvas beneath still
receives the mouse (parallax) across the whole hero. `#features` and
`#membership` re-enable `pointer-events-auto` for hovers/clicks.

---

## 2. Art direction

- **Palette (Tailwind `@theme`):** `apex-black #121212`, `apex-charcoal #1F1F1F`,
  `apex-lime #CCFF00`.
- **Lighting:** one `spotLight` rim from top-right in lime, two colored points
  (violet `#7a00ff`, blue `#0066ff`) for cyber accents + a **procedural**
  `<Environment>` of `Lightformer`s — zero network fetch, instant load, and it
  puts neon streaks on the chrome plates' reflections. Grounding shadow is a
  **baked `<ContactShadows>`** pass (frames=1) instead of a per-frame shadow map.
- **Type:** Bebas Neue, uppercase, tight `leading`; outlined stroke treatment
  (`-webkit-text-stroke` for the "YOUR" line); word-by-word entrance rise.
- **Texture:** glassmorphism nav (`backdrop-blur`, small strip only), neon-glow
  CTA (`shadow-[0_0_20px_rgba(204,255,0,0.35)]`), card hover lift, scroll cue.

---

## 3. Core behaviors

| Behavior                    | Where                        | How                                                                          |
| --------------------------- | ---------------------------- | ---------------------------------------------------------------------------- |
| Hero → left park + spin     | `ScrollWrapper.tsx`          | `useScroll().offset`, ref-eased every frame (no React re-render)             |
| Idle auto-rotate + float    | `GymModel.tsx` / `Float`     | spin `-delta*0.25` on inner group, drei Float bobbing                        |
| Mouse parallax              | `ParallaxTilt.tsx`           | `window.pointermove` → damped `rotation.x/y` (works under the overlay)       |
| Locked camera               | `Scene.tsx` (`LockedCam`)    | camera at `[0,0,8]`; zoom/pan/rotate disabled; polar azimuth restricted      |
| Baked shadow                | `Scene.tsx` + drei           | `<ContactShadows frames={1}>` inside the parallax group (follows the model)  |
| Scroll reveals              | `Reveal.tsx`                 | IntersectionObserver (no RAF), stagger `delay`, fade + lift                  |
| Nav / CTA section jumps     | `Navbar.tsx`, `Membership.tsx` | JS scroll of the drei scroller to exact section offsets                     |
| Preload / loading           | `LoadingScreen.tsx` + `Preload all` + `React.lazy(Scene)` | progress via `useProgress`; heavy chunk code-split out of first paint |

---

## 4. Performance — the "lag" fixes (applied)

Real per-frame costs that were removed in the smoothness pass:

| Before                                    | After                                  | Effect                                  |
| ----------------------------------------- | -------------------------------------- | --------------------------------------- |
| `<Canvas shadows>` + `castShadow` spot    | direct lights + baked `ContactShadows` | no shadow-map pass per frame            |
| `dpr={[1,1.75]}`                          | `dpr={[1,1.5]}` + `<AdaptiveDpr/>`     | ~25% fewer pixels; auto-degrades        |
| `Environment resolution={256}`            | `resolution={128}`                     | smaller bake, fewer shader samples      |
| `backdrop-blur` on whole sections         | removed (translucent bg only)          | no full-viewport compositing each frame |
| 64-seg cylinders / 48-seg tori            | 48 / 32                                | ~30% fewer triangles                     |
| 3 pages eased vertically                  | park model at ~65% scroll              | less perceived motion cost on scroll    |

- **<5MB rule:** this build ships **zero external 3D assets** (the hero "model"
  is ~20 primitives, ~0KB). Drop your Sketchfab/TurboSquid GLB into
  `/public/models`, `useGLTF.preload` it at the top of `Scene.tsx`, and swap
  `GymModel`'s mesh group for a `<primitive>`. Re-compress with `gltf-transform`
  and target ≤ 5MB.
- three.js + drei split into a lazy ~600KB (~170KB gzip) chunk behind
  `React.lazy` + `Suspense`, so the navbar paints instantly.
- The scene is ~20 low-poly meshes with 5 lights: cheap for any real GPU.
  (Headless/SwiftShader numbers are CPU software rendering and not
  representative of hardware FPS.)

---

## 5. Roadmap (next)

1. Real enrollment form in the `#join` CTA band + a Member login modal.
2. Swap the primitive dumbbell for a licensed chrome kettlebell GLB
   (`useGLTF.preload` + `<Suspense>`), then bake reflections offline.
3. Per-section 3D choreography with `scroll.range()` / `scroll.curve()`
   (e.g., camera push-in at the Gallery, reveal second pose at Trainers).
4. CMS-drive `Classes` schedule + live class booking.

## Commands

```bash
npm install      # install
npm run dev      # http://localhost:5173 (or -- --port 5199)
npm run build    # type-check + production bundle
npm run lint     # oxlint
npm run preview  # serve the built bundle
```