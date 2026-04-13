# ChargeTwin Architecture

This document describes the intended architecture of **Voltist ChargeTwin**.

The platform is designed to work equally well as:

- a local simulator for development and debugging
- a GUI-backed operator tool
- a headless multi-instance runtime under orchestration

---

## Architecture Goals

- support one or many simulated charge points
- reuse the same simulation core in all modes
- separate protocol logic from product surface
- keep orchestration concerns outside the simulator core
- make features progressively deliverable by OCPP capability
- support deterministic scenarios and large-scale concurrency

---

## High-Level Architecture

```text
                 +----------------------+
                 |      Studio GUI      |
                 | React / TypeScript   |
                 +----------+-----------+
                            |
                            v
                 +----------------------+
                 |    Control Plane     |
                 | Go API / gRPC / REST |
                 +----------+-----------+
                            |
                  commands / queries / events
                            |
                            v
                 +----------------------+
                 |      Scheduler       |
                 | placement / scaling  |
                 +----------+-----------+
                            |
                            v
        +-----------------------------------------------+
        |               Runtime Workers                  |
        | Rust multi-instance simulator runtime          |
        | hosts many charge point digital twins          |
        +-------------------+---------------------------+
                            |
                            v
                 +----------------------+
                 |   OCPP CSMS / Test   |
                 | external backend(s)  |
                 +----------------------+