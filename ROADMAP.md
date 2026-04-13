## Roadmap Principles

- ship a usable simulator early
- keep the protocol roadmap explicit
- prioritize real operational test value
- build scale and orchestration into the platform from the start
- avoid shallow checkbox support; prefer solid, testable implementation

---

## Versioning Strategy

Recommended versioning model:

- `0.x` while platform and major feature coverage are still evolving
- `1.0.0` once OCPP 1.6 baseline support is stable, documented, and production-usable
- minor versions for feature/profile expansion
- patch versions for fixes and compatibility improvements

---

## Phase 0 — Foundation

### Target
Create the technical base for local development and orchestrated execution.

### Scope
- monorepo structure
- Rust workspace
- Go service scaffolding
- React Studio scaffolding
- local Docker Compose
- PostgreSQL and NATS integration
- schema definitions
- observability baseline
- CI pipelines
- simple single-instance runtime

### Deliverables
- repo and folder structure finalized
- local dependency stack
- first runnable simulator shell
- first instance visible in CLI/logs/UI
- baseline docs

### Exit Criteria
- a developer can run one simulated charger locally
- runtime and control plane communicate
- instance lifecycle is visible end-to-end

### Suggested Release
- `0.1.0`

---

## Phase 1 — Local Simulator MVP

### Target
Turn the platform into a real developer tool before scaling it.

### Scope
- CLI commands for local run
- basic Studio screens
- template loading
- scenario loading
- transport connection handling
- protocol trace logging
- deterministic clock/test hooks

### Deliverables
- usable local single-instance flow
- basic multi-connector model
- scenario runner MVP
- protocol trace inspector MVP

### Exit Criteria
- a developer can run and inspect a realistic local simulator session
- scenarios can alter charger behavior
- logs and traces are usable for debugging

### Suggested Release
- `0.2.0`

---

## Phase 2 — Multi-Instance Runtime and Control Plane

### Target
Enable large-scale concurrent simulation under orchestration.

### Scope
- worker runtime hosting many instances
- control plane APIs
- scheduler service
- worker registration and health
- instance placement
- telemetry/event streaming
- run tracking

### Deliverables
- one worker can host many instances
- control plane can create/start/stop/update instances
- scheduler can assign workloads
- fleet state visible in Studio

### Exit Criteria
- multi-instance orchestration is stable
- instance density is practical
- control flows work via API, not only local CLI

### Suggested Release
- `0.3.0`

---

# OCPP 1.6J Delivery Plan

The core protocol plan is intentionally staged.

## Feature Groups in Scope

- Core
- Firmware Management
- Local Auth List Management
- Reservation
- Smart Charging
- Remote Trigger

These are planned in dependency order and in a way that maximizes product usefulness.

---

## Phase 3 — OCPP 1.6 Core

### Why first
This is the foundation of everything else.

Without a solid Core implementation, advanced feature groups become fragile and less valuable.

### Functional Scope
- BootNotification
- Heartbeat
- StatusNotification
- Authorize
- StartTransaction
- StopTransaction
- MeterValues
- ChangeAvailability
- Reset
- diagnostics-friendly status flows
- single and multi-connector support
- online/offline transitions
- fault states and basic recovery behavior

### Platform Scope
- Core capability matrix
- per-instance config
- basic assertions and fixtures
- interoperability smoke test scenarios

### Testing Scope
- unit coverage for state handling
- integration coverage for message flows
- scenario reproducibility
- CSMS interoperability smoke tests

### Exit Criteria
- stable local and orchestrated Core simulation
- clear message trace visibility
- repeatable test scenarios in CI
- compatibility matrix published

### Suggested Release
- `0.4.0`

---

## Phase 4 — Firmware Management + Remote Trigger

These two features are grouped together because they provide immediate operational value and are relatively self-contained once Core is stable.

### Firmware Management Scope
- UpdateFirmware
- firmware state transitions
- download/install/reboot simulation hooks
- success/failure paths
- timeout and invalid package scenarios
- version/reporting state exposure
- scenario-driven staged firmware behavior

### Remote Trigger Scope
- TriggerMessage support for supported message types
- configurable acceptance/rejection behavior
- trigger-induced message timing behavior
- negative-path and unsupported-trigger scenarios

### Platform Scope
- firmware state view in Studio
- artifact/config model for firmware packages
- scenario packs for update flows
- trigger action panel in GUI/API

### Testing Scope
- firmware happy path
- firmware failure path
- reboot/reconnect behaviors
- supported/unsupported trigger cases
- concurrency tests under many instances

### Exit Criteria
- firmware flows are realistic enough for backend testing
- remote triggers are visible, controllable, and testable
- negative paths are covered

### Suggested Release
- `0.5.0`

---

## Phase 5 — Local Auth List Management + Reservation

These features deepen operational realism and offline/field-behavior testing.

### Local Auth List Management Scope
- SendLocalList
- local list versioning
- full and differential updates
- local authorization cache behavior
- list acceptance/rejection
- out-of-order / stale update handling
- offline/local-auth scenarios
- config hooks for local list enforcement behavior

### Reservation Scope
- ReserveNow
- CancelReservation
- reservation state model
- reservation timeout/expiry
- reservation-to-session transitions
- conflicts and invalid reservation scenarios
- connector-specific reservation behavior

### Platform Scope
- local auth list state inspection
- reservation state view in GUI
- scenario packs for offline auth and reserved-connector use cases
- audit visibility for list version changes

### Testing Scope
- full vs differential list update cases
- stale version rejection
- offline auth fallback behavior
- reservation conflict/expiry cases
- mixed connector state behavior

### Exit Criteria
- local auth list behavior is reproducible and inspectable
- reservation flows are testable at scale
- backend behavior under edge conditions is observable

### Suggested Release
- `0.6.0`

---

## Phase 6 — Smart Charging

### Why later
Smart Charging depends on a strong charger/session model and benefits from established runtime, scenario, and observability infrastructure.

### Functional Scope
- SetChargingProfile
- ClearChargingProfile
- charging profile precedence
- stack level handling
- connector and station-wide limits
- schedule application
- profile conflicts and expiry handling
- realistic current/power enforcement in meter simulation
- dynamic schedule updates during charging

### Digital Twin Scope
- configurable power envelopes
- site constraints
- simulated throttling and load conditions
- profile-aware session behavior

### Platform Scope
- charging profile editor/viewer
- timeline visualization
- schedule assertions
- scenario packs for dynamic load management tests

### Testing Scope
- precedence correctness
- schedule application correctness
- long-running schedule transitions
- load/concurrency tests with many profile updates

### Exit Criteria
- applied limits visibly affect simulated charging behavior
- schedule changes are testable and deterministic
- orchestration handles large fleets with profile changes

### Suggested Release
- `0.7.0`

---

## Phase 7 — Stabilization for 1.0

### Target
Harden the platform into a reliable product baseline.

### Scope
- API stabilization
- RBAC / multi-tenancy
- packaging and installers
- performance tuning
- scale benchmarking
- observability dashboards
- artifact retention
- full compatibility matrix
- docs completion
- release automation hardening

### Exit Criteria
- OCPP 1.6 baseline support is well documented
- platform is operationally stable
- releases are repeatable
- scale characteristics are understood
- major workflows are polished

### Suggested Release
- `1.0.0`

---

## Milestone Summary

| Version | Focus | Outcome |
|---|---|---|
| 0.1.0 | Foundation | Repo, stack, local environment, first runnable shell |
| 0.2.0 | Local MVP | Local CLI/GUI simulator and scenario MVP |
| 0.3.0 | Scale MVP | Multi-instance worker, control plane, scheduler |
| 0.4.0 | OCPP 1.6 Core | Stable Core support |
| 0.5.0 | Firmware + Remote Trigger | Operational flows and control actions |
| 0.6.0 | Local Auth + Reservation | Offline realism and reservation handling |
| 0.7.0 | Smart Charging | Charging profile and load-control behavior |
| 1.0.0 | Production Baseline | Stable, documented, scalable OCPP 1.6 platform |

---

## Cross-Cutting Engineering Tracks

### Track A — Simulation Correctness
Focus on:
- charger state machine correctness
- connector/session lifecycle
- timing model
- meter behavior
- fault transitions

### Track B — Protocol Support
Focus on:
- message handling
- feature gating
- compatibility matrix
- conformance and interoperability tests

### Track C — Runtime and Scale
Focus on:
- worker density
- connection handling
- scheduler placement
- memory/cpu efficiency
- fleet observability

### Track D — Product Experience
Focus on:
- Studio usability
- scenario editing
- trace inspection
- operator workflows
- debugging tools

### Track E — Release Engineering
Focus on:
- CI/CD
- container builds
- automated smoke tests
- packaging
- versioning and changelogs

---

## Definition of Done

A feature is considered done when:

- implementation is complete
- unit and integration tests exist
- protocol traces are visible
- docs are updated
- scenarios/examples exist where relevant
- compatibility matrix is updated
- GUI/API support exists when applicable

---

## Future Expansion After 1.0

Not part of the initial OCPP 1.6 baseline, but potential later tracks include:

- diagnostics and log upload depth
- security extensions
- richer firmware artifact models
- ISO 15118-oriented behavior
- Plug & Charge abstractions
- hardware-in-the-loop
- recorded trace replay
- certification-focused scenario packs