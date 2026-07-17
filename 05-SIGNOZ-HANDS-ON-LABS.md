# SigNoz Hands-On Labs for OSAI

Complete these labs in order. Keep the SigNoz time range at **Last 30 minutes** and refresh after generating telemetry.

## Lab 0 — Verify the platform

```bash
sudo ./START-HERE.sh status
sudo ./START-HERE.sh evidence
```

Goal: distinguish process/container health from actual telemetry health.

## Lab 1 — Find one real Rust trace

```bash
sudo ./START-HERE.sh demo
```

In Traces Explorer:

```text
service.name = 'osai-agent'
```

Open `osai.api.scan_now` or `osai.scan.execute`.

Answer:

- Which span is the root?
- Which child took longest?
- What resource attributes identify the service?
- Is the status OK or ERROR?

## Lab 2 — Understand deterministic demo telemetry

In Traces Explorer:

```text
service.name = 'osai-observability-harness'
```

Open `osai.hackathon.demo.round`, then the expected failed `GET /api/snapshot` span.

Filter:

```text
osai.demo.expected_error = true
```

Use **Related Signals → Logs** and confirm the trace and log share IDs.

## Lab 3 — Query the custom demo metrics

Metrics Explorer:

```text
Metric: osai.demo.request.duration
Group by: url.path
Filter: service.name = 'osai-observability-harness'
```

Then:

```text
Metric: osai.demo.http.error
Aggregation: Sum
Group by: url.path
```

Explain why one unauthenticated request is intentionally an error.

## Lab 4 — Query host metrics

Search the Metrics dropdown for:

```text
system.cpu
system.memory
system.disk
system.filesystem
system.network
process
```

Do not type these prefixes into the filter bar. Select a complete metric name.

Build:

- CPU utilization or CPU-time rate;
- memory utilization;
- filesystem utilization grouped by mountpoint;
- network I/O rate grouped by interface and direction;
- process memory grouped by executable name when available.

## Lab 5 — Analyze logs

Logs Explorer:

```text
service.name = 'osai-agent' AND severity_text IN ('WARN', 'ERROR')
```

Then Docker logs:

```text
service.name = 'osai-docker-stack' AND body ILIKE '%cancel task%'
```

Open one log and identify:

- Resource fields;
- log fields;
- attributes;
- trace context, if present.

## Lab 6 — Controlled Cognee incident

```bash
sudo /opt/osai/OS.rs/scripts/inject-cognee-failure.sh
```

Observe:

- OSAI continues running;
- Cognee-dependent operations fail or queue;
- logs show the dependency failure;
- storage and host telemetry remain available.

Restore:

```bash
sudo /opt/osai/OS.rs/scripts/restore-cognee.sh
```

## Lab 7 — Build OSAI Mission Control

Create panels for:

1. OSAI trace count.
2. OSAI P95 trace duration.
3. Demo expected-error sum.
4. Host memory utilization.
5. Filesystem utilization.
6. Cognee WARN/ERROR log count.
7. llama.cpp cancellation log count.

For every panel, document:

```text
signal
metric or aggregation
filter
group-by
unit
why an operator needs it
```

## Lab 8 — Create an alert

Create a training alert:

```text
Metric: osai.demo.http.error
Condition: Sum > 0
Filter: osai.demo.expected_error = true
Window: 5 minutes
```

Then run the demo again and observe the alert lifecycle.

## Lab 9 — Professional incident method

Use this order for every problem:

```text
1. State the user-visible symptom.
2. Choose the affected service.
3. Check RED: rate, errors, duration.
4. Open a representative trace.
5. Identify the slow/failed span.
6. Open related logs.
7. Compare host/container metrics for the same time.
8. Form a hypothesis supported by evidence.
9. Apply a safe change.
10. Verify before/after telemetry.
```
