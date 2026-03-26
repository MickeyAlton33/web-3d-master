# Deep-Dive: Cosmos Network + Madbox 3D Approaches

---

## SITE 1: Cosmos Network (cosmos.network)

**Stack:** Next.js (App Router) + Three.js r170+ (WebGPU-capable build) + `three-globe` npm package + EffectComposer post-processing + Tailwind CSS v4

### 1. Three-Globe Container Setup

Both globes use the `three-globe` npm package (chunk 804, module 80681). The package is a declarative wrapper around Three.js that creates a `SphereGeometry(100, segments, segments/2)` globe with chainable methods.

```js
// The three-globe library creates its internal globe like this:
// (from chunk 804 / module 80681)
const globeMaterial = new THREE.MeshPhongMaterial({ color: 0x000000 });
const globeMesh = new THREE.Mesh(undefined, globeMaterial); // geometry set later
globeMesh.rotation.y = -Math.PI / 2;

const globeGroup = new THREE.Group();
globeGroup.__globeObjType = "globe";
globeGroup.add(globeMesh);

// Globe initialization (chainable API):
const globe = new ThreeGlobe({ animateIn: false })
  .globeImageUrl("/globe/background/earth-combined.jpg")
  .bumpImageUrl("/globe/background/earth-topology.png")
  .atmosphereColor("#7E7E7E")
  .atmosphereAltitude(0.5);
```

The container is a simple `<div ref={containerRef}>` -- no canvas element in JSX. The renderer creates its own canvas and appends it:

```jsx
// Both globes use this pattern:
<div
  ref={containerRef}
  className={cn(
    "three-globe-container cursor-grab active:cursor-grabbing w-full h-full relative overflow-hidden",
    className,
    { "pointer-events-none": !isInteractive }
  )}
/>
```

### 2. Two Separate Globe Instances (HomepageHero + HomepageCosmosHub)

**Architecture:** Two completely independent globe components, each lazy-loaded with `next/dynamic`, each creating their own renderer, scene, camera, and EffectComposer.

**HomepageHero Globe** (chunk 788, module 89788):
```js
// Lazy loaded in HomepageHero component:
const HeroGlobe = dynamic(
  () => import(/* webpackChunkName: "hero-globe" */ './HeroGlobe'),
  { ssr: false, loading: () => <div className="w-full h-full bg-transparent" /> }
);

// TWO instances rendered -- one for mobile, one for desktop:
<div className="HomepageHero__globe lg:hidden absolute -top-[224px] left-0 w-screen h-screen z-0">
  <HeroGlobe />
</div>
<div className="HomepageHero__globe hidden lg:block absolute top-1/2 -translate-y-1/2 left-[27%] w-full h-[90vh]">
  <HeroGlobe logos={true} />  {/* Desktop gets floating logos */}
</div>
```

The hero globe adds **ring pulse animations** and **floating logo sprites**:
```js
// Ring data for pulsing markers at company locations:
const ringData = [
  { name: "progmat", logo: "/globe/logos/progmat.png", lat: 35.6762, lng: 139.6503,
    maxR: 30, propagationSpeed: 10, repeatPeriod: 1000, color: "rgba(255,100,50,1)" },
  { name: "Ondo Finance", logo: "/globe/logos/ondo.png", lat: 40.705, lng: -73.975,
    maxR: 30, propagationSpeed: 2, repeatPeriod: 1500, color: "rgba(100,200,255,1)" },
  // ... 5 more companies
];

const globe = new ThreeGlobe()
  .globeImageUrl("/globe/background/earth-combined.jpg")
  .bumpImageUrl("/globe/background/earth-topology.png")
  .ringsData(ringData)
  .ringColor(d => t => d.color.replace(/[\d.]+\)$/, `${1 - t})`)) // fade alpha
  .ringMaxRadius("maxR")
  .ringPropagationSpeed("propagationSpeed")
  .ringRepeatPeriod("repeatPeriod")
  .atmosphereColor("#7E7E7E")
  .atmosphereAltitude(0.5);

// Floating logo sprites (when logos=true):
ringData.forEach(point => {
  if (point.logo) {
    // Convert lat/lng to 3D position on sphere
    const globePos = latLngToVector3(point.lat, point.lng, 100);

    // Line connecting globe surface to logo
    const lineGeo = new THREE.BufferGeometry();
    lineGeo.setAttribute('position', new THREE.BufferAttribute(new Float32Array(6), 3));
    const line = new THREE.LineSegments(lineGeo, new THREE.LineBasicMaterial({
      color: 0xffffff, opacity: 0.5, transparent: true
    }));

    // Sprite with texture
    new THREE.TextureLoader().load(point.logo, texture => {
      const spriteMat = new THREE.SpriteMaterial({
        map: texture, sizeAttenuation: true, depthTest: true, depthWrite: false
      });
      const sprite = new THREE.Sprite(spriteMat);
      const aspect = texture.image.width / texture.image.height;
      sprite.scale.set(20 * aspect, 20, 1);
    });
  }
});
```

**HomepageCosmosHub Globe** (chunk 736, module 12736):
```js
// Lazy loaded in HomepageCosmosHub:
const HubGlobe = dynamic(
  () => import(/* webpackChunkName: "hub-globe" */ './HubGlobe'),
  { ssr: false, loading: () => <div className="w-full h-full bg-transparent" /> }
);

// Uses forwardRef + useImperativeHandle for parent control:
const HubGlobe = forwardRef((props, ref) => {
  const stateRef = useRef({
    targetRotation: 100 * Math.PI / 180,
    currentHoverColor: null,
    colorOverlayOpacity: 0
  });

  useImperativeHandle(ref, () => ({
    setGlobeRotation: (degrees) => {
      stateRef.current.targetRotation = degrees * Math.PI / 180;
    },
    setColorOverlay: (hexColor, opacity) => {
      stateRef.current.currentHoverColor = new THREE.Color(hexColor);
      stateRef.current.colorOverlayOpacity = opacity;
    }
  }));
  // ...
});

// Parent controls it via ref on button click:
onClick={() => {
  if (globeRef.current) {
    globeRef.current.setGlobeRotation(feature.rotation);
    globeRef.current.setColorOverlay(
      feature.color,
      0.6 * (feature.color !== "#FFFFFF") // binary: 0 or 0.6
    );
  }
}
```

The HubGlobe has a **color overlay sphere** -- a second transparent `MeshBasicMaterial` sphere rendered on a second pass:
```js
// Overlay sphere for color tinting
const overlaySphere = new THREE.SphereGeometry(100, 64, 64);
const overlayMaterial = new THREE.MeshBasicMaterial({
  color: 0xffffff,
  transparent: true,
  opacity: 0,
  depthWrite: false,
  blending: THREE.AdditiveBlending
});
const overlayMesh = new THREE.Mesh(overlaySphere, overlayMaterial);
overlayScene.add(overlayMesh);

// In animation loop -- lerp color and opacity:
if (state.currentHoverColor) {
  overlayMaterial.color.lerp(state.currentHoverColor, 0.1);
}
overlayMaterial.opacity += (state.colorOverlayOpacity - overlayMaterial.opacity) * 0.05;

// Multi-pass rendering:
composer.render();                    // Main globe via EffectComposer
renderer.autoClear = false;
renderer.clearDepth();
renderer.render(overlayScene, camera); // Overlay sphere on top
renderer.autoClear = true;
```

### 3. OrbitControls Configuration

Both globes use Three.js OrbitControls with nearly identical locked-down settings:

```js
// HubGlobe OrbitControls:
const controls = new OrbitControls(camera, renderer.domElement);
controls.minDistance = 400;    // Locked at exactly 400 -- no zoom
controls.maxDistance = 400;    // Same value = zoom disabled
controls.rotateSpeed = 2;     // Fast rotation response
controls.noZoom = true;        // Explicit zoom disable
controls.noPan = true;         // No panning allowed

// HeroGlobe OrbitControls (slightly different):
const controls = new OrbitControls(camera, renderer.domElement);
controls.minDistance = 101;    // Just above globe radius (100)
controls.maxDistance = 500;    // Allows some zoom range
controls.rotateSpeed = 2;
controls.noZoom = true;        // But still disabled!
controls.noPan = true;

// OrbitControls internals (from chunk 394):
// enableDamping defaults to false (not explicitly set)
// dampingFactor = 0.05 (default)
// autoRotate = false (default)
// mouseButtons = { LEFT: ROTATE, MIDDLE: DOLLY, RIGHT: PAN }
// touches = { ONE: ROTATE, TWO: DOLLY_PAN }
```

### 4. Procedural Globe Geometry (No GLTF)

Everything is code-generated. The globe is a `SphereGeometry`:

```js
// Base globe sphere (from three-globe library):
const segments = Math.max(4, Math.round(360 / globeCurvatureResolution));
globeObj.geometry = new THREE.SphereGeometry(100, segments, segments / 2);
// Default globeCurvatureResolution = 4, so segments = 90, giving SphereGeometry(100, 90, 45)

// Atmosphere glow (custom shader mesh):
const atmosphereMesh = new AtmosphereGlow(globeObj.geometry, {
  color: "#7E7E7E",           // Grey atmosphere
  size: 100 * 0.5,            // 50 units outward  (atmosphereAltitude * radius)
  hollowRadius: 100,          // Inner radius matches globe
  coefficient: 0.1,           // Low base intensity
  power: 3.5                  // Sharp falloff
});
// Geometry is cloned from globe and vertices pushed outward along normals
```

### 5. Tailwind cursor-grab UX Pattern

```jsx
// HeroGlobe container:
<div
  ref={containerRef}
  className={cn(
    "three-globe-container",
    "cursor-grab",                    // Default: open hand
    "active:cursor-grabbing",          // While mouse down: closed hand
    "w-full h-full relative overflow-hidden",
    className,
    { "pointer-events-none": !isReady } // Disabled until intro animation completes
  )}
/>

// HubGlobe container:
<div
  ref={containerRef}
  className={cn(
    "HubGlobe",
    className,
    "w-[200%] lg:w-full",             // 200% width on mobile, full on desktop
    "h-full",
    "translate-x-[-50%] lg:translate-x-0", // Center the 200% on mobile
    "opacity-50 lg:opacity-100"         // Dimmed on mobile
  )}
/>
```

### 6. Custom Shader Code: Dither Post-Processing Effect

The signature Cosmos visual is a **halftone/dither post-processing shader** applied via `EffectComposer > ShaderPass`. Both globes use the identical shader:

```glsl
// VERTEX SHADER
varying vec2 vUv;
void main() {
  vUv = uv;
  gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}

// FRAGMENT SHADER -- Full Dither/Halftone Post-Process
uniform sampler2D tDiffuse;       // Scene render
uniform sampler2D tPattern;       // dot_149.png texture
uniform vec2 resolution;
uniform float patternSize;        // 6 (1x DPI) or 5 (2x DPI)
uniform float threshold;          // 1.0
uniform float contrast;           // 1.0
uniform float exposure;           // 0.5
uniform bool invert;              // false
uniform bool greyscale;           // true
uniform float blending;           // 1.0  (full dither)
uniform bool disable;             // false
uniform float patternType;        // 4 = texture-based
uniform float blackOverlay;       // 0.8  (heavy darkening!)

varying vec2 vUv;

float luma(vec3 color) {
  return dot(color, vec3(0.299, 0.587, 0.114));
}

// Procedural pattern generators (types 0-3):
float generateDotPattern(vec2 coord) {
  vec2 center = fract(coord) - 0.5;
  return 1.0 - smoothstep(0.0, 0.5, length(center));
}

float generateLinePattern(vec2 coord) {
  return sin(coord.y * 6.28318) * 0.5 + 0.5;
}

float generateGridPattern(vec2 coord) {
  vec2 grid = abs(fract(coord) - 0.5);
  return smoothstep(0.4, 0.5, max(grid.x, grid.y));
}

float generateNoisePattern(vec2 coord) {
  return fract(sin(dot(coord, vec2(12.9898, 78.233))) * 43758.5453);
}

void main() {
  if (disable) {
    gl_FragColor = texture2D(tDiffuse, vUv);
    return;
  }

  vec4 tex = texture2D(tDiffuse, vUv);
  vec3 color = tex.rgb;

  // Exposure adjustment
  float exposureFactor = pow(2.0, exposure);
  color *= exposureFactor;

  // Contrast adjustment
  color = ((color - 0.5) * contrast) + 0.5;
  color = clamp(color, 0.0, 1.0);

  // Convert to brightness
  float brightness = greyscale ? luma(color) : (color.r + color.g + color.b) / 3.0;

  // Pattern coordinate
  vec2 patternCoord = vUv * resolution / patternSize;
  float patternThreshold;

  if (patternType < 0.5) {
    patternThreshold = generateDotPattern(patternCoord);
  } else if (patternType < 1.5) {
    patternThreshold = generateLinePattern(patternCoord);
  } else if (patternType < 2.5) {
    patternThreshold = generateGridPattern(patternCoord);
  } else if (patternType < 3.5) {
    patternThreshold = generateNoisePattern(patternCoord);
  } else {
    // Type 4: Texture-based dot pattern (dot_149.png)
    vec2 textureCoord = vUv * resolution / patternSize;
    vec4 patternSample = texture2D(tPattern, textureCoord);
    patternThreshold = patternSample.r; // Red channel as threshold
  }

  patternThreshold *= threshold;

  // Binary dither decision
  float ditherResult = brightness > patternThreshold ? 1.0 : 0.0;

  if (invert) {
    ditherResult = 1.0 - ditherResult;
  }

  vec3 ditherColor = vec3(ditherResult);
  vec3 finalColor = mix(color, ditherColor, blending);

  // Heavy black overlay (0.8 = very dark)
  finalColor = mix(finalColor, vec3(0.0), blackOverlay);

  gl_FragColor = vec4(finalColor, tex.a);
}
```

The pattern texture is loaded once and shared:
```js
let sharedPatternTexture = null;
// ...
if (!sharedPatternTexture) {
  sharedPatternTexture = new THREE.TextureLoader().load("/globe/background/dot_149.png");
  sharedPatternTexture.wrapS = THREE.RepeatWrapping;
  sharedPatternTexture.wrapT = THREE.RepeatWrapping;
  sharedPatternTexture.minFilter = THREE.NearestFilter;
  sharedPatternTexture.magFilter = THREE.NearestFilter;
}
ditherPass.uniforms.tPattern.value = sharedPatternTexture;
```

The EffectComposer pipeline:
```js
const composer = new EffectComposer(renderer);
const renderPass = new RenderPass(scene, camera);
composer.addPass(renderPass);

const ditherPass = new ShaderPass(ditherShader);
ditherPass.uniforms.resolution.value.set(width, height);
ditherPass.uniforms.patternSize.value = devicePixelRatio < 2 ? 6 : 5; // DPI-aware
ditherPass.uniforms.tPattern.value = sharedPatternTexture;
composer.addPass(ditherPass);
```

### 6b. Atmosphere Glow Shader

```glsl
// VERTEX SHADER
uniform float hollowRadius;

varying vec3 vVertexWorldPosition;
varying vec3 vVertexNormal;
varying float vCameraDistanceToObjCenter;
varying float vVertexAngularDistanceToHollowRadius;

void main() {
  vVertexNormal = normalize(normalMatrix * normal);
  vVertexWorldPosition = (modelMatrix * vec4(position, 1.0)).xyz;

  vec4 objCenterViewPosition = modelViewMatrix * vec4(0.0, 0.0, 0.0, 1.0);
  vCameraDistanceToObjCenter = length(objCenterViewPosition);

  float edgeAngle = atan(hollowRadius / vCameraDistanceToObjCenter);
  float vertexAngle = acos(dot(
    normalize(modelViewMatrix * vec4(position, 1.0)),
    normalize(objCenterViewPosition)
  ));
  vVertexAngularDistanceToHollowRadius = vertexAngle - edgeAngle;

  gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}

// FRAGMENT SHADER
uniform vec3 color;
uniform float coefficient;   // 0.1
uniform float power;          // 3.5
uniform float hollowRadius;   // 100

varying vec3 vVertexNormal;
varying vec3 vVertexWorldPosition;
varying float vCameraDistanceToObjCenter;
varying float vVertexAngularDistanceToHollowRadius;

void main() {
  if (vCameraDistanceToObjCenter < hollowRadius) discard; // inside the globe
  if (vVertexAngularDistanceToHollowRadius < 0.0) discard; // within hollow radius

  vec3 worldCameraToVertex = vVertexWorldPosition - cameraPosition;
  vec3 viewCameraToVertex = (viewMatrix * vec4(worldCameraToVertex, 0.0)).xyz;
  viewCameraToVertex = normalize(viewCameraToVertex);

  float intensity = pow(
    coefficient + dot(vVertexNormal, viewCameraToVertex),
    power
  );
  gl_FragColor = vec4(color, intensity);
}
```

### 7. Arc Rendering: CubicBezierCurve3 + TubeGeometry

The `three-globe` library renders arcs using **CubicBezierCurve3** (not QuadraticBezier):

```js
// Arc path generation (from chunk 804):
// 1. Convert start/end lat/lng to 3D points on sphere
// 2. Calculate great-circle interpolation for control points
// 3. Control points elevated above sphere surface for arc height

function createArcCurve(startLatLng, endLatLng, altitude) {
  const startXYZ = latLngToXYZ(startLatLng, startAlt);
  const endXYZ = latLngToXYZ(endLatLng, endAlt);

  // Great-circle interpolator for ground path
  const interpolator = d3GeoInterpolate(startLatLng, endLatLng);

  // Two control points at 25% and 75% along path, elevated by altitude
  const cp1 = latLngToXYZ(interpolator(0.25), computedAltitude);
  const cp2 = latLngToXYZ(interpolator(0.75), computedAltitude);

  return new THREE.CubicBezierCurve3(startXYZ, cp1, cp2, endXYZ);
}

// Rendering decision based on arcStroke:
if (hasStroke) {
  // Thick arcs: TubeGeometry wrapping the curve
  arc.geometry = new THREE.TubeGeometry(
    curve,
    arcCurveResolution,    // segments along path
    strokeWidth / 2,        // tube radius
    arcCircularResolution   // radial segments
  );
  arc.geometry.setAttribute("color", colorAttribute);
  arc.geometry.setAttribute("relDistance", distanceAttribute);
} else {
  // Thin arcs: Line geometry from sampled curve points
  arc.geometry.setFromPoints(curve.getPoints(arcCurveResolution));
}
```

### 8. Entry Animation (Zoom-In Spin)

Both globes have a dramatic intro animation:

```js
// HeroGlobe intro (2.8 seconds):
const intro = {
  active: true,
  startTime: Date.now(),
  duration: 2800,
  initialRotation: 2 * Math.PI + targetRotation,  // Full spin + target
  targetRotation: 100 * Math.PI / 180,             // 100 degrees
  initialScale: 0.001,                              // Start tiny
  targetScale: 1,
  initialCameraZ: 1200,                              // Start far away
  targetCameraZ: 400,
  initialCameraY: 0,
  targetCameraY: isMobile ? 80 : 60                  // Look slightly down
};

// In animation loop -- cubic ease-out:
const elapsed = Math.min((Date.now() - intro.startTime) / intro.duration, 1);
const eased = 1 - Math.pow(1 - elapsed, 3); // Cubic ease-out

const scale = intro.initialScale + (intro.targetScale - intro.initialScale) * eased;
globe.scale.set(scale, scale, scale);
globe.rotation.y = intro.initialRotation - (intro.initialRotation - intro.targetRotation) * eased;
camera.position.z = intro.initialCameraZ - (intro.initialCameraZ - intro.targetCameraZ) * eased;

// After intro, slow continuous rotation:
globe.rotation.y += -0.01 * Math.PI / 180; // ~0.00017 radians/frame
```

---

## SITE 2: Madbox (madbox.io)

**Stack:** Nuxt.js 2 (Vue 2) + Three.js r0.118 (older version, NOT WebGPU) + GSAP 3 + Canvas 2D page transitions + Custom camera controls

### 1. The WebGL Engine: It IS Three.js (old version)

Contrary to what one might expect from the non-Three.js look, Madbox uses **Three.js r0.118** (an older version from ~2020). Evidence from module 626:

```js
// module 626 -- Renderer class
this.instance = new THREE.WebGLRenderer({
  alpha: true,
  antialias: true,
  sortObjects: false
});
this.instance.domElement.style.position = "absolute";
this.instance.domElement.style.top = 0;
this.instance.domElement.style.left = 0;
this.instance.domElement.style.width = "100%";
this.instance.domElement.style.height = "100%";
this.instance.setClearColor("#ffffff", 1);
this.instance.setSize(config.width, config.height);
this.instance.setPixelRatio(config.pixelRatio);
this.instance.physicallyCorrectLights = true;
this.instance.gammaOutPut = true;
this.instance.outputEncoding = THREE.sRGBEncoding;
this.instance.shadowMap.enabled = false;
this.instance.autoClear = false;

// Multi-layer rendering with stencil:
// Layer 0: Main scene
// Layer 1: Borders/outlines
// Layer 2: Overlay
```

The "custom" feel comes from the **architecture wrapping Three.js**, not raw WebGL. It is a **bespoke experience framework** with:
- Custom `EventEmitter` base class
- Resource loader with grouped loading stages ("base", "rest", "optional")
- Custom camera control system (NOT OrbitControls)
- Night mode system
- Matcap material management
- Focus point system for camera animation

```js
// Main Experience class (the "engine"):
class Experience extends EventEmitter {
  constructor({ targetElement }) {
    window.experience = this;
    this.targetElement = targetElement;
    this.time = new Time();
    this.sizes = new Sizes();

    requestAnimationFrame(() => {
      this.setConfig();
      this.setDebug();
      this.setStats();
      this.setScene();       // new THREE.Scene()
      this.setCamera();      // new THREE.PerspectiveCamera(55, aspect, 0.1, 150)
      this.setRenderer();    // Three.js WebGLRenderer + EffectComposer
      this.setResources();   // GLTF/texture loader with progress events

      this.time.on("tick", () => this.update());
    });

    this.sizes.on("resize", () => this.resize());
  }

  update() {
    this.controls?.update();
    this.nightMode?.update();
    this.camera.update();
    this.world?.update();
    this.matcaps?.update();
    this.renderer.update();
    this.focusPoints?.update();
  }
}
```

### 2. Dual-Canvas Architecture

**Canvas 1: WebGL (Three.js)** -- The 3D hero scene (isometric city/office diorama):
```js
// Created by the Renderer class:
this.instance = new THREE.WebGLRenderer({ alpha: true, antialias: true });
this.targetElement.appendChild(this.instance.domElement);
// Positioned absolute, fills the hero section
```

**Canvas 2: Canvas 2D** -- Page transition overlay:
```html
<!-- Vue component renders a fixed-position canvas -->
<canvas class="a-transitionCanvas" data-v-5957afc6></canvas>

<style>
.a-transitionCanvas {
  position: fixed;
  z-index: 999;
  top: 0; left: 0; right: 0; bottom: 0;
  width: 100vw;
  height: 100vh;
  pointer-events: none;
}
</style>
```

The Canvas 2D app:
```js
// Simple 2D rendering framework:
class CanvasApp {
  constructor({ canvasEl, width, height }) {
    this.canvas = canvasEl;
    this.context = this.canvas.getContext("2d");
    this.children = [];
    this.resize(width, height);
  }

  render() {
    this.context.clearRect(0, 0, this.width, this.height);
    for (let i = 0; i < this.children.length; i++) {
      this.children[i].render();
    }
  }
}
```

### 3. CSS Custom Property Mouse Parallax

The "Discover All Games" section uses CSS custom properties driven by JavaScript mouse tracking:

```js
// Vue component methods:
handleMouseMove(event) {
  this.cursor.x = event.clientX;
  this.cursor.y = event.clientY;

  // Normalize to -0.5..+0.5 range within container bounds
  const normalizedX = (this.cursor.x - this.bounds.x) / this.bounds.width - 0.5;
  const normalizedY = (this.cursor.y + window.scrollY - this.offsetTop) / this.bounds.height - 0.5;

  // Set CSS custom properties on the element
  this.$el.style.setProperty("--offsetX", `${5 * normalizedX}px`);
  this.$el.style.setProperty("--offsetY", `${5 * normalizedY}px`);
  this.$el.style.setProperty("--rotateX", `${-10 * normalizedY}deg`);
  this.$el.style.setProperty("--rotateY", `${5 * normalizedX}deg`);
}

// Mounted:
mounted() {
  this.cursor = { x: 0, y: 0 };
  this.computeBounds();
}

computeBounds() {
  this.$nextTick(() => {
    this.bounds = this.$refs.container.getBoundingClientRect();
    this.offsetTop = this.$refs.container.offsetTop;
  });
}
```

The CSS consumes these properties with per-element `--multiplier` values:

```css
/* Container initializes defaults */
.o-homeDiscoverAllGames {
  --offsetX: 0px;
  --offsetY: 0px;
}

/* Images use --multiplier for depth-based parallax */
.o-homeDiscoverAllGames__images__img.-topLeft img,
.o-homeDiscoverAllGames__images__img.-topRight img,
.o-homeDiscoverAllGames__images__img.-bottomLeft img,
.o-homeDiscoverAllGames__images__img.-bottomRight img {
  transition: transform calc(0.2s * var(--multiplier)) cubic-bezier(.215, .61, .355, 1) !important;
  transform: translate3d(
    calc(var(--multiplier) * var(--offsetX)),
    calc(var(--multiplier) * var(--offsetY)),
    0
  );
}

/* Each corner has a different multiplier for depth effect */
.o-homeDiscoverAllGames__images__img.-topLeft    { --multiplier: 4; }
.o-homeDiscoverAllGames__images__img.-topRight   { --multiplier: 3; }
.o-homeDiscoverAllGames__images__img.-bottomLeft { --multiplier: 1.2; }
.o-homeDiscoverAllGames__images__img.-bottomRight { --multiplier: 0.8; }

/* Title has 3D perspective rotation */
.o-homeDiscoverAllGames__title {
  perspective: 1500px;
}

.o-homeDiscoverAllGames__title h2 {
  display: inline-block;
  transform-style: preserve-3d;
  --multiplier: 0.7;
  transform: translate3d(
    calc(var(--multiplier) * var(--offsetX)),
    calc(var(--multiplier) * var(--offsetY)),
    0
  ) rotateX(var(--rotateX)) rotateY(var(--rotateY));
  transition: transform .4s cubic-bezier(.215, .61, .355, 1);
}

/* CTA button also gets parallax + rotation */
.o-homeDiscoverAllGames__ctaContainer {
  --multiplier: 0.5;
  transform-style: preserve-3d;
  transform: translate3d(
    calc(var(--multiplier) * var(--offsetX)),
    calc(var(--multiplier) * var(--offsetY)),
    5px
  ) rotateX(var(--rotateX)) rotateY(var(--rotateY));
  transition: transform .5s cubic-bezier(.215, .61, .355, 1);
}
```

### 4. Stagger Animation via --stagger-index CSS Variable

The mobile menu uses a CSS-driven stagger pattern:

```js
// In the Vue component's mounted hook:
mounted() {
  this.$el.querySelectorAll(".stagger").forEach((el, index) => {
    el.style.setProperty("--stagger-index", index);
  });
}
```

The CSS uses `calc()` to derive delay from the index:

```css
/* Default state: slid off-screen */
.o-mobileMenu .stagger {
  opacity: 0;
  transform: translateX(50px);
  transition: all .4s cubic-bezier(.215, .61, .355, 1);
}

/* Active state: staggered entry */
.o-mobileMenu.-active .stagger {
  opacity: 1;
  transform: translateX(0);
  transition: all .4s cubic-bezier(.215, .61, .355, 1)
    calc(.5s + var(--stagger-index) * 0.04s);
  /* ^ base delay 0.5s + 40ms per item index */
}
```

The template marks elements with the `stagger` class:
```html
<ul class="o-mobileMenu__mainNav">
  <li class="o-mobileMenu__mainNav__item stagger" v-for="item in nav">
    <a :href="item.href">{{ item.label }}</a>
  </li>
</ul>
<ul class="o-mobileMenu__secondaryNav stagger">...</ul>
<div class="stagger">...</div>
<div class="o-mobileMenu__offices__list__item stagger" v-for="office in offices">...</div>
<div class="o-mobileMenu__socials__item stagger" v-for="social in socials">...</div>
```

### 5. Page Transition: Canvas 2D Curved Wipe

The transition is a **curved gradient wipe** using quadratic bezier curves on Canvas 2D:

```js
class TransitionLayer {
  constructor(context, options) {
    this.ctx = context;
    this.enterProgress = 0;    // 0..1 wipe in
    this.bgOpacity = 0;        // White background overlay
    this.leaveProgress = 0;    // 0..1 wipe out

    // Gradient based on destination route
    this.gradient = context.createLinearGradient(0, 0, this.width, 0);
    this.gradient.addColorStop(0, "#ff3891");  // Default pink
    this.gradient.addColorStop(1, "#ffbf04");  // Default orange

    this.curveAmount = Math.min(window.innerWidth, window.innerHeight) / 2;
  }

  setGradient(colors) {
    this.gradient = this.ctx.createLinearGradient(0, 0, this.width, 0);
    this.gradient.addColorStop(0, colors[0]);
    this.gradient.addColorStop(1, colors[1]);
  }

  animateIn() {
    return gsap.fromTo(this, {
      enterProgress: 0,
      leaveProgress: 0,
      bgOpacity: 0
    }, {
      enterProgress: 1,
      bgOpacity: 0.8,
      duration: 0.55,
      ease: "power3.out"
    });
  }

  animateOut() {
    return gsap.fromTo(this, {
      leaveProgress: 0
    }, {
      leaveProgress: 1,
      bgOpacity: 0,
      duration: 0.6,
      overwrite: true,
      ease: "power3.in"
    });
  }

  draw() {
    // Semi-transparent white background
    this.context.save();
    this.context.fillStyle = "white";
    this.context.globalAlpha = this.bgOpacity;
    this.context.fillRect(this.x, this.y, this.width, this.height);
    this.context.restore();

    // Gradient-filled curved shape
    this.context.fillStyle = this.gradient;

    // Top edge: slides down from top (enter) with curve
    const topEdge = this.height * Math.pow(1 - this.enterProgress, 1.25);
    // Bottom edge: slides up from bottom (leave)
    const bottomEdge = this.height * (1 - this.leaveProgress);

    // Curve amounts
    const enterCurve = this.curveAmount * this.enterProgress;
    const leaveCurve = this.curveAmount * this.leaveProgress;

    // Draw the shape with curved top and bottom edges
    this.context.beginPath();
    this.context.moveTo(0, topEdge);
    this.context.quadraticCurveTo(this.width / 2, topEdge - enterCurve, this.width, topEdge);
    this.context.lineTo(this.width, bottomEdge);
    this.context.quadraticCurveTo(this.width / 2, bottomEdge - leaveCurve, 0, bottomEdge);
    this.context.lineTo(0, bottomEdge);
    this.context.fill();
  }
}

// Route-based gradient selection:
const gradientMap = {
  games: "bluePurple",        // #ad00ff -> #38b7ff
  story: "blueGreen",
  jobs: "purplePink",
  contact: "purpleTurquoise",
  default: "pinkOrange"       // #ff3891 -> #ffbf04
};

// Vue component wires it up:
mounted() {
  this.transitionApp = new TransitionApp({
    canvasEl: this.$el,
    width: window.innerWidth,
    height: window.innerHeight
  });

  this.$raf.subscribe(this.transitionApp.render); // rAF loop

  this.$nuxt.$on("pageLeave", () => {
    if (!this.isTransitioning) {
      this.transitionApp.setGradient(getGradientColors(this.currentGradient));
      this.transitionApp.animateIn();
      this.isTransitioning = true;
    }
  });

  this.$nuxt.$on("pageEnter", () => {
    if (this.isTransitioning) {
      this.transitionApp.animateOut();
      this.isTransitioning = false;
    }
  });
}
```

### 6. Shader Code in the Bundle

**Blur Post-Process Shader** (from the Renderer's EffectComposer):

```glsl
// VERTEX
varying vec2 vUv;
void main() {
    vUv = uv;
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}

// FRAGMENT -- Gaussian-like blur
#define M_PI 3.1415926535897932384626433832795
#define uBlurDirections 13.0
#define uIterations 3.0

uniform sampler2D tDiffuse;
uniform vec2 uResolution;
uniform float uBlurOffset;
uniform float uBlurStrength;

varying vec2 vUv;

vec4 blur(sampler2D image, vec2 uv, vec2 resolution, float strength) {
    const float Tau = 6.28318530718;
    vec2 radius = strength / resolution.xy;
    vec4 color = texture2D(image, uv);

    float iterations = 0.0;
    for(float d = 0.0; d < Tau; d += Tau / uBlurDirections) {
        for(float i = 1.0 / uIterations; i <= 1.0; i += 1.0 / uIterations) {
            color += texture2D(image, uv + vec2(cos(d), sin(d)) * radius * i);
            iterations += 1.0;
        }
    }

    color /= (iterations + 1.0);
    return color;
}

void main() {
    // Blur strength increases toward top/bottom edges
    float blurStrength = abs(vUv.y - 0.5) + uBlurOffset;
    blurStrength *= uBlurStrength;
    blurStrength = max(0.0, blurStrength);
    blurStrength = pow(blurStrength, 2.0);

    if(blurStrength > 0.0) {
        gl_FragColor = blur(tDiffuse, vUv, uResolution, blurStrength);
    } else {
        gl_FragColor = texture2D(tDiffuse, vUv);
    }
}
```

**Overlay/Glow Shader** (full-screen quad for vignette):

```glsl
// VERTEX
varying vec2 vUv;
void main() {
    vUv = uv;
    gl_Position = vec4(position, 1.0);  // No projection -- NDC space quad
}

// FRAGMENT
uniform vec3 uColor;
uniform vec2 uPosition;
uniform vec2 uScale;
uniform float uAlpha;

varying vec2 vUv;

void main() {
    vec2 uv = vUv;
    uv -= uPosition;
    uv /= uScale;
    uv += uPosition;
    float strength = distance(uv, uPosition);
    strength = smoothstep(0.0, 1.0, strength);
    strength *= uAlpha;

    gl_FragColor = vec4(uColor, strength);

    #include <tonemapping_fragment>
    #include <encodings_fragment>
}
```

### 7. 3D Camera Controls (Custom, NOT OrbitControls)

Madbox built a custom camera control system with drag, parallax, zoom, and focus:

```js
class Controls extends EventEmitter {
  constructor() {
    // Separate sub-systems
    this.controlsDrag = new ControlsDrag();      // Mouse/touch drag
    this.controlsPosition = new ControlsPosition(); // Mouse position tracking
    this.controlsWheel = new ControlsWheel();      // Scroll wheel
    this.controlsTouchClick = new ControlsTouchClick();

    this.setMap();        // Drag-to-pan mapping
    this.setParallax();   // Mouse hover parallax
    this.setRotation();   // Right-click rotation
    this.setZoom();       // Scroll wheel zoom
    this.setFocus();      // Animated camera focus
    this.setImmersion();  // Full immersive mode
    this.setCursor();     // grab/grabbing cursor
  }

  setMap() {
    this.map = {
      multiplier: 15,
      x: 0, y: 0,
      eased: { multiplier: 0.005, x: 0, y: 0 },
      limits: {
        x: { min: -23, max: 23 },
        y: { min: -3, max: 15 }
      }
    };
  }

  setParallax() {
    this.parallax = {
      multiplier: 1,
      x: 0, y: 0,
      eased: { multiplier: 0.005, x: 0, y: 0 }
    };
  }

  setRotation() {
    this.rotation = {
      base: new THREE.Euler(),
      multiplier: 1,
      x: 0, y: 0,
      eased: { multiplier: 0.005, x: 0, y: 0 },
      limits: {
        x: { min: -0.4, max: 0.4 },
        y: { min: -0.2, max: 0.4 }
      }
    };
    this.rotation.base.reorder("YXZ");
    this.rotation.base.y = 0.5 * Math.PI;
    this.rotation.base.x = 0.15 * -Math.PI;
  }

  update() {
    // Update sub-controllers
    this.controlsDrag.update();
    this.controlsPosition.update();
    this.controlsWheel.update();

    // Drag -> map position (left-click or touch)
    if (this.canInteract && (this.config.touch || this.controlsDrag.button === "left")) {
      this.map.x += this.controlsDrag.delta.normalized.x * this.map.multiplier;
      this.map.y += this.controlsDrag.delta.normalized.y * this.map.multiplier;
    }

    // Clamp and ease
    this.map.x = clamp(this.map.x, this.map.limits.x.min, this.map.limits.x.max);
    this.map.y = clamp(this.map.y, this.map.limits.y.min, this.map.limits.y.max);
    this.map.eased.x += (this.map.x - this.map.eased.x) * this.map.eased.multiplier * this.time.delta;
    this.map.eased.y += (this.map.y - this.map.eased.y) * this.map.eased.multiplier * this.time.delta;

    // Mouse position -> parallax (desktop only)
    if (!this.config.touch) {
      this.parallax.x = this.controlsPosition.normalized.x * this.parallax.multiplier;
      this.parallax.y = this.controlsPosition.normalized.y * this.parallax.multiplier;
      this.parallax.eased.x += (this.parallax.x - this.parallax.eased.x)
        * this.parallax.eased.multiplier * this.time.delta;
      this.parallax.eased.y += (this.parallax.y - this.parallax.eased.y)
        * this.parallax.eased.multiplier * this.time.delta;
    }

    // Apply to camera
    this.camera.position.copy(this.basePosition);
    this.camera.position.z += this.map.eased.x;
    this.camera.position.x -= this.map.eased.y;

    // Zoom (in immersion mode)
    const zoomOffset = this.zoom.direction.clone().multiplyScalar(this.zoom.eased.value);
    this.camera.position.add(zoomOffset);

    // Focus point interpolation
    this.camera.position.lerp(this.focus.position, this.focus.mixStrength);

    // Parallax (local space translate)
    this.camera.translateX(this.parallax.eased.x);
    this.camera.translateY(-this.parallax.eased.y);

    // Rotation
    this.camera.rotation.copy(this.rotation.base);
    this.camera.rotation.x += (this.focus.rotation.x - this.camera.rotation.x) * this.focus.mixStrength;
    this.camera.rotation.y += (this.focus.rotation.y - this.camera.rotation.y) * this.focus.mixStrength;
    this.camera.rotation.y -= this.rotation.eased.x;
    this.camera.rotation.x -= this.rotation.eased.y;
  }

  // Cursor UX:
  setCursor() {
    this.controlsDrag.on("down", () => {
      if (this.canInteract) this.targetElement.style.cursor = "grabbing";
    });
    this.controlsDrag.on("up", () => {
      if (this.canInteract) this.targetElement.style.cursor = "grab";
    });
  }

  // Focus animation (GSAP-driven):
  focus.activate(pointId, duration = 1.4) {
    const point = this.focusPoints.items.find(p => p.id === pointId);
    if (this.focus.active) {
      gsap.to(this.focus.position, { duration, ease: "power2.inOut", ...point.cameraPosition });
      gsap.to(this.focus.rotation, { duration, ease: "power2.inOut", ...point.cameraRotation });
    } else {
      this.focus.position.copy(point.cameraPosition);
      this.focus.rotation.copy(point.cameraRotation);
      gsap.to(this.focus, { duration, ease: "power2.inOut", mixStrength: 1 });

      // Optional blur during focus
      gsap.delayedCall(0.5 * duration, () => {
        this.renderer.drawBlur = true;
        gsap.to(this.renderer.postProcess.finalPass.material.uniforms.uBlurStrength, {
          duration: 0.5 * duration, value: 15
        });
      });
    }
  }
}
```

---

## Key Architectural Differences

| Aspect | Cosmos Network | Madbox |
|--------|---------------|--------|
| Framework | Next.js 14 (App Router) | Nuxt.js 2 (Vue 2) |
| Three.js Version | r170+ (WebGPU build, NodeMaterial) | r0.118 (legacy) |
| 3D Content | Procedural globe (SphereGeometry) | GLTF scene (baked models + matcaps) |
| Post-Processing | Custom dither/halftone shader | Gaussian blur shader |
| Camera Controls | Three.js OrbitControls (locked) | Custom drag/parallax/zoom system |
| Page Transitions | None (SPA with Next.js) | Canvas 2D curved gradient wipe |
| CSS Animations | Tailwind utility classes | CSS custom properties + GSAP |
| Mouse Interaction | OrbitControls drag-to-rotate | CSS custom prop parallax + 3D drag |
| Globe Library | `three-globe` npm package | N/A (no globe) |
| Rendering | EffectComposer + multi-pass | EffectComposer + stencil layers |
