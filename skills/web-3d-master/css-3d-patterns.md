# CSS 3D Patterns

No WebGL needed. These CSS-only techniques create convincing 3D effects that work everywhere.

## Perspective & Transform Basics

### Container Setup
```css
.perspective-container {
  perspective: 1000px;          /* distance from viewer to z=0 plane */
  perspective-origin: 50% 50%;  /* vanishing point */
}
.transform-3d {
  transform-style: preserve-3d; /* children maintain 3D space */
  backface-visibility: hidden;  /* hide back faces */
}
```

## Card Effects

### Cursor-Following 3D Tilt (from Rick Waalders)
```css
.tilt-card {
  perspective: 1000px;
  transform-style: preserve-3d;
}
.tilt-card__inner {
  transition: transform 0.1s ease-out;
  will-change: transform;
}
```
```js
document.querySelectorAll('.tilt-card').forEach(card => {
  const inner = card.querySelector('.tilt-card__inner');

  card.addEventListener('mousemove', (e) => {
    const rect = card.getBoundingClientRect();
    const x = (e.clientX - rect.left) / rect.width - 0.5;
    const y = (e.clientY - rect.top) / rect.height - 0.5;
    inner.style.transform = `
      perspective(1000px)
      rotateY(${x * 15}deg)
      rotateX(${-y * 15}deg)
      scale3d(1.02, 1.02, 1.02)
    `;
  });

  card.addEventListener('mouseleave', () => {
    inner.style.transform = 'perspective(1000px) rotateY(0) rotateX(0) scale3d(1, 1, 1)';
  });
});
```

### 3D Flip Card
```css
.flip-card {
  perspective: 1000px;
  width: 300px;
  height: 400px;
  cursor: pointer;
}
.flip-card__inner {
  position: relative;
  width: 100%;
  height: 100%;
  transition: transform 0.6s cubic-bezier(0.23, 1, 0.32, 1);
  transform-style: preserve-3d;
}
.flip-card:hover .flip-card__inner {
  transform: rotateY(180deg);
}
.flip-card__front,
.flip-card__back {
  position: absolute;
  inset: 0;
  backface-visibility: hidden;
  border-radius: 1rem;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 2rem;
}
.flip-card__back {
  transform: rotateY(180deg);
}
```

### Layered Depth Card
Content elements at different Z positions create real depth.

```css
.depth-card {
  perspective: 800px;
  transform-style: preserve-3d;
}
.depth-card__bg {
  transform: translateZ(-20px) scale(1.04);
}
.depth-card__content {
  transform: translateZ(0);
}
.depth-card__title {
  transform: translateZ(30px);
}
.depth-card__cta {
  transform: translateZ(50px);
}
```

## Rotating Showcases

### 3D Carousel
```css
.carousel-3d {
  perspective: 1200px;
  width: 300px;
  height: 400px;
  margin: 0 auto;
}
.carousel-3d__track {
  width: 100%;
  height: 100%;
  position: relative;
  transform-style: preserve-3d;
  animation: spin 20s linear infinite;
}
.carousel-3d__item {
  position: absolute;
  width: 250px;
  height: 350px;
  backface-visibility: hidden;
  border-radius: 1rem;
  overflow: hidden;
}
/* Distribute items in a circle */
.carousel-3d__item:nth-child(1) { transform: rotateY(0deg)   translateZ(350px); }
.carousel-3d__item:nth-child(2) { transform: rotateY(60deg)  translateZ(350px); }
.carousel-3d__item:nth-child(3) { transform: rotateY(120deg) translateZ(350px); }
.carousel-3d__item:nth-child(4) { transform: rotateY(180deg) translateZ(350px); }
.carousel-3d__item:nth-child(5) { transform: rotateY(240deg) translateZ(350px); }
.carousel-3d__item:nth-child(6) { transform: rotateY(300deg) translateZ(350px); }

@keyframes spin {
  from { transform: rotateY(0deg); }
  to { transform: rotateY(360deg); }
}

/* Pause on hover */
.carousel-3d:hover .carousel-3d__track {
  animation-play-state: paused;
}
```

### 3D Cube
```css
.cube {
  width: 200px;
  height: 200px;
  perspective: 800px;
  margin: 100px auto;
}
.cube__inner {
  width: 100%;
  height: 100%;
  position: relative;
  transform-style: preserve-3d;
  animation: rotateCube 10s linear infinite;
}
.cube__face {
  position: absolute;
  width: 200px;
  height: 200px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 2rem;
  font-weight: 700;
  border: 2px solid rgba(255, 255, 255, 0.2);
}
.cube__face--front  { transform: translateZ(100px); }
.cube__face--back   { transform: rotateY(180deg) translateZ(100px); }
.cube__face--right  { transform: rotateY(90deg) translateZ(100px); }
.cube__face--left   { transform: rotateY(-90deg) translateZ(100px); }
.cube__face--top    { transform: rotateX(90deg) translateZ(100px); }
.cube__face--bottom { transform: rotateX(-90deg) translateZ(100px); }

@keyframes rotateCube {
  from { transform: rotateX(0deg) rotateY(0deg); }
  to { transform: rotateX(360deg) rotateY(360deg); }
}
```

## Parallax Depth

### Mouse-Driven Parallax Layers
```css
.parallax-scene {
  position: relative;
  height: 100vh;
  overflow: hidden;
}
.parallax-layer {
  position: absolute;
  inset: -10%;
  transition: transform 0.15s ease-out;
  will-change: transform;
}
```
```js
const scene = document.querySelector('.parallax-scene');
const layers = scene.querySelectorAll('.parallax-layer');

scene.addEventListener('mousemove', (e) => {
  const x = (e.clientX / window.innerWidth - 0.5) * 2;
  const y = (e.clientY / window.innerHeight - 0.5) * 2;

  layers.forEach((layer, i) => {
    const depth = (i + 1) * 15;
    layer.style.transform = `translate(${x * depth}px, ${y * depth}px)`;
  });
});
```

### Scroll Parallax (CSS-only with scroll-timeline)
```css
@supports (animation-timeline: scroll()) {
  .parallax-slow {
    animation: parallaxUp linear both;
    animation-timeline: scroll();
    animation-range: entry 0% exit 100%;
  }
  @keyframes parallaxUp {
    from { transform: translateY(100px); }
    to { transform: translateY(-100px); }
  }
}
```

## Text in 3D Space

### Perspective Text
```css
.text-3d {
  perspective: 500px;
  display: inline-block;
}
.text-3d span {
  display: inline-block;
  transform: rotateY(-10deg);
  transform-origin: left center;
}
```

### Extruded Text Effect (CSS only)
```css
.text-extruded {
  font-size: clamp(4rem, 12vw, 10rem);
  font-weight: 900;
  color: #fff;
  text-shadow:
    1px 1px 0 #ccc,
    2px 2px 0 #bbb,
    3px 3px 0 #aaa,
    4px 4px 0 #999,
    5px 5px 0 #888,
    6px 6px 0 #777,
    7px 7px 15px rgba(0, 0, 0, 0.3);
}
```

## Accessibility

### Always Respect Reduced Motion
```css
@media (prefers-reduced-motion: reduce) {
  .carousel-3d__track,
  .cube__inner {
    animation: none !important;
  }
  .tilt-card__inner,
  .parallax-layer {
    transition: none !important;
    transform: none !important;
  }
}
```

### Wrap Hover Effects for Touch Devices
```css
@media (hover: hover) {
  .flip-card:hover .flip-card__inner {
    transform: rotateY(180deg);
  }
}
/* On touch: use click instead */
@media (hover: none) {
  .flip-card.flipped .flip-card__inner {
    transform: rotateY(180deg);
  }
}
```
