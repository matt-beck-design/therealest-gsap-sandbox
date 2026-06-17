# The Realest — GSAP Sandbox

**Live:** https://matt-beck-design.github.io/therealest-gsap-sandbox/
**Source:** `sandbox-ideas/orbital-scroll-gallery.html` — single self-contained file (inline CSS + JS).

---

## Architecture

All six effects share a single config-switching engine:

```
CONFIGS[]          Array of config objects, each with init(medias) and destroy()
loadConfig(index)  Calls destroy() on the active config, clears all GSAP props on
                   medias, then calls init() on the new config
medias             Array of 15 <img class="media"> elements in #stage, passed into
                   every config's init()
active             Index of the currently running config (-1 on first load)
```

**Config contract:**
```js
{
  name: 'String',       // Label rendered in the nav pill
  init(medias) { ... }, // Receive shared medias[], set up tweens/listeners
  destroy() { ... }     // Kill all tweens, remove listeners, clean up DOM
}
```

**Key rule:** configs that need body scroll (Dual Carousel, Scroll Flow) must set `document.body.style.overflow = 'auto'` on init and restore it to `''` on destroy. Configs that add a full-screen section hide `#stage` via `display: none` and restore it on destroy.

**Adding a new config:** append an object to `CONFIGS[]` before the closing `]`. The nav pill generates buttons automatically from `cfg.name`.

---

## Shared DOM

```
#stage                       Fixed, inset:0. Home for .media images + .center-logo SVG.
  img.media × 15             Base size: 8vw × 8vw, centered at 50%/50% via offset.
  svg.center-logo            The Realest wordmark, z-index:10, pointer-events:none.
#nav                         Fixed pill at bottom-center. Buttons injected by engine.
```

---

## Config 01 — Orbital

**Controls:** mouse wheel, touch scroll
**GSAP:** `gsap.ticker`, `gsap.quickTo`, `Observer`

### Key variables

| Variable | Default | Effect |
|---|---|---|
| `baseAngles` | evenly spaced `0 → 2π` | starting orbit positions per image |
| `quotients[]` | `(Math.random()-0.5)/2` per image | per-image depth offset; randomized once on init |
| `state.acceleration` | 0 | drives `incr` accumulation per tick |
| `state.rayon` | 0 | controls orbit radius spread + image rotation |
| `setAcceleration` duration | `0.3s` | how quickly acceleration responds to input |
| `setRayon` duration | `0.6s` | how quickly radius spread responds |
| wheel sensitivity | `delta / 800` (accel), `delta / 40` (rayon) | lower divisor = more reactive |
| touch sensitivity | `delta / 200` (accel), `delta / 10` (rayon) | |
| stop timeout | `120ms` | delay before returning to idle after wheel stops |

### Tick formula (runs every frame via `gsap.ticker`)

```js
incr += state.acceleration
angle = (time + incr) / 3 + baseAngles[i]
x = Math.sin(angle) * w / (2.4 + quotients[i] * state.rayon)
y = Math.cos(angle) * h / (2.7 + quotients[i] * state.rayon)
rotate = quotients[i] * state.rayon * 20
```

- `/ 3` on the angle slows the base orbit speed — increase to speed up
- `2.4` / `2.7` are the x/y radius divisors at rest — decrease to widen orbit
- `* 20` on rotate controls max tilt when scrolling

### DOM changes
None. All transforms applied via `gsap.set` directly on the shared `.media` elements.

---

## Config 02 — 3D Sphere

**Controls:** drag (mouse + touch), mouse wheel, trackpad
**GSAP:** `gsap.quickTo`, `Observer`

### Key variables

| Variable | Default | Effect |
|---|---|---|
| `goldenAngle` | `π(3−√5)` | mathematically optimal surface spacing |
| `radius` | `innerWidth * 0.2` (desktop) / `0.3` (mobile) | sphere size in px |
| `perspective` | `innerWidth * 0.5` (desktop) / `0.7` (mobile) | CSS perspective depth |
| `quickY/X` duration | `1s, power2` | rotation smoothing |
| wheel divisor | `/ 10` | lower = faster spin on scroll |
| drag divisor | `/ 4` (mouse) / `1` (touch) | touch is 4× more sensitive than mouse drag |

### Rotation system

Uses a cumulative 3×3 rotation matrix `m[9]`. Each input event builds a new rotation matrix `R` (combined Euler Y + X rotation) and left-multiplies it into `m` via `premultiply3x3`. The positions array holds normalized unit-sphere coordinates per image; final world position is `m × position × radius`.

```js
// Smooth accumulators — these are what quickTo targets
smooth.x  // accumulated X rotation
smooth.y  // accumulated Y rotation

// Inputs feed these accumulators
incrY -= e.deltaY / 10   // wheel vertical → Y spin
incrX -= e.deltaX / 10   // wheel horizontal → X tilt
```

### DOM changes on init
```
#stage
  div.sphere-container       (flex centered, overflow:hidden)
    div.sphere               (transformStyle: preserve-3d)
      img.media × 15         (reparented from #stage; restored on destroy)
  svg.center-logo            (stays in #stage above the container)
```
`is-sphere` class added to `#stage` to activate media size overrides (10vw, object-fit:cover).

---

## Config 03 — Letter Swap

**Controls:** mouseenter on individual letters
**GSAP:** `gsap.to`, `gsap.from`, `gsap.delayedCall`

### Key variables

| Variable | Default | Effect |
|---|---|---|
| word | `"Unrivaled"` | the display word; change in `section.innerHTML` |
| image sources | 8 files from `../img-jerseys/` | cycles via `mediaIndex % length` |
| `.created-media` width | `9vw` | size of the pop-up image |
| pop-up duration | `0.3s, back.out(2)` | entrance scale animation |
| dismissal delay | `1.2s` | `gsap.delayedCall` before image is removed |
| letter offset ease | `back.out(3), 0.3s` | how letters shuffle to avoid overflow |
| `mediaWidth` | `0.095 * innerWidth` | used to calculate overflow; should match `.created-media` CSS width |

### Overflow logic

When a pop-up image is wider than its parent letter, `overflows[i]` stores the excess half-width. `applyLetterOffsets()` calculates a balanced x-shift for every letter so the word stays centered and images don't clip the viewport edge. On image removal, `overflows[i]` resets to 0 and letters animate back.

### DOM changes on init
```
#stage
  section.mwg_effect093      (absolute, grid center)
    p.word                   ("Unrivaled" — each char wrapped in span.letter by JS)
    div.medias
      img.media × 8          (hidden source pool from ../img-jerseys/)
  svg.center-logo            (hidden via is-letter-swap class)
```
Shared stage `.media` elements are hidden (`visibility: hidden`) and restored on destroy.

---

## Config 04 — Dual Carousel

**Controls:** page scroll
**GSAP:** `gsap.to` with `scrollTrigger`, `ScrollTrigger.create`
**Requires:** `document.body.style.overflow = 'auto'`, `theme-dark` class

### Key variables

| Variable | Default | Effect |
|---|---|---|
| `angle` | `14` (degrees) | spacing between items on each circle |
| `totalRot` | `180 + angle * count` | total rotation travel over the scroll distance |
| `pin-height` | `500vh` (CSS) | scroll distance; increase for slower reveal |
| circle size | `80vw` (CSS) | diameter of each ring |
| circle overlap | left: `right: 52%` / right: `left: 52%` | how much circles overlap at center |
| label font | `4vw Inter 500` | scale with circle size |
| image size | `12vw × 12vw` (CSS) | size of each carousel image |
| scrub | `true` | tied 1:1 to scroll position |

### Four tweens

| Tween | Target | What it does |
|---|---|---|
| `t1` | `.parent-circle-left` | rotates the left ring; also holds the pin |
| `t2` | `left .circle p` | counter-rotates labels to stay upright |
| `t3` | `.parent-circle-right` | rotates the right ring |
| `t4` | `right .circle .media` | counter-rotates images to stay upright |

All four share the same `trigger / start / end` so they stay in sync. Only `t1` carries the `pin: container` option.

### Labels (left ring)
`Authentic, Certified, Original, Signed, Game Used, Limited, Graded, Premium, Rare, Iconic` — edit the `.label` paragraphs in `section.innerHTML`.

### DOM changes on init
```
body
  section.mwg_effect040      (inserted before #nav)
    div.pin-height            (500vh scroll track)
      div.container           (100vh, pinned)
        div.parent-circle-left
          div.circle × 10     (each rotated i * 14deg)
            p.label
        div.parent-circle-right
          div.circle × 10
            img.media
  #nav
```

---

## Config 05 — Circular

**Controls:** mouse wheel
**GSAP:** `gsap.quickTo`
**Requires:** `theme-dark` class; does NOT need body scroll

### Key variables

| Variable | Default | Effect |
|---|---|---|
| image list | 14 from `../img/` | duplicated (×2 = 28) to fill the ring |
| container size | `300vw × 300vw` | large enough that the ring extends off-screen |
| container left | `-100vw` | centers the rotation origin on the viewport |
| image size | `20vw × 26vw` (CSS) | tall portrait ratio |
| `yPercent` base | `-50` | images hang from the outer circumference |
| `rotTo` duration | `0.8s, power4` | main rotation smoothing |
| yPercent durations | `1s / 2s / 3s` | three tiers (media-1/2/3) bounce at different speeds |
| wheel divisor (rotation) | `/ 40` | lower = faster spin |
| yPercent bounce value | `-abs(delta/4) - 50` | how far images dip on scroll input |

### Media tiers
Each `.inner-media` receives a random class `media-1`, `media-2`, or `media-3`. Each tier has its own `quickTo` targeting `yPercent`, with different durations (1s / 2s / 3s), so the bounce settles at different rates — creating a ripple across the ring.

### DOM changes on init
```
body
  section.mwg_effect023
    div.header               (3-column grid: "From the Court / The Realest / To the Collection")
    div.container            (300vw × 300vw rotating element)
      div.inner-media × 28   (each rotated to (360/28)*i deg, class media-1/2/3)
        img.media
  #nav
```

---

## Config 06 — Scroll Flow

**Controls:** page scroll
**GSAP:** `gsap.timeline` with `scrollTrigger`, `ScrollTrigger.create`
**Requires:** `document.body.style.overflow = 'auto'`

### Key variables

| Variable | Default | Effect |
|---|---|---|
| images | 18 from `../img/` | all images pass through on one scroll |
| `pin-height` | `500vh` (CSS) | scroll distance; controls how spread-out the stagger feels |
| image size | `20%` width, `aspect-ratio: 1` (CSS) | square |
| `margin-top` | `-10vw` (CSS) | vertically centers images (half of 20vw) |
| `stagger` | `0.1` | delay between each image entering |
| x travel (desktop) | `-1 × clientWidth` | full viewport sweep left |
| x travel (mobile) | `-1.5 × clientWidth` | wider sweep for smaller screens |
| yPercent spread | `(Math.random()-0.5) * 160` | randomized vertical spread on entry |
| scale entrance | `0 → 1` over `0.5` of travel | pop in |
| scale exit | `1 → 0` over `0.5` of travel | pop out |

### Timeline structure

Three `tl.to()` calls all start at the same scroll position (`<` / `<+=0.5` offsets):

```
t=0      xPercent:100 + x: -clientWidth  (sweep left across screen, ease:none, duration:1)
t=0      yPercent: random spread, scale:1  (enter, ease:power1.inOut, duration:0.5)
t=0.5    yPercent:0, scale:0              (exit, ease:power1.inOut, duration:0.5)
```

Two separate ScrollTriggers: `st1` pins the container; `tl.scrollTrigger` scrubs the timeline. Both must be killed independently on destroy.

---

## Images

| Folder | Used by |
|---|---|
| `../img/` | Orbital, 3D Sphere, Dual Carousel, Circular, Scroll Flow (stage medias + inline) |
| `../img-jerseys/` | Letter Swap only |

Paths are relative to `sandbox-ideas/orbital-scroll-gallery.html`, so they resolve to `img/` and `img-jerseys/` at the repo root.

---

## Dependencies

```html
gsap@3.15          core
gsap/Observer      touch + wheel abstraction (Orbital, 3D Sphere)
gsap/ScrollTrigger pin + scrub (Dual Carousel, Scroll Flow)
Inter (Google Fonts) Dual Carousel labels + Circular header
```
