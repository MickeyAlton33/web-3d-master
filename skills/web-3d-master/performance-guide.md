# 3D Performance Guide

Optimization techniques from production 3D websites. The difference between a demo and a shipped product.

## The Performance Hierarchy

From cheapest to most expensive rendering approach:

1. **CSS 3D transforms** -- GPU composited, zero WebGL overhead
2. **Pre-rendered image sequences** -- No runtime rendering, just canvas drawImage
3. **Alpha-channel video** -- Hardware video decoder, no GPU shading
4. **Sprite-sheet animation** -- Single texture, frame lookup
5. **Three.js with matcaps** -- WebGL but no lighting calculations
6. **Three.js with baked lighting** -- Pre-computed, minimal shader work
7. **Three.js with real-time PBR** -- Environment maps, metalness/roughness
8. **Three.js with shadows** -- Shadow maps are expensive
9. **Three.js with post-processing** -- Extra render passes
10. **Three.js with physics** -- CPU-bound simulation each frame

**Always choose the lightest approach that achieves the visual goal.**

## Model Optimization Pipeline (from Superlist)

### 1. Polygon Reduction
```
Blender: Decimate modifier → ratio 0.3-0.5
Or: Houdini PolyReduce SOP with target face count
```

### 2. Invisible Face Culling (from Superlist's Houdini pipeline)
If the camera never sees the back of a model, delete those faces:
```
# Houdini: Delete faces not visible from camera position
# Conceptual approach in Blender:
# Select faces facing away from camera → Delete
# Saves 30-50% of geometry for fixed-viewpoint scenes
```

### 3. Skeletal vs Baked Animation
- **Baked:** Every vertex position stored per frame. Huge files.
- **Skeletal:** Bone transforms stored per frame. Tiny files, same visual result.
- Always use skeletal animation for web delivery.

### 4. Draco Compression
```bash
# Using gltf-pipeline
npx gltf-pipeline -i model.gltf -o model-draco.glb --draco.compressionLevel 7

# Or in Blender: Export as GLB → check "Draco Mesh Compression"
```

### 5. Texture Optimization
```bash
# Convert textures to WebP/AVIF
# Resize to power-of-2 dimensions (512, 1024, 2048)
# Use KTX2 for GPU-compressed textures:
npx gltf-transform ktx2 input.glb output.glb --slots "baseColorTexture"
```

## Runtime Performance

### Pixel Ratio Capping
```js
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
// Never render at 3x -- 4x the pixels of 1.5x for imperceptible quality gain
```

### Conditional Quality
```js
const isMobile = /Android|iPhone|iPad/i.test(navigator.userAgent);
const isLowEnd = navigator.hardwareConcurrency <= 4;

if (isMobile || isLowEnd) {
  renderer.setPixelRatio(1);
  // Reduce particle count
  // Disable post-processing
  // Use simpler materials
  // Fall back to image sequence instead of WebGL
}
```

### Dispose Properly
```js
// When removing 3D content (page navigation, section exit):
function dispose(obj) {
  obj.traverse((child) => {
    if (child.isMesh) {
      child.geometry.dispose();
      if (Array.isArray(child.material)) {
        child.material.forEach(m => m.dispose());
      } else {
        child.material.dispose();
      }
    }
  });
}

// Dispose textures
texture.dispose();

// Dispose render targets
renderTarget.dispose();

// Dispose renderer when leaving page
renderer.dispose();
```

### Frustum Culling
Three.js does this automatically for meshes, but for custom particle systems:
```js
// Disable frustum culling for objects that should always render
// (skyboxes, fullscreen quads):
mesh.frustumCulled = false;
```

### Level of Detail (LOD)
```js
const lod = new THREE.LOD();
lod.addLevel(highDetailMesh, 0);    // close: full detail
lod.addLevel(mediumDetailMesh, 10); // medium distance: fewer polys
lod.addLevel(lowDetailMesh, 30);    // far: minimal polys
scene.add(lod);

// Update in animation loop
lod.update(camera);
```

## Loading Strategy

### Progressive Loading
```js
// Show page immediately with 2D content
// Load 3D only when section is approaching viewport

const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      init3DScene(); // start loading and rendering
      observer.disconnect();
    }
  });
}, { rootMargin: '200px' }); // start loading 200px before visible

observer.observe(document.getElementById('3d-section'));
```

### Asset Preloading with Progress
```js
const manager = new THREE.LoadingManager();
manager.onProgress = (url, loaded, total) => {
  const progress = (loaded / total) * 100;
  document.querySelector('.loader-bar').style.width = `${progress}%`;
};
manager.onLoad = () => {
  document.querySelector('.loader').classList.add('hidden');
};

const gltfLoader = new GLTFLoader(manager);
const textureLoader = new THREE.TextureLoader(manager);
```

### R3F Suspense Loading
```jsx
import { Suspense } from 'react';
import { Html, useProgress } from '@react-three/drei';

function Loader() {
  const { progress } = useProgress();
  return <Html center>{progress.toFixed(0)}%</Html>;
}

<Canvas>
  <Suspense fallback={<Loader />}>
    <Scene />
  </Suspense>
</Canvas>
```

## Mobile Fallbacks

### Detect and Adapt
```js
function get3DCapability() {
  const canvas = document.createElement('canvas');
  const gl = canvas.getContext('webgl2') || canvas.getContext('webgl');

  if (!gl) return 'none';        // no WebGL at all
  if (navigator.hardwareConcurrency <= 2) return 'low';
  if (/Android|iPhone/i.test(navigator.userAgent)) return 'medium';
  return 'high';
}

const capability = get3DCapability();

if (capability === 'none' || capability === 'low') {
  // Show static image or CSS 3D fallback
  showImageFallback();
} else if (capability === 'medium') {
  // Reduced quality WebGL
  initScene({ particles: 1000, shadows: false, postProcessing: false });
} else {
  // Full experience
  initScene({ particles: 10000, shadows: true, postProcessing: true });
}
```

### Image Sequence Fallback for Mobile (from Zoox)
```js
if (isMobile) {
  // Replace WebGL with a video
  const video = document.createElement('video');
  video.src = '/product-orbit-mobile.mp4';
  video.muted = true;
  video.playsInline = true;
  video.autoplay = true;
  video.loop = true;
  container.appendChild(video);
} else {
  // Desktop: full image sequence
  initImageSequence();
}
```

## Performance Monitoring

### Stats.js
```js
import Stats from 'three/addons/libs/stats.module.js';

const stats = new Stats();
document.body.appendChild(stats.dom);

function animate() {
  stats.begin();
  renderer.render(scene, camera);
  stats.end();
  requestAnimationFrame(animate);
}
```

### Renderer Info
```js
// Log draw calls and triangles
console.log('Draw calls:', renderer.info.render.calls);
console.log('Triangles:', renderer.info.render.triangles);
console.log('Textures:', renderer.info.memory.textures);
console.log('Geometries:', renderer.info.memory.geometries);
```

### Performance Budget
| Metric | Target (Desktop) | Target (Mobile) |
|---|---|---|
| Draw calls | < 50 | < 20 |
| Triangles | < 500K | < 100K |
| Texture memory | < 64MB | < 16MB |
| Frame time | < 16ms (60fps) | < 33ms (30fps) |
| Initial load | < 5MB | < 2MB |
| Time to interactive | < 3s | < 5s |
