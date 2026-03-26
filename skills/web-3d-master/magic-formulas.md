# Magic Formulas

The exact replicable recipes that make award-winning 3D sites feel extraordinary. Extracted from deep analysis of 16 production sites through source code inspection, runtime DOM analysis, and live browser examination.

These are not theoretical -- they are reverse-engineered from sites that won Awwwards SOTM/SOTD.

## Formula 1: The uHoverState Displacement (Wildlife Studios)

The LIGHTEST way to make images feel 3D. One float uniform does everything.

**The Magic:** A single `uHoverState` float (0→1 on mouseenter, 1→0 on mouseleave) drives UV displacement on a fullscreen quad. Zero geometry, one texture, maximum impact.

```glsl
// Vertex Shader
attribute vec2 uv;
attribute vec3 position;
uniform mat4 modelViewMatrix;
uniform mat4 projectionMatrix;
varying vec2 vUv;

void main() {
  vUv = uv;
  gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}

// Fragment Shader
precision highp float;
uniform sampler2D tMap;
uniform float uTime;
uniform float uHoverState;
uniform vec2 uResolution;
varying vec2 vUv;

void main() {
  vec2 uv = vUv;

  // Aspect-ratio correction
  float aspect = uResolution.x / uResolution.y;

  // The magic: sine-based UV displacement scaled by hover state
  float displacement = uHoverState * sin(uv.y * 10.0 + uTime * 2.0) * 0.03;
  uv.x += displacement;

  // Optional: slight vertical wobble too
  uv.y += uHoverState * cos(uv.x * 8.0 + uTime * 1.5) * 0.015;

  gl_FragColor = texture2D(tMap, uv);
}
```

```js
// JS: Lerp the hover state
let hoverTarget = 0;
let hoverCurrent = 0;

element.addEventListener('mouseenter', () => { hoverTarget = 1; });
element.addEventListener('mouseleave', () => { hoverTarget = 0; });

function animate() {
  hoverCurrent += (hoverTarget - hoverCurrent) * 0.07; // lerp factor
  material.uniforms.uHoverState.value = hoverCurrent;
  requestAnimationFrame(animate);
}
```

**Why it works:** The sine displacement creates organic, wave-like motion that feels physical. The 0.03 amplitude is the sweet spot -- visible but not nauseating. The lerp factor of 0.07 gives ~200ms of smooth easing.

---

## Formula 2: The GPGPU Refraction (Dorian Lods)

Mouse position feeds a GPU simulation that outputs a displacement texture, which refracts the underlying image with chromatic aberration.

**The Magic:** A ping-pong FBO (framebuffer object) pair. Each frame, the simulation reads the previous state, adds mouse influence, and decays. The output texture drives UV displacement.

```glsl
// GPGPU Simulation Shader (runs on a fullscreen quad, output to FBO)
precision highp float;
uniform vec2 uMouse;          // normalized mouse position (0-1)
uniform float uDecay;         // 0.96 -- how fast ripples fade
uniform float uInfluence;     // 0.15 -- mouse influence radius
uniform sampler2D uPrevState; // previous frame's simulation
varying vec2 vUv;

void main() {
  vec4 prev = texture2D(uPrevState, vUv);

  // Calculate mouse influence
  float dist = distance(vUv, uMouse);
  float influence = smoothstep(uInfluence, 0.0, dist);

  // Direction from mouse to pixel
  vec2 dir = normalize(vUv - uMouse + 0.001) * influence * 0.5;

  // Blend with previous state and decay
  vec2 newDisp = mix(prev.rg, dir, 0.15) * uDecay;

  gl_FragColor = vec4(newDisp, 0.0, 1.0);
}

// Refraction Shader (final render pass)
precision highp float;
uniform sampler2D uTexture;
uniform sampler2D uDisplacement;  // from GPGPU simulation
uniform float uRefractionStrength; // 0.15
varying vec2 vUv;

void main() {
  vec4 disp = texture2D(uDisplacement, vUv);
  vec2 refractedUv = vUv + disp.rg * uRefractionStrength;

  // Chromatic aberration: offset R and B channels slightly
  float r = texture2D(uTexture, refractedUv + disp.rg * 0.008).r;
  float g = texture2D(uTexture, refractedUv).g;
  float b = texture2D(uTexture, refractedUv - disp.rg * 0.008).b;

  gl_FragColor = vec4(r, g, b, 1.0);
}
```

**Key values:** Decay: 0.96, Influence radius: 0.15, Refraction strength: 0.15, Chromatic offset: 0.008

---

## Formula 3: The Persistent Scene (Chipsa + Rhumb)

The 3D canvas NEVER unmounts between page navigations. It morphs.

**Architecture (Barba.js + Three.js):**
```js
// Mount the canvas ONCE at the top level, outside any router
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
document.body.appendChild(renderer.domElement);
renderer.domElement.style.cssText = 'position:fixed;top:0;left:0;width:100%;height:100%;z-index:0;pointer-events:none;';

// Barba.js intercepts all navigation
barba.init({
  transitions: [{
    async leave({ current }) {
      // Animate the 3D scene OUT (e.g., dissolve, zoom, rotate)
      await gsap.to(scene.children[0].material.uniforms.uProgress, {
        value: 1, duration: 0.8, ease: 'power2.inOut'
      });
    },
    async enter({ next }) {
      // Load new 3D content for the incoming page
      await loadSceneForRoute(next.url.path);
      // Animate the 3D scene IN
      await gsap.to(scene.children[0].material.uniforms.uProgress, {
        value: 0, duration: 0.8, ease: 'power2.inOut'
      });
    }
  }]
});
```

**Architecture (R3F + Next.js layout):**
```jsx
// app/layout.tsx -- Canvas at layout level, never re-renders on route change
export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        {/* This Canvas persists across ALL routes */}
        <Canvas style={{ position: 'fixed', inset: 0, zIndex: 0 }}>
          <PersistentScene />
          <RouteCamera /> {/* Camera transitions on route change */}
          <EffectComposer>
            <Bloom intensity={0.5} luminanceThreshold={0.9} />
            <Vignette darkness={0.6} />
          </EffectComposer>
        </Canvas>
        {/* HTML content renders on top */}
        <div style={{ position: 'relative', zIndex: 1 }}>{children}</div>
      </body>
    </html>
  );
}
```

---

## Formula 4: The Multi-Canvas Section Architecture (Brew District / Dr Pepper / J-Vers)

Instead of one monolithic canvas, each page section owns its own WebGL context.

```js
// Create a canvas per section, initialize lazily via IntersectionObserver
document.querySelectorAll('[data-3d-section]').forEach(section => {
  const canvas = document.createElement('canvas');
  section.prepend(canvas);

  let renderer, scene, camera, isInitialized = false;

  const observer = new IntersectionObserver(([entry]) => {
    if (entry.isIntersecting && !isInitialized) {
      // Initialize this section's WebGL context
      renderer = new THREE.WebGLRenderer({ canvas, alpha: true, antialias: true });
      renderer.setSize(section.offsetWidth, section.offsetHeight);
      renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
      scene = new THREE.Scene();
      camera = new THREE.PerspectiveCamera(45, section.offsetWidth / section.offsetHeight);

      // Load section-specific content
      loadSectionContent(section.dataset['3dSection'], scene);
      isInitialized = true;
      animate();
    }
  }, { rootMargin: '200px' });

  observer.observe(section);

  function animate() {
    if (!isInitialized) return;
    renderer.render(scene, camera);
    requestAnimationFrame(animate);
  }
});
```

**Why this beats one big canvas:**
- GPU can cull off-screen canvases (no wasted rendering)
- Each section can have completely different shaders/materials
- Independent lifecycle: dispose when scrolled past, reinit when scrolled back
- Memory pressure is distributed, not concentrated

---

## Formula 5: The Procedural Noise Background (Third Dimension Studio)

67KB total JS. Zero textures. Zero models. Everything is shader-generated.

```glsl
// The complete noise background shader
precision highp float;
uniform float uTime;
uniform vec2 uMouse;
uniform vec2 uResolution;
varying vec2 vUv;

// Simplex 3D noise
vec4 permute(vec4 x) { return mod(((x*34.0)+1.0)*x, 289.0); }
vec4 taylorInvSqrt(vec4 r) { return 1.79284291400159 - 0.85373472095314 * r; }

float snoise(vec3 v) {
  const vec2 C = vec2(1.0/6.0, 1.0/3.0);
  const vec4 D = vec4(0.0, 0.5, 1.0, 2.0);
  vec3 i = floor(v + dot(v, C.yyy));
  vec3 x0 = v - i + dot(i, C.xxx);
  vec3 g = step(x0.yzx, x0.xyz);
  vec3 l = 1.0 - g;
  vec3 i1 = min(g.xyz, l.zxy);
  vec3 i2 = max(g.xyz, l.zxy);
  vec3 x1 = x0 - i1 + C.xxx;
  vec3 x2 = x0 - i2 + C.yyy;
  vec3 x3 = x0 - D.yyy;
  i = mod(i, 289.0);
  vec4 p = permute(permute(permute(
    i.z + vec4(0.0, i1.z, i2.z, 1.0))
    + i.y + vec4(0.0, i1.y, i2.y, 1.0))
    + i.x + vec4(0.0, i1.x, i2.x, 1.0));
  float n_ = 1.0/7.0;
  vec3 ns = n_ * D.wyz - D.xzx;
  vec4 j = p - 49.0 * floor(p * ns.z * ns.z);
  vec4 x_ = floor(j * ns.z);
  vec4 y_ = floor(j - 7.0 * x_);
  vec4 x = x_ * ns.x + ns.yyyy;
  vec4 y = y_ * ns.x + ns.yyyy;
  vec4 h = 1.0 - abs(x) - abs(y);
  vec4 b0 = vec4(x.xy, y.xy);
  vec4 b1 = vec4(x.zw, y.zw);
  vec4 s0 = floor(b0)*2.0 + 1.0;
  vec4 s1 = floor(b1)*2.0 + 1.0;
  vec4 sh = -step(h, vec4(0.0));
  vec4 a0 = b0.xzyw + s0.xzyw*sh.xxyy;
  vec4 a1 = b1.xzyw + s1.xzyw*sh.zzww;
  vec3 p0 = vec3(a0.xy, h.x);
  vec3 p1 = vec3(a0.zw, h.y);
  vec3 p2 = vec3(a1.xy, h.z);
  vec3 p3 = vec3(a1.zw, h.w);
  vec4 norm = taylorInvSqrt(vec4(dot(p0,p0), dot(p1,p1), dot(p2,p2), dot(p3,p3)));
  p0 *= norm.x; p1 *= norm.y; p2 *= norm.z; p3 *= norm.w;
  vec4 m = max(0.6 - vec4(dot(x0,x0), dot(x1,x1), dot(x2,x2), dot(x3,x3)), 0.0);
  m = m * m;
  return 42.0 * dot(m*m, vec4(dot(p0,x0), dot(p1,x1), dot(p2,x2), dot(p3,x3)));
}

// FBM (Fractal Brownian Motion) - layered noise
float fbm(vec3 p) {
  float value = 0.0;
  float amplitude = 0.5;
  float frequency = 1.0;
  for (int i = 0; i < 6; i++) {
    value += amplitude * snoise(p * frequency);
    frequency *= 2.0;
    amplitude *= 0.5;
  }
  return value;
}

void main() {
  vec2 uv = vUv;
  float t = uTime * 0.1;

  // Mouse gravitational warping
  vec2 mouseInfluence = (uMouse - 0.5) * 0.3;
  uv += mouseInfluence * (1.0 - length(uv - 0.5) * 2.0);

  // Layered noise
  float n = fbm(vec3(uv * 3.0, t));
  float n2 = fbm(vec3(uv * 2.0 + 100.0, t * 0.7));

  // Color palette morphing over time
  vec3 color1 = vec3(0.02, 0.0, 0.08);  // deep purple
  vec3 color2 = vec3(0.0, 0.02, 0.06);  // midnight blue
  vec3 color3 = vec3(0.0, 0.04, 0.04);  // dark teal

  float colorMix = sin(t * 0.5) * 0.5 + 0.5;
  vec3 baseColor = mix(color1, color2, colorMix);
  baseColor = mix(baseColor, color3, sin(t * 0.3 + 1.0) * 0.5 + 0.5);

  // Apply noise to color
  vec3 color = baseColor + n * 0.08 + n2 * 0.04;

  gl_FragColor = vec4(color, 1.0);
}
```

**Why 67KB:** No Three.js (uses OGL at ~5KB). No textures. No models. All visual output is mathematical. The entire visual identity is a function of time and mouse position.

---

## Formula 6: The FBO Ping-Pong Motion Blur (Third Dimension / Dorian Lods)

Render to texture, read previous frame, blend for temporal smoothing.

```js
// Create two render targets for ping-pong
const rtA = new THREE.WebGLRenderTarget(width, height, {
  type: THREE.FloatType,
  minFilter: THREE.LinearFilter,
  magFilter: THREE.LinearFilter
});
const rtB = rtA.clone();
let current = rtA, previous = rtB;

// Blend shader reads previous frame + current frame
const blendMaterial = new THREE.ShaderMaterial({
  uniforms: {
    uCurrent: { value: null },
    uPrevious: { value: null },
    uBlend: { value: 0.85 } // 0.85 = heavy trail, 0.5 = subtle
  },
  fragmentShader: `
    uniform sampler2D uCurrent;
    uniform sampler2D uPrevious;
    uniform float uBlend;
    varying vec2 vUv;
    void main() {
      vec4 curr = texture2D(uCurrent, vUv);
      vec4 prev = texture2D(uPrevious, vUv);
      gl_FragColor = mix(curr, prev, uBlend);
    }
  `
});

function animate() {
  // Render new frame to current RT
  renderer.setRenderTarget(current);
  renderer.render(mainScene, camera);

  // Blend current + previous into the screen
  blendMaterial.uniforms.uCurrent.value = current.texture;
  blendMaterial.uniforms.uPrevious.value = previous.texture;
  renderer.setRenderTarget(null);
  renderer.render(blendScene, blendCamera);

  // Swap buffers
  [current, previous] = [previous, current];
  requestAnimationFrame(animate);
}
```

**Key value:** `uBlend: 0.85` creates a heavy "wet paint" trail effect. `0.5` is subtle smoothing. `0.95` is extreme ghosting.

---

## Formula 7: The GL Canvas Overlay (Filippo Ruffini)

A fixed WebGL canvas sits ON TOP of the entire page with `pointer-events: none`. It renders displacement effects on images that the user hovers in the DOM below.

```js
// The singleton pattern
window.app = {
  gl: null,
  init() {
    const canvas = document.createElement('canvas');
    canvas.setAttribute('data-gl', 'c');
    canvas.style.cssText = `
      position: fixed;
      top: 0; left: 0;
      width: 100%; height: 100%;
      z-index: 10;
      pointer-events: none;
    `;
    document.body.appendChild(canvas);

    // Initialize OGL or Three.js on this canvas
    this.gl = new Renderer({ canvas, alpha: true });
    // ... setup camera, scene, displacement plane
  },

  animateIn() {
    // Called after page preloader completes
    gsap.fromTo(this.gl.canvas,
      { opacity: 0 },
      { opacity: 1, duration: 1.2, ease: 'power2.out' }
    );
  },

  // DOM elements below can register for hover effects
  registerHoverTarget(domElement, imageUrl) {
    domElement.addEventListener('mouseenter', () => {
      // Load imageUrl as texture, start displacement effect at element position
      this.startDisplacement(domElement.getBoundingClientRect(), imageUrl);
    });
    domElement.addEventListener('mouseleave', () => {
      this.stopDisplacement();
    });
  }
};
```

**Why this pattern:** The WebGL canvas never interferes with page interaction (pointer-events: none). The displacement effect appears to warp the actual page content but it's a texture copy rendered on the GL layer. Scroll events, clicks, and form inputs all pass through.

---

## Formula 8: The Loading Screen as 3D Preview (Dorian Lods)

Even during asset loading, the WebGL canvas is already rendering. The loading screen IS the first visual impression.

```js
// Start the WebGL renderer IMMEDIATELY, before any 3D assets load
const renderer = new THREE.WebGLRenderer({ antialias: true });
document.body.appendChild(renderer.domElement);

// Render a simple noise shader while heavy assets (Draco, Basis) load
const loadingMaterial = new THREE.ShaderMaterial({
  uniforms: { uTime: { value: 0 }, uProgress: { value: 0 } },
  fragmentShader: `
    uniform float uTime;
    uniform float uProgress;
    varying vec2 vUv;
    // ... noise function here ...
    void main() {
      float n = fbm(vec3(vUv * 3.0, uTime * 0.1));
      vec3 color = vec3(0.01, 0.0, 0.05) + n * 0.03;
      gl_FragColor = vec4(color, 1.0);
    }
  `
});

// Loading manager tracks asset progress
const manager = new THREE.LoadingManager();
manager.onProgress = (url, loaded, total) => {
  loadingMaterial.uniforms.uProgress.value = loaded / total;
};
manager.onLoad = () => {
  // Crossfade from loading shader to full scene
  gsap.to(loadingMaterial.uniforms.uProgress, {
    value: 1, duration: 1.5, ease: 'power2.inOut',
    onComplete: () => switchToMainScene()
  });
};
```

---

## Formula 9: The Scroll-Pinned Sticky Canvas (Brew District / Eco-Pork)

A Three.js canvas stays pinned (position: sticky) while HTML content scrolls past. GSAP ScrollTrigger updates the 3D scene based on scroll progress.

```js
// HTML structure:
// <div class="scroll-runway" style="height: 400vh">
//   <div class="sticky-canvas-wrapper" style="position: sticky; top: 0; height: 100vh">
//     <canvas id="product-canvas"></canvas>
//   </div>
// </div>

// GSAP binds scroll to 3D state
ScrollTrigger.create({
  trigger: '.scroll-runway',
  start: 'top top',
  end: 'bottom bottom',
  scrub: 2, // 2-second smoothing
  onUpdate: (self) => {
    const p = self.progress; // 0 → 1

    // Rotate product based on scroll
    product.rotation.y = p * Math.PI * 2;

    // Move camera
    camera.position.z = 5 - p * 2;
    camera.position.y = Math.sin(p * Math.PI) * 1.5;

    // Change material properties
    product.material.color.lerpColors(goldColor, silverColor, p);
  }
});

// GSAP snap for precise product transitions
ScrollTrigger.create({
  trigger: '.scroll-runway',
  snap: {
    snapTo: [0, 0.25, 0.5, 0.75, 1], // snap to each product
    duration: 0.4,
    ease: 'power2.inOut'
  }
});
```

---

## Formula 10: The Matcap-Only Pipeline (Superlist)

No lights. No shadows. No environment maps. Just matcap textures. Looks amazing, costs almost nothing.

```js
// One matcap texture per model category
const matcapGold = new THREE.TextureLoader().load('/matcap-gold.png');
const matcapSilver = new THREE.TextureLoader().load('/matcap-silver.png');
const matcapDark = new THREE.TextureLoader().load('/matcap-dark.png');

// Apply to loaded GLTF models
gltfLoader.load('/product.glb', (gltf) => {
  gltf.scene.traverse((child) => {
    if (child.isMesh) {
      // Replace all materials with matcap
      child.material = new THREE.MeshMatcapMaterial({
        matcap: matcapGold
      });
    }
  });
  scene.add(gltf.scene);
});

// NO scene.add(light) needed. Zero lighting calculations per frame.
// The matcap texture encodes ALL lighting information.
```

**Creating matcap textures:** Render a sphere in Blender with your desired lighting, export the render as a square PNG (256x256 is sufficient). That PNG IS your matcap.

---

---

## EXTRACTED PRODUCTION VALUES (from source code analysis)

### Dorian Lods -- Exact Glass Material Config
```js
// The actual material values from the production bundle
M_Glass_Box: {
  albedoColor: "#34034f",
  MetalicFactor: 2,
  RoughnessFactor: 0.35,
  IBL: 1.5,
  SheenColor: "#FFFFFF",
  sheenOpacity: 0.75,
  sheenDepth: 8,
  UvScale: 2,
  IOR: 0.4,
  RefractionRoughness: 0.1,
  // 6-wavelength chromatic dispersion IOR values:
  IorR: 1.15, IorY: 1.16, IorG: 1.18,
  IorC: 1.22, IorB: 1.22, IorP: 1.22,
  BackfaceInfluence: 0.58,
  Contrast: 0.35,
  Saturation: 1.2
}
```

### Dorian Lods -- GPGPU Fluid Simulation Values
```
Decay: frame *= 1. - (1.25 * uDt)
Shrink: uv = ((vUv - .5) * (1. - (.35 * uDt))) + .5
Mouse influence: smoothstep(0.05, 0.35, dist), pow 3
Velocity multiplier: uVelocity * 2.
Grid size for text distortion: 80.
```

### Dorian Lods -- Lenis Config
```js
new Lenis({ autoRaf: true, lerp: 0.25 });
// Easing: (t) => Math.min(1, 1.001 - Math.pow(2, -10 * t)) // easeOutExpo
```

### Rhumb Studio -- Camera Presets (exact positions)
```js
const CAMERA_PRESETS = {
  Home:         { position: [3.1, 2.1, -3],   rotationX: -0.1,  rotationY: -0.6 },
  About:        { position: [-20, 2.1, -7],   rotationX: 0,     rotationY: -1.72 },
  Service:      { position: [6, 2.75, -3],     rotationX: 0,     rotationY: -3.14 },
  Contact:      { position: [-9, 2.1, 2],      rotationX: 0,     rotationY: -2.46 },
  Work:         { position: [4.3, 2.1, -4],    rotationX: -0.1,  rotationY: -0.6 },
  "Case Study": { position: [6.3, 2.1, -8],   rotationX: -0.19, rotationY: -1.38 },
  Outside:      { position: [-32.7, 3, 30],    rotationX: 0,     rotationY: -0.84 },
};
// Transition: gsap duration 3s, ease "power2.inOut"
// Responsive FOV: mobile=90, tablet=70, desktop=50
```

### Rhumb Studio -- Post-Processing Stack (exact values)
```js
// NOT Bloom -- that's a common assumption but wrong
{
  vignetteOffset: 0.4,
  vignetteSoftness: 0.4,
  vignetteDarkness: 0.6,
  noiseOpacity: 0.1,           // Noise blend: OVERLAY mode
  resolutionScale: 0.75,       // Render at 75% for performance
  chromaticAberrationOffset: 0.00075  // Very subtle
}
```

### Rhumb Studio -- Mouse Parallax Values
```js
// The exact lerp + breathing values from production
const targetX = 0.0001 * mouseX;   // Very subtle
const targetY = 0.00015 * mouseY;
smoothX += 0.08 * (targetX - smoothX); // Lerp factor 0.08
// Idle breathing:
const breathX = 0.025 * Math.sin(0.5 * time);
const breathY = 0.015 * Math.cos(0.3 * time);
```

### Rhumb Studio -- GPGPU Particle System (384x384 = 147,456 particles)
```js
// Particle init volume:
const scaleX = 2.5, scaleY = 2.0, scaleZ = 2.5;
const radialVariation = mix(0.8, 1.2, random);
// Flow field: 4 wave layers + curl + radial + hash noise
// Advection speed: uDeltaTime * 2.5 * speedVariation
// Soft bounds: radius > 3.0 → pushback 0.94
// Respawn: life >= 1.0 || radius > 4.5
```

### Rhumb Studio -- Water Surface Values
```js
{
  uWaveFrequency: 3,
  uOpacity: 0.7,
  uUvScale: 3,
  uColorShallow: "#6c6a6a",
  uColorDeep: "#3e2813",
  uRoughness: 0.65,
  uMetalness: 0.45,
  uReflectionStrength: 0.8,
  // Geometry: PlaneGeometry(20, 50, 64, 64)
  // Reflection RT: WebGLRenderTarget(512, 512)
}
```

### Rhumb Studio -- Particle Render Colors
```glsl
// Inner (center of point): vec3(0.75, 0.9, 1.0)  -- light blue-white
// Outer (edge of point):   vec3(0.1, 0.2, 0.35)   -- dark blue
// Blending: AdditiveBlending, depthWrite: false
```

---

---

## EXTRACTED PRODUCTION SHADERS (from deep source analysis)

### World Labs -- Gaussian Splat Glass Sphere Shader
```glsl
// The center sphere: refraction + chromatic aberration + animated specular + edge glow
uniform float refractionRatio;     // 0.45
uniform float fresnelPower;        // -0.22
uniform float distortionStrength;  // 0.15
uniform float chromaticAberration; // 0.025
uniform float glowIntensity;       // 1.5

// Chromatic aberration samples R/G/B at different normal offsets:
float rMap = texture(map, refractedUv + chromaticOffset * 1.0).r;
float gMap = texture(map, refractedUv + chromaticOffset * 0.5).g;
float bMap = texture(map, refractedUv - chromaticOffset * 0.5).b;

// Edge glow: pow(edgeFactor, 4.5) * glowIntensity
```

### World Labs -- Scene Transition Portal Reveal
```glsl
// Noise-driven organic portal: the main-to-splat transition
float altNoise = texture(altNoiseMap, uvAr).r;
float noise = texture(altNoiseMap, uvAr + vec2(altNoise * 0.5 + time * 0.1,
    altNoise * 0.25 + time * 0.051)).r;
noise = pow(noise, 0.75);
float distanceField = smoothstep(0.0, 0.25 * pow(noise, 4.5),
    distanceToCenter * noise - progress / min(aspect, 1.0) + 0.3);
```

### Filippo Ruffini -- Complete Glass Refraction Pipeline
```glsl
// The dual-pass glass refraction that made this an Awwwards SOTD
// Pass 1: Render backface normals to a render target
gl_FragColor = vec4(worldNormal, 1.0); // encode normals as RGB

// Pass 2: Use backface normals + scene texture for refraction
vec3 backfaceNormal = texture2D(u_bs, uv).rgb;
vec3 normal = worldNormal * (1.0 - 0.45) - backfaceNormal * 0.45;
vec3 refracted = refract(eyeVector, normal, 1.0 / 1.05); // IOR 1.05
uv += refracted.xy; // THE DISPLACEMENT
float f = pow(1.0 + dot(eyeVector, worldNormal), 3.0); // Fresnel power 3
```

### Filippo Ruffini -- Mouse Velocity Displacement Texture
```js
// 32x32 DataTexture stores mouse velocity. Each frame: decay + paint
for (let i = 0; i < data.length; i += 4) {
  data[i] *= 0.9;     // R: horizontal velocity decay
  data[i + 1] *= 0.9; // G: vertical velocity decay
}
// Paint velocity at cursor position (radius 12 pixels):
data[index] += mouse.vx * 1 * power;
data[index + 1] += mouse.vy * 1 * power;
// Post shader: uv - 0.05 * vec2(tx.rg)  // displace by velocity
```

### Third Dimension Studio -- Complete Noise Background
```glsl
// THE EXACT PRODUCTION SHADER (complete, verified from source bundle)
vec2 pos = vec2(st * 3.0);
float DF = 0.0;

// Layer 1: Time-scrolling noise
vec2 vel = vec2(uTime * 0.1);
DF += snoise(pos + vel) * 0.25 + 0.25;

// Layer 2: Noise-driven angular velocity field (the organic flow)
float a = snoise(pos * vec2(cos(uTime*0.15), sin(uTime*0.1)) * 0.1) * 3.1415;
vel = vec2(cos(a), sin(a));
DF += snoise(pos + vel) * -0.174 + 0.25;

// Color: smoothstep distance field between two palette colors
color = (mix(uColorMain, uColorFirst,
    smoothstep(0.732, -0.082, fract(DF)) * uVisibility)) * 1.5;

// Film grain: fract(cos(dot(p, vec2(23.14069, 2.66514))) * 12345.6789) * 0.05
```

### David Heckhoff -- Holographic Model Reveal
```glsl
// Vertex-driven progress: each vertex maps its Y position to 0-1 reveal
float modelProgress = getModelProgress(worldPos.y);
// Fragment: stripe pattern + fresnel + scan line
float stripes = step(0.5, fract(modelProgress * 20.0));
float scan = smoothstep(0.0, 0.02, abs(modelProgress - uRevealProgress));
color = mix(holoColor, baseColor, stripes * fresnel + scan);
```

### Eco-Pork -- Portal Plane Clipping System
```js
// 6 THREE.Plane objects used as clippingPlanes on meshes
// Animating plane.constant (distance) creates "depth windows"
renderer.localClippingEnabled = true;
const portalPlane = new THREE.Plane(new THREE.Vector3(0, 1, 0), 0);
mesh.material.clippingPlanes = [portalPlane];
// GSAP animates the plane to sweep one model away, revealing another:
gsap.to(portalPlane, { constant: 2, duration: 1.5, ease: "power2.inOut" });
```

### Brew District 24 -- Single GLTF, 6 Product Variants
```js
// ONE model file with named layers per product. Toggle visibility to switch:
gltf.scene.traverse(child => {
  if (child.name === beers[activeIndex].model_layer_name) child.visible = true;
  else if (child.name !== 'can_body') child.visible = false;
});
// 3-phase transition: spin 4*PI + lift (250ms), swap layer (mid-spin), restore (500ms)
// Can body: metalness=1, roughness=0.45 (always visible, shared across all beers)
```

### Brew District 24 -- Sticky Canvas Scroll Binding
```js
// Manual sticky: update canvas Y position on every Lenis scroll event
lenis.on('scroll', ({ scroll }) => {
  gsap.to(".stickyCanWrapper .canvasWrapper", 0, { y: scrollOffset });
  // Scroll percentage drives group rotation:
  group.rotation.z = scrollPercent * Math.PI * 0.5;
  group.rotation.y = scrollPercent * Math.PI * 2;
});
// 100 JPEG frames for sequence canvas, cover-fit via drawImageProp()
```

### Chipsa Design -- R3F View Scissor Technique
```jsx
// ONE Canvas at layout level, multiple Views via scissor tests
// _app.tsx:
<Canvas><View.Port /></Canvas>

// Each section:
<View track={ref}> {/* tracks a DOM element's getBoundingClientRect */}
  <SectionScene />
</View>
// View internally uses gl.scissor() to render only into the tracked region
```

---

## The Meta-Formula: Choosing Your Magic

| Budget | Time | Devices | Best Formula |
|---|---|---|---|
| $0 | 1 day | All | Formula 1 (uHoverState) or Formula 5 (Procedural Noise) |
| $0 | 3 days | Desktop | Formula 6 (FBO Ping-Pong) + Formula 7 (GL Overlay) |
| Low | 1 week | Desktop | Formula 9 (Sticky Canvas) + Formula 10 (Matcap Pipeline) |
| Medium | 2 weeks | Desktop + Mobile | Formula 4 (Multi-Canvas) + Formula 8 (Loading Preview) |
| High | 1 month | All | Formula 3 (Persistent Scene) + Formula 2 (GPGPU Refraction) |
