# 🗺️ Architecture Plan: Netasibas MAP SDK

> ⚠️ **This project is in its infancy.** Nothing described below exists yet unless explicitly marked as implemented. Treat this document as a **vision** and a **roadmap**, not a description of working software.

**Currently implemented:** ✅ Raster test layer (Phase 1 — renders tile coordinates as text on each tile)

**Everything else:** 📋 Planned. Aspirational. Not built. May change.

---

A browser-based geospatial visualization SDK that _someday_ aims to extend MapLibre GL JS with advanced rendering capabilities. Hybrid Rust+WASM / TypeScript architecture for maximum performance.

## 1. Overview 📌

**What it will be:** A MapLibre GL JS extension SDK providing custom layers, sources, and protocols for advanced geospatial visualization.

**What it will do (someday):**

- Fetches tile-based geospatial data from servers (small chunks, many requests)
- Parses and processes tile data using Rust compiled to WASM
- Renders graphics via WebGL2 through MapLibre's GL context
- Exposes a reactive, framework-interop API to consumers

**Key constraint:** MapLibre owns the WebGL2 context and render loop. The SDK is a passive participant that renders only when called.

**Terminology note:** Three terms are used precisely:

- **TypeScript** — the source language we write (types, classes, interfaces). Does not exist at runtime.
- **JavaScript (JS)** — the compiled output of TypeScript; the code that actually executes in the browser.
- **JS runtime** — the browser's JavaScript execution environment (V8/SpiderMonkey/JavaScriptCore), as opposed to the **WASM runtime** where compiled Rust executes. The two runtimes share memory through `ArrayBuffer`s but cannot directly call each other's host APIs.

When the doc says "TypeScript handles X", it means "the TypeScript source code, once compiled to JavaScript, handles X at runtime."

## 2. Architecture 🏗️

> 🚧 This diagram represents the target architecture. Only the raster test layer portion is real today.

```
┌─────────────────────────────────────────────────────────────┐
│  Consuming Application                                      │
│  import { RasterTestLayer } from '@netasibas/map-sdk'       │
├─────────────────────────────────────────────────────────────┤
│  SDK API Layer (TypeScript)                                 │
│  ┌────────────────┐  ┌────────────────┐  ┌───────────────┐  │
│  │ Layer classes  │  │ Source classes │  │ Reactive      │  │
│  │ (CustomLayer   │  │ (addProtocol   │  │ state         │  │
│  │  Interface)    │  │  /addSource)   │  │ (@preact/     │  │
│  │                │  │                │  │  signals)     │  │
│  └───────┬────────┘  └───────┬────────┘  └───────────────┘  │
│          │                   │                              │
│  ┌───────▼───────────────────▼───────────────────────────┐  │
│  │  ResourceManager (singleton per map)                  │  │
│  │  - GL program cache    - Texture pool                 │  │
│  │  - WASM module init    - Buffer pool                  │  │
│  └───────┬───────────────────────────────────────────────┘  │
│          │                                                  │
├──────────┼──────────────────────────────────────────────────┤
│  WASM Layer (Rust → wasm-bindgen)                           │
│  ┌───────▼───────────────────────────────────────────────┐  │
│  │  - PBF/MVT/glTF tile parsing                          │  │
│  │  - Terrain mesh generation                            │  │
│  │  - Spatial indexing (R-tree, quadtree)                │  │
│  │  - Coordinate transforms & clipping                   │  │
│  │  - Returns opaque handles to parsed tile data         │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  External WASM (Google Draco decoder)                       │
│  - Loaded separately by JavaScript; not part of our WASM    │
│  - Used for Draco-compressed mesh data in glTF tiles        │
├─────────────────────────────────────────────────────────────┤
│  MapLibre GL JS (host)                                      │
│  - Owns WebGL2 context and render loop                      │
│  - Calls custom layer render() / onAdd() / onRemove()       │
└─────────────────────────────────────────────────────────────┘
```

## 3. Key Design Decisions 🎯

| Decision            | Choice                                                                                        | Rationale                                                                                            |
| ------------------- | --------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| Render loop         | Passive — `render()` only, `map.triggerRepaint()` for animation                               | MapLibre owns the loop; custom layers must not run their own `requestAnimationFrame`                 |
| GL context          | Share MapLibre's via `onAdd(map, gl)`                                                         | Single context; no second canvas or context                                                          |
| Layer strategy      | Multiple `CustomLayerInterface` instances, one `ResourceManager` shared                       | Allows z-ordering with native MapLibre layers; shared resources avoid duplication                    |
| Custom sources      | `addProtocol()` for fetch interception; `addSourceType()` for full tile lifecycle             | Start simple, add complexity as needed                                                               |
| WASM boundary       | Rust computes → returns opaque handle → JavaScript reads typed array view → `gl.bufferData()` | WASM cannot call GL; near-zero-copy with strict lifetime rules                                       |
| Reactive state      | `@preact/signals` for user-facing properties                                                  | Auto-tracking, batching, framework-interop, ~1.6KB                                                   |
| Language (SDK code) | TypeScript (strict)                                                                           | Type safety for maintainability; compiles to identical runtime code                                  |
| Language (WASM)     | Rust via wasm-bindgen                                                                         | Best WASM toolchain, zero-cost abstractions, memory safety                                           |
| Build (TypeScript)  | Vite library mode                                                                             | Dev server for live map testing + tree-shaking. Rolldown (Vite 8+) optional for faster builds.       |
| Build (Rust)        | wasm-pack                                                                                     | Standard Rust→WASM build pipeline                                                                    |
| Output format       | ESM-only                                                                                      | `exports` map with `sideEffects: false` for optimal tree-shaking. CJS dropped (browser-only target). |
| WebGL version       | WebGL2 only (for now)                                                                         | Universal browser support; WebGPU deferred to future phase                                           |

## 4. Rust/WASM Scope 🦀

> 🚧 No Rust code exists yet. This section defines what Rust _will_ handle once the WASM module is built.

### What Rust will handle (CPU-heavy computation)

| Component                           | Why Rust                                                                        |
| ----------------------------------- | ------------------------------------------------------------------------------- |
| PBF/MVT tile parsing                | Binary protocol decoding is 10-50x faster in Rust (WASM) than in the JS runtime |
| glTF parsing                        | Complex format parsing; Rust crate `gltf` available                             |
| Terrain mesh generation             | DEM → triangle mesh involves significant floating-point computation             |
| Spatial indexing (R-tree, quadtree) | Tree construction and range queries are data-structure-heavy                    |
| Coordinate transforms               | Projection math (EPSG conversions, mercator, globe)                             |
| Geometry clipping                   | Polygon line-clipping against tile boundaries                                   |

### Why not pure JavaScript with Web Workers?

(TypeScript compiles to JavaScript. The comparison below is between JavaScript executed by V8/SpiderMonkey/JavaScriptCore and Rust compiled to WASM.)

- JavaScript is 5-50x slower than compiled WASM for binary parsing and math-heavy workloads
- Web Workers help with parallelism but don't improve per-thread throughput
- WASM can also run inside Web Workers for parallel parsing (best of both worlds)

### What Rust does NOT handle

- **WebGL2 rendering** — WASM cannot call GL functions; all draw calls must go through the JS runtime
- **Network requests** — `fetch()`, `AbortController`, streaming are JS-runtime APIs
- **MapLibre API interaction** — `CustomLayerInterface` is a JS-runtime API contract
- **User-facing configuration** — reactive state, events, layer options
- **Draco mesh decoding** — uses Google's official Draco WASM decoder (same as Three.js/Cesium), loaded separately by JavaScript. Not part of our Rust WASM module. Results are passed to our pipeline as typed arrays.

### External dependencies (not our code)

| Dependency                | Purpose                                          | How integrated                                                       |
| ------------------------- | ------------------------------------------------ | -------------------------------------------------------------------- |
| Google Draco WASM decoder | Decode Draco-compressed meshes in glTF tiles     | Loaded by JavaScript; results passed to our pipeline as typed arrays |
| MapLibre GL JS            | Host map renderer, GL context owner, render loop | Peer dependency; SDK registers custom layers/sources                 |

## 5. JavaScript Runtime Scope 🟨

> 🚧 Only the basic raster test layer JS exists. The full scope below is planned.

### What the SDK's JavaScript code will handle (everything touching the browser/DOM/GPU)

| Component            | Why it must run in the JS runtime                                            |
| -------------------- | ---------------------------------------------------------------------------- |
| WebGL2 rendering     | Only the JS runtime can call GL functions; WASM cannot access the GL context |
| MapLibre integration | `CustomLayerInterface`, `addProtocol`, `addSourceType` are JS-runtime APIs   |
| Network/fetch layer  | `fetch()`, `AbortController`, request scheduling, caching                    |
| User-facing API      | Reactive state, events, configuration objects                                |
| Resource management  | GL program cache, texture pool, buffer pool lifecycle                        |
| Camera/viewport sync | Reading MapLibre's transform, constructing projection matrices               |

### The WASM→JS→GPU data boundary (planned)

```
Rust/WASM:    "Here's the parsed vertex data at this pointer"
     ↓
JavaScript:   "Thanks, I'll upload it to the GPU immediately and draw it"
```

JavaScript (the compiled output of our TypeScript) creates a typed array view into WASM's linear memory and uploads to the GPU. This is near-zero-copy on the JS↔WASM boundary, but **comes with strict lifetime rules**.

```ts
// JS side (compiled from TypeScript): reading WASM output and uploading in the same synchronous block
const { ptr, len } = wasm.parseTile(tileBytes);
const vertices = new Float32Array(wasm.memory.buffer, ptr, len);
gl.bufferData(gl.ARRAY_BUFFER, vertices, gl.STATIC_DRAW);
wasm.freeTile(ptr); // immediately return ownership to WASM
```

#### Critical rule: views into `wasm.memory.buffer` are ephemeral

When WASM memory grows (any `alloc` or any function that may allocate internally), the underlying `ArrayBuffer` is **detached** and **all existing views become invalid and may read garbage**.

**Mandatory invariants:**

1. **Never hold a WASM memory view across any other WASM call.** A second call may grow memory and detach your view.
2. **Never hold a WASM memory view across an `await`.** Other tasks may invoke WASM and detach your buffer.
3. **Use views in the same synchronous block** they were created: create view → upload to GPU → release.
4. **If you must persist data on the JS side**, copy it: `new Float32Array(view)` allocates a fresh JS-owned buffer. This costs one memcpy but is safe.
5. **Memory ownership** is explicit. WASM owns memory allocated by `alloc()`. JavaScript code must call `free()` (or the equivalent) when done. Use a `using` declaration or `try`/`finally` to guarantee cleanup.

#### Recommended pattern: opaque handles

Instead of exposing raw pointers, WASM returns an opaque numeric handle. JavaScript code calls `wasm.getVertexView(handle)` which returns a fresh view _immediately before use_. This makes the lifetime issue impossible to forget:

```ts
const handle = wasm.parseTile(tileBytes); // returns u32 handle
try {
  const vertView = wasm.getVertexView(handle); // fresh view, used immediately
  gl.bufferData(gl.ARRAY_BUFFER, vertView, gl.STATIC_DRAW);
  const idxView = wasm.getIndexView(handle); // another fresh view
  gl.bufferData(gl.ELEMENT_ARRAY_BUFFER, idxView, gl.STATIC_DRAW);
} finally {
  wasm.disposeTile(handle);
}
```

This pattern is enforced by the SDK's WASM wrapper module — consumers of the SDK never see raw pointers.

## 6. Data Flow: Tile Loading Pipeline 🔄

> 🚧 Not implemented. This is the target pipeline design.

```
1. MapLibre requests tile (z,x,y)
   │
2. SDK's custom source.loadTile() or addProtocol handler
   │
3. fetch() on main thread → raw bytes (ArrayBuffer)
   │
4. Pass bytes to WASM: wasm.parseTile(bytes)
   → returns opaque handle (u32)
   │
5. JavaScript reads from WASM memory via handle:
   new Float32Array(wasm.memory.buffer, wasm.vertexPtr(handle), wasm.vertexLen(handle))
   │
6. Upload to GPU:
   gl.bufferData(gl.ARRAY_BUFFER, float32View, gl.STATIC_DRAW)
   │
7. render() draws with cached program + buffers
   │
8. On tile unload:
   wasm.disposeTile(handle)  // free WASM memory
   gl.deleteBuffer(buffer)   // free GPU memory
```

### WASM execution context: main thread vs Worker

WASM can run on the main thread _or_ inside a Web Worker. The choice affects how data reaches the GPU, because the GL context lives only on the main thread.

| Scenario                        | Where WASM runs | Data path                                                                                                          |
| ------------------------------- | --------------- | ------------------------------------------------------------------------------------------------------------------ |
| **Light parsing (default)**     | Main thread     | View into `wasm.memory.buffer` → `gl.bufferData()` in same sync block                                              |
| **Heavy parsing (large tiles)** | Worker thread   | WASM parses in worker → `postMessage` with `Transferable` `ArrayBuffer` → main thread receives → `gl.bufferData()` |

**Worker pipeline rules:**

1. The worker runs its own WASM instance with its own linear memory.
2. The worker must **copy** the parsed result out of WASM memory (`new Uint8Array(view).buffer`) before transferring, since the WASM memory itself is not transferable.
3. The worker `postMessage`s the copied `ArrayBuffer` with it listed in the transfer list — zero-copy across the thread boundary.
4. The main thread receives the `ArrayBuffer`, wraps it in the appropriate typed array view, and uploads to GPU.

This avoids the "WASM memory view across await" footgun entirely, because the data is already a JS-owned `ArrayBuffer` by the time it crosses to main.

**Default recommendation:** Start with WASM on the main thread for simplicity. Move parsing to a worker only if profiling shows parse time blocks the main thread for >4ms per tile.

### Tile fetching patterns (from MapLibre, deck.gl, CesiumJS)

| Concern                   | Approach                                                                                                  |
| ------------------------- | --------------------------------------------------------------------------------------------------------- |
| **Request deduplication** | Single in-flight request per tile key `(z,x,y)`                                                           |
| **Priority queue**        | Distance-from-viewport-center + zoom-level; re-evaluated on camera move                                   |
| **Cancellation**          | `AbortController` per request; cancel out-of-viewport tiles on camera change                              |
| **Memory cache**          | LRU cache keyed by tile ID; configurable max size; evict on memory pressure                               |
| **Worker parsing**        | Optional — see _WASM execution context_ above. Use only when main-thread parse time exceeds frame budget. |
| **Fetch on main**         | `fetch()` stays on main thread (workers have CORS limitations); send raw bytes to WASM/worker for parse   |

### Frame budget: cooperative GPU upload

A single camera move can produce dozens of newly-loaded tiles, each requiring vertex/index buffer uploads and texture uploads. Uploading them all in one frame causes visible jank.

The SDK enforces a per-frame upload budget to keep the main thread responsive:

| Budget                     | Default | Purpose                                                  |
| -------------------------- | ------- | -------------------------------------------------------- |
| **Max upload bytes/frame** | 4 MB    | Caps total `bufferData`/`texImage2D` work per frame      |
| **Max tiles/frame**        | 8       | Caps the number of tile-level uploads regardless of size |
| **Max parse time/frame**   | 4 ms    | Caps WASM parse work before yielding                     |

**Scheduling pattern:**

```ts
// Inside render() — called once per frame by MapLibre
const budget = new FrameBudget(this.config.uploadBudget);
while (budget.hasRoom() && uploadQueue.length > 0) {
  const tile = uploadQueue.shift();
  budget.charge(tile.size);
  this.uploadTile(tile); // gl.bufferData / gl.texImage2D
}
if (uploadQueue.length > 0) {
  map.triggerRepaint(); // keep draining next frame
}
this.drawAll(gl, options); // draw whatever is uploaded
```

**Trade-off:** Tiles appear progressively over several frames during heavy camera moves, rather than all at once after a stall. This is the standard pattern used by MapLibre's `SourceCache` and deck.gl's tile layer.

**Parse scheduling** (when WASM runs on main thread):

- Pull from the highest-priority tile in the request queue
- Run `wasm.parseTile()` only if remaining parse budget permits
- If the parse itself exceeds budget (large tile), accept the overrun _once_ and adjust budget for the next frame
- For sustained overruns, the SDK emits a `'frame-budget-exceeded'` event so consumers can react (lower zoom, simpler style, etc.)

## 7. State Management

Three-tier model matching the access patterns of each state type:

```
┌─────────────────────────────────────┐
│ Consumer-facing (reactive)          │
│ signal: layer.visible              │
│ signal: layer.opacity              │
│ signal: terrain.exaggeration       │
│ signal: source.url                 │
│                                     │
│ → Auto-batched, glitch-free         │
│ → Framework interop (React/Vue)     │
│ → @preact/signals (~1.6KB)         │
├─────────────────────────────────────┤
│ Internal (mutable, hot path)        │
│ camera matrix (per frame)           │
│ GPU buffer state                    │
│ WASM memory pointers                │
│ shader uniform values               │
│                                     │
│ → No reactivity overhead            │
│ → Updated every frame at 60fps      │
├─────────────────────────────────────┤
│ Events (one-off notifications)      │
│ 'tileload', 'tileerror'            │
│ 'click', 'hover'                   │
│ 'contextlost', 'contextrestored'   │
│                                     │
│ → Event emitter pattern             │
│ → Decoupled from rendering          │
└─────────────────────────────────────┘
```

### Why this hybrid approach?

- **Signals for consumer state**: Automatic dependency tracking, glitch-free batched updates, and framework interop without the ceremony of Redux-style unidirectional flow.
- **Mutable internals for hot path**: Camera transforms and GPU state change 60x/sec. Reactivity overhead (dependency tracking, scheduling) is wasteful here.
- **Events for notifications**: Tile load/error, user interactions — discrete occurrences that don't need reactive tracking.

### Rejected alternatives

| Approach                          | Why rejected                                                                                                   |
| --------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| **Unidirectional (Redux-like)**   | Dispatch → render round-trip too slow for 60fps camera updates. Excessive ceremony for rendering state.        |
| **ECS (Entity-Component-System)** | Unfamiliar to web/GIS devs. Wrong abstraction level for tile-based rendering. Over-engineered.                 |
| **Pure observer/events**          | No batching — every property change triggers immediate work. Easy to get stale state. Hard to trace causality. |

## 8. Monorepo Structure

```
map-sdk/
├── packages/
│   ├── core/                        # Main SDK package (@netasibas/map-sdk)
│   │   ├── src/
│   │   │   ├── layers/              # CustomLayerInterface implementations
│   │   │   │   ├── terrain-layer.ts
│   │   │   │   ├── 3dtiles-layer.ts
│   │   │   │   └── vector-overlay-layer.ts
│   │   │   ├── sources/            # Custom source & protocol implementations
│   │   │   │   ├── protocols/
│   │   │   │   │   └── cog-protocol.ts
│   │   │   │   └── sources/
│   │   │   │       └── custom-tile-source.ts
│   │   │   ├── gl/                  # Thin WebGL2 helpers
│   │   │   │   ├── device.ts        # GL context wrapper
│   │   │   │   ├── program.ts       # Shader program compile + cache
│   │   │   │   ├── buffer.ts        # Vertex/index buffer management
│   │   │   │   ├── texture.ts       # Texture pool + lifecycle
│   │   │   │   └── vao.ts           # VAO management
│   │   │   ├── state/               # Reactive state primitives
│   │   │   │   └── signals.ts       # @preact/signals wrappers
│   │   │   ├── resource-manager.ts  # Singleton per-map: WASM, GL cache, texture pool
│   │   │   ├── tile-queue.ts        # Priority-based tile request scheduler
│   │   │   ├── tile-cache.ts        # LRU tile memory cache
│   │   │   └── index.ts
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── vite.config.ts
│   │
│   ├── wasm-core/                   # Rust package (@asm/wasm-core)
│   │   ├── src/
│   │   │   ├── lib.rs               # wasm-bindgen exports
│   │   │   ├── parsers/             # PBF, MVT, glTF
│   │   │   │   ├── mod.rs
│   │   │   │   ├── pbf.rs
│   │   │   │   ├── mvt.rs
│   │   │   │   └── gltf.rs
│   │   │   ├── mesh/                # Terrain mesh generation
│   │   │   │   ├── mod.rs
│   │   │   │   └── terrain.rs
│   │   │   ├── spatial/             # R-tree, quadtree, clipping
│   │   │   │   ├── mod.rs
│   │   │   │   ├── rtree.rs
│   │   │   │   └── quadtree.rs
│   │   │   └── transform/           # Coordinate transforms
│   │   │       ├── mod.rs
│   │   │       └── projection.rs
│   │   ├── Cargo.toml
│   │   └── tests/
│   │
│   └── playground/                    # Demo/test application
│       ├── src/
│       │   ├── main.ts
│       │   └── style.css
│       ├── index.html
│       └── vite.config.ts            # Vite app mode for live map testing
│
├── Cargo.toml                        # Rust workspace
├── package.json                      # pnpm workspace root
├── pnpm-workspace.yaml
└── tsconfig.base.json
```

## 9. Build & Packaging

### TypeScript source → JavaScript output (packages/core)

- **Bundler:** Vite library mode (Rollup by default; Rolldown in Vite 8+ is optional)
- **Output:** ESM-only (browser-only target; no CJS needed)
- **Tree-shaking:** `sideEffects: false` in package.json; multiple entry points via `exports` map
- **Dev server:** Vite dev mode serves `packages/playground` for live map rendering during development
- **GLSL shaders:** vite-plugin-glsl or raw string imports

### Rust (packages/wasm-core)

- **Toolchain:** wasm-pack with wasm-bindgen
- **Target:** `wasm32-unknown-unknown`
- **Output:** WASM binary + JS glue code
- **Optimization:** `--release` profile with LTO and `opt-level = "z"` for minimal binary size

### Package exports

```json
{
  "name": "@netasibas/map-sdk",
  "type": "module",
  "sideEffects": false,
  "exports": {
    ".": "./dist/index.js",
    "./layers": "./dist/layers/index.js",
    "./sources": "./dist/sources/index.js",
    "./wasm": "./dist/wasm/asm_wasm_core.wasm"
  }
}
```

### Consumer usage

```ts
import { TerrainLayer, ThreeDTilesLayer } from "@netasibas/map-sdk/layers";
import { CogProtocol } from "@netasibas/map-sdk/sources";
import { initWasm } from "@netasibas/map-sdk";

await initWasm(); // loads WASM module once

map.addProtocol("cog", CogProtocol);
map.addLayer(new TerrainLayer({ exaggeration: 1.5 }));
map.addLayer(new ThreeDTilesLayer({ url: "https://..." }), "buildings");
```

## 10. MapLibre Integration Constraints

### CustomLayerInterface contract

| Method                   | Called when                 | Purpose                                 |
| ------------------------ | --------------------------- | --------------------------------------- |
| `onAdd(map, gl)`         | Layer added to map          | Init GL resources, WASM module, shaders |
| `render(gl, options)`    | Every frame                 | Draw using MapLibre's GL context        |
| `prerender(gl, options)` | Every frame (before render) | Offscreen FBO passes                    |
| `onRemove(map, gl)`      | Layer removed               | Clean up GL resources, WASM memory      |

### Critical constraints

| Constraint                        | Implication                                                                                          |
| --------------------------------- | ---------------------------------------------------------------------------------------------------- |
| **MapLibre owns the render loop** | Never call `requestAnimationFrame`. Call `map.triggerRepaint()` when you need a new frame.           |
| **Single GL context**             | Use `map.painter.context.gl` passed in `onAdd`. No second canvas or context.                         |
| **Premultiplied alpha blending**  | MapLibre sets `blendFunc(ONE, ONE_MINUS_SRC_ALPHA)`. Override with `blendFuncSeparate` if needed.    |
| **Depth buffer**                  | `"3d"` renderingMode shares depth with native layers. `"2d"` mode needs prerender FBO trick.         |
| **Context loss**                  | Must handle `webglcontextlost` / `webglcontextrestored`. Recreate all GL resources on restore.       |
| **Projection support**            | Must handle both mercator and globe projections. Use `getProjectionData()` for tile-based rendering. |
| **Not serializable**              | Custom layers cannot be expressed in MapLibre's style JSON. SDK manages its own config persistence.  |
| **GL state not preserved**        | Cannot assume any GL state beyond blending/depth setup. Set everything you need before drawing.      |

### Custom sources: addProtocol vs addSourceType

| Approach          | When to use                                                                                                                                            |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `addProtocol()`   | Custom fetch/decode for a URL scheme (e.g., `cog://`, `pmtiles://`). Lighter weight.                                                                   |
| `addSourceType()` | Full custom tile lifecycle with custom `loadTile`, `abortTile`, `unloadTile`. Heavier but integrates with MapLibre's tile cache and viewport requests. |

**Recommendation:** Start with `addProtocol()` for custom fetch. Use `addSourceType()` only when you need custom tile lifecycle management beyond what standard sources provide.

## 11. Layer Strategy

### Multiple CustomLayerInterface instances with shared ResourceManager

Each distinct layer type registers as a separate custom layer. This allows consumers to:

- Interleave SDK layers with native MapLibre layers using `beforeId`
- Toggle individual layers via `map.setLayoutProperty(id, 'visibility', ...)`
- Remove individual layers via `map.removeLayer(id)`

A shared `ResourceManager` (singleton per map instance) provides:

- GL program cache (compiled shaders shared across layers)
- WASM module instance (loaded once, shared)
- Texture pool (shared atlas for icons/patterns)
- Buffer pool (reusable GPU buffers)

### Layer grouping

| Layer type          | Custom layer or group?   | Why                                                 |
| ------------------- | ------------------------ | --------------------------------------------------- |
| Terrain + 3D Tiles  | **Single grouped layer** | Share depth buffer, terrain mesh, elevation queries |
| Raster overlay      | **Separate layer**       | Independent z-ordering, simple rendering            |
| Vector tile overlay | **Separate layer**       | Interleaves naturally with native labels            |

### Resource sharing pattern

```ts
// ResourceManager is created per-map and shared across layers
class ResourceManager {
  private static instances = new WeakMap<Map, ResourceManager>();

  static for(map: Map): ResourceManager {
    let rm = this.instances.get(map);
    if (!rm) {
      rm = new ResourceManager(map.painter.context.gl);
      this.instances.set(map, rm);
    }
    return rm;
  }

  programCache: ProgramCache;
  wasmModule: AsmWasm;
  texturePool: TexturePool;
  bufferPool: BufferPool;
}
```

## 12. Error Handling & Degradation

A production SDK must define explicit behavior for every failure mode. Silent failures are unacceptable.

### Failure modes & responses

| Failure                | Detection                                             | Response                                                                                                                       |
| ---------------------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| **WASM fails to load** | `initWasm()` rejects (network, CSP, corrupted module) | Throw a typed `WasmInitError`. SDK is unusable; consumer must handle. No fallback to JavaScript parser (not in scope).         |
| **WASM runtime panic** | `RuntimeError` from WASM call                         | Catch in JavaScript wrapper, emit `'wasm-error'` event with tile context, mark tile as failed, do not crash the map.           |
| **Tile fetch fails**   | `fetch()` rejects or non-2xx                          | Retry with exponential backoff (max 3 attempts). After exhaustion, emit `'tileerror'` and skip the tile. Other tiles continue. |
| **Tile parse fails**   | WASM returns error code                               | Emit `'tileerror'` with tile coords + reason. Tile shows as blank/missing.                                                     |
| **GPU out of memory**  | `gl.getError()` returns `OUT_OF_MEMORY` after upload  | Evict oldest tiles from texture/buffer pool, retry once. If still failing, emit `'gpu-memory-pressure'` and pause new uploads. |
| **WebGL2 unavailable** | `canvas.getContext('webgl2')` returns null            | MapLibre fails first; SDK does nothing. Document the minimum requirement.                                                      |
| **Context loss**       | `webglcontextlost` event                              | Cancel in-flight uploads, mark all GL handles invalid. MapLibre handles the event; SDK reacts via the same hook.               |
| **Context restored**   | `webglcontextrestored` event                          | Recreate program cache, re-upload textures from cache, re-upload tile buffers from WASM (re-parse if data was discarded).      |

### Error contract

All SDK errors are typed and extend a common `AsmSdkError` base class:

```ts
class AsmSdkError extends Error {
  readonly code: ErrorCode; // enum: WASM_INIT, TILE_FETCH, TILE_PARSE, GPU_OOM, ...
  readonly context?: Record<string, unknown>; // tile coords, URL, etc.
  readonly cause?: unknown;
}
```

This lets consumers `switch(err.code)` rather than parse error messages.

### Degradation guarantees

- A failure in **one tile** never affects **other tiles** or the map's responsiveness.
- A failure in **one layer** never affects **other layers**.
- The SDK never throws synchronously from `render()` — all failures during render are caught and logged via the observability hook.
- The SDK never crashes the MapLibre instance.

### Out of scope

- Pure-JavaScript parsing fallback if WASM fails (would double the code surface).
- WebGL1 fallback (MapLibre v5+ requires WebGL2).
- Automatic recovery from sustained GPU memory pressure beyond simple eviction.

## 13. Observability

The SDK is consumed by other teams. It must expose performance and diagnostic data without forcing a specific logging or metrics framework.

### Hooks (consumer-pluggable)

```ts
interface AsmSdkObservability {
  // Structured logging — consumer routes to console, datadog, sentry, etc.
  logger?: (level: 'debug' | 'info' | 'warn' | 'error', event: string, data?: object) => void

  // Performance counters — consumer reads these on a timer or on demand
  metrics?: (metric: MetricName, value: number, tags?: Record<string, string>) => void

  // Optional tracing — for spans like "parseTile", "uploadTile", "renderLayer"
  trace?: (span: string, durationMs: number, attrs?: Record<string, unknown>) => void
}

const map = new Map({ ... })
configureAsmSdk({ observability: { logger: myLogger, metrics: myMetrics } })
```

### Metrics emitted

| Metric                      | Type                     | Description                                                                                                         |
| --------------------------- | ------------------------ | ------------------------------------------------------------------------------------------------------------------- |
| `asm.tile.fetch.duration`   | histogram (ms)           | Time from `fetch()` start to bytes available                                                                        |
| `asm.tile.parse.duration`   | histogram (ms)           | WASM parse time per tile                                                                                            |
| `asm.tile.upload.duration`  | histogram (ms)           | GPU upload time per tile                                                                                            |
| `asm.tile.cache.hit`        | counter                  | Tile served from memory cache                                                                                       |
| `asm.tile.cache.miss`       | counter                  | Tile not in cache, fetched                                                                                          |
| `asm.tile.cache.evict`      | counter                  | Tile evicted from LRU                                                                                               |
| `asm.tile.error`            | counter (tagged by kind) | Tile failed to fetch/parse                                                                                          |
| `asm.frame.draw.duration`   | histogram (ms)           | Time spent in `render()` per frame                                                                                  |
| `asm.frame.budget.exceeded` | counter                  | Frame ran over upload/parse budget                                                                                  |
| `asm.gpu.memory.bytes`      | gauge                    | Estimated GPU memory in use by SDK (sum of uploaded buffer/texture sizes; WebGL2 does not expose actual GPU memory) |
| `asm.wasm.memory.bytes`     | gauge                    | WASM linear memory size                                                                                             |

### Debug overlay (optional, dev-only)

The SDK exports an optional `<AsmDebugOverlay>` (vanilla DOM, no framework) that renders a stats HUD showing current values of the gauges and histograms. It subscribes to the same observability hook internally. Behind a separate entry point `@netasibas/map-sdk/debug` so it tree-shakes out of production bundles.

### Default behavior

If no observability hook is configured, the SDK is **silent** — no `console.log`, no stats, no overhead. This is non-negotiable for an SDK.

## 14. Implementation Priority

### Phase 1: Raster Test Layer

1. Monorepo setup (pnpm workspace, Vite, TypeScript configs)
2. `RasterTestLayer` — renders tile coordinates (z/x/y) as text on each tile using a Canvas-generated raster protocol
3. Playground app with live MapLibre map to verify visually

### Phase 2: Foundation

4. Thin WebGL2 helpers (`gl/` module: device, program, buffer, texture, vao)
5. ResourceManager singleton
6. "Hello world" custom layer (validates GL context sharing, render loop, resource lifecycle)

### Phase 3: Data Pipeline

7. Tile request scheduler with priority queue and cancellation
8. LRU tile cache
9. `addProtocol()` handler for custom fetch
10. WASM module init + first parser (PBF or MVT)

### Phase 4: First Real Layer

11. Terrain layer (validates full pipeline: fetch → WASM parse → mesh gen → GPU upload → render)
12. Reactive state integration (@preact/signals)

### Phase 5: Expansion

13. Additional WASM parsers (glTF); Google Draco decoder integration for compressed meshes
14. Raster/vector overlay layers
15. 3D Tiles support (see _Out of scope_ below — likely a multi-phase effort of its own)
16. WebGPU evaluation (future)

## 15. Out of Scope (for this plan)

The following are acknowledged but deliberately not detailed here. They will get their own design docs when reached.

### 3D Tiles support (OGC 3D Tiles)

3D Tiles is a substantial standard on its own: hierarchical LOD, screen-space-error refinement (REPLACE vs ADD), nested tilesets, multiple content formats (glTF, B3DM, I3DM, PNTS), implicit tiling, metadata extensions. A naive "3D Tiles layer" understates the work by an order of magnitude.

When tackled, narrow the initial scope: target **Cesium 3D Tiles 1.1 with glTF content only**, REPLACE refinement only, no metadata extensions. Defer everything else.

### MapLibre's built-in terrain

MapLibre already has first-class terrain via `map.setTerrain()` with elevation queries through `map.queryTerrainElevation()`. Before building a custom terrain layer, evaluate whether MapLibre's built-in terrain is sufficient. Build custom terrain only when:

- The DEM format isn't supported by MapLibre
- Custom shading/coloring beyond MapLibre's hillshade is required
- Elevation needs to be shared with other custom layers (e.g., 3D Tiles draped on terrain) in a way MapLibre's API doesn't expose

If MapLibre's terrain suffices, replace the "Terrain layer" milestone with "Custom DEM source + addProtocol" — much simpler.

### Versioning & API stability

The SDK is a public contract for consumers. A full versioning policy will be defined when the first stable release approaches. Minimum commitments:

- **Semver** strictly enforced
- Public API explicitly marked; everything else lives under an `/internal` subpath and is not exported from the package root
- Breaking changes require a deprecation cycle of at least one minor version with `@deprecated` JSDoc tags before removal
- WASM ABI is internal; consumers never interact with raw WASM exports
