# Contract: Save File Format

**Feature**: `001-workspace-vertical-slice` | **Date**: 2026-11-01

Saves are Serde-serialized `SaveGame` values encoded as MessagePack and compressed with Zstd
(Constitution Principle VI, FR-014, FR-015).

---

## 1. File layout

```text
┌──────────────────────────────────────────────┐
│ Zstd frame                                   │
│  └── MessagePack map (named fields)          │
│       ├── schema_version: u32                │
│       ├── game_version: str                  │
│       ├── seed: u64                          │
│       ├── clock: { ... }                     │
│       ├── map: { ... }                       │
│       ├── buildings: [ ... ]                 │
│       ├── roads: [ ... ]                     │
│       ├── vehicles: [ ... ]                  │
│       ├── shipments: [ ... ]                 │
│       └── next_id: u32                       │
└──────────────────────────────────────────────┘
```

**Encoding**: `rmp_serde::to_vec_named` (named fields, so field reordering is safe).
**Compression**: `zstd::stream::encode_all(payload, 3)`.
**Extension**: `.mpsave`.
**Default location**: platform data directory under `metropolis/saves/`.

---

## 2. Root schema (`schema_version = 1`)

| Field | Type | Notes |
|-------|------|-------|
| `schema_version` | `u32` | `1` |
| `game_version` | `String` | `env!("CARGO_PKG_VERSION")` at write time |
| `seed` | `u64` | world seed; required to reconstruct `RngStreams` |
| `clock` | `ClockSnapshot` | see §3 |
| `map` | `MapSnapshot` | see §4 |
| `buildings` | `Vec<BuildingSnapshot>` | see §5 |
| `roads` | `Vec<TileSnapshot>` | see §6 |
| `vehicles` | `Vec<VehicleSnapshot>` | see §7 |
| `shipments` | `Vec<ShipmentSnapshot>` | see §8 |
| `next_id` | `u32` | id allocator high-water mark |

---

## 3. `ClockSnapshot`

| Field | Type |
|-------|------|
| `tick` | `u64` |
| `game_minute` | `u64` |
| `game_hour` | `u64` |
| `game_day` | `u64` |

**Rule**: `game_minute == tick / 60`, `game_hour == game_minute / 60`,
`game_day == game_hour / 24`. A save violating this is `SaveError::InvalidState`.

---

## 4. `MapSnapshot`

| Field | Type | Notes |
|-------|------|-------|
| `width` | `u32` | 512 |
| `height` | `u32` | 512 |
| `chunks` | `Vec<ChunkSnapshot>` | row-major, 256 entries |

`ChunkSnapshot`:

| Field | Type | Notes |
|-------|------|-------|
| `coord` | `{ x: i32, y: i32 }` | |
| `terrain` | `Vec<u16>` | exactly 1024 entries |
| `buildings` | `Vec<u32>` | exactly 1024 entries |

**Rule**: `chunks.len() == (width / 32) * (height / 32)` and every `terrain`/`buildings` vector
has exactly `32 * 32` entries. Violations are `SaveError::InvalidState`.

---

## 5. `BuildingSnapshot`

| Field | Type | Notes |
|-------|------|-------|
| `id` | `u32` | |
| `type_id` | `u32` | index into the catalog |
| `origin` | `{ x: i32, y: i32 }` | |
| `rotation` | `u8` | `0..=3` |
| `level` | `u8` | |
| `inventory` | `Vec<{ commodity: u32, amount: u32 }>` | |
| `production` | `Option<{ progress_ticks: u32, interval_ticks: u32, active: bool }>` | |

**Rule**: `type_id` MUST be a valid catalog index at load time; otherwise
`SaveError::InvalidState` (the catalog is not stored in the save — it is content, not state).

---

## 6. `TileSnapshot`

| Field | Type |
|-------|------|
| `x` | `i32` |
| `y` | `i32` |

**Rule**: every road tile MUST be inside the map and MUST have terrain != `Water`.

---

## 7. `VehicleSnapshot`

| Field | Type | Notes |
|-------|------|-------|
| `id` | `u32` | |
| `kind` | `u32` | catalog index |
| `cargo` | `Vec<{ commodity: u32, amount: u32 }>` | |
| `state` | `u8` | `0=Idle, 1=ToPickup, 2=Loading, 3=ToDelivery, 4=Unloading, 5=Blocked` |
| `route` | `Option<Vec<TileSnapshot>>` | |
| `route_index` | `usize` | |
| `progress` | `f32` | `0.0..=1.0` |

**Rule**: when `route` is `Some`, every tile MUST be a road tile present in `roads`, and
`route_index < route.len()`. Violations are `SaveError::InvalidState`.

---

## 8. `ShipmentSnapshot`

| Field | Type | Notes |
|-------|------|-------|
| `id` | `u32` | |
| `commodity` | `u32` | catalog index |
| `amount` | `u32` | |
| `source` | `u32` | `BuildingId` |
| `destination` | `u32` | `BuildingId` |
| `state` | `u8` | `0=Pending, 1=Assigned, 2=InTransit, 3=Delivered, 4=Failed` |
| `assigned_vehicle` | `Option<u32>` | |

**Rule**: `source` and `destination` MUST resolve to buildings present in `buildings`. When
`state == Assigned` or `InTransit`, `assigned_vehicle` MUST be `Some` and resolve to a vehicle
present in `vehicles`.

---

## 9. Version handling

| Condition | Behavior |
|-----------|----------|
| `schema_version == SCHEMA_VERSION` | deserialize directly |
| `schema_version < SCHEMA_VERSION` | run the migration chain `migrate::v{n}_to_v{n+1}`; if any step is missing, return `SaveError::UnsupportedVersion { found, supported }` |
| `schema_version > SCHEMA_VERSION` | return `SaveError::UnsupportedVersion { found, supported }` |
| Zstd decode failure | `SaveError::Corrupt("zstd: …")` |
| MessagePack decode failure | `SaveError::Corrupt("msgpack: …")` |
| Any §3–§8 rule violated | `SaveError::InvalidState("…")` |

**Contract**: on any `Err`, the running game state MUST be unchanged. `restore` builds the new
state into a staging structure and only swaps it in after full validation.

---

## 10. Determinism hash

```rust
pub fn state_hash(&self) -> u64;
```

**Contract**: hashes the canonical MessagePack encoding of `SaveGame` (uncompressed, named
fields) with `xxhash`-style 64-bit hashing. Two runs from the same seed and input sequence MUST
produce the same `state_hash` (SC-007). The hash is computed over the same bytes that would be
written to disk, so the determinism test and the save round-trip test share one definition of
state.

---

## 11. Size and time budget

| Scenario | Target |
|----------|--------|
| 100 buildings + 20 vehicles + full 512 × 512 terrain | save < 5 s, load < 5 s (SC-008) |
| Typical slice save | < 1 MB compressed |

**Rule**: terrain is the dominant payload (256 chunks × 1024 × 2 bytes ≈ 512 KB uncompressed).
Zstd level 3 compresses the mostly-uniform terrain well below the 1 MB target. If a save exceeds
5 MB, the writer MUST log a warning naming the largest section.