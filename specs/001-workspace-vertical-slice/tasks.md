# Tasks: Workspace Foundation & Vertical Slice

**Input**: Design documents from `/specs/001-workspace-vertical-slice/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/](./contracts/), [quickstart.md](./quickstart.md)

**Tests**: Included. The constitution's "Development Workflow & Quality Gates" section mandates headless unit tests, cross-crate integration tests, determinism tests, and a browser smoke test, so test tasks are required rather than optional.

**Organization**: Tasks are grouped by user story so each story is independently implementable and testable.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3, US4)
- Every task names an exact file path

## Path Conventions

Cargo workspace at repository root: `apps/game/`, `crates/metropolis_*/`, `assets/`, `data/`, `tools/asset_pipeline/`, `tests/`, `.github/workflows/`.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Convert the single-package hello-world into the constitution-mandated Cargo workspace.

- [ ] T001 Create the workspace root `Cargo.toml` with `[workspace]` members (`apps/game`, `crates/metropolis_core`, `crates/metropolis_world`, `crates/metropolis_render`, `crates/metropolis_transport`, `crates/metropolis_economy`, `crates/metropolis_logistics`, `crates/metropolis_population`, `crates/metropolis_save`, `tools/asset_pipeline`), `resolver = "3"`, and a `[workspace.dependencies]` table pinning every version from plan.md Technical Context
- [ ] T002 Delete the legacy single-package `src/main.rs` and `src/` directory, and remove the old `[package]` section from the root `Cargo.toml`
- [ ] T003 [P] Create `rust-toolchain.toml` pinning `channel = "1.99.1"`, `components = ["rustfmt", "clippy"]`, `targets = ["wasm32-unknown-unknown"]`
- [ ] T004 [P] Create `rustfmt.toml` (`edition = "2024"`, `max_width = 100`) and `clippy.toml` (`avoid-breaking-exported-api = false`)
- [ ] T005 [P] Update `.gitignore` to cover `/target`, `/dist`, `*.mpsave`, and `.DS_Store`
- [ ] T006 Create the directory skeleton `assets/{buildings,terrain,roads,vehicles,ui}/`, `data/{buildings,industries,commodities,vehicles}/`, `tools/asset_pipeline/src/`, `tests/`, and `web/` with `.gitkeep` placeholders where empty
- [ ] T007 [P] Create `crates/metropolis_core/Cargo.toml` and `crates/metropolis_core/src/lib.rs` with the headless Bevy subcrate dependency set from research.md R1
- [ ] T008 [P] Create `crates/metropolis_world/Cargo.toml` and `crates/metropolis_world/src/lib.rs` with the headless dependency set
- [ ] T009 [P] Create `crates/metropolis_transport/Cargo.toml` and `crates/metropolis_transport/src/lib.rs` with the headless dependency set
- [ ] T010 [P] Create `crates/metropolis_economy/Cargo.toml` and `crates/metropolis_economy/src/lib.rs` with the headless dependency set
- [ ] T011 [P] Create `crates/metropolis_logistics/Cargo.toml` and `crates/metropolis_logistics/src/lib.rs` with the headless dependency set
- [ ] T012 [P] Create `crates/metropolis_population/Cargo.toml` and `crates/metropolis_population/src/lib.rs` with the headless dependency set
- [ ] T013 [P] Create `crates/metropolis_save/Cargo.toml` and `crates/metropolis_save/src/lib.rs` with `serde`, `rmp-serde`, `zstd` added
- [ ] T014 [P] Create `crates/metropolis_render/Cargo.toml` and `crates/metropolis_render/src/lib.rs` depending on the `bevy` facade with `default-features = false, features = ["std", "2d", "png", "webgl2"]` per research.md R2
- [ ] T015 Create `apps/game/Cargo.toml` (binary `metropolis`) and `apps/game/src/main.rs` with a minimal `App` that opens a window, plus `apps/game/src/web.rs` with the `wasm_bindgen(start)` entry point and `console_error_panic_hook`
- [ ] T016 [P] Create `tools/asset_pipeline/Cargo.toml` and `tools/asset_pipeline/src/main.rs` with a `clap`-free `std::env::args` subcommand dispatcher for `pack-atlas` and `validate-data`
- [ ] T017 Verify the empty workspace builds by running `cargo build --workspace` and `cargo test --workspace` against the root `Cargo.toml` and confirming both succeed

**Checkpoint**: Workspace compiles with all crates present and no rendering dependency in simulation crates.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core primitives, the dependency-boundary gate, and the headless harness that every user story depends on.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [ ] T018 [P] Implement `TileCoord`, `ChunkCoord`, `BuildingId`, `VehicleId`, `ShipmentId`, and `Rotation` in `crates/metropolis_core/src/ids.rs` per data-model.md §1
- [ ] T019 [P] Implement `SimClock` and the `advance_clock` system in `crates/metropolis_core/src/clock.rs` with `TICK_MS = 100` and `TICKS_PER_GAME_MINUTE = 60`
- [ ] T020 [P] Implement `RngStreams` with `ChaCha8Rng` per-stream seeding in `crates/metropolis_core/src/rng.rs` per data-model.md §1 and research.md R7
- [ ] T021 [P] Implement `SimEvent`, `SimEvents`, and the `drain_sim_events` system in `crates/metropolis_core/src/events.rs`
- [ ] T022 [P] Implement `SimError` and `CorePlugin` (registering `SimClock`, `RngStreams`, `SimEvents`, and `advance_clock` first in `FixedUpdate`) in `crates/metropolis_core/src/error.rs` and `crates/metropolis_core/src/lib.rs`
- [ ] T023 Implement the headless `App` builder in `crates/metropolis_core/src/harness.rs` that constructs an `App` with `bevy_app` defaults, sets `Time<Fixed>` to 100 ms, and registers the simulation plugins without any window or renderer
- [ ] T024 [P] Add the dependency-boundary guard script `.github/workflows/ci.yml` step running the `cargo tree` grep from contracts/sim-api.md §1 and failing on any match
- [ ] T025 [P] Add `.github/workflows/ci.yml` with `cargo fmt --check`, `cargo clippy --workspace --all-targets -- -D warnings`, and `cargo test --workspace`
- [ ] T026 [P] Add `.github/workflows/wasm.yml` building `metropolis` for `wasm32-unknown-unknown` and running `wasm-bindgen --target web`
- [ ] T027 Create `web/index.html` loading the `wasm-bindgen` output into a full-viewport canvas
- [ ] T028 Create `tools/build_web.sh` performing the release WASM build, `wasm-bindgen` bundling into `dist/`, and copying `web/index.html`
- [ ] T029 Verify the boundary gate locally: run the `cargo tree` command from contracts/sim-api.md §1 and confirm no `wgpu`/`winit`/`image`/`bevy_render`/`bevy_window`/`bevy_asset`/`bevy_log` appears

**Checkpoint**: Foundation ready — `cargo test --workspace` passes, the boundary gate is green, and the headless `App` builder exists.

---

## Phase 3: User Story 1 — Playable Isometric Map in the Browser (Priority: P1) 🎯 MVP

**Goal**: A 512 × 512 chunked isometric map renders on desktop and in the browser, with smooth pan/zoom and exact pointer-to-tile picking.

**Independent Test**: Launch the game on desktop and in a browser, pan corner to corner, zoom to both extremes, and confirm no gaps/seams, ≥ 60 FPS desktop / ≥ 30 FPS browser, and that the highlighted tile is always the tile under the pointer.

### Tests for User Story 1

- [ ] T030 [P] [US1] Write the `grid_to_world`/`world_to_grid` round-trip property test over a sampled grid including all four corners in `crates/metropolis_world/src/grid.rs` (`#[cfg(test)] mod tests`)
- [ ] T031 [P] [US1] Write the chunk-index and chunk-bounds unit tests in `crates/metropolis_world/src/chunk.rs` (`#[cfg(test)] mod tests`)
- [ ] T032 [P] [US1] Write the visible-chunk culling unit test asserting only chunks intersecting the camera rect plus one-chunk margin are selected in `crates/metropolis_render/src/culling.rs` (`#[cfg(test)] mod tests`)

### Implementation for User Story 1

- [ ] T033 [P] [US1] Implement `CHUNK_SIZE`, `MAP_WIDTH`, `MAP_HEIGHT`, `TILE_WIDTH`, `TILE_HEIGHT`, `grid_to_world`, and `world_to_grid` in `crates/metropolis_world/src/grid.rs` per contracts/sim-api.md §4
- [ ] T034 [P] [US1] Implement `TerrainKind` and deterministic terrain generation from `RngStreams::terrain` in `crates/metropolis_world/src/terrain.rs`
- [ ] T035 [US1] Implement `Chunk` with `Vec<u16>` terrain and `Vec<u32>` buildings storage in `crates/metropolis_world/src/chunk.rs` per data-model.md §2
- [ ] T036 [US1] Implement `WorldMap` with `generate(seed)`, `tile()`, and `chunk()` in `crates/metropolis_world/src/map.rs` per data-model.md §2
- [ ] T037 [US1] Implement `WorldPlugin` registering `WorldMap` and the `apply_placement_requests` system slot in `crates/metropolis_world/src/lib.rs`
- [ ] T038 [P] [US1] Implement the isometric camera with pan (WASD + middle-drag) and zoom (scroll wheel, clamped) in `crates/metropolis_render/src/camera.rs`
- [ ] T039 [P] [US1] Implement chunk visibility selection from the camera rect plus one-chunk margin in `crates/metropolis_render/src/culling.rs` per research.md R4
- [ ] T040 [P] [US1] Implement the texture-atlas sprite batching layer that spawns/despawns chunk entities as they enter/leave the visible set in `crates/metropolis_render/src/batching.rs`
- [ ] T041 [US1] Implement terrain chunk rendering (one batched sprite layer per visible chunk, depth key `x + y`) in `crates/metropolis_render/src/isometric.rs`
- [ ] T042 [US1] Implement pointer picking via `Camera::viewport_to_world_2d` followed by `world_to_grid` in `crates/metropolis_render/src/picking.rs` per research.md R12
- [ ] T043 [US1] Implement `RenderPlugin` wiring camera, culling, batching, and picking into `crates/metropolis_render/src/lib.rs`
- [ ] T044 [US1] Implement `apps/game/src/plugins/render_plugin.rs` and `apps/game/src/plugins/input_plugin.rs` registering the render and input systems in `Update`
- [ ] T045 [US1] Wire `apps/game/src/main.rs` to build the full `App` (simulation plugins + render plugins) and add the `--headless` flag that skips window/render setup
- [ ] T046 [P] [US1] Author placeholder terrain sprites in `assets/terrain/` and generate the atlas plus layout descriptor with `tools/asset_pipeline`
- [ ] T047 [US1] Add an on-screen FPS counter and visible-chunk count overlay in `apps/game/src/plugins/ui_plugin.rs` for SC-002 verification
- [ ] T048 [US1] Validate VS-1 from quickstart.md on desktop and in the browser and record the measured FPS and load times

**Checkpoint**: User Story 1 is fully functional and independently testable — the map is navigable on both targets.

---

## Phase 4: User Story 2 — Place Buildings and Roads (Priority: P2)

**Goal**: Players select a building, preview its rotated footprint, place it on valid terrain, and draw roads that form a routable network; invalid placements are rejected with a reason.

**Independent Test**: Place several buildings of different footprints, rotate them, draw a connecting road, and confirm every placement is reflected in map state and every invalid placement is rejected with the correct reason.

### Tests for User Story 2

- [ ] T049 [P] [US2] Write placement validation tests covering out-of-bounds, occupied, water, and unknown-type rejections in `tests/placement.rs`
- [ ] T050 [P] [US2] Write road-connection tests asserting the connection bitmask is recomputed correctly for straight, corner, T, and cross junctions in `crates/metropolis_transport/src/graph.rs` (`#[cfg(test)] mod tests`)

### Implementation for User Story 2

- [ ] T051 [P] [US2] Implement `BuildingDef`, `CommodityDef`, `IndustryDef`, `VehicleDef`, and `Catalog` with `load_from_dir` and `validate` in `crates/metropolis_economy/src/commodity.rs` and `crates/metropolis_economy/src/lib.rs` per contracts/data-files.md
- [ ] T052 [P] [US2] Author the slice content files `data/buildings/{farm,mill,bakery,warehouse,road}.ron`, `data/commodities/{grain,flour,bread}.ron`, `data/industries/{grain_flour,bread}.ron`, and `data/vehicles/truck.ron` per contracts/data-files.md §7
- [ ] T053 [US2] Implement `validate_placement` returning `PlacementError` in `crates/metropolis_world/src/placement.rs` per data-model.md §2
- [ ] T054 [US2] Implement the `Building` component, `PlacementRequest` queue, and the `apply_placement_requests` system that writes building ids into `Chunk::buildings` in `crates/metropolis_world/src/placement.rs`
- [ ] T055 [P] [US2] Implement `RoadTile`, `RoadGraph`, and the `rebuild_road_connections` system in `crates/metropolis_transport/src/graph.rs` per data-model.md §4
- [ ] T056 [US2] Implement road drawing (drag from A to B produces an L-shaped or straight path) and road removal in `crates/metropolis_transport/src/graph.rs`
- [ ] T057 [US2] Implement `TransportPlugin` registering `RoadGraph`, `RouteCache`, `rebuild_road_connections`, and `invalidate_route_cache` in `crates/metropolis_transport/src/lib.rs`
- [ ] T058 [US2] Implement the building catalog UI panel and the placement preview (footprint outline, green/red validity tint, `R` to rotate) in `apps/game/src/plugins/ui_plugin.rs`
- [ ] T059 [US2] Implement building sprite rendering with rotation and isometric depth sorting in `crates/metropolis_render/src/isometric.rs`
- [ ] T060 [US2] Implement road tile sprite rendering with connection-aware variants in `crates/metropolis_render/src/isometric.rs`
- [ ] T061 [P] [US2] Author placeholder building and road sprites in `assets/buildings/` and `assets/roads/` and regenerate the atlas
- [ ] T062 [US2] Surface placement rejection reasons in the UI (toast or inline label) in `apps/game/src/plugins/ui_plugin.rs`
- [ ] T063 [US2] Validate VS-2 from quickstart.md and confirm a building plus connecting road can be placed in under 30 seconds

**Checkpoint**: User Stories 1 and 2 both work independently — the map is navigable and editable.

---

## Phase 5: User Story 3 — Working Production Chain with Freight Delivery (Priority: P3)

**Goal**: A truck routes over the road network to move grain from the farm to the mill and flour from the mill to the bakery, and the bakery produces bread only after flour is delivered; removing a road stops production.

**Independent Test**: Build farm, mill, bakery, and warehouse with connecting roads, observe a truck complete a delivery, confirm downstream output increases only after delivery, then remove a road segment and confirm production halts.

### Tests for User Story 3

- [ ] T064 [P] [US3] Write A* routing tests covering a valid path, an unreachable destination returning `NoPath`, and cache invalidation after a road removal in `tests/routing.rs`
- [ ] T065 [P] [US3] Write the headless slice integration test asserting at least one delivery and one downstream production cycle within 3600 ticks in `tests/headless_slice.rs`
- [ ] T066 [P] [US3] Write the off-screen parity test asserting production and delivery rates match when the chain is outside the camera rect in `tests/headless_slice.rs`

### Implementation for User Story 3

- [ ] T067 [P] [US3] Implement the `Router` trait and `AStarRouter` with a binary-heap open set in `crates/metropolis_transport/src/routing.rs` per research.md R5
- [ ] T068 [P] [US3] Implement `Route`, `RouteCache`, and graph-version invalidation in `crates/metropolis_transport/src/routing.rs` per data-model.md §4
- [ ] T069 [US3] Implement the `Vehicle` component, `VehicleState` transitions, and the `move_vehicles` system at 10 Hz in `crates/metropolis_transport/src/vehicle.rs` per data-model.md §5
- [ ] T070 [US3] Implement `Inventory` with capacity-bounded `add`/`remove`/`available` in `crates/metropolis_logistics/src/inventory.rs` per contracts/sim-api.md §4
- [ ] T071 [US3] Implement `Shipment`, `ShipmentState`, and `DispatchQueue` in `crates/metropolis_logistics/src/shipment.rs` per data-model.md §7
- [ ] T072 [US3] Implement the `dispatch_shipments` system at 1 Hz that assigns idle vehicles only when a valid route exists in `crates/metropolis_logistics/src/dispatch.rs`
- [ ] T073 [US3] Implement shipment failure handling when a route is invalidated mid-trip (cargo returns to source, shipment returns to `Pending`) in `crates/metropolis_logistics/src/dispatch.rs`
- [ ] T074 [US3] Implement `LogisticsPlugin` registering `DispatchQueue` and the dispatch systems in `crates/metropolis_logistics/src/lib.rs`
- [ ] T075 [US3] Implement the `Production` component and the `run_production` system gated on `SimClock::game_minute` in `crates/metropolis_economy/src/production.rs` per data-model.md §3
- [ ] T076 [US3] Implement input reservation and output deposit against `Inventory` in `crates/metropolis_economy/src/production.rs`
- [ ] T077 [US3] Implement `EconomyPlugin` registering `Catalog`, `Production`, and the production/economy systems in `crates/metropolis_economy/src/lib.rs`
- [ ] T078 [US3] Implement the `--headless --seed --ticks --scenario` CLI mode emitting the JSON summary from contracts/sim-api.md §5 in `apps/game/src/main.rs`
- [ ] T079 [US3] Implement vehicle sprite rendering with interpolation between tiles in `crates/metropolis_render/src/isometric.rs`
- [ ] T080 [P] [US3] Author the placeholder truck sprite in `assets/vehicles/` and regenerate the atlas
- [ ] T081 [US3] Implement a building inspector panel showing inventory and production progress in `apps/game/src/plugins/ui_plugin.rs`
- [ ] T082 [US3] Validate VS-3 and VS-6 from quickstart.md, including the road-removal halt within one game minute

**Checkpoint**: All three gameplay stories work independently — the vertical slice loop is proven end to end.

---

## Phase 6: User Story 4 — Save and Resume a City (Priority: P4)

**Goal**: A player saves a city mid-delivery and later loads it, resuming with identical buildings, roads, inventories, and in-progress deliveries.

**Independent Test**: Save a city while a truck is between two tiles, restart the game, load the save, and confirm the city and the in-progress delivery resume identically.

### Tests for User Story 4

- [ ] T083 [P] [US4] Write the save/load round-trip test asserting 100% state restoration for 100 buildings and 20 vehicles in `tests/save_roundtrip.rs`
- [ ] T084 [P] [US4] Write the determinism test running the same seed twice and asserting identical `state_hash` in `tests/determinism.rs`
- [ ] T085 [P] [US4] Write the version-handling tests for a newer `schema_version`, a truncated payload, and a referential-integrity violation in `crates/metropolis_save/src/schema.rs` (`#[cfg(test)] mod tests`)

### Implementation for User Story 4

- [ ] T086 [P] [US4] Implement `SaveGame` and all snapshot structs with `serde` derives in `crates/metropolis_save/src/schema.rs` per contracts/save-format.md
- [ ] T087 [US4] Implement `SaveGame::capture` reading `WorldMap`, buildings, roads, vehicles, shipments, and `SimClock` from the `App` in `crates/metropolis_save/src/persistence.rs`
- [ ] T088 [US4] Implement `SaveGame::restore` with staging-then-swap atomicity in `crates/metropolis_save/src/persistence.rs`
- [ ] T089 [US4] Implement `save_to_path` and `load_from_path` using `rmp_serde::to_vec_named` and `zstd::stream::encode_all` at level 3 in `crates/metropolis_save/src/persistence.rs` per research.md R8
- [ ] T090 [US4] Implement `SaveError` and the version dispatch plus migration chain scaffold in `crates/metropolis_save/src/migrate.rs` per contracts/save-format.md §9
- [ ] T091 [US4] Implement `state_hash` over the canonical uncompressed MessagePack encoding in `crates/metropolis_save/src/persistence.rs` per contracts/save-format.md §10
- [ ] T092 [US4] Implement `SavePlugin` registering the save/load systems and the default save directory resolution in `crates/metropolis_save/src/lib.rs`
- [ ] T093 [US4] Implement the save/load UI (save button, load list, error toast) in `apps/game/src/plugins/ui_plugin.rs`
- [ ] T094 [US4] Validate VS-4 and VS-5 from quickstart.md, including the mid-tile save and the corrupt-file error path

**Checkpoint**: All four user stories are independently functional — the vertical slice is complete.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Performance evidence, documentation, and the remaining quality gates.

- [ ] T095 [P] Add the `--scenario stress` harness mode generating 100 buildings and 20 vehicles in `apps/game/src/main.rs` for SC-008 measurement
- [ ] T096 [P] Add a `criterion` benchmark for the simulation tick measuring the 50 ms per 100 ms budget in `crates/metropolis_core/benches/tick.rs`
- [ ] T097 Measure and record desktop and browser FPS, tick time, and save/load time against the Principle IV budgets in `specs/001-workspace-vertical-slice/quickstart.md` §6
- [ ] T098 [P] Implement `asset_pipeline validate-data data/` and wire it into `.github/workflows/ci.yml`
- [ ] T099 [P] Implement `asset_pipeline pack-atlas assets/` producing the atlas PNG and `.ron` layout descriptor
- [ ] T100 [P] Write `README.md` documenting the workspace layout, build/run/test commands, and the simulation/rendering boundary
- [ ] T101 Run the full quickstart.md validation (VS-1 through VS-6) on desktop and in the browser and record results
- [ ] T102 Run the final quality gates from `.github/workflows/ci.yml` (`cargo fmt --check`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace`, and the `cargo tree` boundary gate) and confirm all pass

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately
- **Foundational (Phase 2)**: Depends on Phase 1 — BLOCKS all user stories
- **User Stories (Phases 3–6)**: All depend on Phase 2 completion
- **Polish (Phase 7)**: Depends on all four user stories

### User Story Dependencies

- **US1 (P1)**: Starts after Phase 2 — no dependency on other stories
- **US2 (P2)**: Starts after Phase 2 — needs `WorldMap` from US1 (T036) for placement validation, but its own tests can run against a synthetic map
- **US3 (P3)**: Starts after Phase 2 — needs `RoadGraph` from US2 (T055) and `Catalog` from US2 (T051)
- **US4 (P4)**: Starts after Phase 2 — needs the entity types from US1–US3 to snapshot, so it is sequenced last in practice

### Within Each User Story

- Tests are written first and MUST fail before implementation
- Models before services; services before systems; systems before UI
- Core implementation before integration
- Story complete and validated before moving to the next priority

### Parallel Opportunities

- T003–T005 (toolchain/config files) run in parallel
- T007–T014 (crate manifests) run in parallel
- T018–T022 (core primitives) run in parallel
- T024–T026 (CI workflows) run in parallel
- T030–T032 (US1 tests) run in parallel
- T033–T034, T038–T040 (US1 independent modules) run in parallel
- T049–T050 (US2 tests) run in parallel
- T051–T052, T055 (US2 catalog and road graph) run in parallel
- T064–T066 (US3 tests) run in parallel
- T067–T068, T070 (US3 routing and inventory) run in parallel
- T083–T085 (US4 tests) run in parallel
- T095–T096, T098–T100 (polish) run in parallel

---

## Parallel Example: User Story 1

```bash
# Launch all US1 tests together:
Task: "grid_to_world/world_to_grid round-trip property test in crates/metropolis_world/src/grid.rs"
Task: "chunk-index and chunk-bounds unit tests in crates/metropolis_world/src/chunk.rs"
Task: "visible-chunk culling unit test in crates/metropolis_render/src/culling.rs"

# Launch independent US1 modules together:
Task: "grid math in crates/metropolis_world/src/grid.rs"
Task: "terrain generation in crates/metropolis_world/src/terrain.rs"
Task: "camera in crates/metropolis_render/src/camera.rs"
Task: "culling in crates/metropolis_render/src/culling.rs"
Task: "batching in crates/metropolis_render/src/batching.rs"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL — blocks all stories)
3. Complete Phase 3: User Story 1
4. **STOP and VALIDATE**: run VS-1 from quickstart.md on desktop and in the browser
5. Demo the navigable isometric map

### Incremental Delivery

1. Setup + Foundational → workspace compiles, boundary gate green
2. US1 → navigable map (MVP)
3. US2 → building and road editing
4. US3 → working production chain (the slice's proof point)
5. US4 → save/resume
6. Polish → performance evidence and documentation

### Parallel Team Strategy

1. Team completes Setup + Foundational together
2. Once Foundational is done:
   - Developer A: US1 (world + render)
   - Developer B: US2 (catalog + placement + roads)
   - Developer C: US3 (routing + logistics + production)
   - Developer D: US4 (save format + determinism)
3. Stories integrate at the checkpoints

---

## Notes

- `[P]` tasks touch different files and have no incomplete dependencies
- `[Story]` labels map every story-phase task to its user story for traceability
- Every task names an exact file path so it is executable without further context
- Commit after each task or logical group
- Stop at any checkpoint to validate a story independently
- Avoid: vague tasks, same-file conflicts, cross-story dependencies that break independence