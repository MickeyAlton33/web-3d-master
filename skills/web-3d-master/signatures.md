# 3D Signature Techniques

The ONE standout technique from each analyzed site -- the thing that makes it memorable. These are the tricks worth stealing.

## Frontier Techniques

### Gaussian Splatting on the Web (World Labs)
The cutting edge of web 3D. Renders photorealistic AI-generated 3D scenes without traditional mesh/texture pipelines. Uses `.spz` compressed splat format with a 100K splat budget per scene and WebP preview thumbnails for progressive loading.

```js
// Conceptual pattern -- Gaussian splatting viewer
// Uses custom WebGL1 renderer (no Three.js needed)
// .spz files are ~1-2MB per scene vs ~50MB for equivalent mesh+textures

// Progressive loading pattern:
// 1. Show pre-rendered WebP 360° thumbnail immediately
// 2. Initialize WebGL context
// 3. Stream .spz splat data
// 4. Crossfade from thumbnail to live render
```

**When to use:** Photorealistic scene viewing, AI-generated 3D content, virtual tours.

### Full PBR-IBL Pipeline with Draco + Basis (Dorian Lods)
The complete production-grade WebGL material pipeline using a custom "glxp" engine built on OGL:

```
Pipeline:
1. Draco WASM → compressed mesh decompression (Web Workers)
2. Basis Universal → GPU-compressed textures (Web Workers)
3. BRDF LUT → pre-computed lighting lookup table
4. Split-sum environment maps → diffuse + specular at multiple MIP levels
5. MSDF text → resolution-independent text in WebGL
```

```js
// BRDF LUT + Environment Map PBR setup
const brdfLUT = loadTexture('/brdfLUT.png');
const envDiffuse = loadTexture('/environment/diffuse.png');
const envSpecular = [
  loadTexture('/environment/specular-0.png'),  // roughest
  loadTexture('/environment/specular-1.png'),
  loadTexture('/environment/specular-2.png'),
  loadTexture('/environment/specular-3.png'),
  loadTexture('/environment/specular-4.png'),  // smoothest
];

// Fragment shader uses split-sum approximation:
// color = diffuseIBL * (F0 * brdf.x + brdf.y) + specularIBL * F0
```

**When to use:** High-fidelity product visualization, architectural rendering, luxury brand 3D.

### Alpha-Channel Video Pipeline (ChainGPT Labs)
Pre-render 3D in Cinema 4D/Octane, export as transparent video, overlay on 2D page. Looks like real-time 3D but costs zero GPU at runtime.

```bash
# Dual-codec pipeline for cross-browser transparency:

# Chrome/Firefox: WebM VP9 with alpha
ffmpeg -i render_with_alpha.mov \
  -c:v libvpx-vp9 -pix_fmt yuva420p \
  -deadline best -crf 32 -b:v 1700k -an \
  output.webm

# Safari: HEVC with alpha (via Apple Compressor)
# Export .mov with ProRes 4444 → Compressor → HEVC with alpha
```

```html
<video autoplay muted loop playsinline style="mix-blend-mode: normal;">
  <source src="/character.webm" type="video/webm; codecs=vp9">
  <source src="/character.mp4" type="video/mp4; codecs=hvc1">
</video>
```

**When to use:** Character animations, product reveals with complex motion, cinematic overlays.

## Library-Specific Signatures

### OGL as Three.js Replacement (Rhumb, Clay Boan, J-Vers, Dorian Lods)
OGL has emerged as the preferred WebGL library for creative portfolios -- 4 out of 7 analyzed portfolio sites use it over Three.js. It's ~5KB vs Three.js's ~600KB.

```js
import { Renderer, Camera, Program, Mesh, Plane } from 'ogl';

const renderer = new Renderer({ alpha: true, antialias: true });
const gl = renderer.gl;
document.body.appendChild(gl.canvas);

const camera = new Camera(gl, { fov: 45 });
camera.position.z = 5;

const geometry = new Plane(gl, { width: 2, height: 2 });
const program = new Program(gl, {
  vertex: `
    attribute vec3 position;
    attribute vec2 uv;
    uniform mat4 modelViewMatrix;
    uniform mat4 projectionMatrix;
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragment: `
    precision highp float;
    uniform float uTime;
    varying vec2 vUv;
    void main() {
      vec3 color = vec3(vUv, sin(uTime) * 0.5 + 0.5);
      gl_FragColor = vec4(color, 1.0);
    }
  `,
  uniforms: { uTime: { value: 0 } }
});

const mesh = new Mesh(gl, { geometry, program });

requestAnimationFrame(function update(t) {
  requestAnimationFrame(update);
  program.uniforms.uTime.value = t * 0.001;
  renderer.render({ scene: mesh, camera });
});
```

**When to use:** Creative portfolios, shader-driven backgrounds, when bundle size matters.

### Multiple Independent WebGL Canvases (J-Vers)
Instead of one monolithic full-page canvas, use 5+ separate WebGL2 contexts -- each dedicated to a specific effect:

```
Page structure:
├── canvasWrapper (hero background shader)
├── stickyAnimCanWrapper (sticky scroll-driven 3D animation)
├── canvasSequenceWrapper (frame-by-frame 3D playback)
├── canvasWrapper (mid-page parallax effect)
└── canvasWrapper (footer ambient effect)
```

**When to use:** Long-scroll pages with multiple distinct 3D sections. Better performance than one huge scene because the browser can cull off-screen canvases.

### Scroll-Driven Skeletal GLTF Animation (Superlist)
The gold standard for scroll-bound 3D. Key insight: use skeletal animation instead of baked vertex animation to reduce file size by 10x.

```js
// The Superlist pipeline:
// 1. Model in Blender with armature
// 2. Houdini: delete faces not visible from camera angle
// 3. Export as GLB with Draco compression
// 4. Apply matcap material (no scene lights needed)
// 5. GSAP ScrollTrigger scrubs animation.time

const action = mixer.clipAction(gltf.animations[0]);
action.play();
action.paused = true;

gsap.to(action, {
  time: action.getClip().duration,
  scrollTrigger: { trigger: el, scrub: 2, start: 'top top', end: 'bottom bottom' }
});
```

**When to use:** Product showcases, landing page storytelling, any scroll-driven 3D narrative.

### Sprite-Sheet 3D (Hanai World)
Pre-render 3D frames in Cinema 4D, assemble into a sprite sheet, play back on canvas. Zero WebGL overhead. Apple uses this same technique for AirPods Pro.

```js
// 120 frames of a 3D globe rotation, packed in a 10x12 grid
const cols = 10, rows = 12, totalFrames = 120;
const frameW = 512, frameH = 512;

function drawFrame(ctx, spriteSheet, scrollProgress) {
  const frame = Math.floor(scrollProgress * (totalFrames - 1));
  const col = frame % cols;
  const row = Math.floor(frame / cols);
  ctx.drawImage(spriteSheet, col * frameW, row * frameH, frameW, frameH, 0, 0, canvas.width, canvas.height);
}
```

**When to use:** When you need photorealistic 3D but can't afford WebGL (mobile, low-end devices, broad compatibility).

### Canvas 2D Film Grain Overlay (Chipsa)
Not WebGL -- just a fullscreen `<canvas>` with 2D context rendering noise at 60fps. Adds tremendous cinematic depth for zero WebGL cost.

```js
const canvas = document.createElement('canvas');
canvas.style.cssText = 'position:fixed;inset:0;pointer-events:none;z-index:9999;opacity:0.05;';
document.body.appendChild(canvas);
const ctx = canvas.getContext('2d');

function renderGrain() {
  canvas.width = window.innerWidth;
  canvas.height = window.innerHeight;
  const imageData = ctx.createImageData(canvas.width, canvas.height);
  const data = imageData.data;
  for (let i = 0; i < data.length; i += 4) {
    const v = Math.random() * 255;
    data[i] = data[i+1] = data[i+2] = v;
    data[i+3] = 255;
  }
  ctx.putImageData(imageData, 0, 0);
  requestAnimationFrame(renderGrain);
}
renderGrain();
```

**When to use:** Any dark-themed site. Adds analog texture with almost no performance cost.

### Draco WASM from Google CDN (Hardik Bhansali)
Load Draco WASM decoder from Google's `gstatic.com` -- it's likely already cached in the user's browser from Google Maps or other 3D sites.

```js
const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath('https://www.gstatic.com/draco/versioned/decoders/1.5.6/');
// This URL is the same one Google Maps uses -- high chance of cache hit

const gltfLoader = new GLTFLoader();
gltfLoader.setDRACOLoader(dracoLoader);
```

**When to use:** Always, when loading Draco-compressed models. The Google CDN path gives you free caching.

### Cursor-Proximity Font Weight (Jack Redley)
Calculate distance from cursor to each character, map to CSS font-weight. Makes flat text feel three-dimensional.

```js
document.querySelectorAll('.magnetic-text span').forEach(char => {
  document.addEventListener('mousemove', (e) => {
    const rect = char.getBoundingClientRect();
    const cx = rect.left + rect.width / 2;
    const cy = rect.top + rect.height / 2;
    const dist = Math.sqrt((e.clientX - cx) ** 2 + (e.clientY - cy) ** 2);
    const weight = Math.max(100, Math.min(800, 800 - dist * 2));
    char.style.fontWeight = weight;
  });
});
```

**When to use:** Typography-focused portfolios, creative agency sites, anywhere text is the hero.

### Spline + Webflow Scroll Binding (STR8FIRE)
Embed Spline 3D scenes in Webflow and bind scroll/hover to object transforms via Webflow's interaction timeline.

```html
<!-- Embed Spline in Webflow -->
<spline-viewer url="https://prod.spline.design/SCENE_ID/scene.splinecode"></spline-viewer>
```

```js
// Webflow Interactions bind scroll position to Spline object properties:
// positionX, positionY, positionZ
// rotationX, rotationY, rotationZ
// scale (uniform or per-axis)
// Material properties (color, opacity, metalness)
```

**When to use:** Quick prototyping, designer-led 3D, when custom WebGL code isn't feasible.

### Image Sequence + ffmpeg Pipeline (Zoox)
Pre-render 180 frames of a product orbit, compress with ffmpeg, play back on canvas tied to scroll. Photorealistic quality impossible to achieve in real-time WebGL.

```bash
# Server-side frame pipeline
ffmpeg -i product_orbit.mp4 -vf "scale=1920:-1" -q:v 2 frames/product-%04d.webp

# Or from Blender renders:
ffmpeg -i render_%04d.png -c:v libwebp -quality 85 frames/product-%04d.webp
```

```js
// Client: draw frame based on scroll position
const progress = scrollY / (document.body.scrollHeight - window.innerHeight);
const frame = Math.floor(progress * (frameCount - 1));
ctx.drawImage(images[frame], 0, 0, canvas.width, canvas.height);
```

**When to use:** Product pages, automotive, hardware -- anything where photorealism matters more than interactivity.

## New Signatures (from deep-dive round 2)

### GPGPU Refraction Distortion on Hover (Dorian Lods)
Mouse position feeds a GPU simulation that outputs a displacement texture, which a shader uses to refract the underlying image. The effect has inertia and decay -- feels liquid.

```glsl
// Fragment: sample displacement from GPGPU pass, offset UVs
vec4 disp = texture2D(uDisplacement, vUv);
vec2 refractedUv = vUv + disp.rg * uRefractionStrength;
float r = texture2D(uTexture, refractedUv + disp.rg * 0.01).r;
float g = texture2D(uTexture, refractedUv).g;
float b = texture2D(uTexture, refractedUv - disp.rg * 0.01).b;
gl_FragColor = vec4(r, g, b, 1.0);
```

**When to use:** Portfolio image hover effects, interactive backgrounds, anywhere you want liquid-glass feel.

### Persistent 3D Scene Across Page Routes (Chipsa Design)
The Three.js canvas never reloads between pages. Barba.js intercepts navigation, GSAP animates transition, Three.js scene morphs. Feels like navigating through one continuous 3D space.

**When to use:** Multi-page portfolios, agency sites, anywhere route changes should feel spatial.

### 81-Shader Custom Pipeline (David Heckhoff)
The most shader-dense portfolio found: 81 GLSL programs, 129 render targets, displacement + fresnel + matcap + refraction + transmission. Vue 3 + Three.js + GSAP. Awwwards Honorable Mention.

**When to use:** When the portfolio IS the demo reel -- every visual element custom-shaded.

### Persistent R3F Scene with Route-Driven Camera (Rhumb Studio)
R3F Canvas mounted at layout level, never unmounts. Route changes trigger camera presets. GPU particles + water simulation + PBR + post-processing remain alive across all pages. KTX2 textures + baked lighting.

**When to use:** Studio/agency sites where the 3D environment IS the brand.

### Interactive Three.js Globe with OrbitControls (Cosmos Network)
Dual procedural Three.js globes with drag-to-rotate. No GLTF models -- geometry is code-generated. Tailwind `cursor-grab` classes for UX feedback.

**When to use:** Blockchain, network visualization, global data dashboards.

### uHoverState Displacement Shader (Wildlife Studios)
The lightest "3D feel" technique: a single float uniform (0→1 on mouseenter, 1→0 on mouseleave) drives UV displacement. Zero geometry, one fullscreen quad, one texture.

```glsl
uniform float uHoverState;
uniform float uTime;
uniform sampler2D tMap;
varying vec2 vUv;
void main() {
  vec2 uv = vUv;
  float dist = uHoverState * sin(uv.y * 10.0 + uTime) * 0.03;
  uv.x += dist;
  gl_FragColor = texture2D(tMap, uv);
}
```

**When to use:** Any portfolio. Maximum visual impact for minimum GPU cost.

### Rive State Machines for Interactive Heroes (Jasper AI)
Rive `.riv` files replace Lottie/video for interactive, reactive hero animations. State machine integration allows branching animation paths based on scroll/hover/click.

**When to use:** SaaS heroes that need to be interactive but not WebGL-heavy.

### Section-Scoped WebGL Canvases (Dr Pepper, Brew District 24, J-Vers)
Instead of one monolithic canvas, each page section owns its own WebGL context. Enables independent shaders per section and GPU memory isolation.

**When to use:** Long-scroll product pages with distinct 3D sections.

### Variable Font Weight as 3D (Casa di Solare)
Animate `font-variation-settings` on hover -- type itself becomes the 3D object. Combined with pre-rendered C4D/Redshift sequences for hero moments.

**When to use:** Type foundry sites, editorial, anywhere typography is the hero.

## Decision Matrix: Which Signature for Your Project?

| Project Type | Best Signature | Why |
|---|---|---|
| AI/Tech landing page | Gaussian Splatting or Alpha Video | Cutting-edge feel, photorealistic |
| Product showcase | Image Sequence or Scroll-Driven GLTF | User controls the reveal via scroll |
| Creative portfolio | OGL + Custom Shaders or Multiple Canvases | Lightweight, unique, full control |
| Developer portfolio | 81-Shader Pipeline or GPGPU Refraction | Demonstrates deep technical skill |
| SaaS/startup | Spline embed or Rive State Machines | Quick to implement, interactive |
| Luxury brand | Full PBR-IBL or Pre-rendered Video | Maximum visual fidelity |
| Agency site | Persistent R3F Scene Across Routes | The 3D environment IS the brand |
| Blockchain/network | Interactive Globe with OrbitControls | Data visualization as hero |
| Mobile-first | Sprite Sheet or CSS 3D | Zero WebGL, works everywhere |
| Cinematic experience | Alpha Video + Grain Overlay | Film-quality at zero GPU cost |
| Image hover effects | uHoverState Displacement | Maximum impact, minimum cost |
| Type foundry | Variable Font Weight Animation | Typography becomes the 3D |
| Multi-product page | Section-Scoped WebGL Canvases | Independent 3D per section |
