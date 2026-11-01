# Phase 0 Research: Workspace Foundation & Vertical Slice

**Feature**: `001-workspace-vertical-slice` | **Date**: 2026-11-01

All `NEEDS CLARIFICATION` markers from the plan's Technical Context are resolved below. Each
entry records the decision, the rationale, and the alternatives that were evaluated and rejected.

---

## R1. Headless Bevy dependency set for simulation crates

**Decision**: Simulation crates (`metropolis_core`, `metropolis_world`,
`metropolis_transport`, `metropolis_economy`, `metropolis_logistics`,
`metropolis_population`, `metropolis_save`) depend on individual Bevy subcrates with
`default-features = false`, never on the `bevy` facade:

```toml
bevy_app       = { version = "0.20.1", default-features = false, features = ["std"] }
bevy_ecs       = { version = "0.20.1", default-features = false, features = ["std", "bevy_reflect"] }
bevy_math      = { version = "0.20.1", default-features = false, features = ["std"] }
bevy_time      = { version = "0.20.1", default-features = false, features = ["std"] }
bevy_transform = { version = "0.20.1", default-features = false, features = ["std"] }
bevy_platform  = { version = "0.20.1", default-features = false, features = ["std"] }
bevy_tasks     = { version = "0.20.1", default-features = false }
bevy_reflect   = { version = "0.20.1", default-features = false, features = ["std"] }
bevy_state     = { version = "0.20.1", default-features = false }
bevy_diagnostic = { version = "0.20.1", default-features = false }
```

**Rationale**: A dependency audit of every Bevy 0.20.1 subcrate on crates.io shows that the
required (non-optional) transitive closure of the Bevy crate set contains **only `bevy_*`
crates** — every non-Bevy dependency (`wgpu`, `winit`, `image`, `glam`, `serde`, `tracing`,
`wasm-bindgen`, `web-sys`, …) is behind an optional feature. The non-Bevy crates that *are*
pulled in by the headless subset above are small utilities only:

| Subcrate | Required non-Bevy dependencies |
|----------|-------------------------------|
| `bevy_app` | `downcast-rs`, `log`, `thiserror`, `variadics_please` |
| `bevy_ecs` | `arrayvec`, `bitflags`, `bumpalo`, `concurrent-queue`, `derive_more`, `fixedbitset`, `indexmap`, `log`, `nonmax`, `slotmap`, `smallvec`, `thiserror`, `variadics_please` |
| `bevy_math` | `derive_more`, `glam`, `thiserror`, `variadics_please` |
| `bevy_time` | `log` |
| `bevy_transform` | `derive_more`, `thiserror` |
| `bevy_platform` | `foldhash`, `futures-channel`, `portable-atomic`, `portable-atomic-util`, `spin`, `windows-sys` |
| `bevy_tasks` | `async-task`, `atomic-waker`, `crossbeam-queue`, `derive_more`, `futures-lite`, `heapless`, `web-task` |
| `bevy_reflect` | `assert_type_match`, `derive_more`, `disqualified`, `downcast-rs`, `erased-serde`, `foldhash`, `serde`, `thiserror`, `variadics_please` |
| `bevy_diagnostic` | `atomic-waker`, `const-fnv1a-hash`, `log` |
| `bevy_state` | `log`, `variadics_please` |

No `wgpu`, no `winit`, no `image`, no `bevy_render`, no `bevy_window`, no `bevy_asset`. This
satisfies Constitution Principle I by construction rather than by convention.

**Explicitly excluded from simulation crates** (each would drag in graphics or platform code):

| Crate | Why excluded |
|-------|--------------|
| `bevy_log` | Hard-depends on `tracing-subscriber`, `tracing-wasm`, `tracing-oslog`, `android_log-sys`; use `tracing` directly instead |
| `bevy_asset` | Required deps include `async-io`, `blake3`, `ron`, `uuid`, `wasm-bindgen`, `web-sys` |
| `bevy_window` | Required dep `raw-window-handle`; also a required dep of `bevy_camera`/`bevy_picking` |
| `bevy_image` | Required deps include `image`, `wgpu-types`, `guillotiere`, `rectangle-pack` |
| `bevy_render` | Required deps include `wgpu`, `wgpu-types`, `web-sys`, `wasm-bindgen` |
| `bevy_camera`, `bevy_mesh`, `bevy_shader`, `bevy_material`, `bevy_light` | All require `wgpu-types` |
| `bevy_input`, `bevy_input_focus`, `bevy_picking` | Window/pointer oriented; input belongs to `apps/game` |

**Alternatives considered**:
- *Use the `bevy` facade with `default-features = false` in simulation crates.* Rejected: the
  facade's feature graph is coarse (`2d`, `ui`, `default_platform`), and a single accidental
  feature unification from `apps/game` would silently pull `wgpu` into the simulation build.
  Depending on subcrates directly makes the boundary explicit and auditable.
- *Write the simulation with no Bevy at all (own ECS).* Rejected: the constitution fixes Bevy
  ECS as the entity model, and re-implementing scheduling, queries, and change detection is
  unjustified complexity.
- *Use `bevy_ecs` only, with a hand-rolled scheduler.* Rejected: `bevy_app`'s `App`/`Plugin`/
  `FixedUpdate` machinery is exactly what Principle II requires, and it is headless-safe.

**Verification task**: CI runs `cargo tree -p metropolis_world -e normal` (and the other sim
crates) and fails if the output contains `wgpu`, `winit`, `image`, `bevy_render`,
`bevy_window`, `bevy_asset`, or `bevy_log`. This converts the audit above into an enforced gate.

---

## R2. Rendering and application dependency set

**Decision**: Only `apps/game` and `crates/metropolis_render` depend on the `bevy` facade:

```toml
bevy = { version = "0.20.1", default-features = false, features = [
  "std", "2d", "ui", "png", "webgl2", "default_font", "multi_threaded",
] }
```

`webgpu` is an additional opt-in feature (`--features webgpu`) that swaps the backend for
browsers that support it.

**Rationale**: `2d` brings `bevy_sprite`/`bevy_sprite_render` for isometric sprite batching;
`ui` brings `bevy_ui`/`bevy_ui_render` for the in-game interface; `png` enables the image loader
for placeholder sprites; `webgl2` is the browser baseline required by Principle IV (WebGL2
fallback); `default_font` avoids shipping a font asset for the slice; `multi_threaded` is safe
here because it only affects the render/app layer, and the browser build still runs the
simulation single-threaded (see R6). `default-features = false` drops `3d` and `audio`, which
the slice does not use, keeping the WASM binary small.

**Alternatives considered**:
- *Default features.* Rejected: pulls `3d` (PBR, shadows, glTF) and `audio` (rodio) into a 2D
  game, inflating the WASM payload and build time.
- *`webgpu` only.* Rejected: excludes browsers without WebGPU, violating the WebGL2 fallback
  requirement.
- *Custom `wgpu` renderer instead of `bevy_sprite`.* Rejected for the slice: sprite batching
  with a texture atlas already meets the culling/batching requirement, and a custom renderer is
  premature optimization (Principle V).

---

## R3. WASM build tooling

**Decision**: Use **`wasm-bindgen-cli` + a static `index.html`** driven by a small
`tools/build_web.sh` script, not Trunk and not `wasm-pack`.

**Rationale**: Neither `wasm-bindgen`, `trunk`, nor `wasm-pack` is currently installed on the
development machine, so any choice requires an install step. `wasm-bindgen-cli` is the lowest
common denominator: it is the tool both Trunk and `wasm-pack` wrap, it has no opinion about
project layout, and it produces exactly the `*.js` + `*.wasm` pair that a hand-written
`index.html` can load. This keeps the build reproducible in CI with a single
`cargo install wasm-bindgen-cli --version <pinned>` step and avoids a second build system.

**Alternatives considered**:
- *Trunk.* Rejected: adds a Rust-native bundler with its own config format and asset pipeline
  for a project that needs one HTML file and one WASM module; also an extra binary to pin.
- *`wasm-pack`.* Rejected: oriented toward npm-published library packages; it generates
  `package.json` scaffolding the game does not need.
- *`wasm32-wasip1` + WASI runtime.* Rejected: no DOM access, so no canvas or input.

**Install steps** (documented in `quickstart.md`):
`rustup target add wasm32-unknown-unknown` (already present) and
`cargo install wasm-bindgen-cli --version 0.2.130`.

---

## R4. Chunked isometric rendering strategy

**Decision**: Render each visible chunk as a **single batched sprite layer** built from a
texture atlas, with per-chunk visibility computed from the camera frustum expanded by one chunk
of margin. Terrain tiles are drawn from a `TextureAtlas` via `Sprite`/`TextureAtlasLayout`;
buildings are drawn as separate sprites sorted by isometric depth key `(x + y)`.

**Rationale**: `bevy_sprite` already batches sprites that share an atlas and pipeline, so the
per-chunk work is limited to spawning/despawning chunk entities as they enter and leave the
margin. A 512 × 512 map is 256 chunks; at typical zoom only 20–40 are visible, so the render
cost tracks the viewport rather than the world. Depth sorting by `(x + y)` is the standard
isometric painter's-order key and matches the 128 × 64 tile geometry.

**Alternatives considered**:
- *One entity per tile for the whole map.* Rejected: 262,144 entities, far beyond the entity
  budget and a guaranteed frame-rate collapse.
- *Custom mesh per chunk (one quad mesh with a tile texture array).* Rejected for the slice:
  more code, and sprite batching already meets the target. Revisit only if profiling shows
  sprite overhead dominating.
- *Render-to-texture chunk caching.* Rejected: premature; adds GPU memory management for no
  measured gain at slice scale.

---

## R5. Routing algorithm for the slice

**Decision**: Plain **A\*** over the road tile graph, with a binary-heap open set and a
`HashMap<TileCoord, u32>` cost map, plus a route cache keyed by `(source, destination)` that is
invalidated whenever a road tile is added or removed.

**Rationale**: The slice has one small road network and a handful of vehicles. Hierarchical A*
only pays off on large graphs with many repeated long-distance queries; introducing it now
would add a cluster/portal abstraction that cannot be validated at slice scale (Principle V).
The route cache plus invalidation is the part that actually matters for correctness — it is what
makes "remove a road segment → production stops" (SC-006) observable — so it is built now and
the algorithm can be swapped later behind the same `Router` trait.

**Alternatives considered**:
- *Hierarchical A\* immediately.* Rejected: unjustified complexity before the slice is proven;
  the constitution lists it as the target algorithm, not a slice requirement.
- *Dijkstra.* Rejected: no heuristic, strictly slower, no benefit on a uniform-cost grid.
- *Flow-field / all-pairs precomputation.* Rejected: memory-heavy and invalidated constantly by
  road edits.

---

## R6. Fixed-timestep scheduling in Bevy 0.20

**Decision**: One `FixedUpdate` schedule at **100 ms** (10 Hz) as the base tick. Higher-frequency
systems (vehicle movement at 10 Hz) run every tick; lower-frequency systems run on accumulators
inside the tick:

| System | Frequency | Implementation |
|--------|-----------|----------------|
| Vehicle movement | 10 Hz | every `FixedUpdate` tick |
| Traffic updates | 2 Hz | every 5th tick |
| Logistics dispatch | 1 Hz | every 10th tick |
| Industry production | per game minute | `SimClock` accumulator (1 game minute = 60 ticks) |
| Economy | per game hour | `SimClock` accumulator |
| Population | per game day | `SimClock` accumulator |

`SimClock` is a resource holding `tick: u64`, `game_minute: u64`, `game_hour: u64`,
`game_day: u64`, advanced by a single `advance_clock` system ordered first in `FixedUpdate`.
Systems read the clock rather than counting frames, so the schedule is data-driven and testable.

**Rationale**: A 100 ms base tick satisfies the "simulation tick < 50 ms per 100 ms tick" budget
with headroom, and 10 Hz is the highest frequency the constitution requires, so no system needs
a sub-tick loop. Accumulators keep the tick count deterministic regardless of how many systems
are registered.

**Alternatives considered**:
- *Multiple `FixedUpdate` schedules at different rates.* Rejected: Bevy's fixed schedule is
  single-rate; running several would require manual `run_if` gating anyway, with less clarity.
- *`Update` with delta accumulation.* Rejected: violates Principle II (render-delta-driven
  simulation).
- *60 Hz fixed tick.* Rejected: 6× the simulation work for no behavioral benefit, and it
  endangers the 50 ms budget on the browser target.

**Browser backgrounding**: Bevy's fixed timestep clamps catch-up steps, so a backgrounded tab
resumes without a time jump. The slice additionally caps catch-up at 10 steps per frame to keep
the first frame after resume within budget (spec edge case: "browser tab is backgrounded").

---

## R7. Determinism strategy

**Decision**: Store all randomness in a `RngStreams` resource holding named
`ChaCha8Rng` instances (`terrain`, `logistics`, `traffic`), each seeded from the world seed via
a fixed derivation (`seed ^ stream_id`). Systems take `ResMut<RngStreams>` and draw only from
their own stream. Determinism is verified by a `state_hash()` function that hashes the canonical
serialized form of simulation state (map, buildings, roads, inventories, vehicles, clock) and by
an integration test that runs the same seed twice and asserts equal hashes.

**Rationale**: Per-stream RNGs prevent one system's draw count from perturbing another's
sequence, which is the usual source of non-determinism when systems are added or reordered.
Hashing the serialized form reuses the save schema, so the determinism test also exercises
serialization.

**Alternatives considered**:
- *One global RNG.* Rejected: any change in system order or draw count shifts every downstream
  value, making determinism tests brittle.
- *`rand::thread_rng`.* Rejected: explicitly forbidden by Principle II.
- *Hash of in-memory structs via `bytemuck`.* Rejected: padding bytes and `HashMap` iteration
  order make it unstable; the serialized form is canonical.

---

## R8. Save format and versioning

**Decision**: A `SaveGame` struct serialized with `rmp_serde::to_vec_named` (named fields, so
the format survives field reordering), compressed with `zstd::stream::encode_all` at level 3,
written to a single `.mpsave` file. The uncompressed header carries
`schema_version: u32` (starting at `1`) and `game_version: String`. Loading dispatches on
`schema_version`: equal → deserialize; lower → run a migration chain; higher → return
`SaveError::UnsupportedVersion` with the found and supported versions.

**Rationale**: Named-field MessagePack is self-describing enough to migrate and compact enough
for the 5-second budget; Zstd level 3 is the standard speed/ratio knee. A single file keeps the
slice's save path trivial while the version header satisfies Principle VI.

**Alternatives considered**:
- *`bincode`.* Rejected: not self-describing, so migrations require positional knowledge.
- *JSON.* Rejected: 5–10× larger and slower; the constitution mandates MessagePack.
- *Zstd level 19.* Rejected: compression time dominates and endangers the 5-second budget.
- *Separate header file.* Rejected: two-file saves are not atomic and can desync.

---

## R9. Data-file format for content definitions

**Decision**: Use **RON** (`.ron`) for `data/` content files, deserialized with `ron` into
`BuildingDef`, `IndustryDef`, `CommodityDef`, and `VehicleDef` structs. Files are loaded at
startup by `metropolis_core::content` and validated (unique ids, referenced commodities exist,
footprints non-empty, production intervals > 0). Each file carries a `schema_version` field.

**Rationale**: RON is Rust-native, supports comments and trailing commas (important for
hand-authored content), and is already in the dependency graph via `bevy_asset`'s `ron`
dependency in the app layer — so no new crate family is introduced. It is human-diffable in
review, which matters for balance changes.

**Alternatives considered**:
- *TOML.* Rejected: awkward for nested arrays of structs (footprints, input/output lists).
- *JSON.* Rejected: no comments, no trailing commas, verbose for nested content.
- *MessagePack.* Rejected: not human-editable, defeating the purpose of `data/`.
- *Hard-coded Rust constants.* Rejected: violates Principle VI.

---

## R10. Asset pipeline for placeholder art

**Decision**: Ship hand-authored placeholder PNGs in `assets/` and provide
`tools/asset_pipeline` as a binary that (a) packs `assets/**/*.png` into a single atlas plus a
`.ron` layout descriptor and (b) validates every file in `data/`. The slice runs against the
committed placeholder atlas; the tool is run manually and its output committed.

**Rationale**: The spec states art is placeholder-quality and final art comes later. A minimal
packer keeps the atlas format under project control without committing to a third-party packer,
and the data validator gives CI a cheap content-integrity gate.

**Alternatives considered**:
- *`bevy_asset_loader` / `bevy_embedded_assets`.* Rejected: extra dependencies for behavior the
  slice does not need.
- *Runtime atlas generation.* Rejected: adds startup cost against the SC-001 budget.
- *No tool at all.* Rejected: the constitution mandates `tools/`, and a validator is genuinely
  useful for `data/`.

---

## R11. Headless test harness shape

**Decision**: The headless harness is an **integration test** (`tests/headless_slice.rs`) plus a
`--headless` mode on the `metropolis` binary. Both build a `bevy_app::App` with
`MinimalPlugins`-equivalent manual setup (no `bevy_winit`, no `bevy_render`), insert the
simulation plugins, and step `FixedUpdate` a fixed number of times. The integration test asserts
the full chain outcome; the binary mode is for manual profiling.

**Rationale**: An integration test is what CI needs and what SC-010 describes. Reusing the same
plugin set as the real app guarantees the tested code path is the shipped one. `MinimalPlugins`
lives in `bevy_internal`/`bevy_app`, so the harness stays inside the headless dependency set.

**Alternatives considered**:
- *A separate `apps/headless` binary crate.* Rejected: duplicates plugin wiring and drifts from
  the real app.
- *Unit tests only.* Rejected: cannot prove cross-crate integration (FR-017, SC-010).

---

## R12. Pointer picking correctness

**Decision**: Convert the cursor position from viewport space to world space using the camera's
`world_to_viewport` inverse (`Camera::viewport_to_world_2d`), then apply `world_to_grid` and
floor. Validate with a round-trip test: for a grid of sample tiles, `grid_to_world` →
`world_to_grid` must return the original tile, including at maximum zoom-out.

**Rationale**: The spec's edge case list includes "maximum zoom-out" and SC-003 requires 100%
correctness across sampled positions. The round-trip property test is the cheapest way to prove
it, and it catches the classic off-by-half-tile error in isometric picking.

**Alternatives considered**:
- *Screen-space tile lookup table.* Rejected: memory-heavy and breaks under zoom.
- *Ray-cast against tile quads.* Rejected: requires render-side geometry the simulation must not
  own (Principle I).

---

## R13. Parallelism policy

**Decision**: Ship the slice **single-threaded**. `rayon` is declared as an optional dependency
behind a non-default `parallel` feature on `metropolis_world` and `metropolis_transport`, and is
never enabled for `wasm32` targets.

**Rationale**: Principle IV forbids assuming browser multithreading and requires benchmark
evidence before adding parallelism. The slice's workload (a few buildings, one truck) is far
below the threshold where parallelism helps, and adding it now would make determinism harder to
prove.

**Alternatives considered**:
- *Enable Rayon everywhere.* Rejected: violates Principle IV and breaks the WASM build without
  shared-memory configuration.
- *`bevy_tasks` parallel iterators.* Rejected for the slice: same determinism and browser
  concerns; revisit with benchmarks.

---

## R14. CI gates

**Decision**: Two GitHub Actions workflows:
- `ci.yml` — `cargo fmt --check`, `cargo clippy --workspace --all-targets -- -D warnings`,
  `cargo test --workspace`, the headless slice test, and the `cargo tree` dependency guard from
  R1.
- `wasm.yml` — `cargo build -p metropolis --target wasm32-unknown-unknown --release`,
  `wasm-bindgen` bundling, and a headless-browser smoke test that loads the page and asserts the
  canvas initializes.

**Rationale**: These are exactly the gates the constitution's "Development Workflow & Quality
Gates" section requires, split so the fast native checks fail early and the slower WASM job runs
in parallel.

**Alternatives considered**:
- *Single workflow.* Rejected: the WASM job is the slow one; separating it keeps PR feedback
  fast.
- *Skip the browser smoke test.* Rejected: the constitution requires a browser smoke test before
  a milestone is declared complete.

---

## Resolved unknowns summary

| Plan field | Resolution |
|------------|------------|
| Language/Version | Rust 1.99.1, edition 2024 (R1, R3) |
| Primary Dependencies | Bevy 0.20.1 headless subcrates + facade split (R1, R2); serde/rmp-serde/zstd (R8); rand_chacha (R7); wasm-bindgen (R3) |
| Storage | Local `.mpsave` files, MessagePack + Zstd, versioned (R8); RON content in `data/` (R9) |
| Testing | `cargo test` + headless harness + determinism hash test (R7, R11) |
| Target Platform | Native + `wasm32-unknown-unknown`, WebGL2 baseline (R2, R3) |
| Project Type | Cargo workspace, 1 app + 8 crates + 1 tool (plan Project Structure) |
| Performance Goals | Constitution Principle IV budgets; culling + batching (R4); single-threaded (R13) |
| Constraints | Dependency guard (R1), fixed tick (R6), seeded RNG (R7), versioned saves (R8) |
| Scale/Scope | One 512 × 512 map, 5 building types, 1 vehicle, 3 commodities, 1 chain |

No `NEEDS CLARIFICATION` markers remain.