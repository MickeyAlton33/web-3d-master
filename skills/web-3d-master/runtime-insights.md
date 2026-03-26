# Runtime Insights

Live browser examination findings -- what's actually running in the DOM, not just what the source code suggests.

## Live 3D Census (from browser examination)

### Sites with Active WebGL Canvases
| Site | Canvases | Videos | Running Anims | Key Runtime Feature |
|---|---|---|---|---|
| **ChainGPT Labs** | 1 (Unicorn Studio) | 3 (WebM alpha) | 52 | 51 CSS particles + Unicorn Studio WebGL footer |
| **Cosmos Network** | 2 | 0 | 2 | Dual WebGL contexts for hero + background |
| **Spline Design** | 3 | 12 | 2 | Triple canvas for their own product demos |
| **Madbox** | 2 | 2 | 9 | Dual WebGL for gaming showcase |
| **Third Dimension Studio** | 1 | 0 | 0 | Single fullscreen WebGL takeover |
| **Filippo Ruffini** | 1 | 0 | 0 | Single OGL canvas background |
| **World (Worldcoin)** | 1 | 1 | 0 | Single canvas for the globe/orb |
| **Reform Collective** | 1 | 8 | 20 | WebGL background + 8 video showcases |

### Sites Using Video Instead of Runtime 3D
| Site | Videos | Approach |
|---|---|---|
| **Reform Collective** | 8 | Pre-rendered showreel clips as project previews |
| **Spline Design** | 12 | Product demo videos alongside live canvases |
| **ChainGPT Labs** | 3 | Alpha-channel WebM overlays (the 3D characters) |
| **Dr Pepper** | video hero | Full-screen brand video |
| **Brew District 24** | video | Product showcase |

### Key Runtime Findings

**1. The 52-Animation Site (ChainGPT Labs)**
At any given moment, ChainGPT Labs has 52 CSS animations running simultaneously:
- 51 are CSS particle bubbles (`.bubble` elements with per-element custom properties for randomized motion)
- 1 is the Unicorn Studio WebGL canvas
- 3 WebM alpha-channel videos play on top of the 2D layout

This means the 3D illusion comes from **layered 2D effects** (CSS particles + transparent video) with only ONE actual WebGL element. The lesson: you don't need a full Three.js scene when layered CSS + video can achieve 80% of the effect.

**2. OGL Dominance for Portfolios**
Multiple portfolio sites (Filippo Ruffini, Rhumb Studio, Third Dimension) use a single OGL canvas as a fullscreen background. The canvas renders a shader-driven effect (often a noise/distortion pattern or image transition effect), and HTML content sits on top with `pointer-events` managed carefully.

Pattern:
```
Layer 1: OGL canvas (fullscreen, z-index: 0)
Layer 2: HTML content (z-index: 1, pointer-events: auto)
Layer 3: Custom cursor div (z-index: 999, pointer-events: none)
```

**3. Spline's Triple Canvas**
Spline Design's own homepage uses 3 separate canvases -- not one giant scene. Each canvas is scoped to a specific section of the page, allowing the browser to GPU-cull off-screen canvases and preventing one heavy scene from blocking the entire page.

**4. Cosmos Network's Dual WebGL**
Two separate WebGL2 contexts: one for the animated hero (likely a particle/mesh network visualization) and one for a background effect. This confirms J-Vers' multi-canvas pattern is a widespread production technique.

**5. Reform Collective: 20 Animations + Lenis**
Reform runs 20 simultaneous CSS animations with Lenis smooth scrolling and an active WebGL canvas. The 8 video elements are lazy-loaded project showcases -- they only play when scrolled into view, preserving the animation budget for the WebGL background.

## Runtime Performance Patterns

### CSS Particle Systems Are Free
ChainGPT's 51 CSS particles run at 60fps because CSS animations are GPU-composited. The pattern:

```css
.bubble {
  position: absolute;
  border-radius: 50%;
  background: radial-gradient(circle, rgba(255,255,255,0.1), transparent);
  animation: rise var(--duration) var(--delay) infinite ease-out;
  /* Each bubble gets unique values via inline style custom properties */
}
@keyframes rise {
  0% { transform: translateY(100vh) scale(0); opacity: 0; }
  50% { opacity: 0.5; }
  100% { transform: translateY(-10vh) scale(1); opacity: 0; }
}
```

50+ elements with unique `--duration` (5-20s) and `--delay` (0-10s) create organic, non-repeating motion without any JavaScript.

### Video Is Cheaper Than WebGL
Multiple sites that appear to be "3D" are actually pre-rendered video:
- **ChainGPT**: 3D robot characters are transparent WebM video, not runtime models
- **Reform Collective**: Project showcases are video, not live renders
- **Dr Pepper**: Full-screen product animation is video

The performance hierarchy in practice:
1. CSS animations (~0% CPU, GPU-composited)
2. Video playback (~2% CPU, hardware decoded)
3. Single OGL/WebGL canvas (~5-15% CPU)
4. Three.js with models + lighting (~15-40% CPU)
5. Multiple WebGL contexts (~20-50% CPU)

### Lazy WebGL Initialization
Sites like Clay Boan (OGL in Nuxt) include the WebGL library but don't initialize a canvas until needed. This means:
- Initial page load has zero WebGL overhead
- Canvas is created only when the user scrolls to a 3D section
- If the user never reaches that section, zero GPU resources are consumed

```js
// Lazy WebGL pattern
const observer = new IntersectionObserver((entries) => {
  if (entries[0].isIntersecting) {
    initWebGLScene(); // only now create the canvas + renderer
    observer.disconnect();
  }
}, { rootMargin: '200px' });
observer.observe(document.getElementById('3d-section'));
```

### Hidden Scrollbar Is Universal
Every 3D site examined hides the native scrollbar:
```css
body { scrollbar-width: none; }
body::-webkit-scrollbar { display: none; }
```
Combined with Lenis smooth scrolling, this creates a seamless 3D-first experience where the browser chrome doesn't break the immersion.

## Integration Patterns Observed in Production

### Pattern A: Full-Page WebGL Background (Most Common)
```
┌──────────────────────────────┐
│  Canvas (100vw × 100vh)      │ ← WebGL shader/particles
│  z-index: 0                  │
│  ┌──────────────────────┐    │
│  │  HTML Content         │   │ ← position: relative; z-index: 1
│  │  (text, images, nav)  │   │
│  └──────────────────────┘    │
│  ┌──────────────────────┐    │
│  │  Custom Cursor        │   │ ← position: fixed; z-index: 999
│  │  (pointer-events: none│   │
│  └──────────────────────┘    │
└──────────────────────────────┘
```
Used by: Rhumb, Filippo Ruffini, Third Dimension, Hardik Bhansali

### Pattern B: Sectional Canvases (Growing Trend)
```
┌──────────────────────────────┐
│  Hero Section                │
│  ┌─────────────────────┐     │
│  │  Canvas A (hero 3D)  │    │
│  └─────────────────────┘     │
├──────────────────────────────┤
│  Content Section (HTML only) │
├──────────────────────────────┤
│  ┌─────────────────────┐     │
│  │  Canvas B (sticky 3D)│    │ ← position: sticky; top: 0
│  └─────────────────────┘     │
├──────────────────────────────┤
│  More Content (HTML)         │
├──────────────────────────────┤
│  ┌─────────────────────┐     │
│  │  Canvas C (footer 3D)│    │
│  └─────────────────────┘     │
└──────────────────────────────┘
```
Used by: J-Vers (5 canvases), Spline Design (3 canvases), Cosmos (2 canvases)

### Pattern C: Video + CSS Particle Overlay (Cheapest "3D")
```
┌──────────────────────────────┐
│  Video (autoplay, muted)     │ ← pre-rendered 3D, hardware decoded
│  ┌──────────────────────┐    │
│  │  50+ CSS Particles    │   │ ← GPU-composited animations
│  │  (position: absolute) │   │
│  └──────────────────────┘    │
│  ┌──────────────────────┐    │
│  │  HTML Content         │   │
│  └──────────────────────┘    │
│  ┌──────────────────────┐    │
│  │  Alpha Video Overlay  │   │ ← WebM with transparency
│  └──────────────────────┘    │
└──────────────────────────────┘
```
Used by: ChainGPT Labs (the most impressive "fake" 3D site)

### Pattern D: Unicorn Studio / No-Code WebGL
A growing pattern where the WebGL layer is created via a visual editor (Unicorn Studio, Spline), not hand-coded. The editor outputs optimized WebGL that's embedded via a `<script>` tag or iframe.

```js
// Unicorn Studio initialization
UnicornStudio.init().then(scene => {
  // Scene responds to mouse position automatically
  // Scroll binding is configured in the editor
});
```

Used by: ChainGPT Labs (footer), multiple SaaS sites

## Key Takeaway

**The most impressive "3D" websites often use the LEAST runtime 3D.** ChainGPT Labs looks incredibly 3D-rich but runs only 1 WebGL canvas -- everything else is CSS particles and transparent video. The Zoox product showcase is entirely pre-rendered images. Even Superlist uses matcaps instead of real lighting.

The production-proven approach: **pre-render where possible, use CSS for atmosphere, reserve WebGL for the ONE interactive element that needs it.**
