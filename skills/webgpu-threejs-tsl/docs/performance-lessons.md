# Performance Lessons

Measured lessons from a production WebGPU Three.js scene (~600 meshes, ~190
materials, 26 lights, MRT bloom). Every claim was verified on three r185.
Numbers come from an Intel Gen12 iGPU via Dawn D3D12: ratios transfer,
absolutes vary. Last updated: 2026-07-14 (second pass — six controlled
experiments, adversarially verified).

## 1. Startup: pipeline compilation is THE cost

- The first render compiles every render pipeline synchronously on the main
  thread. One frame can freeze for tens of seconds on big scenes (measured:
  23s). `requestAnimationFrame` stops firing; CSS compositor animations keep
  running (use them for loading UI).
- **`passNode.compileAsync(renderer)`** is the fix when rendering through
  `pass()`/`RenderPipeline` (MRT, bloom...). It pins the pass's render target
  and MRT before compiling so cache keys match the real render. Under the
  hood Dawn uses `createRenderPipelineAsync` (compiles on its thread pool, main
  thread stays free).
  - A bare `renderer.compileAsync(scene, camera)` compiles the
    direct-to-canvas pipelines — if you render through a pass, that work is
    thrown away entirely.
  - Async compile frees the main thread but wall-clock stays similar; the win
    is a live UI + no frozen tab, and you can show honest progress.
- **Set `scene.environment` BEFORE the big compile.** The envMap changes every
  material's shader variant: compile-without-env then recompile-with-env is
  two full rounds; the first is 100% wasted (measured: ~11s thrown away).
- **Disable `frustumCulled` during `compileAsync`** (restore after). Otherwise
  only what the camera sees compiles, and every player turn hits fresh
  pipeline stalls spread over the first seconds of gameplay.
- After compiling, do a **warm-up sweep**: render N orientations, one per real
  rAF frame (chaining renders in a single task does NOT let Chrome's GPU
  process finish its lazy per-orientation work; a CPU profile shows the main
  thread idle during the stall). With pipelines precompiled the sweep
  only warms bind groups/uniform buffers; 8 steps at 45° cover a fov-60 16:9
  camera with overlap.
- After compiling, the sweep's cost is a step function, not linear: on this
  scene the segment was flat from 8→4 steps (~2600ms both), fell off a cliff
  from 4→2 (~2650→~780ms, −70%), and 2 vs 0 overlapped. That says where the
  cost lives, NOT that fewer steps are free — the safe minimum (no turn frame
  over 50ms) was only verified for one rotation direction. Establish the safe
  minimum across all turn directions before cutting steps.
- **Chrome caches compiled pipelines on disk per browser profile**
  (`DawnWebGPUCache` in the profile dir), automatically, no flags. Measured:
  first visit 14s of compile → second visit 1.3s (−91%). Incognito/private
  windows and cleared profiles pay full price every time. Design first-visit
  UX for the worst case.
  - The cache is keyed **per program/variant**, not per file or session, so
    day-to-day dev edits mostly stay warm. Measured: a uniform/color change
    (same WGSL) does not invalidate anything (~1.4s compile, ~10× under the
    ~11.4s cold run); a change that forks the shader (`side: DoubleSide`)
    recompiles only that variant (+350-575ms, not the full ~11s); reverting
    the fork finds the original still cached (no LRU eviction seen). Editing
    uniforms during development costs nothing; only structural forks pay,
    and they pay little.

## 2. Lights are shader code, not just runtime cost

- Every non-batched light is **unrolled into the fragment shader of every
  program**. 26 lights × ~99 programs ≈ 4.9MB of WGSL. Compile time scales
  with total WGSL size, not just program count.
- **`DynamicLighting`** (`three/addons/lighting/DynamicLighting.js`, r185+)
  batches shadowless Point/Spot/Directional/Hemisphere lights into uniform
  arrays + a loop: light count stops affecting shader size (4.9MB → 1.7MB
  measured, image identical). Mind the `maxPointLights` etc. caps: lights
  beyond the cap silently fall back to the unrolled path. `ClusteredLighting`
  exists for many-light scenes with depth complexity.
- **`ClusteredLighting` vs `DynamicLighting` is a startup-vs-runtime trade,
  measured.** At 35 lights, swapping to `ClusteredLighting` compiled +12
  programs (78→90) and +440KB of WGSL (deterministic, 2/2 runs) and started
  slower every time — but nearly doubled resting FPS (~35 → ~55-66 on the
  iGPU, corroborated independently by the app's stats overlay) with an
  image-identical scene. Treat magnitudes as order-of-magnitude (thermal
  drift); the cross-over light count wasn't measured. Rule of thumb:
  Clustered pays a heavier compile to win runtime; if startup is your
  bottleneck and lights fit the batch caps, DynamicLighting starts faster.
- **Not batchable**: shadow-casting lights, `RectAreaLight`, projected spots
  (`.map`), node lights. They stay unrolled in EVERY program.
- **`RectAreaLight` is the most expensive light in three.js**: LTC
  evaluation = 2 texture tables + 2 full `LTC_Evaluate` calls (each: 4 edge
  form-factors, 8 cross products, matrix build) per fragment, in every
  program. One instance taxes the whole scene. Near-equivalents: emissive
  panel + short-range batched PointLights. It also **never casts shadows**
  (no area light in three.js does).
- **Shadows: SpotLight = 1 map; PointLight = a 6-face cube** sampled per
  pixel. Measured on a real scene: 8 point-shadow cubes = 240→75fps; the same
  coverage with cone-down SpotLights was free. A solid default rule: shadows
  only via SpotLight; PointLights only illuminate (fake occlusion by
  shortening `distance` so the light dies before the floor).
- Shadow sampling (PCF) also lives in every program per shadow light, and each
  shadow map eats a texture slot (WebGL fallback budget: 16 per shader; LTC
  eats 2). Set a scene-wide shadow budget (the scene above runs on 3) and
  treat raising it as an architectural decision.
- `light.layers` does **NOT** filter illumination per object, only camera
  visibility. The only way to occlude light is `castShadow`.

## 3. What actually forks a shader program (r185, verified in source)

Programs dedupe by the **generated WGSL string** (`Pipelines.js` keys
`programs.vertex/fragment` by code). `RenderObject.getMaterialCacheKey()`:

- **Uniforms (do NOT fork)**: `color` (any `Color`), `roughness`, `metalness`,
  `emissive` color/intensity, `envMapIntensity`, `normalScale`, `anisotropy`
  *value*... every scalar collapses to zero/non-zero in the key. Two hundred
  materials differing only in tint = one program. For tint variants use
  `material.clone()`: it shares textures AND the program.
- **DO fork a program**: presence of each map (`map`, `normalMap`,
  `alphaMap`...), `side` (kept exact; DoubleSide flips normals in code),
  scalar zero↔non-zero transitions (`clearcoat: 0` vs `0.001`!), material
  class features (Physical's clearcoat/anisotropy/sheen), skinning,
  `InstancedMesh` (its uuid is in the key, so every InstancedMesh gets its
  own program), vertex colors, flat shading, fog on/off, and
  `object.receiveShadow` (an object-level flag: the same material splits
  into with/without shadow-sampling variants).
- Because `receiveShadow` is per-object, a material shared across objects
  with **mixed** flags forks into both variants. Homogenizing the flag
  collapsed the split on this scene: −4 programs (78→74), −3 pipelines,
  deterministic in 2/2 runs, no new contact shadows in the captured views.
  Census-only win — the compile-time effect fell below the noise floor.
  Treat it as variant-count hygiene, not a compile-time lever.
- **Fork a pipeline but not a program**: `transparent`/blending, cull mode,
  depth state, MRT target formats, sample count.
- The classic WebGL trick of giving every material a dummy 1×1 white map to
  homogenize permutations is **contraindicated** in the node system: it saves
  ~1 program and adds texture fetches to hundreds of draws.
- `MeshPhysicalMaterial` compiles the heaviest programs (50-65KB WGSL each).
  Use it only where clearcoat/aniso visibly pays (lacquered wood, glass);
  matte surfaces read identically on `MeshStandardMaterial` (a barely-visible
  `anisotropy: 0.3` on a matte floor cost a whole heavy program).

## 4. Environment capture (CubeCamera) multiplies variants

- Every material visible during a cube capture compiles a **second pipeline
  variant** (different render context, no MRT). Hide from the capture whatever
  won't be missed in reflections: `userData.hideFromEnv` pattern + hide
  emissive-of-color meshes (`emissiveIntensity` defaults to 1 on ALL
  materials; filter by emissive color+intensity, or filtering "intensity > 0"
  hides everything and the PMREM comes out black).
- Lit lamp shades / glowing panels captured up close become giant smeared
  discs with false parallax in glossy floors. Hide them during capture too,
  and dim lights whose glow would be reflected where real occluders (a bar
  counter) should block them: envMaps know nothing about occlusion.
- A `scene.overrideMaterial` set only during the cube-capture pass collapses
  the capture-pass variants — measured: 78→71 programs (−7), programsSize
  −23%, deterministic in 2/2 runs, compile segment ~−34% (~−4s) and total
  startup ~−30%. Cheaper visually than feared: at a floor-grazing angle where
  the reflection would show it, override vs control captures were
  indistinguishable. Other angles and higher resolutions untested, so eyeball
  it before shipping — but "the reflection loses material nuance" was not
  observed here.

## 5. Lifecycle: the leak that slows every reload

- **`renderer.dispose()` does NOT destroy the `GPUDevice`.** Each hot-reload
  (Vite HMR remount) or SPA remount leaks a whole device in the browser's GPU
  process, which is shared and survives page reloads. Symptom: startups get
  progressively slower, GPU process memory balloons (seen at 900MB). Fix:
  `renderer.backend?.device?.destroy?.()` in your dispose path, and call
  dispose from **`pagehide`** too (F5 never runs framework unmount hooks).
  Killing the tab does not reset the GPU process; restarting the browser does.
- Give components a real `dispose()` (stop the rAF loop, remove listeners,
  destroy GUI, then renderer + device) and call it on unmount AND pagehide
  (idempotent, since both can fire).

## 6. The WebGL2 fallback is a different performance world

- WebGPU needs a **secure context**: `http://` over LAN silently falls back to
  WebGL2 (localhost is fine). Log the real backend
  (`renderer.backend.isWebGPUBackend`) and surface it in your debug UI; people
  WILL test over LAN or in browsers without WebGPU and report "it's slow".
- Firefox/private windows: private mode may disable GPU acceleration
  (anti-fingerprinting): don't tune for numbers measured there.
- The fallback **does have async pipeline compile**: three's WebGL2 backend
  routes `compileAsync` through `KHR_parallel_shader_compile`, and using it
  matters as much as on WebGPU. Measured with WebGL2 forced: with
  compileAsync the main thread froze ~3.7-4.4s during compile; the
  synchronous path froze it a sustained ~9-10s (+4.6 to +6.2s of extra
  block, 2/2 runs, two independent signals). As on WebGPU, async buys a
  live thread, not less wall-clock — the cost reorganizes across segments.
- The fallback still compiles slower overall (2-4× the WebGPU startup) and
  has far less headroom. Budget a "fallback diet" branch: shadows off
  **before** compiling (programs are born without PCF code; measured
  35→60fps), hide big translucent overdraw geometry (visible light cones:
  −35% load time), cap pixelRatio (DPR 2 sank a HiDPI laptop to 16fps;
  1.0-1.5 is the range).
- Modern headless Chrome exposes WebGPU by default. To test the fallback,
  hide the API: `page.evaluateOnNewDocument(() => Object.defineProperty(
  navigator, 'gpu', { get: () => undefined }))`.

## 7. Measure, don't guess

- Instrument startup segments (assets / env capture / compile / warm-up) with
  `performance.now()` marks; print one JSON line and stash it on `window`.
  Any machine can then report its real breakdown by pasting one console line.
- A `?sin=piece1,piece2` query param that skips scene pieces turns load-time
  attribution into a 5-minute bisection.
- Headless puppeteer with `--use-angle=d3d11` gets the **real GPU** via Dawn
  (verify with `adapter.info` from a secure page; `about:blank` has no
  `navigator.gpu`). Compilation is CPU-side (Dawn/DXC): headless compile
  times match the user's machine.
- Measure FPS deltas with a calibrated noise floor (5 unchanged runs first);
  single-run FPS on a busy machine swings ±7fps. Sequential runs only: two
  headless Chromes compete for CPU and poison timings.
- When hunting a light/material culprit: automated bisection (toggle each
  light, read back pixel brightness of the suspect region) beats hypotheses:
  it found in one pass what four educated guesses missed.
