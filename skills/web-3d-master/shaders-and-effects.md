# Shaders & Visual Effects

GLSL shader patterns and post-processing techniques from award-winning 3D websites.

## GLSL Fundamentals

### Vertex Shader Template
```glsl
uniform float uTime;
uniform vec2 uMouse;

varying vec2 vUv;
varying vec3 vPosition;
varying vec3 vNormal;

void main() {
  vUv = uv;
  vPosition = position;
  vNormal = normalize(normalMatrix * normal);

  vec3 pos = position;
  // Displacement goes here
  // pos += normal * sin(uTime + position.y * 3.0) * 0.1;

  gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
}
```

### Fragment Shader Template
```glsl
uniform float uTime;
uniform vec2 uResolution;
uniform sampler2D uTexture;

varying vec2 vUv;
varying vec3 vNormal;

void main() {
  vec3 color = vec3(0.0);
  // Color logic goes here
  gl_FragColor = vec4(color, 1.0);
}
```

## Common Shader Effects

### Noise-Based Displacement (organic mesh animation)
```glsl
// Vertex shader
uniform float uTime;
uniform float uAmplitude; // 0.1 - 0.5

// Classic simplex noise function (include a noise library)
float snoise(vec3 v) { /* ... simplex noise implementation ... */ }

void main() {
  vec3 pos = position;
  float noise = snoise(vec3(position * 2.0 + uTime * 0.5));
  pos += normal * noise * uAmplitude;

  gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
}
```

### Fresnel Effect (edge glow)
```glsl
// Fragment shader
varying vec3 vNormal;

void main() {
  vec3 viewDir = normalize(cameraPosition - vPosition);
  float fresnel = pow(1.0 - dot(viewDir, vNormal), 3.0);

  vec3 baseColor = vec3(0.1, 0.1, 0.2);
  vec3 glowColor = vec3(0.3, 0.5, 1.0);
  vec3 color = mix(baseColor, glowColor, fresnel);

  gl_FragColor = vec4(color, 1.0);
}
```

### Gradient Color Based on Position
```glsl
varying vec3 vPosition;

void main() {
  float t = (vPosition.y + 1.0) * 0.5; // normalize -1..1 to 0..1
  vec3 colorA = vec3(0.1, 0.0, 0.3); // dark purple
  vec3 colorB = vec3(0.0, 0.8, 1.0); // cyan
  vec3 color = mix(colorA, colorB, t);

  gl_FragColor = vec4(color, 1.0);
}
```

### Holographic / Iridescent Effect
```glsl
varying vec3 vNormal;
varying vec3 vPosition;

void main() {
  vec3 viewDir = normalize(cameraPosition - vPosition);
  float fresnel = pow(1.0 - abs(dot(viewDir, vNormal)), 2.0);

  // Rainbow based on normal direction + time
  float hue = dot(vNormal, vec3(1.0, 0.5, 0.0)) * 0.5 + uTime * 0.1;
  vec3 color = vec3(
    sin(hue * 6.28) * 0.5 + 0.5,
    sin((hue + 0.33) * 6.28) * 0.5 + 0.5,
    sin((hue + 0.66) * 6.28) * 0.5 + 0.5
  );

  color = mix(vec3(0.0), color, fresnel);
  gl_FragColor = vec4(color, 0.8);
}
```

### Scroll-Driven Shader Transition
```js
// JS: bind scroll progress to shader uniform
const material = new THREE.ShaderMaterial({
  uniforms: {
    uProgress: { value: 0 },
    uTexture1: { value: textureA },
    uTexture2: { value: textureB }
  },
  fragmentShader: `
    uniform float uProgress;
    uniform sampler2D uTexture1;
    uniform sampler2D uTexture2;
    varying vec2 vUv;

    void main() {
      vec4 texA = texture2D(uTexture1, vUv);
      vec4 texB = texture2D(uTexture2, vUv);
      gl_FragColor = mix(texA, texB, uProgress);
    }
  `
});

// GSAP ScrollTrigger binding
gsap.to(material.uniforms.uProgress, {
  value: 1,
  scrollTrigger: { trigger: '#section', scrub: true }
});
```

## Post-Processing

### EffectComposer Setup (Vanilla Three.js)
```js
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';

const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));

const bloomPass = new UnrealBloomPass(
  new THREE.Vector2(window.innerWidth, window.innerHeight),
  0.5,   // strength
  0.4,   // radius
  0.85   // threshold
);
composer.addPass(bloomPass);

// Replace renderer.render with composer.render in animation loop
function animate() {
  composer.render();
  requestAnimationFrame(animate);
}
```

### Post-Processing with R3F
```jsx
import { EffectComposer, Bloom, Vignette, ChromaticAberration } from '@react-three/postprocessing';

<EffectComposer>
  <Bloom luminanceThreshold={0.9} luminanceSmoothing={0.025} intensity={0.5} />
  <Vignette offset={0.3} darkness={0.6} />
  <ChromaticAberration offset={[0.001, 0.001]} />
</EffectComposer>
```

### Custom Post-Processing Pass
```js
const customPass = {
  uniforms: {
    tDiffuse: { value: null },
    uTime: { value: 0 },
    uIntensity: { value: 0.5 }
  },
  vertexShader: `
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    uniform sampler2D tDiffuse;
    uniform float uTime;
    uniform float uIntensity;
    varying vec2 vUv;

    void main() {
      vec2 uv = vUv;
      // Barrel distortion
      vec2 center = uv - 0.5;
      float dist = length(center);
      uv += center * dist * dist * uIntensity;

      gl_FragColor = texture2D(tDiffuse, uv);
    }
  `
};
```

## Background Effects (No 3D Models)

### Gradient Mesh Background
```glsl
// Full-screen quad fragment shader
uniform float uTime;
varying vec2 vUv;

void main() {
  vec2 uv = vUv;

  // Animated gradient mesh
  float noise1 = sin(uv.x * 3.0 + uTime * 0.5) * sin(uv.y * 2.0 + uTime * 0.3);
  float noise2 = sin(uv.x * 5.0 - uTime * 0.4) * sin(uv.y * 4.0 + uTime * 0.6);

  vec3 color1 = vec3(0.1, 0.0, 0.3); // deep purple
  vec3 color2 = vec3(0.0, 0.3, 0.6); // ocean blue
  vec3 color3 = vec3(0.0, 0.6, 0.4); // teal

  vec3 color = mix(color1, color2, noise1 * 0.5 + 0.5);
  color = mix(color, color3, noise2 * 0.5 + 0.5);

  gl_FragColor = vec4(color, 1.0);
}
```

### Animated Noise Background
```glsl
// Simplex noise-based flowing background
uniform float uTime;
varying vec2 vUv;

// Include your noise function here
float snoise(vec3 v) { /* ... */ }

void main() {
  float n = snoise(vec3(vUv * 3.0, uTime * 0.2));
  vec3 color = mix(vec3(0.02, 0.02, 0.05), vec3(0.1, 0.05, 0.2), n * 0.5 + 0.5);
  gl_FragColor = vec4(color, 1.0);
}
```
