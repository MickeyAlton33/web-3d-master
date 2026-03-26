# Three.js Patterns

Production-ready Three.js patterns extracted from award-winning 3D websites.

## Scene Setup

### Standard Scene Template
```js
import * as THREE from 'three';

const canvas = document.querySelector('#webgl');
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 0, 5);

const renderer = new THREE.WebGLRenderer({
  canvas,
  antialias: true,
  alpha: true,                    // transparent background
  powerPreference: 'high-performance'
});
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // cap at 2x for performance
renderer.outputColorSpace = THREE.SRGBColorSpace;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.0;

// Resize handler
window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});

// Animation loop
function animate() {
  requestAnimationFrame(animate);
  renderer.render(scene, camera);
}
animate();
```

### R3F Scene Template
```jsx
import { Canvas } from '@react-three/fiber';
import { Environment, Preload } from '@react-three/drei';

export default function App() {
  return (
    <Canvas
      camera={{ position: [0, 0, 5], fov: 45 }}
      gl={{
        antialias: true,
        alpha: true,
        powerPreference: 'high-performance',
        outputColorSpace: THREE.SRGBColorSpace,
        toneMapping: THREE.ACESFilmicToneMapping
      }}
      dpr={[1, 2]}
    >
      <Environment preset="studio" />
      <Scene />
      <Preload all />
    </Canvas>
  );
}
```

## Model Loading

### GLTF/GLB Loading (Vanilla)
```js
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';

const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath('https://www.gstatic.com/draco/versioned/decoders/1.5.6/');

const gltfLoader = new GLTFLoader();
gltfLoader.setDRACOLoader(dracoLoader);

gltfLoader.load('/model.glb', (gltf) => {
  const model = gltf.scene;
  model.traverse((child) => {
    if (child.isMesh) {
      child.castShadow = true;
      child.receiveShadow = true;
    }
  });
  scene.add(model);
});
```

### GLTF Loading with R3F + Drei
```jsx
import { useGLTF } from '@react-three/drei';

function Model({ url, ...props }) {
  const { scene } = useGLTF(url);
  return <primitive object={scene} {...props} />;
}

// Preload for instant display
useGLTF.preload('/model.glb');
```

## Materials

### Matcap Material (from Superlist)
No lights needed -- lighting is baked into a spherical texture lookup. Extremely performant.

```js
const matcapTexture = new THREE.TextureLoader().load('/matcap-gold.png');
const material = new THREE.MeshMatcapMaterial({ matcap: matcapTexture });

// Apply to all meshes in a loaded model
model.traverse((child) => {
  if (child.isMesh) {
    child.material = new THREE.MeshMatcapMaterial({ matcap: matcapTexture });
  }
});
```

### PBR Material with Environment Map
```js
import { RGBELoader } from 'three/addons/loaders/RGBELoader.js';

new RGBELoader().load('/environment.hdr', (texture) => {
  texture.mapping = THREE.EquirectangularReflectionMapping;
  scene.environment = texture;
  // scene.background = texture; // optional: also use as background
});

const material = new THREE.MeshStandardMaterial({
  color: 0xffffff,
  metalness: 0.9,
  roughness: 0.1,
  envMapIntensity: 1.5
});
```

### Custom Shader Material
```js
const material = new THREE.ShaderMaterial({
  vertexShader: `
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    uniform float uTime;
    varying vec2 vUv;
    void main() {
      vec3 color = vec3(vUv, sin(uTime) * 0.5 + 0.5);
      gl_FragColor = vec4(color, 1.0);
    }
  `,
  uniforms: {
    uTime: { value: 0 }
  }
});

// Update in animation loop
function animate() {
  material.uniforms.uTime.value = performance.now() * 0.001;
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
```

## Lighting

### Studio Lighting Setup
```js
// Key light
const keyLight = new THREE.DirectionalLight(0xffffff, 1.5);
keyLight.position.set(5, 5, 5);
keyLight.castShadow = true;
keyLight.shadow.mapSize.set(2048, 2048);
scene.add(keyLight);

// Fill light
const fillLight = new THREE.DirectionalLight(0x8888ff, 0.5);
fillLight.position.set(-5, 0, -5);
scene.add(fillLight);

// Ambient
const ambientLight = new THREE.AmbientLight(0xffffff, 0.3);
scene.add(ambientLight);
```

### Environment-Only Lighting (R3F)
```jsx
import { Environment } from '@react-three/drei';

// Presets: studio, city, sunset, dawn, night, forest, apartment, warehouse, park, lobby
<Environment preset="studio" />

// Custom HDR
<Environment files="/custom-env.hdr" />

// Contact shadows for grounding
import { ContactShadows } from '@react-three/drei';
<ContactShadows position={[0, -1, 0]} opacity={0.4} scale={10} blur={2.5} />
```

## Camera Patterns

### Orbit Controls
```js
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.dampingFactor = 0.05;
controls.maxPolarAngle = Math.PI / 2; // prevent going below ground
controls.minDistance = 2;
controls.maxDistance = 10;
controls.autoRotate = true;
controls.autoRotateSpeed = 1;
```

### Mouse-Following Camera (from portfolios)
```js
let targetX = 0, targetY = 0;
document.addEventListener('mousemove', (e) => {
  targetX = (e.clientX / window.innerWidth - 0.5) * 2;
  targetY = (e.clientY / window.innerHeight - 0.5) * 2;
});

function animate() {
  camera.position.x += (targetX * 0.5 - camera.position.x) * 0.05;
  camera.position.y += (-targetY * 0.5 - camera.position.y) * 0.05;
  camera.lookAt(0, 0, 0);
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
```

## Animation

### Scroll-Driven GLTF Animation (from Superlist)
```js
import gsap from 'gsap';
import { ScrollTrigger } from 'gsap/ScrollTrigger';
gsap.registerPlugin(ScrollTrigger);

// After loading GLTF with animations
const mixer = new THREE.AnimationMixer(model);
const action = mixer.clipAction(gltf.animations[0]);
action.play();
action.paused = true; // we control it via scroll

gsap.to(action, {
  time: action.getClip().duration,
  scrollTrigger: {
    trigger: '#section-3d',
    start: 'top top',
    end: 'bottom bottom',
    scrub: 2   // 2-second smoothing
  }
});

// Update mixer in animation loop
const clock = new THREE.Clock();
function animate() {
  mixer.update(clock.getDelta());
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
```

### Floating Animation (R3F)
```jsx
import { Float } from '@react-three/drei';

<Float
  speed={1.5}          // animation speed
  rotationIntensity={0.4}  // max rotation
  floatIntensity={0.6}     // max float distance
  floatingRange={[-0.1, 0.1]}  // y-axis range
>
  <Model url="/product.glb" />
</Float>
```

### Procedural Animation
```js
function animate() {
  const time = performance.now() * 0.001;

  // Gentle rotation
  model.rotation.y = Math.sin(time * 0.3) * 0.2;

  // Bobbing
  model.position.y = Math.sin(time * 0.5) * 0.1;

  // Breathing scale
  const breathe = 1 + Math.sin(time * 0.8) * 0.02;
  model.scale.set(breathe, breathe, breathe);

  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
```
