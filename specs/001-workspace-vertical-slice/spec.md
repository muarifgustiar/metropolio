# Feature Specification: Workspace Foundation & Vertical Slice

**Feature Branch**: `001-workspace-vertical-slice`

**Created**: 2026-11-01

**Status**: Draft

**Input**: User description: "Scaffold the Cargo workspace (`apps/game`, `crates/*`, `assets/`, `data/`, `tools/`, `tests/`) → `/speckit-specify` for the workspace/vertical-slice feature, then `/speckit-plan`."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Playable Isometric Map in the Browser (Priority: P1)

A player opens the game in a web browser and sees a 2.5D isometric city map. They can pan and
zoom the camera, and the map stays smooth while moving across the whole world. The same build
also runs as a desktop application.

**Why this priority**: Nothing else in the game can be demonstrated or validated without a
running, navigable world. This is the first milestone and the foundation every later story
builds on.

**Independent Test**: Launch the game (desktop and browser), pan and zoom across the full map,
and confirm the world renders correctly with no visible gaps, seams, or frame-rate collapse.

**Acceptance Scenarios**:

1. **Given** the game is launched, **When** the world loads, **Then** an isometric map of the
   full 512 × 512 tile world is displayed with terrain visible in every direction.
2. **Given** the map is displayed, **When** the player pans and zooms, **Then** the camera
   moves smoothly and the map remains visually continuous with no missing or duplicated tiles.
3. **Given** the player is viewing a corner of the map, **When** they pan to the opposite
   corner, **Then** only the visible portion of the world is drawn and frame rate stays within
   the target range.
4. **Given** the player hovers the pointer over a tile, **When** the pointer moves, **Then** the
   tile under the pointer is correctly identified (highlight follows the pointer exactly).

---

### User Story 2 - Place Buildings and Roads (Priority: P2)

A player selects a building from a catalog, places it on the map, and rotates it to face the
desired direction. They then draw roads between buildings. Invalid placements (occupied tiles,
water, out of bounds) are rejected with clear feedback.

**Why this priority**: Building and road placement is the core interaction of a city-builder and
is the prerequisite for any transport or production behavior.

**Independent Test**: Place several buildings of different footprints, rotate them, draw a road
connecting them, and confirm the map state reflects every placement and rejects invalid ones.

**Acceptance Scenarios**:

1. **Given** the player has selected a building, **When** they move the pointer over the map,
   **Then** a placement preview shows the footprint and whether the placement is valid.
2. **Given** a valid preview, **When** the player confirms placement, **Then** the building
   appears on the map and occupies its footprint.
3. **Given** a building is selected for placement, **When** the player rotates it, **Then** the
   preview and the placed building use the rotated orientation.
4. **Given** a tile is already occupied, is water, or is outside the map, **When** the player
   attempts to place a building there, **Then** the placement is rejected and the reason is
   shown.
5. **Given** the player is in road mode, **When** they drag between two points, **Then** a road
   is created along the drawn path and connects to adjacent roads.

---

### User Story 3 - Working Production Chain with Freight Delivery (Priority: P3)

A player builds a farm, a mill, a bakery, and a warehouse, connects them with roads, and watches
a truck carry goods along the road network. The bakery produces bread only after flour has
actually been delivered by truck; if the road is removed, production stops.

**Why this priority**: This is the vertical slice's proof point — it demonstrates that
transport, logistics, and production are genuinely integrated rather than simulated in
isolation.

**Independent Test**: Build the four buildings and connecting roads, observe a truck complete a
delivery, and confirm the downstream building's output increases only after delivery. Remove a
road segment and confirm production halts.

**Acceptance Scenarios**:

1. **Given** a farm, mill, bakery, and warehouse connected by roads, **When** the simulation
   runs, **Then** a truck travels from the warehouse to the farm, picks up raw material, and
   delivers it to the mill.
2. **Given** the mill has received raw material, **When** the production interval elapses,
   **Then** the mill produces flour and its output inventory increases.
3. **Given** the bakery has no flour, **When** the simulation runs, **Then** the bakery does not
   produce bread until flour is delivered.
4. **Given** a road segment on the delivery route is removed, **When** the simulation runs,
   **Then** the truck can no longer complete the route and production downstream stops.
5. **Given** the player is viewing a different part of the map, **When** the simulation runs,
   **Then** production and deliveries continue at the same rate as when visible.

---

### User Story 4 - Save and Resume a City (Priority: P4)

A player saves their city and later loads it, resuming with the exact same buildings, roads,
inventories, and in-progress deliveries.

**Why this priority**: Saves protect player investment and are required to validate that the
simulation state is complete and serializable. It is lower priority than the slice itself but
must land before the slice is considered done.

**Independent Test**: Save a city mid-delivery, restart the game, load the save, and confirm the
city and in-progress delivery resume identically.

**Acceptance Scenarios**:

1. **Given** a city with buildings, roads, and inventories, **When** the player saves, **Then**
   the save completes within the target time and reports success.
2. **Given** a saved city, **When** the player loads it in a fresh session, **Then** all
   buildings, roads, inventories, and in-progress deliveries are restored.
3. **Given** a save produced by an older schema version, **When** the player loads it, **Then**
   the game either migrates it or reports a clear, actionable error without corrupting state.

---

### Edge Cases

- The player attempts to place a building partially outside the map boundary.
- The player draws a road that loops back on itself or crosses an existing road.
- A delivery route becomes unreachable while a truck is en route (road removed mid-trip).
- A building is demolished while it holds inventory or has a pending order.
- The player saves while a truck is between two tiles.
- The browser tab is backgrounded and the simulation resumes later without a time jump.
- The player zooms out to the maximum level where the whole map is visible.
- The save file is truncated or corrupted.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The project MUST be organized as a multi-package workspace with a single
  application entry point and separate packages for core, world, rendering, transport,
  economy, logistics, population, and save concerns, plus dedicated folders for assets, game
  data, tooling, and integration tests.
- **FR-002**: The simulation packages MUST be usable without any rendering, windowing, or
  graphics dependency, and MUST be runnable headlessly for automated testing.
- **FR-003**: The game MUST build and run both as a desktop application and as a browser
  application.
- **FR-004**: The world MUST be a 512 × 512 tile isometric map divided into fixed-size chunks,
  with only visible chunks drawn.
- **FR-005**: The camera MUST support panning and zooming across the entire map, and pointer
  position MUST map to the correct tile.
- **FR-006**: Players MUST be able to select a building from a catalog, preview its footprint,
  rotate it, and place it on valid terrain.
- **FR-007**: The system MUST reject placements on occupied tiles, water, or out-of-bounds
  tiles, and MUST explain the rejection.
- **FR-008**: Players MUST be able to draw roads between points, and roads MUST connect to
  adjacent road tiles to form a routable network.
- **FR-009**: Vehicles MUST route over the road network and move along their route at a fixed
  simulation rate.
- **FR-010**: Buildings MUST produce output only when their required inputs are present in
  inventory.
- **FR-011**: Logistics MUST dispatch vehicles to move goods between buildings according to
  orders, and MUST stop when no valid route exists.
- **FR-012**: Simulation MUST continue for all buildings and vehicles regardless of whether they
  are visible on screen.
- **FR-013**: Simulation MUST advance on a fixed timestep so results do not depend on frame
  rate, and MUST be reproducible from the same starting state and seed.
- **FR-014**: Players MUST be able to save and load a city, including buildings, roads,
  inventories, and in-progress deliveries.
- **FR-015**: Saves MUST carry a schema version, and loading an incompatible save MUST produce a
  clear error rather than corrupt state.
- **FR-016**: Building, industry, commodity, and vehicle definitions MUST be loaded from
  versioned data files rather than hard-coded in simulation logic.
- **FR-017**: The system MUST provide a headless test harness that runs the simulation without a
  display for automated verification.

### Key Entities *(include if feature involves data)*

- **World Map**: The full playable area; composed of chunks; each tile has terrain type and
  occupancy.
- **Chunk**: A fixed-size square region of the map; the unit of rendering and culling.
- **Building**: A placed structure with a type, footprint, orientation, level, and inventory.
- **Building Definition**: Catalog entry describing footprint, cost, inputs, outputs, and
  production interval.
- **Road Network**: Graph of connected road tiles used for routing.
- **Vehicle**: A moving carrier with a route, cargo, capacity, and current position.
- **Order / Shipment**: A request to move a quantity of a commodity from a source building to a
  destination building.
- **Inventory**: Per-building stock of commodities with capacity limits.
- **Commodity**: A tradeable good (e.g., grain, flour, bread).
- **Save Game**: A versioned snapshot of all simulation state.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A new player can launch the game and see a navigable isometric map within 10
  seconds on a mid-range desktop and within 20 seconds in a browser.
- **SC-002**: Panning and zooming across the full map maintains at least 60 frames per second on
  desktop and at least 30 frames per second in a browser.
- **SC-003**: Pointer-to-tile selection is correct for 100% of sampled positions across the map,
  including at maximum zoom-out.
- **SC-004**: A player can place a building and draw a connecting road in under 30 seconds
  without reading documentation.
- **SC-005**: The full farm → mill → bakery → warehouse chain completes at least one delivery
  and one downstream production cycle within 5 minutes of game time.
- **SC-006**: Removing a road segment stops downstream production within one simulation minute.
- **SC-007**: Simulation results are identical across two runs from the same starting state and
  seed (100% state match).
- **SC-008**: Saving and loading a city with 100 buildings and 20 vehicles completes in under 5
  seconds each, with 100% of state restored.
- **SC-009**: Simulation continues at the same rate when the affected area is off-screen
  (within 5% of on-screen rate).
- **SC-010**: The headless test suite runs the full vertical slice scenario without a display
  and passes deterministically.

## Assumptions

- The target player is a single-player city-builder fan on a mid-range desktop or laptop
  browser; mobile is out of scope for this feature.
- The vertical slice uses one fixed map with a small catalog (farm, mill, bakery, warehouse,
  road, truck) rather than a full building set.
- Terrain is procedurally generated or loaded from a fixed data file; terrain editing is out of
  scope.
- Economy, population, zoning, rail, and ships are out of scope for this feature and belong to
  later milestones.
- Cloud saves and accounts are out of scope; only local save/load is included.
- The player has a modern browser with WebGPU or WebGL2 support.
- Asset art for the slice is placeholder-quality; final art is produced later.
- The project constitution governs architecture and performance constraints and takes
  precedence over this specification where they conflict.
