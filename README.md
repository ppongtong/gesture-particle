# Gesture Particle

An interactive 3D particle system controlled entirely by hand gestures — no mouse, no keyboard. Point, swipe, pinch, or open your palm and 10,000 particles respond in real time.

---

## Demo

🔗 **[ppongtong.github.io/gesture-particle](https://ppongtong.github.io/gesture-particle)**

Open your webcam, make a gesture, watch the magic happen.

| Gesture               | Formation | Feel                                   |
| --------------------- | --------- | -------------------------------------- |
| ✊ Fist               | Sphere    | Particles compress into a glowing orb  |
| 🖐️ Open hand          | Scatter   | Explosion outward into space           |
| ☝️ Point (index only) | Vortex    | Spinning tornado column                |
| ✌️ Peace sign         | Wave      | Animated ripple surface                |
| 🤌 Pinch              | Implode   | Everything collapses to your fingertip |
| 👋 Fast swipe         | Push      | Kinetic burst in the swipe direction   |

Move your hand around the frame and the entire formation follows.

---

## How to Run

Because the project fetches all dependencies directly from CDNs and uses no local imports, you can run it straight from your file system! **No local web server is required.**

1. Clone or download this repository.
2. Double-click the `index.html` file to open it in your browser.
3. Allow camera access when prompted.

*(If you prefer, you can still serve it locally using `python3 -m http.server 8080` or `npx serve .`)*

> **Browser support:** Chrome 89+, Edge 89+, Safari 16.4+. Requires a webcam and good lighting.

---

## How It Works

The project is a single `index.html` file — no build step, no dependencies to install. Everything loads from CDN at runtime.

### Tech Stack

| Library                                                                           | Version | Role                              |
| --------------------------------------------------------------------------------- | ------- | --------------------------------- |
| [Three.js](https://threejs.org)                                                   | r160    | 3D rendering via WebGL            |
| [MediaPipe Hands](https://ai.google.dev/edge/mediapipe/solutions/guide)           | 0.4     | Real-time hand landmark detection |
| [MediaPipe Camera Utils](https://www.npmjs.com/package/@mediapipe/camera_utils)   | 0.3     | Webcam feed management            |
| [MediaPipe Drawing Utils](https://www.npmjs.com/package/@mediapipe/drawing_utils) | 0.3     | Hand skeleton overlay             |

---

### 1. Hand Tracking — MediaPipe Hands

[MediaPipe Hands](https://ai.google.dev/edge/mediapipe/solutions/guide) is a machine learning pipeline from Google that detects **21 hand landmarks** per frame directly in the browser using a WASM model. No server, no cloud — everything runs locally on your CPU/GPU.

```
Landmark indices used:
  0  — Wrist
  4  — Thumb tip
  6  — Index finger PIP (middle joint)
  8  — Index finger tip
  10 — Middle finger PIP
  12 — Middle finger tip
  14 — Ring finger PIP
  16 — Ring finger tip
  18 — Pinky PIP
  20 — Pinky tip
```

Each landmark is a normalised `{x, y, z}` coordinate in 0–1 space relative to the video frame.

**Finger extension detection:**
A finger is considered "extended" when its tip sits above its middle joint in the frame — i.e. `tip.y < pip.y - threshold`. Because MediaPipe's Y-axis increases downward, a smaller Y value means higher on screen.

```js
function isExtended(landmarks, tipIndex, pipIndex) {
  return landmarks[tipIndex].y < landmarks[pipIndex].y - 0.03;
}
```

**Gesture classification** checks which fingers are extended, then maps combinations to modes:

```
0 fingers extended  →  fist   →  sphere
4 fingers extended  →  open   →  scatter
index only          →  point  →  vortex
index + middle      →  peace  →  wave
thumb–index dist < 0.06  →  pinch  →  implode
```

A **stabilisation buffer** (4 consecutive matching frames) prevents flickering between gestures.

**Swipe detection** tracks the last 5 hand positions. If the velocity exceeds a threshold, a one-shot impulse is applied to every particle in that direction.

---

### 2. Coordinate Mapping — Webcam to 3D World

MediaPipe returns normalised 0–1 coordinates. Three.js uses a world-space coordinate system centred at the origin. The mapping:

```js
// Flip X because the webcam feed is CSS-mirrored
worldX = (0.5 - landmark.x) * 82; // → roughly −41 to +41
worldY = (0.5 - landmark.y) * 55; // → roughly −27 to +27
```

The hand position is smoothed each frame with a lerp (`factor 0.18`) so the particle formations glide rather than snap.

---

### 3. Particle System — Three.js + Custom GLSL

10,000 particles are stored in flat `Float32Array` buffers for maximum CPU performance:

```
positions   Float32Array(N × 3)  — current x, y, z
velocities  Float32Array(N × 3)  — per-particle velocity
targets     Float32Array(N × 3)  — where each particle wants to be
colors      Float32Array(N × 3)  — per-particle RGB
sizes       Float32Array(N)      — per-particle sprite size
```

These are uploaded to the GPU as `THREE.BufferAttribute`s on a `THREE.Points` object.

**Vertex shader** sizes each point sprite by its distance from the camera, giving a natural perspective falloff:

```glsl
attribute float aSize;
attribute vec3  aColor;
varying   vec3  vColor;

void main() {
  vColor = aColor;
  vec4 mv = modelViewMatrix * vec4(position, 1.0);
  gl_PointSize = aSize * (320.0 / -mv.z);
  gl_Position  = projectionMatrix * mv;
}
```

**Fragment shader** draws each particle as a soft circular sprite with a bright core and a wider halo:

```glsl
varying vec3 vColor;

void main() {
  vec2  uv   = gl_PointCoord - 0.5;
  float dist = length(uv);
  if (dist > 0.5) discard;

  float core  = 1.0 - smoothstep(0.0, 0.18, dist);
  float halo  = 1.0 - smoothstep(0.0, 0.50, dist);
  float alpha = core * 0.85 + halo * 0.30;

  gl_FragColor = vec4(vColor * (1.0 + core * 2.2), alpha);
}
```

With `THREE.AdditiveBlending` enabled, overlapping particles accumulate brightness — creating the bloom glow effect without any post-processing pass.

---

### 4. Spring Physics

Each particle is pulled toward its target position by a spring force, then damped to simulate drag. This runs every frame on the CPU:

```js
const dx = target.x - position.x;
const dy = target.y - position.y;
const dz = target.z - position.z;

velocity.x = velocity.x * DAMPING + dx * SPRING_K; // SPRING_K = 0.052
velocity.y = velocity.y * DAMPING + dy * SPRING_K; // DAMPING  = 0.87
velocity.z = velocity.z * DAMPING + dz * SPRING_K;

position.x += velocity.x;
position.y += velocity.y;
position.z += velocity.z;
```

A tiny random noise term (`± 0.004`) is added to velocity each frame to keep particles feeling alive even when stationary.

When the gesture changes, only the `targets` array is updated — the spring physics handles the transition smoothly, with heavier/faster particles arriving first and stragglers trailing behind organically.

---

### 5. Formation Algorithms

**Sphere** — [Fibonacci sphere](https://arxiv.org/abs/0912.4540) for even surface distribution:

```js
const lat = Math.acos(1 - (2 * (i + 0.5)) / N);
const lon = (2 * Math.PI * i) / GOLDEN_RATIO;
target = {
  x: r * sin(lat) * cos(lon),
  y: r * cos(lat),
  z: r * sin(lat) * sin(lon),
};
```

**Vortex** — helical spiral with time-based rotation:

```js
const angle = (i / N) * Math.PI * 28 + time * 2.5;
const radius = 4 + (i / N) * 22;
const height = (i / N - 0.5) * VORTEX_HEIGHT;
target = { x: radius * cos(angle), y: height, z: radius * sin(angle) };
```

**Wave** — particles arranged on a 2D grid, Y driven by animated sine × cosine:

```js
target.y = sin(x * 0.22 + time * 2.2) * cos(z * 0.22 + time * 1.6) * 12;
```

**Scatter** — random positions inside a large sphere, computed once on mode entry.

**Implode** — all targets set to the hand position; particles rush to the fingertip.

---

### 6. Colour System

Each particle has a stable `hue` offset (`0–1`) assigned at creation. When the gesture changes, the palette target (stored as HSL) smoothly lerps toward a new value each frame:

| Formation | Colour              |
| --------- | ------------------- |
| Sphere    | Cyan `hsl(190°)`    |
| Scatter   | Gold `hsl(32°)`     |
| Vortex    | Violet `hsl(273°)`  |
| Wave      | Teal `hsl(155°)`    |
| Implode   | Magenta `hsl(338°)` |

The per-particle hue offset (±7%) keeps neighbouring particles slightly different shades, breaking up flat-looking uniform colour.

---

## Project Structure

```
gesture-particle/
├── index.html     — everything: HTML, CSS, GLSL shaders, JS
└── favicon.svg    — SVG favicon (particle sphere icon)
```

No build tools. No `node_modules`. No config files. The entire experience is one HTML file.

---

## License

MIT
