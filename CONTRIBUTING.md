# Contributing to ChargeTwin

Thank you for your interest in contributing. This document covers how to get set up, what kinds of contributions are most useful, and how the review process works.

---

## Getting set up

### Prerequisites

- Rust 1.78+ (`rustup` recommended)
- Node.js 20+ (for the GUI)
- A running CSMS for integration testing, or use the test harness (see below)

### Build

```bash
git clone https://github.com/volt-ist/chargetwin.git
cd chargetwin

# Build all crates
cargo build

# Run unit tests
cargo test

# Build the GUI
cd ui && npm install && npm run build && cd ..
```

### Run against a local CSMS

If you don't have a CSMS, you can use [SteVe](https://github.com/steve-community/steve) or [OCPP-J test server](https://github.com/lbbrhzn/ocpp) locally during development.

```bash
cargo run --bin chargetwin -- start --csms ws://localhost:9000 --id CP-DEV-01
```

---

## What to contribute

### Bug reports

Open a GitHub issue. Please include:

- ChargeTwin version or commit SHA
- OCPP version in use (`--ocpp` flag)
- CSMS name and version if relevant
- A minimal reproducer (scenario YAML or CLI command)
- What you expected vs what happened
- Logs (`--log-level debug` output if possible)

### OCPP message coverage

The [ROADMAP.md](ROADMAP.md) lists which profiles and messages are planned per milestone. PRs that implement a missing message type are very welcome. Please:

- Implement the full request/response pair
- Add unit tests for the state machine transition
- Add an integration test scenario in `scenarios/`
- Update the OCPP support table in `README.md`

### Scenario library

The `scenarios/` directory contains example scenarios. Useful additions include:

- Realistic charging patterns (overnight slow charge, DC fast charge, interrupted session)
- Edge cases (auth failure, CSMS-initiated stop, heartbeat timeout, firmware update sequence)
- Fault injection patterns (network drop mid-transaction, malformed response handling)

Scenario files should include a `description` field and a comment block explaining what the scenario tests and what a passing result looks like.

### Documentation

Good documentation is a feature. Improvements to `docs/`, `README.md`, `ARCHITECTURE.md`, or `ROADMAP.md` are welcome. If you find something confusing or out of date, a PR is better than an issue.

---

## Pull request process

1. Fork the repository and create a branch from `main`
2. Make your changes with tests
3. Run `cargo test` and `cargo clippy -- -D warnings` — both must pass
4. For GUI changes, run `npm run lint` in `ui/`
5. Open a PR with a clear description of what the change does and why
6. Link any related issues

PRs are reviewed by maintainers. Expect feedback within a few days. Small, focused PRs are merged faster than large ones.

---

## Code style

- Rust: `cargo fmt` before committing. Clippy warnings are treated as errors in CI.
- TypeScript: Prettier and ESLint are configured in `ui/`. Run `npm run lint` before committing.
- Commit messages: short imperative subject line (`Add StopTransaction handler`), body if the change needs explanation.

---

## License

By contributing, you agree that your contributions will be licensed under the [MIT License](LICENSE).
