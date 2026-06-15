# The Realest — GSAP Sandbox

Interactive explorations of GSAP animation effects, adapted with The Realest brand imagery. Switch between configs using the floating pill nav.

**Live:** https://matt-beck-design.github.io/therealest-gsap-sandbox/

---

## Explorations

### 1. Orbital
Images orbit the center of the screen in an elliptical path, continuously animating via the GSAP ticker. Scrolling or swiping injects acceleration — the orbit speeds up, images fan out radially, and slowly settles back to its resting rhythm once input stops. Each image has a slight randomized depth offset so the group feels three-dimensional without true 3D transforms.

### 2. 3D Sphere
All images are distributed across the surface of a virtual sphere using the golden angle, a mathematically optimal spacing algorithm that avoids clustering. The sphere is rotated by a 3×3 rotation matrix that responds to drag and scroll input, giving it physical weight — fast drags spin it, and it coasts to a stop. The result reads as a true rotating globe of product imagery.

### 3. Letter Swap
The word **"Unrivaled"** sits large on screen. Hovering each letter triggers a product image that bursts out of the letter — scaling and fading in, then disappearing after a short delay. Letters track how far their pop-up images overflow the viewport edge and shift horizontally to compensate, keeping every reveal fully visible regardless of which letter is hovered.

### 4. Dual Carousel
A full-screen scroll-driven experience with two rotating rings. The left ring cycles through text labels (Authentic, Certified, Signed, Game Used, etc.); the right ring cycles through product images. Both rings rotate at different speeds as you scroll, with each item counter-rotating to stay upright. The entire section is pinned for the duration of the scroll travel so the animation plays out before the page continues.

### 5. Circular
A single massive container (300vw × 300vw) centered in the viewport rotates continuously via wheel input. Inside, images are distributed evenly around the circumference at varying depths (three size tiers). Scrolling spins the whole assembly; images bounce vertically as the wheel fires, creating a subtle ripple through the group. The effect reads as an infinite rotating carousel in every direction at once.

### 6. Scroll Flow
A scroll-triggered sequence where product images sweep across the viewport in a staggered wave. Each image starts off-screen to the right, crosses through center, and fades out left — their vertical positions are randomized so they layer and overlap naturally. Images scale up from nothing at entry and back down at exit. The whole sequence is pinned to the viewport and scrubs precisely with scroll position.

---

*Built with [GSAP 3.15](https://gsap.com) — ScrollTrigger, Observer, ticker.*
