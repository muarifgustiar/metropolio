# Implementation Plan: Workspace Foundation & Vertical Slice

**Branch**: `001-workspace-vertical-slice` | **Date**: 2026-11-01 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-workspace-vertical-slice/spec.md`

## Summary

Stand up the Metropolis Cargo workspace exactly as the constitution prescribes (`apps/game` plus
eight `crates/metropolis_*` packages, with `assets/`, `data/`, `tools/`, `tests/`), then prove the
architecture with a playable vertical slice: a 512 × 512 chunked isometric map with pan/zoom and
correct pointer picking, building and road placement with validation, a farm → mill → bakery →
warehouse production chain driven by a truck that actually routes over the road graph, and a
versioned MessagePack + Zstd save/load round-trip.

The technical approach is a **two-layer split enforced by Cargo dependency boundaries**:
simulation crates depend only on headless Bevy subcrates (`bevy_app`, `bevy_ecs`, `bevy_math`,
`bevy_time`, `bevy_transform`, `bevy_platform`, `bevy_tasks`, `bevy_reflect`, `bevy_state`,
`bevy_diagnostic`) with `default-features = false`, so no `wgpu`, `winit`, `image`, or
`bevy_asset` can enter the simulation dependency graph. Rendering and the Bevy application sit
above that boundary and are the only crates allowed to depend on the `bevy` facade with
`2d`/`ui` features. Simulation advances on `FixedUpdate` at a 100 ms tick with sub-rate
accumulators for the 10 Hz / 2 Hz / 1 Hz / per-game-minute systems, seeded `rand_chacha` streams,
and a headless harness that runs the full slice scenario in CI without a display.

## Technical Context

**Language/Version**: Rust 1.99.1, edition 2024 (pinned via `rust-toolchain.toml`)

**Primary Dependencies**:
- Bevy 0.20.1 — `bevy` facade (rendering/app crates only) and headless subcrates
  (`bevy_app`, `bevy_ecs`, `bevy_math`, `bevy_time`, `bevy_transform`, `bevy_platform`,
  `bevy_tasks`, `bevy_reflect`, `bevy_state`, `bevy_diagnostic`) for simulation crates
- `serde` 1.0 + `rmp-serde` 1.3 (MessagePack) + `zstd` 0.14 (compressed saves)
- `rand` 0.10 + `rand_chacha` 0.10 (seeded, deterministic RNG)
- `thiserror` 2.0 (error types), `tracing` 0.1 (structured logging in sim crates)
- `wasm-bindgen` 0.2 + `console_error_panic_hook` 0.1 (browser entry point)
- `rayon` 1.12 — native-only, behind a non-default `parallel` feature (not enabled in the slice)

**Storage**: Local filesystem saves — MessagePack payload compressed with Zstd, carrying an
explicit `schema_version`. Game content lives in versioned data files under `data/`
(`buildings/`, `industries/`, `commodities/`, `vehicles/`). No database; cloud persistence
(Go + Echo, PostgreSQL, S3/MinIO) is explicitly out of scope for this feature.

**Testing**: `cargo test` (unit tests per crate + integration tests in `tests/`), a headless
simulation harness for end-to-end slice verification, determinism tests comparing state hashes
across two identical seeded runs, and `cargo clippy` / `cargo fmt --check` as gates.

**Target Platform**: Desktop (macOS/Linux/Windows native) and browser via
`wasm32-unknown-unknown` + WebAssembly, with WebGL2 as the baseline graphics backend and WebGPU
as an opt-in feature.

**Project Type**: Cargo workspace — one application binary (`apps/game`) plus eight library
crates, one headless harness binary, and one asset-pipeline tool.

**Performance Goals** (from Constitution Principle IV):
- 512 × 512 map, 10,000 buildings, 5,000 vehicles, 100,000 aggregated population, 1,000 industries
- 60 FPS desktop, 30–60 FPS browser
- Simulation tick < 50 ms per 100 ms tick
- Save/load < 5 s
- Slice-level targets: SC-001 (map visible ≤ 10 s desktop / ≤ 20 s browser), SC-002 (60/30 FPS),
  SC-008 (save + load of 100 buildings / 20 vehicles < 5 s each)

**Constraints**:
- Simulation crates MUST NOT link `wgpu`, `winit`, `image`, `bevy_render`, `bevy_window`,
  `bevy_asset`, or `bevy_log` (Constitution Principle I). Enforced by dependency review plus a
  `cargo tree` assertion in CI.
- All simulation on `FixedUpdate`; no render-delta-driven simulation (Principle II).
- Seeded `rand_chacha` only; no thread-local or unseeded RNG (Principle II).
- No entity-per-citizen; aggregate population structures (Principle III).
- `CHUNK_SIZE = 32`; terrain as `Vec<u16>`, building ids as `Vec<u32>` (Principle III).
- Browser build MUST be single-threaded by default; Rayon is native-only and opt-in
  (Principle IV).
- Isometric tiles 128 × 64 with the canonical `grid_to_world` / `world_to_grid` transforms.
- Godot / GDScript / GDExtension forbidden.
- Saves MUST be Serde + MessagePack + Zstd with a schema version (Principle VI).

**Scale/Scope**: One fixed 512 × 512 map (256 chunks of 32 × 32), a catalog of five building
types (farm, mill, bakery, warehouse, road) plus one vehicle type (truck), three commodities
(grain, flour, bread), and one production chain. Economy, population, zoning, rail, ships, and
cloud saves are out of scope.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Gate | Pre-Design | Post-Design |
|-----------|------|------------|-------------|
| I. Simulation–Rendering Separation (NON-NEGOTIABLE) | Simulation crates depend only on headless Bevy subcrates; no rendering/windowing/asset APIs; headless harness exists | PASS — dependency set chosen from headless subcrates only; `metropolis_render`/`metropolis_app` are the sole consumers of the `bevy` facade | PASS — [research.md](./research.md) R1 fixes the exact feature flags; [contracts/sim-api.md](./contracts/sim-api.md) exposes only plain-data types; [quickstart.md](./quickstart.md) runs the slice headlessly |
| II. Deterministic Fixed-Timestep Simulation | `FixedUpdate` at 100 ms; frequency-scheduled systems; seeded `rand_chacha` in world state | PASS — tick schedule defined in Technical Context; RNG stored as a resource | PASS — [data-model.md](./data-model.md) defines `SimClock` + `RngStreams`; [contracts/sim-api.md](./contracts/sim-api.md) fixes the tick order; determinism test in [quickstart.md](./quickstart.md) |
| III. Data-Oriented ECS Discipline | Entities only for buildings/vehicles/shipments; aggregate population; `CHUNK_SIZE = 32`; contiguous chunk storage | PASS — no population entities in scope; chunk layout fixed | PASS — [data-model.md](./data-model.md) specifies `Vec<u16>`/`Vec<u32>` chunk storage and component-only data |
| IV. Browser-First Performance Budgets | Targets measured; visible-chunk culling; atlas batching; single-threaded browser baseline | PASS — culling and batching are in the render design; Rayon is a non-default native feature | PASS — [quickstart.md](./quickstart.md) includes the perf and WASM smoke validation scenarios |
| V. Vertical Slice Before Scale | Deliverable is one map, few buildings, roads, freight, one working chain | PASS — scope is exactly the slice; economy/population/rail deferred | PASS — [data-model.md](./data-model.md) and [contracts/](./contracts/) cover only slice entities |
| VI. Reproducible, Versioned Game State | Serde + MessagePack + Zstd with schema version; data-driven content in `data/` | PASS — save format and `data/` layout fixed | PASS — [contracts/save-format.md](./contracts/save-format.md) and [contracts/data-files.md](./contracts/data-files.md) define both schemas |

**Result**: No violations. Complexity Tracking is empty.

## Project Structure

### Documentation (this feature)

```text
specs/001-workspace-vertical-slice/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command)
├── quickstart.md        # Phase 1 output (/speckit-plan command)
├── contracts/           # Phase 1 output (/speckit-plan command)
│   ├── sim-api.md
│   ├── data-files.md
│   └── save-format.md
├── checklists/
│   └── requirements.md
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)

```text
Cargo.toml                       # [workspace] members + shared [workspace.dependencies]
Cargo.lock
rust-toolchain.toml              # channel = "1.99.1", targets = native + wasm32-unknown-unknown
rustfmt.toml
clippy.toml
.gitignore

apps/
└── game/
    ├── Cargo.toml               # bin: metropolis; deps: bevy (2d, ui, webgl2), all crates
    └── src/
        ├── main.rs              # Bevy App bootstrap, plugin registration
        ├── plugins/
        │   ├── mod.rs
        │   ├── render_plugin.rs
        │   ├── input_plugin.rs
        │   └── ui_plugin.rs
        └── web.rs               # wasm-bindgen entry point + panic hook

crates/
├── metropolis_core/             # shared primitives: ids, time, events, errors, rng
│   └── src/{lib.rs, ids.rs, clock.rs, events.rs, rng.rs, error.rs}
├── metropolis_world/            # map, chunks, terrain, grid math, occupancy, placement
│   └── src/{lib.rs, chunk.rs, terrain.rs, grid.rs, map.rs, placement.rs}
├── metropolis_render/           # isometric rendering, camera, chunk culling, batching
│   └── src/{lib.rs, isometric.rs, camera.rs, batching.rs, culling.rs, picking.rs}
├── metropolis_transport/        # road graph, A* routing, vehicle movement
│   └── src/{lib.rs, graph.rs, routing.rs, vehicle.rs}
├── metropolis_economy/          # commodities, production, demand
│   └── src/{lib.rs, commodity.rs, production.rs, demand.rs}
├── metropolis_logistics/        # inventory, orders, shipments, dispatch
│   └── src/{lib.rs, inventory.rs, shipment.rs, dispatch.rs}
├── metropolis_population/       # aggregate population (stub for this feature)
│   └── src/{lib.rs, aggregate.rs}
└── metropolis_save/             # MessagePack + Zstd persistence, schema versioning
    └── src/{lib.rs, persistence.rs, schema.rs, migrate.rs}

assets/
├── buildings/                   # placeholder isometric sprites (PNG)
├── terrain/
├── roads/
├── vehicles/
└── ui/

data/
├── buildings/{farm,mill,bakery,warehouse,road}.ron
├── industries/{grain_flour,bread}.ron
├── commodities/{grain,flour,bread}.ron
└── vehicles/truck.ron

tools/
└── asset_pipeline/              # bin: atlas packer + data-file validator
    ├── Cargo.toml
    └── src/main.rs

tests/
├── headless_slice.rs            # full farm→mill→bakery→warehouse scenario, no display
├── determinism.rs               # same seed twice → identical state hash
├── save_roundtrip.rs            # save/load restores 100% of state
├── routing.rs                   # A* correctness + unreachable-route handling
└── placement.rs                 # footprint/rotation/occupancy validation

.github/
└── workflows/
    ├── ci.yml                   # fmt, clippy, test, headless slice, cargo tree guard
    └── wasm.yml                 # wasm32 build + browser smoke test
```

**Structure Decision**: The workspace layout is mandated verbatim by the constitution's
"Technology Stack & Architecture Constraints" section, so no alternative layout was considered.
The only additions beyond the mandated tree are: `apps/game/src/plugins/` (Bevy plugin
registration split out of `main.rs`), `apps/game/src/web.rs` (WASM entry point), a
`tools/asset_pipeline` binary crate, and a `tests/` directory holding cross-crate integration
tests. `metropolis_population` is scaffolded with an aggregate-only stub because the constitution
fixes the crate list, but population behavior is out of scope for this feature.

## Complexity Tracking

> No Constitution Check violations. No entries.

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| — | — | — |