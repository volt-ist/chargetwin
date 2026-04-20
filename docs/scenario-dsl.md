# Scenario DSL Reference

ChargeTwin scenarios are declarative YAML files that describe a sequence of charge point behaviours. There is no embedded scripting runtime — all logic is expressed through built-in action types, conditionals, and timing directives.

> **Status:** This document describes the v0.3 DSL. Actions marked 🔵 are planned for a later milestone.

---

## File structure

```yaml
name: string           # required — unique scenario name
version: "1"           # required — DSL schema version
description: string    # optional

charger:
  ocpp_version: "1.6"  # required — "1.6" | "2.0.1" | "2.1"
  vendor: string        # optional — BootNotification.chargePointVendor
  model: string         # optional — BootNotification.chargePointModel
  serial: string        # optional — BootNotification.chargePointSerialNumber

steps:
  - action: ...         # one or more steps
```

---

## Actions

### `connect`

Opens a WebSocket connection to the CSMS.

```yaml
- action: connect
  csms: "ws://localhost:9000"     # required
  path_suffix: "/CP-001"          # optional — appended to csms URL
  tls: false                      # optional — default false
```

---

### `disconnect`

Closes the WebSocket connection.

```yaml
- action: disconnect
```

---

### `boot`

Sends a BootNotification and asserts the expected response.

```yaml
- action: boot
  expect_response: RegistrationStatus.Accepted   # optional — default Accepted
  retry_interval_seconds: 30                      # optional — if Pending, retry after N seconds
```

---

### `heartbeat`

Sends a Heartbeat message.

```yaml
- action: heartbeat
```

---

### `authorize`

Sends an Authorize request.

```yaml
- action: authorize
  id_tag: "DEADBEEF"                            # required
  expect_response: AuthorizationStatus.Accepted  # optional — default Accepted
```

---

### `start_transaction`

Sends a StartTransaction request.

```yaml
- action: start_transaction
  connector_id: 1            # required
  id_tag: "DEADBEEF"         # required
  meter_start: 0             # optional — Wh, default 0
```

---

### `stop_transaction`

Sends a StopTransaction request.

```yaml
- action: stop_transaction
  reason: Local              # optional — Local | Remote | EVDisconnected | HardReset | PowerLoss | Reboot | Other
  id_tag: "DEADBEEF"         # optional
```

---

### `meter_values`

Emits MeterValues messages at a regular interval for a given duration, simulating energy delivery.

```yaml
- action: meter_values
  connector_id: 1            # optional — default 1
  interval_seconds: 30       # required — how often to send MeterValues
  duration_seconds: 300      # required — total duration
  energy_kwh: 15.4           # required — total energy to deliver over the duration
  voltage: 230               # optional — V, default 230
  current: 16                # optional — A, default derived from power
```

---

### `status_notification`

Sends a StatusNotification.

```yaml
- action: status_notification
  connector_id: 1
  status: Charging           # Available | Preparing | Charging | SuspendedEV | SuspendedEVSE | Finishing | Reserved | Unavailable | Faulted
  error_code: NoError        # optional — default NoError
```

---

### `wait`

Pauses scenario execution.

```yaml
- action: wait
  seconds: 10                # fixed wait

# or with a random range
- action: wait
  seconds_min: 5
  seconds_max: 30
  seed: 42                   # optional — for deterministic random
```

---

### `fault`

Injects a fault condition into the simulated charge point or connection.

```yaml
# Disconnect the WebSocket abruptly
- action: fault
  type: disconnect

# Drop the next N outbound messages
- action: fault
  type: drop_messages
  count: 3

# Add latency to the next N outbound messages
- action: fault
  type: delay_messages
  count: 5
  delay_ms: 2000

# Send a malformed OCPP message
- action: fault
  type: malformed_message
```

---

### `assert`

Asserts a condition on a metric or state value. Fails the scenario if the condition is not met.

```yaml
- action: assert
  metric: transaction.energy_wh
  gte: 15000

- action: assert
  metric: session.duration_seconds
  lte: 400

- action: assert
  state: connector.status
  eq: Finishing

- action: assert
  metric: messages.sent.count
  gt: 0
```

**Available metrics:**

| Metric | Description |
|---|---|
| `transaction.energy_wh` | Total energy delivered in current transaction (Wh) |
| `transaction.duration_seconds` | Duration of current transaction |
| `session.duration_seconds` | Duration from connect to current step |
| `messages.sent.count` | Total OCPP messages sent |
| `messages.received.count` | Total OCPP messages received |
| `messages.errors.count` | Total OCPP CallError messages received |

**Available states:**

| State | Values |
|---|---|
| `connector.status` | `Available`, `Preparing`, `Charging`, `Finishing`, `Unavailable`, `Faulted` |
| `charger.registration_status` | `Accepted`, `Pending`, `Rejected` |
| `charger.connected` | `true`, `false` |

---

### `repeat` 🔵

Repeats a block of steps N times. Planned for v0.6.

```yaml
- action: repeat
  count: 10
  steps:
    - action: start_transaction
      connector_id: 1
      id_tag: "DEADBEEF"
    - action: meter_values
      interval_seconds: 30
      duration_seconds: 120
      energy_kwh: 5.0
    - action: stop_transaction
    - action: wait
      seconds: 60
```

---

### `sequence` 🔵

Runs another scenario file as a sub-sequence. Planned for v0.6.

```yaml
- action: sequence
  file: ./scenarios/basic-auth.yaml
```

---

## Full example

```yaml
name: standard-session
version: "1"
description: >
  A standard charge session with authorization, metering,
  and a clean disconnect. Asserts energy delivery.

charger:
  ocpp_version: "1.6"
  vendor: "Voltist"
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
    meter_start: 0

  - action: meter_values
    interval_seconds: 30
    duration_seconds: 300
    energy_kwh: 15.4

  - action: assert
    metric: transaction.energy_wh
    gte: 15000

  - action: stop_transaction
    reason: Local

  - action: wait
    seconds: 2

  - action: disconnect
```
