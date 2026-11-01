# Quickstart: Workspace Foundation & Vertical Slice

**Feature**: `001-workspace-vertical-slice` | **Date**: 2026-11-01

Runnable validation guide. It proves the feature works end to end; implementation details live in
[tasks.md](./tasks.md) and the implementation phase.

---

## 1. Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| Rust | 1.99.1 (pinned by `rust-toolchain.toml`) | `rustup toolchain install 1.99.1` |
| WASM target | `wasm32-unknown-unknown` | `rustup target add wasm32-unknown-unknown` |
| `wasm-bindgen-cli` | 0.2.130 | `cargo install wasm-bindgen-cli --version 0.2.130` |
| `zstd` (optional, for inspecting saves) | any | `brew install zstd` |

Verify:

```bash
rustc --version          # rustc 1.99.1
cargo --version          # cargo 1.99.1
rustup target list --installed | grep wasm32-unknown-unknown
wasm-bindgen --version   # wasm-bindgen 0.2.130
```

---

## 2. Build and quality gates

```bash
cargo fmt --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```

**Expected**: all three succeed with no warnings. `cargo test` includes the headless slice,
determinism, save round-trip, routing, and placement integration tests.

### Dependency boundary gate (Constitution Principle I)

```bash
cargo tree -p metropolis_world -p metropolis_transport -p metropolis_economy \
           -p metropolis_logistics -p metropolis_population -p metropolis_save \
           -e normal --prefix none \
  | grep -E '^(wgpu|winit|image|bevy_render|bevy_window|bevy_asset|bevy_log) '
```

**Expected**: no output (exit code 1 from `grep`). Any match is a Principle I violation.

---

## 3. Run the game (desktop)

```bash
cargo run -p metropolis
```

**Expected**: a window opens showing the isometric map. Pan with WASD or middle-drag, zoom with
the scroll wheel.

---

## 4. Run the game (browser)

```bash
cargo build -p metropolis --target wasm32-unknown-unknown --release
wasm-bindgen --target web --out-dir dist \
  target/wasm32-unknown-unknown/release/metropolis.wasm
cp web/index.html dist/index.html
python3 -m http.server 8080 --directory dist
```

Open <http://localhost:8080>.

**Expected**: the same map renders in the browser; pan and zoom work; the console shows no
panics.

---

## 5. Validation scenarios

Each scenario maps to a success criterion from [spec.md](./spec.md).

### VS-1 — Navigable isometric map (SC-001, SC-002, SC-003)

1. Launch the game (desktop and browser).
2. Pan from one map corner to the opposite corner.
3. Zoom from maximum zoom-in to maximum zoom-out.
4. Hover the pointer over tiles at several zoom levels, including maximum zoom-out.

**Expected**:
- The map is visible within 10 s (desktop) / 20 s (browser).
- No visible gaps, seams, or duplicated tiles at any zoom level.
- Frame rate stays at ≥ 60 FPS desktop / ≥ 30 FPS browser (check the on-screen FPS counter).
- The highlighted tile is exactly the tile under the pointer at every sampled position.

### VS-2 — Building and road placement (SC-004)

1. Select `farm` from the catalog.
2. Move the pointer over the map — a footprint preview appears, green when valid, red when not.
3. Rotate the preview with `R`; the footprint swaps width and height.
4. Confirm placement on grass.
5. Attempt placement on water, on an occupied tile, and partially outside the map.

**Expected**:
- The preview tracks the pointer exactly.
- Rotation changes the preview and the placed building's orientation.
- Valid placement succeeds; each invalid placement is rejected with a specific reason
  (`Water`, `Occupied`, `OutOfBounds`).
- A building and a connecting road can be placed in under 30 s without documentation.

### VS-3 — Working production chain (SC-005, SC-006, SC-009)

1. Place a farm, a mill, a bakery, and a warehouse.
2. Draw roads connecting all four.
3. Let the simulation run for 5 minutes of game time.

**Expected**:
- A truck travels from the warehouse to the farm, loads grain, and delivers it to the mill.
- The mill produces flour only after grain arrives.
- The bakery produces bread only after flour arrives.
- At least one delivery and one downstream production cycle complete within 5 game minutes.
- Pan the camera far away so the chain is off-screen; production and deliveries continue at the
  same rate (within 5%).
- Remove a road segment on the delivery route; downstream production stops within one game
  minute.

### VS-4 — Save and resume (SC-008)

1. Build a city with ~100 buildings and ~20 vehicles (or use the `--scenario stress` harness
   mode).
2. Save mid-delivery, while a truck is between two tiles.
3. Quit and relaunch.
4. Load the save.

**Expected**:
- Save completes in under 5 s and reports success.
- Load completes in under 5 s.
- All buildings, roads, inventories, and in-progress deliveries are restored exactly, including
  the truck's position between tiles.
- Loading a save with a bumped `schema_version` reports a clear `UnsupportedVersion` error and
  leaves the running game untouched.
- Loading a truncated file reports `Corrupt` and leaves the running game untouched.

### VS-5 — Determinism (SC-007, SC-010)

```bash
cargo test --test determinism
cargo test --test headless_slice
```

**Expected**: both pass. The determinism test runs the same seed twice and asserts identical
`state_hash` values. The headless slice test runs the full farm → mill → bakery → warehouse
scenario with no display and passes deterministically.

### VS-6 — Headless harness

```bash
cargo run -p metropolis -- --headless --seed 42 --ticks 3600 --scenario slice
```

**Expected**: JSON on stdout:

```json
{ "ticks": 3600, "deliveries": 3, "production_cycles": 5, "state_hash": 12345678901234567890 }
```

Running the same command twice produces the same `state_hash`. Exit code `0`.

---

## 6. Performance validation (Constitution Principle IV)

```bash
cargo run -p metropolis --release -- --headless --seed 42 --ticks 6000 --scenario stress
```

**Expected** (printed by the harness):
- Simulation tick < 50 ms per 100 ms tick.
- Save/load of the stress scenario < 5 s each.

For rendering, run the release build and read the on-screen FPS counter while panning across the
full map at maximum zoom-out.

---

## 7. Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `error: failed to run custom build command for wgpu` | Missing graphics toolchain | Install platform graphics dev packages |
| Blank canvas in browser | `wasm-bindgen` output not copied to `dist/` | Re-run the `wasm-bindgen` step |
| `CatalogError` at startup | Invalid `data/` file | Run `cargo run -p asset_pipeline -- validate-data data/` |
| `SaveError::UnsupportedVersion` | Save from a newer build | Use a save produced by the current `SCHEMA_VERSION` |
| `cargo tree` gate fails | A rendering crate leaked into a simulation crate | Remove the dependency; see [contracts/sim-api.md](./contracts/sim-api.md) §1 |