# Contract: Content Data Files (`data/`)

**Feature**: `001-workspace-vertical-slice` | **Date**: 2026-11-01

All game content is defined in versioned RON files under `data/` and loaded at startup
(Constitution Principle VI, FR-016). No building, commodity, industry, or vehicle definition may
be hard-coded in simulation logic.

---

## 1. Directory layout

```text
data/
├── buildings/
│   ├── farm.ron
│   ├── mill.ron
│   ├── bakery.ron
│   ├── warehouse.ron
│   └── road.ron
├── industries/
│   ├── grain_flour.ron
│   └── bread.ron
├── commodities/
│   ├── grain.ron
│   ├── flour.ron
│   └── bread.ron
└── vehicles/
    └── truck.ron
```

**Contract**: every `*.ron` file in these four directories MUST deserialize into its
corresponding definition type. A file that fails to parse or validate aborts startup with a
`CatalogError` naming the file and the reason — files are never silently skipped.

---

## 2. Common envelope

Every data file carries a schema version so content can evolve independently of code:

```ron
(
    schema_version: 1,
    // ... type-specific fields
)
```

**Contract**: `schema_version` MUST equal `1` for this feature. A higher value is rejected with
`CatalogError::UnsupportedSchema { file, found, supported }`.

---

## 3. `CommodityDef` — `data/commodities/*.ron`

```ron
(
    schema_version: 1,
    id: "grain",
    display_name: "Grain",
    base_value: 4,
)
```

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| `schema_version` | `u32` | yes | must be `1` |
| `id` | `String` | yes | non-empty, unique across all commodities, `[a-z0-9_]+` |
| `display_name` | `String` | yes | non-empty |
| `base_value` | `u32` | yes | `> 0` |

---

## 4. `BuildingDef` — `data/buildings/*.ron`

```ron
(
    schema_version: 1,
    id: "mill",
    display_name: "Flour Mill",
    footprint: (3, 3),
    cost: 250,
    inputs: [(commodity: "grain", amount: 2)],
    outputs: [(commodity: "flour", amount: 1)],
    production_interval_minutes: 2,
    storage_capacity: 40,
    requires_road: true,
)
```

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| `schema_version` | `u32` | yes | must be `1` |
| `id` | `String` | yes | non-empty, unique across all buildings, `[a-z0-9_]+` |
| `display_name` | `String` | yes | non-empty |
| `footprint` | `(u8, u8)` | yes | both components `>= 1` and `<= 8` |
| `cost` | `u32` | yes | `>= 0` |
| `inputs` | `Vec<CommodityAmount>` | yes | may be empty; every `commodity` must resolve |
| `outputs` | `Vec<CommodityAmount>` | yes | may be empty; every `commodity` must resolve |
| `production_interval_minutes` | `u32` | yes | `> 0` when `outputs` is non-empty; `0` allowed otherwise |
| `storage_capacity` | `u32` | yes | `> 0` |
| `requires_road` | `bool` | yes | — |

**Cross-field rule**: a building with non-empty `inputs` MUST have `storage_capacity` at least
the sum of its input amounts, otherwise it can never start a cycle.

---

## 5. `IndustryDef` — `data/industries/*.ron`

```ron
(
    schema_version: 1,
    id: "grain_flour",
    building_type: "mill",
    inputs: [(commodity: "grain", amount: 2)],
    outputs: [(commodity: "flour", amount: 1)],
    interval_minutes: 2,
)
```

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| `schema_version` | `u32` | yes | must be `1` |
| `id` | `String` | yes | non-empty, unique across all industries |
| `building_type` | `String` | yes | must resolve to a `BuildingDef.id` |
| `inputs` | `Vec<CommodityAmount>` | yes | every `commodity` must resolve |
| `outputs` | `Vec<CommodityAmount>` | yes | non-empty; every `commodity` must resolve |
| `interval_minutes` | `u32` | yes | `> 0` |

**Cross-field rule**: `inputs`, `outputs`, and `interval_minutes` MUST match the referenced
`BuildingDef` exactly. A mismatch is a `CatalogError::IndustryMismatch`.

---

## 6. `VehicleDef` — `data/vehicles/*.ron`

```ron
(
    schema_version: 1,
    id: "truck",
    display_name: "Delivery Truck",
    capacity: 20,
    speed_tiles_per_tick: 0.25,
)
```

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| `schema_version` | `u32` | yes | must be `1` |
| `id` | `String` | yes | non-empty, unique across all vehicles |
| `display_name` | `String` | yes | non-empty |
| `capacity` | `u32` | yes | `> 0` |
| `speed_tiles_per_tick` | `f32` | yes | `> 0.0` and `<= 1.0` |

**Rule**: `speed_tiles_per_tick <= 1.0` guarantees a vehicle advances at most one tile per tick,
which keeps `Vehicle::progress` bounded and the movement deterministic.

---

## 7. Slice content set

The vertical slice ships exactly these definitions:

| File | `id` | Footprint | Inputs | Outputs | Interval |
|------|------|-----------|--------|---------|----------|
| `buildings/farm.ron` | `farm` | (3, 3) | — | grain × 4 | 2 min |
| `buildings/mill.ron` | `mill` | (3, 3) | grain × 2 | flour × 1 | 2 min |
| `buildings/bakery.ron` | `bakery` | (2, 2) | flour × 1 | bread × 2 | 3 min |
| `buildings/warehouse.ron` | `warehouse` | (4, 4) | — | — | 0 |
| `buildings/road.ron` | `road` | (1, 1) | — | — | 0 |
| `commodities/grain.ron` | `grain` | — | — | — | — |
| `commodities/flour.ron` | `flour` | — | — | — | — |
| `commodities/bread.ron` | `bread` | — | — | — | — |
| `industries/grain_flour.ron` | `grain_flour` | — | grain × 2 | flour × 1 | 2 min |
| `industries/bread.ron` | `bread` | — | flour × 1 | bread × 2 | 3 min |
| `vehicles/truck.ron` | `truck` | — | — | — | — |

**Contract**: the chain MUST be satisfiable — the farm produces grain faster than the mill
consumes it, and the mill produces flour faster than the bakery consumes it, so the slice
reaches steady state within SC-005's 5-minute window.

---

## 8. Validation entry point

```rust
pub fn validate_catalog_dir(dir: &std::path::Path) -> Result<Catalog, CatalogError>;
```

**Contract**: returns `Ok` only when every file parses, every `schema_version` is supported,
every id is unique, and every cross-reference resolves. `tools/asset_pipeline` exposes this as
`asset_pipeline validate-data data/` so CI can gate content changes without building the game.