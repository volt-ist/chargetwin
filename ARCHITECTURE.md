# ChargeTwin — Architecture

This document describes the internal architecture of ChargeTwin: how the components are structured, how they communicate, and where the key boundaries lie.

---

## Guiding principles

- **Headless first.** The simulation engine has no GUI dependency. The browser GUI is a client of the local HTTP server, not embedded in the engine.
- **Single engine, many surfaces.** The same simulation core powers the CLI, the orchestrator, and the GUI. There is no separate "fleet engine" — the orchestrator manages a pool of the same charge point instances used in local mode.
- **Single binary, clean internal boundaries.** v1 ships as one self-contained binary. Components communicate through well-defined internal interfaces (traits and channels), not network calls. This makes it straightforward to extract the orchestrator into a separate service in a future release without redesigning the protocol between them.
- **Multi-instance efficiency.** The engine is designed to run many charge point instances concurrently in a single process using async Tokio tasks, not threads-per-instance. This keeps memory usage low and enables fleet simulation without cluster overhead.
- **Deterministic scenarios.** The scenario executor is a pure state machine over a declarative YAML description. Given the same scenario and CSMS responses, execution is reproducible.
- **Profile-led delivery.** OCPP capability is added in complete profiles, not individual messages. Each profile is tested end-to-end before shipping.

---

## Component map

```
┌─────────────────────────────────────────────────────────────────┐
│                        chargetwin binary                        │
│                                                                 │
│  ┌──────────────┐    ┌──────────────────────────────────────┐  │
│  │     CLI      │    │           Local server (Axum)        │  │
│  │  (commands)  │    │   HTTP REST + WebSocket event stream  │  │
│  └──────┬───────┘    └────────────────┬─────────────────────┘  │
│         │                             │                          │
│         ▼                             ▼                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Orchestrator                           │   │
│  │   instance registry · scenario scheduler · metrics bus   │   │
│  └────────────────────────┬─────────────────────────────────┘   │
│                           │  spawns N                            │
│              ┌────────────┴────────────┐                        │
│              ▼                         ▼                        │
│      ┌──────────────┐         ┌──────────────┐                  │
│      │  CP instance │   ...   │  CP instance │                  │
│      │   (core)     │         │   (core)     │                  │
│      └──────┬───────┘         └──────┬───────┘                  │
│             │                        │                           │
│             ▼                        ▼                           │
│      ┌──────────────┐         ┌──────────────┐                  │
│      │ OCPP adapter │         │ OCPP adapter │                  │
│      │  (per ver.)  │         │  (per ver.)  │                  │
│      └──────┬───────┘         └──────┬───────┘                  │
└─────────────┼────────────────────────┼────────────────────────-─┘
              │ WebSocket              │ WebSocket
              ▼                        ▼
           CSMS                      CSMS
```

---

## Components

### Simulation core (`chargetwin-core`)

The simulation core models a single charge point. It is completely independent of transport, GUI, and orchestration concerns.

**Responsibilities:**
- Maintains the charge point state machine (availability, connector state, session state, transaction state)
- Exposes a command interface: `ChargePoint::handle(command) -> Effect`
- Emits typed events for every state transition (e.g. `SessionStarted`, `MeterValueRecorded`, `FaultRaised`)
- Enforces OCPP-defined state transition rules; invalid transitions return errors, not panics

**Design:**
- Pure state machine — no I/O, no networking, no timers
- Fully synchronous internally; Tokio integration happens in the adapter layer
- One `ChargePoint` struct per simulated charge point; cheap to instantiate

**Key types:**
```
ChargePoint          — the state machine
ChargePointState     — availability, connector, session, transaction
ChargePointCommand   — inputs (e.g. AuthorizeRequest, RemoteStartTransaction)
ChargePointEffect    — outputs (e.g. SendMessage, EmitEvent, SetTimer)
ChargePointEvent     — observable state changes for subscribers
```

### OCPP adapter (`chargetwin-ocpp`)

Translates between the typed core state machine and the wire format of a specific OCPP version.

**Responsibilities:**
- Encodes outbound core effects into OCPP JSON (or XML for 1.6 SOAP, if ever needed)
- Decodes inbound OCPP messages from the CSMS into core commands
- Manages the WebSocket connection lifecycle (connect, reconnect, TLS)
- Handles OCPP call/response correlation (message IDs, timeouts)
- Is versioned: there is one adapter implementation per supported OCPP version

**Design:**
- Adapters are selected at startup based on the configured OCPP version
- The core state machine never sees raw OCPP JSON — only typed commands and effects
- This boundary is the key isolation point: adding OCPP 2.0.1 support means writing a new adapter, not modifying the core

### Scenario engine (`chargetwin-scenario`)

Executes declarative YAML scenarios against a charge point instance.

**Responsibilities:**
- Parses and validates scenario YAML against the DSL schema
- Drives the charge point through the scenario step-by-step
- Handles timing directives (`wait`, `interval`, `duration`)
- Evaluates `assert` steps and records pass/fail results
- Emits structured scenario events (step started, step passed, step failed, scenario complete)

**Design:**
- The scenario executor is itself a state machine over the step list
- No embedded scripting runtime in v1. All conditional logic is expressed through built-in step types
- Deterministic given the same CSMS responses: random behaviour (e.g. `random_delay`) is seeded and reproducible
- Scenarios are composed: a scenario can reference another as a sub-sequence

**DSL action types (v0.x):**

| Action | Description |
|---|---|
| `connect` | Open WebSocket to CSMS |
| `disconnect` | Close WebSocket |
| `boot` | Send BootNotification, assert expected response |
| `authorize` | Send Authorize, assert expected response |
| `start_transaction` | Send StartTransaction |
| `stop_transaction` | Send StopTransaction |
| `meter_values` | Emit MeterValues at interval for a duration |
| `trigger` | Send TriggerMessage |
| `wait` | Sleep for a fixed or random duration |
| `fault` | Inject a fault (disconnect, malformed message, delay) |
| `assert` | Assert a metric or state condition |
| `repeat` | Repeat a block N times |
| `sequence` | Inline sub-sequence of steps |

### Orchestrator (`chargetwin-cli` + core orchestration module)

Manages a pool of charge point instances and coordinates their lifecycle.

**Responsibilities:**
- Spawns and destroys charge point instances (each runs as a Tokio task)
- Maintains an instance registry (ID, status, scenario, metrics)
- Assigns scenarios to instances individually or in bulk
- Aggregates metrics across the fleet
- Exposes the orchestration API (via the local server) for CLI and GUI access

**Design:**
- In v1, the orchestrator runs in the same process as the instances it manages
- Communication between the orchestrator and instances uses Tokio channels (not network calls)
- The instance registry is an in-memory concurrent map; persistence is not required in v1
- Fleet-level metrics are aggregated in-process before export to Prometheus

### Local server (`chargetwin-server`)

An Axum-based HTTP and WebSocket server that exposes the orchestrator and instance state to the CLI and GUI.

**Responsibilities:**
- REST API for instance lifecycle management (create, start, stop, delete)
- REST API for scenario management (list, upload, assign, run)
- WebSocket endpoint for real-time event streaming (per-instance and fleet-level)
- Serves the static GUI assets in local mode
- Prometheus `/metrics` endpoint

**API surface (v0.x):**

```
GET    /api/instances                    list all instances
POST   /api/instances                    create instance(s)
GET    /api/instances/:id                get instance state
DELETE /api/instances/:id                stop and remove instance
POST   /api/instances/:id/scenario       assign and start a scenario
GET    /api/scenarios                    list available scenarios
POST   /api/scenarios                    upload a scenario file
GET    /metrics                          Prometheus metrics
WS     /api/events                       fleet-level event stream
WS     /api/instances/:id/events         per-instance event stream
```

### GUI (`ui/`)

A React/TypeScript browser application that talks to the local server over HTTP and WebSocket.

**Responsibilities:**
- Display instance list with live status
- Show full OCPP message trace per instance
- Visualise connector and session state
- Allow scenario upload, assignment, and execution from the browser
- Show fleet-level metrics dashboard

**Design:**
- Stateless: all state lives in the Rust server. The GUI subscribes to WebSocket events and reflects server state
- No direct OCPP protocol knowledge in the GUI; it only consumes the typed events emitted by the server
- TanStack Query for REST calls; Zustand for local UI state; React for rendering
- Vite for development and build
- In v1, the GUI is served as static assets by the local server (no separate hosting needed)
- Future: optionally wrap with Tauri for desktop distribution without a browser requirement

---

## Data flow: single charge point session

```
User (CLI or GUI)
  │
  │  POST /api/instances  { id: "CP-001", ocpp: "1.6", csms: "ws://..." }
  ▼
Local server (Axum)
  │
  │  orchestrator.create(config)
  ▼
Orchestrator
  │
  │  spawns Tokio task
  ▼
CP instance (ChargePoint + OCPP adapter)
  │
  │  connect → ws://csms:9000/ocpp/CP-001
  ▼
CSMS

--- OCPP exchange begins ---

CSMS → [BootNotification response] → OCPP adapter
  │  decode → BootNotificationResponse { status: Accepted }
  ▼
ChargePoint.handle(BootNotificationResponse)
  │  state transition: Pending → Accepted
  │  emits: ChargePointEvent::Registered
  ▼
Orchestrator (via channel)
  │  records event, updates registry
  ▼
Local server (WebSocket push)
  │  { type: "event", instance: "CP-001", event: "Registered" }
  ▼
GUI / CLI subscriber
```

---

## Data flow: scenario execution

```
User
  │  POST /api/instances/CP-001/scenario  { scenario: "peak-load.yaml" }
  ▼
Local server → Orchestrator
  │  scenario_executor.run(scenario, cp_instance)
  ▼
Scenario executor
  │  step: boot → ChargePoint.handle(SendBootNotification)
  │  step: authorize → ...
  │  step: start_transaction → ...
  │  step: meter_values (interval 30s, duration 120s) → ...
  │  step: assert { metric: transaction.energy_wh, gte: 15000 }
  │  step: stop_transaction → ...
  ▼
Scenario result
  │  { status: passed, steps: 8, assertions: 1/1 }
  ▼
WebSocket event stream → GUI / CLI
```

---

## Crate structure

```
chargetwin/
├── crates/
│   ├── chargetwin-core/       # ChargePoint state machine (no I/O)
│   │   ├── src/state.rs       # State types
│   │   ├── src/machine.rs     # Transition logic
│   │   └── src/events.rs      # Emitted events
│   │
│   ├── chargetwin-ocpp/       # OCPP wire format and WS transport
│   │   ├── src/v16/           # OCPP 1.6 types and adapter
│   │   ├── src/v201/          # OCPP 2.0.1 types and adapter
│   │   └── src/transport.rs   # WebSocket lifecycle
│   │
│   ├── chargetwin-scenario/   # Scenario DSL and executor
│   │   ├── src/dsl.rs         # Schema and parser
│   │   └── src/executor.rs    # Step runner
│   │
│   ├── chargetwin-server/     # Axum HTTP/WS server
│   │   ├── src/api.rs         # REST routes
│   │   ├── src/events.rs      # WebSocket event broadcasting
│   │   └── src/metrics.rs     # Prometheus endpoint
│   │
│   └── chargetwin-cli/        # Binary entry point, CLI, orchestrator
│       ├── src/main.rs
│       ├── src/commands/      # CLI subcommands
│       └── src/orchestrator/  # Instance pool and registry
│
└── ui/                        # React GUI
    ├── src/
    │   ├── api/               # HTTP client + WS subscription hooks
    │   ├── components/        # UI components
    │   └── stores/            # Zustand stores
    └── vite.config.ts
```

---

## Future: splitting into services

The internal boundaries described above are designed to make service extraction low-risk when the time comes.

| Today (v1) | Future split |
|---|---|
| Core + adapter in same process as orchestrator | Core/adapter as a standalone `chargetwin-engine` service |
| Orchestrator in same binary as CLI | Orchestrator as `chargetwin-orchestrator` service with gRPC/NATS interface |
| Metrics aggregated in-process | External metrics collector (Prometheus remote write) |
| Event stream over local WebSocket | Event bus over NATS JetStream |

The key design decisions that enable this transition without a rewrite are: the typed channel interface between orchestrator and instances (maps cleanly to gRPC), the event model (maps cleanly to NATS subjects), and the REST API surface (unchanged from the outside, just moves to a different process).
