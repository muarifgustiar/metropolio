<!--
Sync Impact Report
==================
Version change: 1.0.0 → 1.1.0
Bump rationale: MINOR — materially expanded platform compatibility, security, and validation
  governance to incorporate the supplied master requirements.

Modified principles:
  - IV. Browser-First Performance Budgets → IV. Cross-Platform Compatibility & Measured Performance
  - VI. Reproducible, Versioned Game State → VI. Reproducible, Validated Game State

Added sections:
  - Security, Data, and Platform Boundaries
  - Platform support and release validation added to Development Workflow & Quality Gates

Removed sections: none

Deferred TODOs:
  - The supplied prompt's Phase 1 implementation request is a feature intent, not governance;
    define or reconcile it through $speckit-specify before implementation.
  - Android and iOS support requirements need a scoped feature specification; they are outside
    the current vertical-slice assumptions.
-->

# Metropolis Constitution

## Core Principles

### I. Simulation–Rendering Separation (NON-NEGOTIABLE)

The simulation MUST NOT depend on rendering. Simulation crates (`metropolis_sim` and its
domain crates) MUST NOT import Bevy rendering, windowing, or asset APIs, and MUST run headless
for automated testing. A factory, vehicle, or shipment MUST continue to progress while outside
the camera viewport. Rendering MAY read simulation state; it MUST NOT own authoritative game
state. Gameplay orchestration (`metropolis_game`) and the Bevy application (`metropolis_app`)
sit above both layers and MUST NOT be required by the simulation.

Rationale: headless simulation enables deterministic tests, fast CI, and future server-side or
batch execution without a GPU.

### II. Deterministic Fixed-Timestep Simulation

All simulation systems MUST run on a fixed timestep, never on render delta. Systems MUST be
scheduled by frequency rather than executed every frame. Randomness MUST come from seeded
generators stored in world state; no system may use unseeded or thread-local RNG. Given the
same seed, initial state, and input sequence, a simulation run MUST produce equivalent results.
The project MUST define explicit rules for deterministic ordering and floating-point behavior
where those affect simulation results.

Rationale: determinism makes bugs reproducible, saves replayable, and performance regressions
measurable.

### III. Data-Oriented ECS Discipline

ECS entities MUST be reserved for objects with lifecycle and direct interaction (buildings,
vehicles, stations, shipments). Aggregate populations MUST be stored as compact data
structures, not one entity per citizen. Components MUST be plain data; behavior lives in
systems. Chunked world data MUST use contiguous, cache-friendly storage with a fixed
`CHUNK_SIZE` of 32. Hot-path allocations inside per-tick systems MUST be avoided or pooled.

Rationale: scalable simulation depends on data locality, bounded work, and clear ownership of
state.

### IV. Cross-Platform Compatibility & Measured Performance

Platform compatibility MUST be considered from the start of each feature. Core simulation MUST
not contain platform-specific behavior. Before selecting a dependency or claiming support for a
target, maintainers MUST verify compatibility with the selected engine version and target
platform. Unsupported features MUST have a documented limitation and compatible alternative
where the feature is required. A platform is supported only after its application is packaged
and runtime-validated; successful compilation alone is insufficient.

Performance targets are acceptance criteria only when adopted by an approved feature
specification. Performance-sensitive changes MUST be measured against that specification.
Browser and mobile implementations MUST account for platform limits in memory, threading,
storage, lifecycle, and rendering, and MUST NOT assume native filesystem access or shared-memory
threads. Optimization MUST follow profiling evidence.

Rationale: a shared codebase can still encounter different runtime and packaging constraints on
desktop, browser, and mobile.

### V. Vertical Slice Before Scale

Work MUST proceed through approved milestones in order, and the first gameplay deliverable
MUST be a working end-to-end vertical slice. Large-scale simulation MUST NOT begin before the
slice is proven. Scope beyond the current milestone MUST be deferred rather than partially
implemented.

Rationale: integration risk in supply-chain games is concentrated in the end-to-end loop, not
in any single subsystem.

### VI. Reproducible, Validated Game State

Persisted state MUST use a versioned schema. Loading older state MUST migrate it or fail with a
clear, actionable error; it MUST NOT silently corrupt state. Save operations MUST protect
existing data against partial writes where the platform permits. Simulation-relevant content
MUST be defined in validated, versioned data files rather than hard-coded in simulation
systems. Optional backend services MUST NOT be required for offline single-player gameplay.

Rationale: players invest time in a city, so save integrity and validated content are essential.

## Security, Data, and Platform Boundaries

External game data and save files MUST be validated before use and MUST have resource limits
against malformed inputs. Asset and save handling MUST prevent path traversal. `unsafe` Rust
MUST be avoided unless justified, and every unsafe block MUST document its safety invariant.
Third-party dependencies MUST be reviewed for platform compatibility and security. Backend
credentials MUST NOT be embedded in game binaries; backend communication MUST use HTTPS and
cloud-save ownership MUST be validated.

Platform adapters MUST own operating-system, browser, and mobile APIs, persistence, and
lifecycle handling. The simulation MUST communicate through platform-independent commands
and state. No blocking I/O may run on the main rendering thread.

## Technology Stack & Architecture Constraints

The project uses Rust stable and Bevy for ECS, rendering, and the game loop. The repository
MUST remain a Cargo workspace with an application entry point and separate packages for core,
world, rendering, transport, economy, logistics, population, and save concerns, plus dedicated
folders for assets, data, tooling, and integration tests. The simulation core MUST remain
headless and independent of graphics, windowing, and platform APIs. Circular crate dependencies
are prohibited.

Specific Bevy versions, graphics backends, serialization formats, compression algorithms,
pathfinding strategies, backend technologies, and platform performance targets MUST be chosen
in feature plans based on verified compatibility and measured requirements. Optional services
may support accounts or cloud saves but MUST NOT be required for offline play.

## Development Workflow & Quality Gates

- Every change MUST be validated with `cargo fmt --check`, `cargo clippy` (no new warnings),
  and `cargo test` before merge.
- Simulation logic MUST have headless unit tests; cross-package behavior MUST have integration
  tests under `tests/`.
- Determinism MUST be covered by tests that run the same seed and inputs twice and compare
  resulting state.
- Performance-sensitive changes MUST include benchmark or profiling evidence against targets
  adopted by the relevant feature specification.
- Browser builds MUST be compiled and smoke-tested in a browser before a browser milestone is
  declared complete. Mobile and desktop support claims MUST include packaging and runtime
  validation on the applicable simulator, emulator, or device.
- Dependency security and license checks MUST run in CI. CI MUST validate the targets required
  by the current milestone; unsupported or unavailable runtime checks MUST be reported clearly.
- Pull requests MUST state which approved milestone they advance and MUST NOT introduce work
  from a later milestone.
- AI-assisted changes MUST inspect the repository first, verify APIs against the selected
  dependency version, report validation results accurately, and document known platform
  limitations. Untested platforms MUST NOT be represented as working.

## Governance

This constitution supersedes other development practices when they conflict. Amendments require
a written proposal describing the change and rationale, explicit approval by the project
maintainer, and an update to this file with a semantic version bump and amended date. Dependent
templates and commands read this file at runtime and MUST NOT be edited as part of an amendment.

Versioning follows semantic versioning: MAJOR for backward-incompatible governance or principle
removals/redefinitions, MINOR for new principles or materially expanded guidance, and PATCH for
clarifications and non-semantic refinements.

Every pull request and review MUST verify compliance with the Core Principles. Any deviation
MUST be documented and justified in the pull request; unjustified complexity or non-compliance
blocks merge. Runtime development guidance lives in this file and the repository README.

**Version**: 1.1.0 | **Ratified**: 2026-11-01 | **Last Amended**: 2026-11-01
