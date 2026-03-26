# Scroll-Driven 3D

Patterns for binding 3D content to scroll position -- the most common technique on award-winning 3D websites.

## GSAP ScrollTrigger + Three.js

The gold standard stack used by Superlist, SVZ Design, and most Awwwards winners.

### Basic Scroll-Bound Transform
```js
import gsap from 'gsap';
import { ScrollTrigger } from 'gsap/ScrollTrigger';
gsap.registerPlugin(ScrollTrigger);

// Rotate model based on scroll
gsap.to(model.rotation, {
  y: Math.PI * 2,
  scrollTrigger: {
    trigger: '#section-product',
    start: 'top bottom',
    end: 'bottom top',
    scrub: 2  // 2-second lag for smooth interpolation
  }
});

// Scale model based on scroll
gsap.fromTo(model.scale,
  { x: 0.5, y: 0.5, z: 0.5 },
  {
    x: 1, y: 1, z: 1,
    scrollTrigger: {
      trigger: '#section-reveal',
      start: 'top 80%',
      end: 'top 20%',
      scrub: 1
    }
  }
);
```

### Scroll-Driven Animation Playback (from Superlist)
Bind a GLTF's baked animation timeline to scroll position.

```js
const mixer = new THREE.AnimationMixer(model);
const action = mixer.clipAction(gltf.animations[0]);
action.play();
action.paused = true;

// Map scroll to animation time
const proxy = { time: 0 };
gsap.to(proxy, {
  time: action.getClip().duration,
  ease: 'none',
  scrollTrigger: {
    trigger: '#scroll-runway',
    start: 'top top',
    end: 'bottom bottom',
    scrub: 2,
    onUpdate: () => {
      action.time = proxy.time;
      mixer.update(0); // delta=0 since we set time directly
    }
  }
});
```

### Scroll-Driven Camera Path
```js
// Define camera positions along a path
const cameraPath = [
  { pos: [0, 0, 10], lookAt: [0, 0, 0] },   // start
  { pos: [5, 3, 5],  lookAt: [0, 1, 0] },    // mid
  { pos: [0, 8, 2],  lookAt: [0, 0, 0] },    // end
];

const progress = { value: 0 };
gsap.to(progress, {
  value: 1,
  scrollTrigger: {
    trigger: '#camera-section',
    start: 'top top',
    end: 'bottom bottom',
    scrub: 2
  },
  onUpdate: () => {
    const t = progress.value;
    const idx = Math.min(Math.floor(t * (cameraPath.length - 1)), cameraPath.length - 2);
    const localT = (t * (cameraPath.length - 1)) - idx;

    const from = cameraPath[idx];
    const to = cameraPath[idx + 1];

    camera.position.lerpVectors(
      new THREE.Vector3(...from.pos),
      new THREE.Vector3(...to.pos),
      localT
    );
    const lookTarget = new THREE.Vector3().lerpVectors(
      new THREE.Vector3(...from.lookAt),
      new THREE.Vector3(...to.lookAt),
      localT
    );
    camera.lookAt(lookTarget);
  }
});
```

## Image Sequence Animation (Apple/Zoox Style)

Pre-render 3D frames offline, then play them back on a canvas driven by scroll.

### Canvas Image Sequence
```js
const canvas = document.getElementById('sequence-canvas');
const ctx = canvas.getContext('2d');
const frameCount = 180;
const images = [];

// Preload all frames
for (let i = 0; i < frameCount; i++) {
  const img = new Image();
  img.src = `/frames/product-${String(i).padStart(4, '0')}.webp`;
  images.push(img);
}

// Draw frame based on scroll
function updateCanvas() {
  const scrollFraction = window.scrollY / (document.body.scrollHeight - window.innerHeight);
  const frameIndex = Math.min(frameCount - 1, Math.floor(scrollFraction * frameCount));

  if (images[frameIndex]?.complete) {
    canvas.width = canvas.offsetWidth * window.devicePixelRatio;
    canvas.height = canvas.offsetHeight * window.devicePixelRatio;
    ctx.scale(window.devicePixelRatio, window.devicePixelRatio);
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    ctx.drawImage(images[frameIndex], 0, 0, canvas.offsetWidth, canvas.offsetHeight);
  }
  requestAnimationFrame(updateCanvas);
}
updateCanvas();
```

### Generating Frames with FFmpeg
```bash
# Extract frames from a 3D render video
ffmpeg -i product_orbit.mp4 -vf "scale=1920:-1" -q:v 2 frames/product-%04d.webp

# Or from a Blender render:
# Set output to PNG sequence in Blender, then convert:
ffmpeg -i render_%04d.png -c:v libwebp -quality 85 frames/product-%04d.webp
```

## Sprite Sheet Animation (from Hanai World)

A single image contains all frames in a grid -- lighter than individual files.

```js
const canvas = document.getElementById('sprite-canvas');
const ctx = canvas.getContext('2d');
const spriteSheet = new Image();
spriteSheet.src = '/sprites/planet-sheet.webp';

const cols = 10;       // columns in sprite sheet
const rows = 12;       // rows
const totalFrames = cols * rows;
const frameW = 512;    // single frame width
const frameH = 512;    // single frame height

window.addEventListener('scroll', () => {
  const progress = window.scrollY / (document.body.scrollHeight - window.innerHeight);
  const frameIndex = Math.floor(progress * (totalFrames - 1));
  const col = frameIndex % cols;
  const row = Math.floor(frameIndex / cols);

  ctx.clearRect(0, 0, canvas.width, canvas.height);
  ctx.drawImage(
    spriteSheet,
    col * frameW, row * frameH, frameW, frameH,  // source
    0, 0, canvas.width, canvas.height              // destination
  );
});
```

## Alpha-Channel Video Overlay (from ChainGPT Labs)

Pre-rendered 3D with transparency overlaid on 2D content.

```html
<!-- WebM with alpha for Chrome/Firefox, HEVC for Safari -->
<video autoplay muted loop playsinline>
  <source src="/3d-character.webm" type="video/webm; codecs=vp9">
  <source src="/3d-character.mp4" type="video/mp4; codecs=hvc1">
</video>
```

```bash
# FFmpeg: render with alpha channel to WebM
ffmpeg -i render_with_alpha.mov \
  -c:v libvpx-vp9 \
  -pix_fmt yuva420p \
  -deadline best \
  -crf 32 \
  -b:v 1700k \
  -an \
  output.webm

# For Safari (HEVC with alpha): use Apple Compressor
```

```css
video {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  object-fit: contain;
  pointer-events: none;
  mix-blend-mode: normal; /* alpha handles transparency */
}
```

## Lenis Smooth Scroll + Three.js

The standard smooth scroll library paired with GSAP.

```js
import Lenis from '@studio-freight/lenis';

const lenis = new Lenis({
  duration: 1.2,
  easing: (t) => Math.min(1, 1.001 - Math.pow(2, -10 * t)),
  smoothWheel: true,
  smoothTouch: false  // native scroll on mobile
});

// Sync with GSAP ScrollTrigger
lenis.on('scroll', ScrollTrigger.update);
gsap.ticker.add((time) => lenis.raf(time * 1000));
gsap.ticker.lagSmoothing(0);

// Sync with Three.js animation loop
function animate(time) {
  lenis.raf(time);
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
requestAnimationFrame(animate);
```

## Spline + Scroll Binding

### Spline Viewer with Scroll Events
```html
<spline-viewer
  url="https://prod.spline.design/SCENE_ID/scene.splinecode"
  loading-anim
></spline-viewer>
```

```js
// Access Spline runtime for scroll binding
const viewer = document.querySelector('spline-viewer');
viewer.addEventListener('load', (e) => {
  const app = e.target.spline;

  window.addEventListener('scroll', () => {
    const progress = window.scrollY / (document.body.scrollHeight - window.innerHeight);
    // Rotate object by name
    const obj = app.findObjectByName('Hero Model');
    if (obj) {
      obj.rotation.y = progress * Math.PI * 2;
      obj.position.y = progress * -5;
    }
  });
});
```

---

## Case Study: Rose Island (rose-island.co)

**Stack:** Three.js r138 + GSAP 3.10.4 + Webflow CMS + Netlify-hosted JS bundle

Rose Island is a scroll-driven 3D storytelling site about a micronation in the Adriatic Sea.
The 3D scene (GLB model of the island platform on procedural ocean water) is driven through
a camera path by GSAP ScrollTrigger as the user scrolls through the narrative.

### Architecture: Webflow Layout + External 3D Engine

The key architectural insight is the separation of concerns:
- **Webflow** provides the HTML layout, CMS content, responsive CSS, and hosting
- **Netlify** hosts the external 3D engine bundle (isla-rosa.netlify.app/main.js)
- The bundle is a Vite build containing GSAP, Three.js, GLTFLoader, Water shader, and Sky

```html
<!-- Webflow w-embed pattern: embed Three.js canvas inside CMS page -->
<div class="canvas__embed w-embed">
  <canvas class="webgl"></canvas>
</div>

<!-- External 3D engine loaded from Netlify -->
<script src="https://isla-rosa.netlify.app/main.js" type="module"></script>
```

The `.canvas__embed` div uses `w-embed` (Webflow's custom code embed block) to
inject a raw `<canvas>` element that Webflow's visual editor cannot create natively.
On mobile (max-width: 768px), `pointer-events: none` is applied to the canvas
so touch scrolling passes through to the page.

### GLB Model Loading: The .glb.txt Workaround

Webflow restricts file uploads to known types and does not allow `.glb` files.
Rose Island works around this by renaming the file to `.glb.txt` and uploading
it to Webflow's CDN as a "text" file. The GLTFLoader does not care about the
file extension -- it parses the binary content regardless.

```js
// Webflow CDN serves the GLB disguised as .txt
const gltfLoader = new GLTFLoader(loadingManager);
gltfLoader.load(
  'https://uploads-ssl.webflow.com/.../model.glb.txt',
  (gltf) => {
    // Assign custom PBR materials by mesh name
    const pontons = gltf.scene.children.find(c => c.name === 'pontons');
    pontons.material = new MeshStandardMaterial({ map: bakedPontonsMap, roughness: roughnessMap, metalness: 1 });

    const base = gltf.scene.children.find(c => c.name === 'base');
    base.material = new MeshStandardMaterial({ map: bakedBaseMap, normalMap: baseNormal });

    const main = gltf.scene.children.find(c => c.name === 'main');
    main.material = new MeshStandardMaterial({ map: bakedMainMap });

    const secondary = gltf.scene.children.find(c => c.name === 'secondary');
    secondary.material = new MeshStandardMaterial({ map: bakedSecondaryMap });

    const curtains = gltf.scene.children.find(c => c.name === 'curtains');
    curtains.material = new MeshStandardMaterial({ color: '#46423F' });

    scene.add(gltf.scene);
  }
);
```

All textures are also hosted on Webflow's CDN as standard image files (JPG).
The model uses 5 named mesh parts, each with baked textures created offline
in Blender. This is much cheaper than realtime PBR with multiple lights.

### Procedural Ocean with Three.js Water Shader

The island sits on a procedural water plane using Three.js's built-in Water class:

```js
const waterGeometry = new PlaneGeometry(15000, 15000);
const water = new Water(waterGeometry, {
  textureWidth: 512,
  textureHeight: 512,
  waterNormals: new TextureLoader().load('/waternormal.jpeg', (tex) => {
    tex.wrapS = tex.wrapT = RepeatWrapping;
  }),
  sunDirection: new Vector3(),
  sunColor: '#EFC27A',
  waterColor: 0x007587,
  distortionScale: 5,
  fog: scene.fog !== undefined
});
water.rotation.x = -Math.PI / 2;
water.position.y = -3.1;
water.material.uniforms.size.value = 15;
scene.add(water);

// Animate water in render loop (not scroll-driven -- always moves)
function animate() {
  water.material.uniforms.time.value += 1 / 110;
  camera.lookAt(0, 0, 0);
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
```

### Time-of-Day Dynamic Sky

The Sky shader adapts to the user's local time (Europe/Madrid timezone displayed):

```js
const sky = new Sky();
sky.scale.setScalar(10000);
scene.add(sky);

const hour = new Date().getHours();
let skyParams = {};

if (hour >= 0 && hour <= 5) {
  skyParams = { elevation: 10, azimuth: -180, turbidity: 0, rayleigh: 0.02,
                mieCoefficient: 1.4, mieDirectionalG: 0.27 };
} else if (hour >= 6 && hour <= 11) {
  skyParams = { elevation: 8.9, azimuth: -33, turbidity: 0.09, rayleigh: 0.47,
                mieCoefficient: 1.4, mieDirectionalG: 0.27 };
} else if (hour >= 12 && hour <= 20) {
  skyParams = { elevation: 170, azimuth: -126, turbidity: 0.19, rayleigh: 1.5,
                mieCoefficient: 1.4, mieDirectionalG: 0.27 };
} else {
  skyParams = { elevation: 3.1, azimuth: -131, turbidity: 0.04, rayleigh: 2,
                mieCoefficient: 10, mieDirectionalG: 0 };
}

// Apply sky params and generate environment map for PBR reflections
const pmremGenerator = new PMREMGenerator(renderer);
const sunPosition = new Vector3();
const phi = MathUtils.degToRad(90 - skyParams.elevation);
const theta = MathUtils.degToRad(skyParams.azimuth);
sunPosition.setFromSphericalCoords(1, phi, theta);
sky.material.uniforms.sunPosition.value.copy(sunPosition);
water.material.uniforms.sunDirection.value.copy(sunPosition).normalize();

const envMap = pmremGenerator.fromScene(sky);
scene.environment = envMap.texture;
```

### GSAP ScrollTrigger Camera Drive + Pin

The core scroll interaction uses GSAP matchMedia for responsive behavior.
The camera starts at position (27, 6, 58) looking at origin, and scrolls
to (-15, -2, 40) as the hero section passes. The canvas itself is pinned
during scroll using ScrollTrigger's `pin` option.

```js
// Responsive scroll triggers -- desktop only
ScrollTrigger.matchMedia({
  '(min-width: 768px)': function() {
    // 1. Drive camera position from hero scroll
    gsap.to(camera.position, {
      x: -15,
      y: -2,
      z: 40,
      immediateRender: false,
      scrollTrigger: {
        trigger: '.section--hero',
        start: 'top top',
        end: 'bottom top',
        scrub: true         // direct 1:1 scroll binding, no lag
      }
    });

    // 2. Pin the canvas in place while hero section scrolls
    gsap.to('.canvas__embed', {
      ease: 'none',
      scrollTrigger: {
        trigger: '.section--hero',
        start: 'top top',
        end: 'bottom bottom',
        scrub: true,
        pin: '.canvas__embed'  // canvas stays fixed while content scrolls
      }
    });

    // 3. Slide canvas out when narrative sections begin
    gsap.to('.canvas__embed', {
      yPercent: 100,           // slide down off screen
      ease: 'none',
      scrollTrigger: {
        trigger: '.section__wrapper',
        start: 'top bottom',
        end: 'top top',
        scrub: true
      }
    });
  },
  '(max-width: 767px)': function() {
    // Mobile: no scroll-driven camera, static view
  }
});
```

The camera path pattern here is simple (single gsap.to) because the camera always
looks at origin via `camera.lookAt(0, 0, 0)` in the render loop. For more complex
multi-waypoint paths, see the camera path pattern above.

### Mouse Parallax on Camera Rig

A subtle parallax effect adds life. The camera is parented to a Group, and mouse
movement rotates that group slightly. This creates a gentle floating feel without
fighting the scroll-driven position.

```js
const cameraGroup = new Group();
cameraGroup.add(camera);
scene.add(cameraGroup);

let mouseX = 0, mouseY = 0;
let targetRotY = 0, targetRotZ = 0;
const halfW = window.innerWidth / 2;
const halfH = window.innerHeight / 2;

document.addEventListener('mousemove', (e) => {
  mouseX = e.clientX - halfW;
  mouseY = e.clientY - halfH;
});

function animate() {
  targetRotY = mouseX * 0.0001;
  targetRotZ = mouseY * 0.0001;
  // Smooth interpolation (0.04 = ~4% per frame)
  cameraGroup.rotation.y += 0.04 * (-targetRotY - cameraGroup.rotation.y);
  cameraGroup.rotation.z += 0.04 * (-targetRotZ - cameraGroup.rotation.z);

  water.material.uniforms.time.value += 1 / 110;
  camera.lookAt(0, 0, 0);
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
```

### Loading Orchestration

A LoadingManager tracks all asset progress and orchestrates the intro animation:

```js
const loadingManager = new LoadingManager(
  // onLoad: all assets ready
  () => {
    gsap.delayedCall(0.5, playIntroAnimation);
    gsap.delayedCall(5, () => {
      document.body.classList.remove('disable-scroll'); // unlock scrolling
    });
  },
  // onProgress: update loading bar
  (url, loaded, total) => {
    const progress = loaded / total;
    loadingBar.style.transform = `scaleX(${progress})`;
  }
);

function playIntroAnimation() {
  window.scrollTo(0, 0);
  camera.position.set(27, 6, 58); // final position

  gsap.timeline()
    .from(camera.position, {
      x: 27, y: 700, z: 1250,    // start far away, zoom in
      duration: 4,
      ease: 'Quart.easeInOut'
    }, 0)
    .to('.loading__columns', {
      yPercent: -100,
      duration: 1.2,
      ease: 'Quart.easeInOut',
      stagger: { from: 'random', amount: 0.6 }
    }, 0)
    .from('.header__letters', {
      yPercent: 25, duration: 0.8, stagger: 0.1,
      opacity: 0, ease: 'Quart.easeInOut'
    }, 2.5);
}
```

---

## Case Study: J-Vers (j-vers.com)

**Stack:** Next.js App Router + React Three Fiber (R3F) + Three.js + GSAP + Lenis + Zustand

J-Vers is NOT using OGL with 5 raw WebGL2 canvases. It uses a single React Three Fiber
canvas via a "tunnel" architecture where multiple React component trees render into one
shared WebGL context. The visual richness comes from Three.js ShaderMaterials, a cursor
touch-trail texture, Simplex noise, and GSAP scroll-driven DOM animations with parallax.

### The WebGL Tunnel Architecture

Instead of multiple canvases (which would waste GPU memory), J-Vers uses a React "tunnel"
pattern. A single R3F `<Canvas>` renders all 3D content, while React portals (tunnels)
allow components anywhere in the DOM tree to inject 3D meshes into that single scene.

```jsx
// WebGLProvider.jsx -- wraps the app, provides a single shared canvas
import { createContext, useState, useEffect, useContext } from 'react';
import dynamic from 'next/dynamic';
import tunnel from 'tunnel-rat'; // React portal tunnels

// Lazy-load the canvas (SSR-safe)
const WebGLCanvas = dynamic(() =>
  import('./WebGLCanvas').then(m => m.WebGLCanvas),
  { ssr: false }
);

const WebGLContext = createContext({});

export function WebGLProvider({ children, root = true, force = true }) {
  const [WebGLTunnel] = useState(() => tunnel());  // 3D content tunnel
  const [DOMTunnel] = useState(() => tunnel());     // HTML overlay tunnel

  return (
    <WebGLContext.Provider value={{ WebGLTunnel, DOMTunnel }}>
      <WebGLCanvas />
      {children}
    </WebGLContext.Provider>
  );
}

export function useWebGL() {
  return useContext(WebGLContext);
}
```

```jsx
// WebGLCanvas.jsx -- the single R3F canvas
import { Canvas, useFrame, useThree } from '@react-three/fiber';
import { useWebGL } from './WebGLProvider';

function RenderLoop({ render = true }) {
  const { advance } = useThree();
  // Lenis-synced render: use requestAnimationFrame callback
  useAnimationFrame((time) => {
    if (render) advance(time / 1000);
  });
  return null;
}

export function WebGLCanvas({ render = true }) {
  const { WebGLTunnel, DOMTunnel } = useWebGL();

  return (
    <div className="webgl">
      <Canvas
        gl={{ precision: 'highp', powerPreference: 'high-performance',
              antialias: true, alpha: true }}
        dpr={[1, 2]}
        orthographic
        camera={{ position: [0, 0, 5000], near: 0.001, far: 10000, zoom: 1 }}
        frameloop="never"        // manual frame advance, synced with Lenis
        flat
        eventSource={document.documentElement}
        eventPrefix="client"
      >
        <RenderLoop render={render} />
        <WebGLTunnel.Out />      {/* 3D content from anywhere renders here */}
      </Canvas>
      <DOMTunnel.Out />          {/* HTML overlays */}
    </div>
  );
}
```

Key design decisions:
- `orthographic` camera with `zoom: 1` -- treats 3D scene like 2D layers with depth
- `frameloop="never"` -- no wasted frames; only renders when Lenis scroll ticks
- `flat` -- no tone mapping, shader colors map 1:1 to screen
- Single canvas, components inject via tunnels from anywhere in React tree

### Tunnel Usage: Any Component Can Inject 3D

```jsx
// ParallaxImages.jsx -- a section component that injects a 3D mesh
import { useWebGL } from './WebGLProvider';
import { useRect } from './hooks/useRect';
import { TransformContext } from './TransformContext';

function ParallaxImages({ className, ...props }) {
  const [ref, rect] = useRect();  // track DOM element position/size

  return (
    <div ref={ref} className={className}>
      <WebGLPortal>
        <ImageMesh rect={rect} {...props} />
      </WebGLPortal>
    </div>
  );
}

// WebGLPortal: renders children into the shared canvas via tunnel
function WebGLPortal({ children }) {
  const { WebGLTunnel } = useWebGL();
  const TransformProvider = useContext(TransformContext);
  const id = useId();

  if (WebGLTunnel) {
    return (
      <WebGLTunnel.In>
        <TransformProvider key={id}>
          {children}
        </TransformProvider>
      </WebGLTunnel.In>
    );
  }
  return null;
}
```

### Custom ShaderMaterial Factory

J-Vers uses a factory function to create typed ShaderMaterial classes with
getter/setter uniforms:

```js
import { ShaderMaterial, UniformsUtils, MathUtils } from 'three';

function createShaderMaterial(defaults, vertexShader, fragmentShader, init) {
  const MaterialClass = class extends ShaderMaterial {
    constructor(overrides = {}) {
      const entries = Object.entries(defaults);
      super({
        uniforms: entries.reduce((acc, [key, value]) => {
          const uniform = UniformsUtils.clone({ [key]: { value } });
          return { ...acc, ...uniform };
        }, {}),
        vertexShader,
        fragmentShader
      });

      this.key = '';

      // Create getters/setters for each uniform
      // so you can do: material.uProgress = 0.5
      // instead of: material.uniforms.uProgress.value = 0.5
      entries.forEach(([key]) => {
        Object.defineProperty(this, key, {
          get: () => this.uniforms[key].value,
          set: (v) => { this.uniforms[key].value = v; }
        });
      });

      Object.assign(this, overrides);
      if (init) init(this);
    }
  };

  MaterialClass.key = MathUtils.generateUUID();
  return MaterialClass;
}
```

### Cursor Touch-Trail Texture

The interactive cursor effect creates a 2D canvas trail texture that is used
as a displacement/effect uniform in shaders:

```js
import { Texture } from 'three';

class TouchTexture {
  constructor({
    size = 256, maxAge = 750, radius = 0.3, intensity = 0.2,
    interpolate = 0, smoothing = 0, minForce = 0.3, blend = 'screen',
    ease = t => Math.sqrt(1 - Math.pow(t - 1, 2))
  } = {}) {
    this.size = size;
    this.maxAge = maxAge;
    this.radius = radius;
    this.intensity = intensity;
    this.ease = ease;
    this.trail = [];
    this.force = 0;

    // Create offscreen canvas for trail rendering
    this.canvas = document.createElement('canvas');
    this.canvas.width = this.canvas.height = size;
    this.ctx = this.canvas.getContext('2d');
    this.ctx.fillStyle = 'black';
    this.ctx.fillRect(0, 0, size, size);
    this.texture = new Texture(this.canvas);
  }

  update(delta) {
    this.clear();
    this.trail.forEach((point, i) => {
      point.age += 1000 * delta;
      if (point.age > this.maxAge) this.trail.splice(i, 1);
    });
    this.trail.forEach(point => this.drawTouch(point));
    this.texture.needsUpdate = true;
  }

  addTouch(uv) {
    // uv comes from raycaster hit on a plane mesh
    this.trail.push({ x: uv.x, y: uv.y, age: 0, force: this.force });
  }

  drawTouch(point) {
    const pos = { x: point.x * this.size, y: (1 - point.y) * this.size };
    let intensity = point.age < 0.3 * this.maxAge
      ? this.ease(point.age / (0.3 * this.maxAge))
      : this.ease(1 - (point.age - 0.3 * this.maxAge) / (0.7 * this.maxAge));
    intensity *= point.force;

    const radius = this.size * this.radius * intensity;
    this.ctx.globalCompositeOperation = this.blend;
    const gradient = this.ctx.createRadialGradient(
      pos.x, pos.y, Math.max(0, 0.25 * radius),
      pos.x, pos.y, Math.max(0, radius)
    );
    gradient.addColorStop(0, `rgba(255, 255, 255, ${this.intensity})`);
    gradient.addColorStop(1, 'rgba(0, 0, 0, 0.0)');
    this.ctx.beginPath();
    this.ctx.fillStyle = gradient;
    this.ctx.arc(pos.x, pos.y, Math.max(0, radius), 0, Math.PI * 2);
    this.ctx.fill();
  }
}

// React hook wrapper
function useTouchTexture(options = {}) {
  const texture = useMemo(() => new TouchTexture(options), []);
  useFrame((_, delta) => texture.update(delta));
  const onPointerMove = useCallback((e) => texture.addTouch(e.uv), [texture]);
  return [texture.texture, onPointerMove];
}
```

### Simplex Noise for Organic Motion

Used for procedural distortion in shaders:

```js
// 2D Simplex noise generator (from the site's module 6669)
function createNoise2D(random = Math.random) {
  const perm = new Uint8Array(512);
  for (let i = 0; i < 256; i++) perm[i] = i;
  for (let i = 0; i < 255; i++) {
    const j = i + ~~(random() * (256 - i));
    [perm[i], perm[j]] = [perm[j], perm[i]];
  }
  for (let i = 256; i < 512; i++) perm[i] = perm[i - 256];

  const grad2 = new Float64Array([1,1,-1,1,1,-1,-1,-1,1,0,-1,0,1,0,-1,0,0,1,0,-1,0,1,0,-1]);
  const gradX = new Float64Array(perm).map(v => grad2[v % 12 * 2]);
  const gradY = new Float64Array(perm).map(v => grad2[v % 12 * 2 + 1]);

  return function noise2D(x, y) {
    // Standard 2D simplex noise implementation
    // Returns value in [-1, 1] range
    const F2 = 0.5 * (Math.sqrt(3) - 1);
    const G2 = (3 - Math.sqrt(3)) / 6;
    // ... (standard simplex algorithm)
  };
}
```

### GSAP Scroll Animations (Hero Section)

The hero section uses GSAP ScrollTrigger for parallax and reveal animations,
while the 3D canvas renders independently:

```jsx
useEffect(() => {
  const ctx = gsap.context(() => {
    // 1. Hero background parallax (50% speed)
    if (heroBackgroundRef.current) {
      gsap.fromTo(heroBackgroundRef.current,
        { yPercent: 0 },
        {
          yPercent: 50,
          ease: 'none',
          scrollTrigger: {
            trigger: sectionRef.current,
            start: 'top top',
            end: '+=100%',
            scrub: true
          }
        }
      );
    }

    // 2. Dark overlay fades in as you scroll past hero
    gsap.fromTo(overlayRef.current,
      { opacity: 0 },
      {
        opacity: 0.4,
        ease: 'none',
        scrollTrigger: {
          trigger: sectionRef.current,
          start: 'top top',
          end: 'bottom top',
          scrub: true
        }
      }
    );
  });

  return () => ctx.revert(); // cleanup on unmount
}, []);
```

### Sticky Card Stack Pattern

Cards stack on top of each other as the user scrolls, using CSS sticky positioning
combined with CSS custom properties for offset:

```jsx
function StickyCardStack({ cards }) {
  const cardCount = cards.length - 1;
  const spacing = 7; // rem

  return (
    <section style={{ '--pt': `${cardCount * spacing}rem` }}>
      {cards.map((card, index) => {
        const offset = cardCount * spacing - index * spacing;
        return (
          <div
            key={card.id}
            className="sticky-card-holder"
            style={{ '--top': `${cardCount * spacing + 6}rem` }}
          >
            <div
              className="sticky-card-block"
              style={{ '--offset': `${-offset}rem` }}
            >
              <CardContent card={card} index={index + 1} />
            </div>
          </div>
        );
      })}
    </section>
  );
}
```

```css
.sticky-card-holder {
  position: sticky;
  top: var(--top);
}

.sticky-card-block {
  transform: translateY(var(--offset));
}
```

### Video Hero with Loop Handoff

The hero plays a one-shot intro video, then seamlessly switches to a looping video:

```jsx
function HeroVideo({ media, loopMedia }) {
  const mainRef = useRef(null);
  const loopRef = useRef(null);

  useEffect(() => {
    const main = mainRef.current;
    const loop = loopRef.current;
    if (!main || !loop) return;

    main.play().catch(console.error);

    const onEnded = () => {
      loop.play().catch(console.error);
      loop.style.visibility = 'visible';
      main.style.visibility = 'hidden';
    };

    main.addEventListener('ended', onEnded);
    return () => main.removeEventListener('ended', onEnded);
  }, []);

  return (
    <div className="w-full h-full">
      <video ref={mainRef} className="absolute inset-0 w-full h-full object-cover"
             src={media.url} muted playsInline />
      <video ref={loopRef} className="absolute inset-0 w-full h-full object-cover"
             src={loopMedia.url} loop muted playsInline style={{ visibility: 'hidden' }} />
    </div>
  );
}
```

### Color Themes for 3D Objects

The site defines color palettes per shape/section that feed into shader uniforms:

```js
const themes = {
  pink: {
    hemisphereGround: '#fbeef8', hemisphereTop: '#0040ff',
    materialBase: '#f1BDFF', emissive: '#df6bff', pointLight: '#f0f0ff'
  },
  whirl: {
    materialBase: '#f4e3fe', hemisphereGround: '#7860a9',
    hemisphereTop: '#a28ecd', emissive: '#faa', pointLight: '#f0f0ff'
  },
  blue: {
    materialBase: '#A3ACDA', emissive: '#000',
    pointLight: '#DCDFEB', hemisphereGround: '#fff', hemisphereTop: '#151C40'
  },
  star: {
    materialBase: '#2450ff',
    pointLightFront: '#6e73b4', pointLightBack: '#0055ff'
  },
  torus: {
    ambientLight: '#113adf', materialBase: '#7d8dcf',
    materialSecond: '#fff', emissive: '#fff',
    pointLightFront: '#646796', pointLightBack: '#c5a7fb'
  }
};
```

### Page Transitions with Async Leave/Enter

J-Vers uses a custom transition system that coordinates GSAP leave/enter
animations with Next.js route changes:

```jsx
const TransitionContext = createContext({ stage: 'none', navigate: () => {}, isReady: false });

function TransitionProvider({ children, leave, enter, auto = false }) {
  const router = useRouter();
  const pathname = usePathname();
  const [stage, setStage] = useState('none'); // 'none' | 'leaving' | 'entering'

  const navigate = useCallback(async (href, fromPath, method = 'push') => {
    setStage('leaving');
    const cleanup = await leave(() => router[method](href));
    // leave() runs GSAP exit animation, then calls the router navigate callback
  }, [leave, router]);

  useEffect(() => {
    if (stage === 'entering') {
      enter(() => setStage('none'));
    }
  }, [stage, enter]);

  // When pathname changes (route completed), trigger 'entering'
  useEffect(() => () => {
    if (stage === 'leaving') setStage('entering');
  }, [stage, pathname]);

  return (
    <TransitionContext.Provider value={{ stage, navigate, isReady: stage !== 'entering' }}>
      {children}
    </TransitionContext.Provider>
  );
}
```

---

## Key Contrasts: Rose Island vs J-Vers

| Aspect | Rose Island | J-Vers |
|--------|------------|--------|
| **Framework** | Webflow (static HTML) | Next.js App Router (React) |
| **3D Library** | Three.js vanilla | React Three Fiber + Three.js |
| **Canvas Count** | 1 raw canvas | 1 R3F canvas (tunnel pattern) |
| **Scroll Engine** | GSAP ScrollTrigger direct | Lenis + GSAP + manual R3F frame advance |
| **3D Content** | GLB model + Water shader + Sky | ShaderMaterials + touch texture + noise |
| **Scroll Binding** | gsap.to(camera.position) with scrub | GSAP for DOM, R3F useFrame for 3D |
| **Pin Strategy** | ScrollTrigger pin on canvas div | CSS sticky + Lenis smooth scroll |
| **Build** | Vite bundle on Netlify | Next.js webpack chunks |
| **Key Trick** | .glb.txt for Webflow CDN | tunnel-rat for single-canvas multi-component |
| **Camera** | Perspective, scroll-driven position | Orthographic, fixed (2D-layer approach) |
| **Video** | None | Hero intro + loop handoff |
