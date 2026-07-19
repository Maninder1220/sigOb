# Complete OSAI v2.2.4 Observability Audit

## Scope reviewed

The review covered the complete package, including:

- root lifecycle scripts and documentation;
- `osai-agent` Rust API, scanner, Ask engine, workers, supervisor, UI, Compose files, migrations, and knowledge files;
- HostScope workspace, OTLP metrics exporter, systemd deployment, MCP, diagnostics, collectors, redaction, and schemas;
- OpenTelemetry Collector receivers, processors, exporters, persistent offsets, host mounts, Docker socket, and Compose settings;
- SigNoz install/repair/verification scripts, Foundry deployment, MCP server, and systemd bridges;
- connected, air-gap, CI, Terraform/OpenTofu, cleanup, demo, and validation paths.

## Main finding

The transport was healthy, but the model was difficult for a new operator to prove because:

1. OSAI emitted traces but no application metrics.
2. Application logs reached SigNoz through journald/filelog instead of the same OpenTelemetry context as traces.
3. The UI did not expose the Trace ID for an Ask response.
4. Trace View shows root traces, while important Ask operations are child spans; stale demo filters hid real application spans.
5. The generic HTTP middleware root and the `osai.api.ask` span were not explained as parent/child.
6. Cognee recall failure and Qwen success could occur in the same request, but the user had no single correlated view.
7. Collector self-health metrics were not exported to SigNoz.

## Standardized correction

v2.2.5 uses this contract:

```text
Browser-generated request ID
        │
        ▼
osai-agent Rust
  ├─ OTLP traces
  ├─ OTLP correlated logs
  └─ OTLP application metrics
        │
        ▼
127.0.0.1:14318 OSAI Collector
  ├─ app signals
  ├─ HostScope metrics/context
  ├─ hostmetrics
  ├─ Docker metrics/logs
  └─ Collector self-health
        │
        ▼
SigNoz OTLP → ClickHouse → Explorer / dashboards / alerts / MCP
```

## OSAI-agent role

OSAI-agent owns request and business-operation observability. It knows:

- when `/api/ask` started and ended;
- whether AI was requested and actually used;
- planning and evidence-building durations;
- PostgreSQL retrieval status;
- Cognee recall status;
- Qwen model, token counts, finish reason, and latency;
- whether the final response was AI, fallback, or error.

## HostScope role

HostScope owns host state and health context. It knows:

- CPU topology and load;
- memory and paging;
- filesystem and network counters;
- process counts;
- OS/security/runtime context;
- collection issues;
- bounded recent diagnostic scans.

HostScope cannot know the semantic stages inside one Ask OSAI request. OSAI-agent cannot replace HostScope's broad host-state collection.

## Cognee finding from the observed request

The supplied Cognee logs show recall was routed, but returned `DatasetNotFoundError: No datasets found` and HTTP 404. This is a memory prerequisite problem. It does not mean Qwen failed; OSAI can continue with current Rust/PostgreSQL evidence and Qwen.

## Files changed

```text
osai-agent/Cargo.toml
osai-agent/src/telemetry.rs
osai-agent/src/main.rs
osai-agent/src/ask.rs
osai-agent/web/app.js
observability/collector/config.yaml
observability/scripts/install-signoz.sh
observability/scripts/apply-telemetry-update.sh
observability/scripts/verify.sh
scripts/install-airgap.sh
scripts/apply-standardized-observability.sh
START-HERE.sh
README.md
ARCHITECTURE.md
docs/10-OBSERVABILITY-FROM-ZERO.md
docs/11-ASK-OSAI-ONE-REQUEST-JOURNEY.md
docs/12-SIGNAL-OWNERSHIP-STANDARD.md
```
