# ChargeTwin — Roadmap

The roadmap is organised around two axes: **OCPP profile coverage** and **platform capabilities**. Each milestone ships a complete, usable increment — not a partial implementation of everything.

Milestones are not date-bound at this stage. They are sequenced by dependency and value.

---

## v0.1 — Foundations

**Goal:** A working single charge point simulator that speaks OCPP 1.6J Core profile end-to-end. Useful to a developer who wants to test a CSMS against a basic charger today.

**OCPP 1.6J:**
- [ ] Core profile — full message set
    - BootNotification / Heartbeat
    - Authorize
    - StartTransaction / StopTransaction
    - MeterValues (basic)
    - StatusNotification
    - ChangeAvailability, ChangeConfiguration, GetConfiguration
    - Reset, UnlockConnector

**Platform:**
- [ ] Single charge point CLI (`chargetwin start --csms ... --id ...`)
- [ ] WebSocket transport (OCPP-J)
- [ ] Typed OCPP 1.6 message schema (Serde, validated on decode)
- [ ] Structured JSON logging (`tracing`)
- [ ] Minimal configuration via YAML file or CLI flags
- [ ] GitHub Actions CI (build + test)

**Not in scope:** GUI, orchestration, scenarios, fleet mode.

---

## v0.2 — Local dev experience

**Goal:** A developer can open a browser, see their simulated charge point's live state, send commands, and inspect every OCPP message. The local GUI is production-quality from day one.

**OCPP 1.6J:**
- [ ] Firmware Management profile
    - UpdateFirmware, FirmwareStatusNotification
- [ ] Remote Trigger profile
    - TriggerMessage (all triggerable messages)
- [ ] Local Auth List Management profile
    - SendLocalList, GetLocalListVersion

**Platform:**
- [ ] Local HTTP/WebSocket server (Axum)
- [ ] REST API for instance management
- [ ] WebSocket event stream (per-instance)
- [ ] React/TypeScript GUI (Vite)
    - Instance status panel (availability, connector, session)
    - Full OCPP message trace (sent/received, timestamps, raw JSON)
    - Basic command panel (Authorize, Reset, UnlockConnector)
- [ ] GUI served as static assets by the local binary
- [ ] Prometheus `/metrics` endpoint (per-instance counters)

---

## v0.3 — Scenarios + Smart Charging

**Goal:** Scenarios can be scripted in YAML and run headlessly. Smart Charging makes the simulator useful for CSMS teams building load management features.

**OCPP 1.6J:**
- [ ] Smart Charging profile
    - SetChargingProfile, ClearChargingProfile, GetCompositeSchedule
    - Charging schedule enforcement in the simulated session

**Platform:**
- [ ] Scenario DSL (YAML) — first version
    - Actions: `connect`, `boot`, `authorize`, `start_transaction`, `stop_transaction`, `meter_values`, `wait`, `disconnect`
    - Assertions: `assert` on metrics and state
    - Fault injection: `fault` (disconnect, delay, malformed message)
- [ ] Scenario executor integrated with charge point core
- [ ] Scenario result output: structured JSON (pass/fail, step-by-step)
- [ ] `chargetwin run --scenario ./scenarios/basic-session.yaml` command
- [ ] Bundled example scenarios

---

## v0.4 — Fleet mode

**Goal:** Hundreds of charge point instances can be spawned from a single command and driven by scenarios. The orchestrator is operational.

**Platform:**
- [ ] Orchestrator (in-process, same binary)
    - Instance pool (Tokio tasks, concurrency-safe registry)
    - Bulk instance create/start/stop
    - Per-instance scenario assignment
    - Fleet-level metrics aggregation
- [ ] Fleet CLI (`chargetwin orchestrate --count N --scenario ...`)
- [ ] WebSocket fleet event stream (`/api/events`)
- [ ] GUI fleet dashboard
    - Instance list with live status
    - Fleet-level metrics (active sessions, messages/sec, error rate)
    - Scenario run status per instance
- [ ] Docker image
- [ ] Docker Compose stack (chargetwin + Prometheus + Grafana)
- [ ] Bundled Grafana dashboards

---

## v0.5 — OCPP 2.0.1 foundation

**Goal:** The same developer and fleet experience is available for OCPP 2.0.1, starting with the most operationally important profiles.

**OCPP 2.0.1:**
- [ ] Core / Provisioning
- [ ] Authorization
- [ ] Transactions
- [ ] Firmware Management
- [ ] Remote Trigger
- [ ] Device Model (partial)
- [ ] Smart Charging (partial)

**Platform:**
- [ ] `--ocpp 2.0.1` flag wires up the 2.0.1 OCPP adapter
- [ ] Scenario DSL extended for 2.0.1 action types
- [ ] OCPP version shown in GUI instance panel

---

## v0.6 — Observability + security

**Goal:** ChargeTwin is production-ready for continuous integration environments. Security extensions are implemented. Traces make it easy to debug CSMS behaviour against the simulator.

**OCPP 1.6J + 2.0.1:**
- [ ] Security extensions (1.6 Security Whitepaper)
    - TLS for WebSocket connections
    - Certificate-based authentication
    - SecurityEventNotification

**Platform:**
- [ ] OpenTelemetry trace export (spans per OCPP call)
- [ ] Helm chart for Kubernetes deployment (fleet mode)
- [ ] Scenario `repeat`, `sequence`, and sub-scenario composition
- [ ] `chargetwin events --follow` formatted CLI output
- [ ] Scenario library expanded (offline recovery, auth failure loop, firmware update cycle)

---

## v1.0 — Production milestone

**Goal:** ChargeTwin is feature-complete for OCPP 1.6J and 2.0.1, robustly tested, and suitable as a primary testing tool for CSMS teams.

**OCPP 2.0.1:**
- [ ] Full profile coverage (Reservation, Display, DataTransfer, Local Auth List)
- [ ] ISO 15118 / Plug & Charge (basic)

**OCPP 2.1:**
- [ ] Core / Provisioning
- [ ] Transactions
- [ ] V2G / bidirectional charging (initial)

**Platform:**
- [ ] OCPP conformance test runner (OCTT-compatible assertions)
- [ ] Plugin interface for custom charge point behaviours (Rust trait, dynamically loaded)
- [ ] Optional Tauri packaging for desktop distribution
- [ ] Stable REST API (v1 contract, changelog-tracked)
- [ ] Comprehensive documentation site

---

## Beyond v1.0

Ideas tracked but not scheduled:

- **Cloud-hosted fleet runner** — run fleet scenarios against a CSMS without local infrastructure
- **SaaS conformance testing** — submit a CSMS endpoint, receive a full OCPP conformance report
- **OCPP message fuzzer** — systematic mutation of message fields for robustness testing
- **CSMS stub** — a minimal built-in CSMS for fully self-contained local testing
- **gRPC / NATS orchestration interface** — extract the orchestrator into a separate service for large-scale lab deployments

---

## Principles that don't change

- Ship complete profiles, not partial message sets
- The scenario DSL stays declarative and YAML-based in v1.x; no general scripting runtime
- The single-binary model is maintained until there is a concrete operational reason to split
- The internal component boundaries are maintained even before the split, so the eventual extraction is a refactor, not a rewrite
