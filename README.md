# ChargeTwin

**ChargeTwin** is a [Volt.ist](https://github.com/volt-ist) charge point digital twin and simulator platform for EV charging systems.

It is built for two primary use cases:

- **Developer / QA mode** — run one or a few simulated charge points locally with a CLI or browser-based GUI
- **Fleet / lab mode** — run many charge point instances concurrently under an orchestrator

ChargeTwin is designed to support **progressive OCPP capability delivery**, starting with **OCPP 1.6J** and expanding through operationally valuable profiles and features toward OCPP 2.0.1 and 2.1.

---

## Why ChargeTwin

Most charge point simulators stop at basic protocol replay. ChargeTwin goes further:

- Realistic charge point state modeling (full OCPP state machine)
- Repeatable, declarative scenario execution via YAML
- Large-scale multi-instance simulation from a single binary
- Headless and GUI-driven operation from the same engine
- Orchestration support for automated labs and CI pipelines
- A clean internal boundary between simulation core and transport, making the path to split services straightforward when the time comes

---

## Modes

### Local interactive mode

Used by developers, integrators, and QA teams.

```
chargetwin start --csms ws://localhost:9000 --id CP-001
chargetwin gui        # opens browser UI at http://localhost:7000
```

Typical usage:

- Start a simulated charge point and connect to a CSMS
- Inspect full OCPP message traces in the browser GUI
- Inject faults and delays mid-session
- Test edge cases against specific OCPP profiles
- Watch connector and session state update in real time

### Headless orchestrated mode

Used for scale testing, regression testing, and automation.

```
chargetwin orchestrate --count 500 --csms ws://staging.csms.io --scenario ./scenarios/peak-load.yaml
chargetwin events --follow
```

Typical usage:

- Spawn hundreds of charge point instances from a single process
- Assign YAML scenarios per-instance or across the fleet
- Run under scheduler or CI control with structured output
- Stream telemetry and OCPP traces
- Collect pass/fail assertions from scenario execution

---

## Tech Stack

| Layer                    | Technology                        | Notes                                              |
|--------------------------|-----------------------------------|----------------------------------------------------|
| **Simulation engine**    | Rust, Tokio                       | Async, low memory, many concurrent WS connections  |
| **OCPP state machine**   | Custom (Rust)                     | Typed, per-version, per-profile                    |
| **WebSocket transport**  | `tokio-tungstenite`               | Client-side WS to CSMS                             |
| **Local HTTP/WS server** | Axum                              | GUI and CLI talk to this                           |
| **Orchestrator**         | Rust (same binary)                | Worker thread pool, instance registry              |
| **Scenario engine**      | Rust + YAML DSL                   | Declarative, deterministic, no embedded scripting  |
| **GUI**                  | React + TypeScript                | Vite, TanStack Query, Zustand                      |
| **Metrics**              | Prometheus + OpenTelemetry        | Exposed at `/metrics`                              |
| **Logs**                 | Structured JSON (`tracing`)       | Compatible with Grafana Loki                       |
| **Config**               | YAML / environment variables      | Single config file per run                         |
| **Packaging**            | Single binary (Rust) + static GUI | Optional Tauri wrap for desktop distribution later |
| **Container**            | Docker, Helm chart                | For fleet/lab deployment                           |
| **CI**                   | GitHub Actions                    | Build, test, OCPP conformance checks               |

> The simulation engine and orchestrator ship as a **single self-contained binary** in v1. The internal boundary between components is clean enough to extract into separate services in a future release without redesigning the protocol between them.

---

## OCPP Support

ChargeTwin targets full OCPP 1.6J profile coverage first, then expands to 2.0.1 and 2.1.

| Profile / Feature          | OCPP 1.6J | OCPP 2.0.1 | OCPP 2.1   |
|----------------------------|-----------|------------|------------|
| Core                       | ✅ v0.1    | 🔵 v0.4    | 🟡 v0.6    |
| Firmware Management        | ✅ v0.1    | 🔵 v0.4    | 🟡 v0.6    |
| Local Auth List Management | ✅ v0.2    | 🔵 v0.4    | 🟡 v0.6    |
| Reservation                | ✅ v0.2    | 🔵 v0.5    | 🟡 planned |
| Smart Charging             | ✅ v0.3    | 🔵 v0.5    | 🟡 planned |
| Remote Trigger             | ✅ v0.2    | 🔵 v0.4    | 🟡 planned |
| Security extensions        | 🔵 v0.3   | 🔵 v0.5    | 🟡 planned |
| ISO 15118 / Plug & Charge  | —         | 🟡 v1.x    | 🟡 v1.x    |
| V2G / bidirectional        | —         | —          | 🟡 v1.x    |

✅ Stable · 🔵 Planned · 🟡 Future

---

## Scenario DSL (YAML)

Scenarios are declarative YAML files that describe a sequence of charge point behaviors. There is no embedded scripting runtime — logic is expressed through built-in action types, conditionals, and timing directives.

```yaml
# scenarios/basic-session.yaml
name: basic-session
version: "1"
charger:
  ocpp_version: "1.6"
  vendor: "Volt.ist"
  model: "SimCP-1"

steps:
  - action: connect
    csms: "ws://localhost:9000"

  - action: boot
    expect_response: RegistrationStatus.Accepted

  - action: authorize
    id_tag: "DEADBEEF"
    expect_response: AuthorizationStatus.Accepted

  - action: start_transaction
    connector_id: 1
    id_tag: "DEADBEEF"

  - action: meter_values
    interval_seconds: 30
    duration_seconds: 120
    energy_kwh: 15.4

  - action: stop_transaction
    reason: Local

  - action: assert
    metric: transaction.energy_wh
    gte: 15000

  - action: disconnect
```

See [`docs/scenario-dsl.md`](docs/scenario-dsl.md) for the full action reference.

---

## Quickstart

### Prerequisites

- Rust 1.78+ (`rustup` recommended)
- Node.js 20+ (for the GUI)
- A running CSMS, or use a local one for testing

### Build from source

```bash
git clone https://github.com/volt-ist/chargetwin.git
cd chargetwin

# Build the binary
cargo build --release

# Build the GUI
cd ui && npm install && npm run build && cd ..

# Run a single simulated charge point
./target/release/chargetwin start \
  --csms ws://localhost:9000 \
  --id CP-001 \
  --ocpp 1.6

# Open the browser GUI
./target/release/chargetwin gui
# → http://localhost:7000
```

### Docker

```bash
docker pull ghcr.io/volt-ist/chargetwin:latest

docker run --rm ghcr.io/volt-ist/chargetwin \
  --csms ws://host.docker.internal:9000 \
  --id CP-DOCKER-01
```

### Fleet mode

```bash
chargetwin orchestrate \
  --count 500 \
  --csms ws://staging.csms.io \
  --scenario ./scenarios/peak-load.yaml \
  --metrics-port 9090
```

---

## Project Structure

```
chargetwin/
├── crates/
│   ├── chargetwin-core/     # Simulation engine, OCPP state machines
│   ├── chargetwin-ocpp/     # OCPP message types, parsers (per version)
│   ├── chargetwin-scenario/ # Scenario DSL parser and executor
│   ├── chargetwin-server/   # Local HTTP/WebSocket server (Axum)
│   └── chargetwin-cli/      # CLI entry point and orchestrator
├── ui/                      # React GUI (Vite + TypeScript)
├── scenarios/               # Bundled example scenarios
├── docs/                    # Extended documentation
├── docker/                  # Dockerfile and Compose stack
└── .github/workflows/       # CI pipelines
```

---

## Contributing

ChargeTwin is open source and welcomes contributions.

- **Bug reports** — open a GitHub issue with your OCPP version, CSMS details, and a minimal reproducer
- **OCPP message coverage** — PRs implementing missing messages with matching tests are always welcome
- **Scenario library** — share useful simulation scenarios (peak load, offline recovery, auth failure loops, etc.)
- **Documentation** — good docs are a feature; help improve guides, examples, and the OCPP matrix

Please read [`CONTRIBUTING.md`](CONTRIBUTING.md) before opening a PR.

---

## Roadmap

See [`ROADMAP.md`](ROADMAP.md) for the full milestone plan.

---

## License

MIT — see [`LICENSE`](LICENSE).

Built by [Volt.ist](https://github.com/volt-ist).
