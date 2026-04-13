# ChargeTwin

**ChargeTwin** is a **Voltist** charge point digital twin and simulator platform for EV charging systems.

It is built for two primary use cases:

- **Developer / QA mode** — run one or a few simulated charge points locally with CLI or GUI
- **Fleet / lab mode** — run many charge point instances concurrently under an orchestrator

ChargeTwin is designed to support **progressive OCPP capability delivery**, starting with **OCPP 1.6J**, then expanding through operationally valuable profiles and features.

## Why ChargeTwin

Most charge point simulators stop at basic protocol replay. ChargeTwin is intended to go further:

- realistic charge point state modeling
- repeatable scenario execution
- large-scale multi-instance simulation
- headless and GUI-driven operation
- orchestration support for automated labs and CI
- a clean path from protocol basics to richer feature coverage

## Initial Scope

The first major protocol target is **OCPP 1.6J** with phased support for:

- Core
- Firmware Management
- Local Auth List Management
- Reservation
- Smart Charging
- Remote Trigger

This gives the platform a strong practical testing surface before deeper security and ISO 15118-oriented work.

## Product Modes

### Local interactive mode
Used by developers, integrators, and QA teams.

Typical usage:
- start a simulated charge point
- connect to a CSMS
- inspect messages
- inject faults
- test edge cases
- watch connector/session state in the GUI

### Headless orchestrated mode
Used for scale testing, regression testing, and automation.

Typical usage:
- create hundreds of charger instances
- assign templates and scenarios
- run under scheduler/orchestrator control
- stream telemetry and protocol traces
- collect artifacts and assertions

## Core Principles

- **Headless first** — GUI is a client, not the engine
- **Single engine, many surfaces** — same simulation core powers CLI, worker, and GUI
- **Multi-instance efficiency** — many simulators per worker process
- **Deterministic scenarios** — reproducible behavior when needed
- **Profile-led delivery** — clear staged rollout by OCPP capability

## Planned Stack

### Simulation engine
- **Rust**
- Tokio
- Serde
- Tracing
- WebSocket transport layer

### Control plane
- **Go**
- REST and/or gRPC
- NATS integration
- PostgreSQL

### Studio / GUI
- **React + TypeScript**
- Vite
- TanStack Query
- Zustand
- optional Tauri packaging

### Infrastructure
- NATS / JetStream
- PostgreSQL
- Docker
- Kubernetes
- Prometheus
- Grafana
- Loki