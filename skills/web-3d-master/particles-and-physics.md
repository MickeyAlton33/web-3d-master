# Particles & Physics

Particle systems, instanced rendering, and physics integration patterns.

## Particle Systems

### Basic Points System
```js
const particleCount = 5000;
const positions = new Float32Array(particleCount * 3);
const scales = new Float32Array(particleCount);

for (let i = 0; i < particleCount; i++) {
  positions[i * 3] = (Math.random() - 0.5) * 20;
  positions[i * 3 + 1] = (Math.random() - 0.5) * 20;
  positions[i * 3 + 2] = (Math.random() - 0.5) * 20;
  scales[i] = Math.random();
}

const geometry = new THREE.BufferGeometry();
geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
geometry.setAttribute('aScale', new THREE.BufferAttribute(scales, 1));

const material = new THREE.ShaderMaterial({
  vertexShader: `
    attribute float aScale;
    uniform float uTime;
    varying float vAlpha;

    void main() {
      vec3 pos = position;
      pos.y += sin(uTime * 0.5 + position.x * 0.5) * 0.3;

      vec4 mvPosition = modelViewMatrix * vec4(pos, 1.0);
      gl_PointSize = aScale * 50.0 * (1.0 / -mvPosition.z);
      gl_Position = projectionMatrix * mvPosition;

      vAlpha = aScale;
    }
  `,
  fragmentShader: `
    varying float vAlpha;

    void main() {
      float d = distance(gl_PointCoord, vec2(0.5));
      if (d > 0.5) discard;
      float alpha = (1.0 - smoothstep(0.3, 0.5, d)) * vAlpha;
      gl_FragColor = vec4(1.0, 1.0, 1.0, alpha);
    }
  `,
  uniforms: { uTime: { value: 0 } },
  transparent: true,
  depthWrite: false,
  blending: THREE.AdditiveBlending
});

const particles = new THREE.Points(geometry, material);
scene.add(particles);
```

### Instanced Rendering (100K+ Objects)
From Offscreen Canvas -- render massive numbers of identical meshes with one draw call.

```js
const count = 100000;
const geometry = new THREE.SphereGeometry(0.05, 8, 8);
const material = new THREE.MeshStandardMaterial({ color: 0xffffff });
const mesh = new THREE.InstancedMesh(geometry, material, count);

const dummy = new THREE.Object3D();
const color = new THREE.Color();

for (let i = 0; i < count; i++) {
  dummy.position.set(
    (Math.random() - 0.5) * 50,
    (Math.random() - 0.5) * 50,
    (Math.random() - 0.5) * 50
  );
  dummy.updateMatrix();
  mesh.setMatrixAt(i, dummy.matrix);
  mesh.setColorAt(i, color.setHSL(Math.random(), 0.7, 0.5));
}
mesh.instanceMatrix.needsUpdate = true;
mesh.instanceColor.needsUpdate = true;
scene.add(mesh);
```

### R3F Instanced Rendering
```jsx
import { useRef, useMemo } from 'react';
import { useFrame } from '@react-three/fiber';
import * as THREE from 'three';

function Particles({ count = 10000 }) {
  const meshRef = useRef();
  const dummy = useMemo(() => new THREE.Object3D(), []);

  const particles = useMemo(() => {
    const temp = [];
    for (let i = 0; i < count; i++) {
      temp.push({
        position: [(Math.random() - 0.5) * 20, (Math.random() - 0.5) * 20, (Math.random() - 0.5) * 20],
        speed: Math.random() * 0.01 + 0.005
      });
    }
    return temp;
  }, [count]);

  useFrame((state) => {
    particles.forEach((p, i) => {
      dummy.position.set(...p.position);
      dummy.position.y += Math.sin(state.clock.elapsedTime * p.speed * 100) * 0.01;
      dummy.updateMatrix();
      meshRef.current.setMatrixAt(i, dummy.matrix);
    });
    meshRef.current.instanceMatrix.needsUpdate = true;
  });

  return (
    <instancedMesh ref={meshRef} args={[null, null, count]}>
      <sphereGeometry args={[0.03, 6, 6]} />
      <meshBasicMaterial color="#ffffff" />
    </instancedMesh>
  );
}
```

### GPU Particles (Shader-Driven)
```js
// Encode particle behavior entirely in shaders for maximum performance
const material = new THREE.ShaderMaterial({
  vertexShader: `
    attribute vec3 aVelocity;
    attribute float aLife;
    uniform float uTime;

    varying float vLife;

    void main() {
      vLife = fract(aLife + uTime * 0.1);

      vec3 pos = position + aVelocity * vLife * 10.0;
      pos.y -= vLife * vLife * 5.0; // gravity

      vec4 mvPos = modelViewMatrix * vec4(pos, 1.0);
      gl_PointSize = (1.0 - vLife) * 30.0 * (1.0 / -mvPos.z);
      gl_Position = projectionMatrix * mvPos;
    }
  `,
  fragmentShader: `
    varying float vLife;

    void main() {
      float d = distance(gl_PointCoord, vec2(0.5));
      if (d > 0.5) discard;
      float alpha = (1.0 - vLife) * (1.0 - smoothstep(0.3, 0.5, d));
      gl_FragColor = vec4(1.0, 0.6, 0.2, alpha);
    }
  `,
  transparent: true,
  depthWrite: false,
  blending: THREE.AdditiveBlending,
  uniforms: { uTime: { value: 0 } }
});
```

## Physics

### Rapier Physics with R3F
```jsx
import { Physics, RigidBody } from '@react-three/rapier';

function PhysicsScene() {
  return (
    <Physics gravity={[0, -9.81, 0]}>
      {/* Static ground */}
      <RigidBody type="fixed">
        <mesh position={[0, -1, 0]}>
          <boxGeometry args={[20, 0.5, 20]} />
          <meshStandardMaterial color="#333" />
        </mesh>
      </RigidBody>

      {/* Falling cubes */}
      {Array.from({ length: 50 }).map((_, i) => (
        <RigidBody key={i} position={[
          (Math.random() - 0.5) * 4,
          Math.random() * 10 + 5,
          (Math.random() - 0.5) * 4
        ]}>
          <mesh>
            <boxGeometry args={[0.5, 0.5, 0.5]} />
            <meshStandardMaterial color={`hsl(${Math.random() * 360}, 70%, 50%)`} />
          </mesh>
        </RigidBody>
      ))}
    </Physics>
  );
}
```

### Cannon-ES with Vanilla Three.js
```js
import * as CANNON from 'cannon-es';

const world = new CANNON.World({ gravity: new CANNON.Vec3(0, -9.82, 0) });

// Ground
const groundBody = new CANNON.Body({
  mass: 0,
  shape: new CANNON.Plane()
});
groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
world.addBody(groundBody);

// Falling sphere
const sphereBody = new CANNON.Body({
  mass: 1,
  shape: new CANNON.Sphere(0.5),
  position: new CANNON.Vec3(0, 10, 0)
});
world.addBody(sphereBody);

// Three.js mesh
const sphereMesh = new THREE.Mesh(
  new THREE.SphereGeometry(0.5),
  new THREE.MeshStandardMaterial({ color: 0xff4444 })
);
scene.add(sphereMesh);

// Sync in animation loop
function animate() {
  world.step(1 / 60);
  sphereMesh.position.copy(sphereBody.position);
  sphereMesh.quaternion.copy(sphereBody.quaternion);
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
```

## CSS-Only Particle Effects

### Floating Particles (Pure CSS)
```css
.particle-container {
  position: fixed;
  inset: 0;
  pointer-events: none;
  overflow: hidden;
  z-index: -1;
}
.particle {
  position: absolute;
  width: var(--size, 4px);
  height: var(--size, 4px);
  background: rgba(255, 255, 255, 0.3);
  border-radius: 50%;
  animation: float var(--duration, 10s) var(--delay, 0s) infinite linear;
}
@keyframes float {
  0% { transform: translateY(100vh) translateX(0) rotate(0deg); opacity: 0; }
  10% { opacity: 1; }
  90% { opacity: 1; }
  100% { transform: translateY(-10vh) translateX(var(--drift, 50px)) rotate(360deg); opacity: 0; }
}
```

```html
<!-- Generate 20+ particles with random properties -->
<div class="particle-container">
  <div class="particle" style="--size:3px; --duration:12s; --delay:0s; --drift:30px; left:10%"></div>
  <div class="particle" style="--size:5px; --duration:8s; --delay:2s; --drift:-40px; left:30%"></div>
  <div class="particle" style="--size:2px; --duration:15s; --delay:1s; --drift:60px; left:55%"></div>
  <!-- ... more particles ... -->
</div>
```
