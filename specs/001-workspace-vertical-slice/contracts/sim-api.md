# Contract: Simulation API Surface

**Feature**: `001-workspace-vertical-slice` | **Date**: 2026-11-01

This contract defines the boundary between the simulation layer and everything above it
(rendering, application, tests). It is the enforcement point for Constitution Principle I: the
types below are plain data and Bevy ECS primitives only — no rendering, windowing, or asset
types may appear in any signature.

---

## 1. Crate dependency contract

| Crate | May depend on | MUST NOT depend on |
|-------|---------------|--------------------|
| `metropolis_core` | `bevy_app`, `bevy_ecs`, `bevy_math`, `bevy_platform`, `bevy_reflect`, `serde`, `rand`, `rand_chacha`, `thiserror`, `tracing` | `bevy` facade, `bevy_render`, `bevy_window`, `bevy_asset`, `bevy_image`, `bevy_log`, `wgpu`, `winit` |
| `metropolis_world` | `metropolis_core` + the same headless set | same as above |
| `metropolis_transport` | `metropolis_core`, `metropolis_world` + headless set | same as above |
| `metropolis_economy` | `metropolis_core`, `metropolis_world` + headless set | same as above |
| `metropolis_logistics` | `metropolis_core`, `metropolis_world`, `metropolis_transport`, `metropolis_economy` + headless set | same as above |
| `metropolis_population` | `metropolis_core` + headless set | same as above |
| `metropolis_save` | all simulation crates + `serde`, `rmp-serde`, `zstd` | same as above |
| `metropolis_render` | `bevy` facade (`2d`, `png`, `webgl2`), `metropolis_core`, `metropolis_world` | — |
| `apps/game` | `bevy` facade (`2d`, `ui`, `png`, `webgl2`, `default_font`), all crates | — |

**Enforcement**: CI runs

```bash
cargo tree -p metropolis_world -p metropolis_transport -p metropolis_economy \
           -p metropolis_logistics -p metropolis_population -p metropolis_save \
           -e normal --prefix none \
  | grep -E '^(wgpu|winit|image|bevy_render|bevy_window|bevy_asset|bevy_log) ' \
  && exit 1 || exit 0
```

A match fails the build.

---

## 2. Plugin contract

Each simulation crate exposes exactly one plugin, and the plugin MUST be constructible without
any rendering context:

```rust
// metropolis_world
pub struct WorldPlugin { pub config: WorldConfig }
impl bevy_app::Plugin for WorldPlugin { /* ... */ }

// metropolis_transport
pub struct TransportPlugin;
// metropolis_economy
pub struct EconomyPlugin;
// metropolis_logistics
pub struct LogisticsPlugin;
// metropolis_population
pub struct PopulationPlugin;
// metropolis_save
pub struct SavePlugin;
```

**Contract**: `App::new().add_plugins((WorldPlugin { .. }, TransportPlugin, EconomyPlugin,
LogisticsPlugin, PopulationPlugin, SavePlugin))` MUST build and run `FixedUpdate` with no
window, no renderer, and no asset server. This is the headless harness (FR-002, FR-017).

---

## 3. Schedule contract

`FixedUpdate` runs at 100 ms. System order within a tick is fixed and MUST NOT be reordered
without updating this contract:

| Order | System | Crate | Frequency |
|-------|--------|-------|-----------|
| 1 | `advance_clock` | `metropolis_core` | every tick |
| 2 | `apply_placement_requests` | `metropolis_world` | every tick |
| 3 | `rebuild_road_connections` | `metropolis_transport` | every tick (only when dirty) |
| 4 | `invalidate_route_cache` | `metropolis_transport` | every tick (only when graph version changed) |
| 5 | `move_vehicles` | `metropolis_transport` | every tick (10 Hz) |
| 6 | `update_traffic` | `metropolis_transport` | every 5th tick (2 Hz) |
| 7 | `dispatch_shipments` | `metropolis_logistics` | every 10th tick (1 Hz) |
| 8 | `run_production` | `metropolis_economy` | every tick, gated on `SimClock::game_minute` change |
| 9 | `run_economy` | `metropolis_economy` | every tick, gated on `game_hour` change |
| 10 | `run_population` | `metropolis_population` | every tick, gated on `game_day` change |
| 11 | `drain_sim_events` | `metropolis_core` | every tick, last |

**Contract**: systems 2–10 MUST NOT read `Time` (render delta). They read `SimClock` only.

---

## 4. Public API surface (per crate)

### `metropolis_core`

```rust
pub const TICK_MS: u64 = 100;
pub const TICKS_PER_GAME_MINUTE: u64 = 60;

pub struct TileCoord { pub x: i32, pub y: i32 }
pub struct ChunkCoord { pub x: i32, pub y: i32 }
pub struct BuildingId(pub u32);
pub struct VehicleId(pub u32);
pub struct ShipmentId(pub u32);

pub struct SimClock { /* fields per data-model.md */ }
pub struct RngStreams { /* fields per data-model.md */ }
impl RngStreams { pub fn from_seed(seed: u64) -> Self; }

pub enum SimEvent { /* per data-model.md */ }
pub struct SimEvents(pub Vec<SimEvent>);

pub enum SimError { /* ... */ }
```

### `metropolis_world`

```rust
pub const CHUNK_SIZE: usize = 32;
pub const MAP_WIDTH: u32 = 512;
pub const MAP_HEIGHT: u32 = 512;
pub const TILE_WIDTH: f32 = 128.0;
pub const TILE_HEIGHT: f32 = 64.0;

pub fn grid_to_world(x: i32, y: i32) -> bevy_math::Vec2;
pub fn world_to_grid(position: bevy_math::Vec2) -> TileCoord;

pub struct WorldMap { /* ... */ }
impl WorldMap {
    pub fn generate(seed: u64) -> Self;
    pub fn tile(&self, coord: TileCoord) -> Option<TileRef<'_>>;
    pub fn chunk(&self, coord: ChunkCoord) -> Option<&Chunk>;
}

pub fn validate_placement(
    map: &WorldMap,
    catalog: &Catalog,
    request: &PlacementRequest,
) -> Result<(), PlacementError>;
```

**Contract**: `grid_to_world` and `world_to_grid` MUST be exact inverses for every in-bounds
tile: `world_to_grid(grid_to_world(x, y)) == TileCoord { x, y }`. This is asserted by a property
test over a sampled grid including map corners.

### `metropolis_transport`

```rust
pub trait Router {
    fn route(&self, graph: &RoadGraph, from: TileCoord, to: TileCoord)
        -> Result<Route, RoutingError>;
}

pub struct AStarRouter;
impl Router for AStarRouter { /* ... */ }

pub struct RoadGraph { /* ... */ }
pub struct RouteCache { /* ... */ }
pub struct Vehicle { /* ... */ }
```

**Contract**: `Router::route` MUST return `Err(RoutingError::NoPath)` — never panic — when the
destination is unreachable. Removing a road tile MUST bump `RoadGraph::version`.

### `metropolis_economy`

```rust
pub struct Catalog { /* ... */ }
impl Catalog {
    pub fn load_from_dir(dir: &std::path::Path) -> Result<Self, CatalogError>;
    pub fn validate(&self) -> Result<(), CatalogError>;
}
pub struct Production { /* ... */ }
```

**Contract**: `Catalog::load_from_dir` MUST fail with a descriptive `CatalogError` on any
invalid data file rather than silently skipping it.

### `metropolis_logistics`

```rust
pub struct Inventory { /* ... */ }
impl Inventory {
    pub fn add(&mut self, commodity: CommodityId, amount: u32) -> u32;
    pub fn remove(&mut self, commodity: CommodityId, amount: u32) -> u32;
    pub fn available(&self, commodity: CommodityId) -> u32;
}
pub struct Shipment { /* ... */ }
```

**Contract**: `add`/`remove` return the amount actually transferred and never exceed capacity or
available stock. Partial transfers are legal.

### `metropolis_save`

```rust
pub const SCHEMA_VERSION: u32 = 1;

pub struct SaveGame { /* ... */ }
impl SaveGame {
    pub fn capture(app: &mut bevy_app::App) -> Self;
    pub fn restore(self, app: &mut bevy_app::App) -> Result<(), SaveError>;
    pub fn state_hash(&self) -> u64;
}

pub fn save_to_path(game: &SaveGame, path: &std::path::Path) -> Result<(), SaveError>;
pub fn load_from_path(path: &std::path::Path) -> Result<SaveGame, SaveError>;

pub enum SaveError {
    Io(std::io::Error),
    Corrupt(String),
    UnsupportedVersion { found: u32, supported: u32 },
    InvalidState(String),
}
```

**Contract**: `restore` MUST be atomic — on `Err`, the `App` state is unchanged.

---

## 5. Headless harness contract

The harness is `tests/headless_slice.rs` and the `metropolis --headless` binary mode.

**Inputs**: seed (`u64`), tick count (`u64`), scenario name.

**Behavior**:
1. Build an `App` with the simulation plugins only (no `bevy_winit`, no `bevy_render`).
2. Load the catalog from `data/`.
3. Apply the scenario's scripted placements (farm, mill, bakery, warehouse, roads).
4. Step `FixedUpdate` `tick_count` times.
5. Emit a JSON summary to stdout: `{ "ticks": u64, "deliveries": u32, "production_cycles": u32,
   "state_hash": u64 }`.

**Exit codes**: `0` success, `1` scenario assertion failed, `2` setup error (missing data file,
invalid catalog).

**Contract**: the harness MUST NOT require a display, a GPU, or network access, and MUST produce
identical `state_hash` for identical inputs (SC-007, SC-010).