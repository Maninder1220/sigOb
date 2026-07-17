# OSAI Flight Recorder

OSAI Flight Recorder is a self-hosted operations and observability platform built around a Rust AI operations agent. It combines OSAI, PostgreSQL, RustFS, Cognee, llama.cpp/Qwen, HostScope, OpenTelemetry Collector, SigNoz, ClickHouse, and the SigNoz MCP server on one Linux machine.

> **Current runtime model:** OSAI's stateful dependencies run in Docker, while the Rust `osai-all` supervisor and HostScope normally run as native systemd services. This is intentional: native processes can inspect the real host, while containers keep databases, memory services, inference, and observability isolated.

## 1. What your running containers mean

Your current container list represents three groups:

| Group | Containers | Responsibility |
|---|---|---|
| OSAI dependencies | `osai-postgres`, `osai-rustfs`, `osai-cognee`, `osai-llama` | OSAI history, object evidence, long-term memory, and local Qwen inference |
| Telemetry transport | `osai-signoz-collection-agent`, `signoz-ingester-1` | Receive, normalize, batch, and forward OTLP traces, metrics, and logs |
| SigNoz backend | `signoz-signoz-0`, ClickHouse, Keeper, metastore PostgreSQL, `signoz-mcp` | Store telemetry, query it, visualize it, alert on it, and expose MCP analysis |

The main Rust application is normally **not** visible in `docker ps`. Verify it with:

```bash
sudo systemctl status osai-agent.service --no-pager
sudo systemctl status hostscope.service --no-pager
ps -ef | grep -E 'osai-(all|agent|storage-worker|cognee-ingest)|hostscope' | grep -v grep
```

## 2. Signal ownership: controlled, not duplicated

Use one authoritative producer for each type of information.

| Signal or data class | Authoritative producer | Transport into SigNoz | SigNoz identity |
|---|---|---|---|
| OSAI request and dependency traces | Rust `tracing` + OpenTelemetry bridge | OTLP/HTTP to `127.0.0.1:14318` | `service.name=osai-agent`, `osai-supervisor`, workers |
| OSAI application logs | Rust structured `tracing` events in journald | journald bridge → Collector `filelog` | `service.name=osai-agent` |
| Container stdout/stderr logs | Docker JSON log files | Collector `filelog/docker` | `service.name=osai-docker-stack` plus log file/container ID context |
| Standard host time-series | Collector `hostmetrics` receiver | Internal Collector pipeline | `service.name=osai-host` |
| Container CPU, memory, network, block I/O | Collector `docker_stats` receiver | Internal Collector pipeline | `service.name=osai-docker-stack` |
| Rich host context and collection quality | Native HostScope | OTLP metrics + change-only summary events | `service.name=hostscope-agent` and `hostscope-context` |
| Complete HostScope snapshot | Native HostScope JSON/MCP | Local file and read-only MCP; not continuously copied into telemetry | `/var/lib/hostscope/latest.json` |
| Deterministic demo correlation | Demo telemetry script | OTLP/HTTP to `127.0.0.1:14318` | `service.name=osai-demo` |

This prevents the most common mistake: exporting the same Rust logs both directly through OTLP and again from journald. OSAI exports **traces directly**, while logs are harvested from journald.

## 3. End-to-end telemetry architecture

```mermaid
flowchart LR
    Browser[Browser / API client] --> OSAI[OSAI Rust systemd service]
    OSAI --> Postgres[(OSAI PostgreSQL)]
    OSAI --> RustFS[(RustFS)]
    OSAI --> Cognee[Cognee container]
    Cognee --> Llama[llama.cpp + Qwen]

    OSAI -- OTLP traces --> Agent[OSAI OTel collection agent :14318]
    OSAI -- journald --> JournalFile[/var/log/osai/otel/osai-agent.json]
    JournalFile --> Agent
    DockerLogs[Docker JSON logs] --> Agent
    DockerAPI[Docker socket stats] --> Agent
    HostFS[/proc /sys host filesystem] --> Agent
    HostScope[Native HostScope] -- OTLP metrics --> Agent
    HostScope -- changed context event --> Agent

    Agent -- OTLP/gRPC --> Ingest[SigNoz ingester :4317]
    Ingest --> ClickHouse[(ClickHouse)]
    ClickHouse --> SigNoz[SigNoz UI :8080]
    SigNoz --> MCP[SigNoz MCP :18000]
```

### Why port `14318` exists

SigNoz already owns host ports `4317` and `4318`. The OSAI collection agent therefore exposes a separate loopback-only OTLP/HTTP receiver on `127.0.0.1:14318`, then forwards data to SigNoz on `127.0.0.1:4317`.

- Native OSAI/HostScope: `http://127.0.0.1:14318`
- Containerized application using host networking: `http://127.0.0.1:14318`
- Containerized application using bridge networking: add `host.docker.internal:host-gateway`, then use `http://host.docker.internal:14318`
- Direct SigNoz ingestion, bypassing the OSAI agent: `http://<host>:4318` for OTLP/HTTP or `<host>:4317` for OTLP/gRPC

Do not use `127.0.0.1:4318` from an ordinary bridge-network container unless SigNoz is running in that same container. Inside a container, `127.0.0.1` means the container itself.

## 4. Instrumentation already present in OSAI

The Rust workspace already contains:

- OpenTelemetry SDK and OTLP/HTTP trace exporter.
- `tracing-opentelemetry` bridge.
- Named spans for scans, API actions, PostgreSQL, RustFS, Cognee, and Qwen calls.
- Resource attributes for service name, namespace, version, environment, and telemetry distribution.
- Provider shutdown through `TelemetryGuard` so buffered spans are flushed.

Important files:

```text
osai-agent/src/telemetry.rs        OTLP trace provider and tracing subscriber
osai-agent/src/main.rs             API spans
osai-agent/src/collector/scanner.rs scan spans
osai-agent/src/ask.rs              Cognee, PostgreSQL, and gen_ai spans
osai-agent/src/bin/*.rs            supervisor and worker spans
observability/collector/config.yaml signal routing and enrichment
```

The Rust code uses OpenTelemetry Rust `0.32`; do not paste a `0.31` dependency block over the existing compatible dependency set.

## 5. Start or apply the observability configuration

For a complete installation:

```bash
sudo ./START-HERE.sh up onprem
```

For an already running installation after replacing the project files, apply only the telemetry layer:

```bash
cd /opt/osai/OS.rs
sudo ./START-HERE.sh telemetry-apply
```

This installs the HostScope context timer, validates and recreates only `osai-signoz-collection-agent`, and restarts the native exporters. It does **not** recreate SigNoz, ClickHouse, PostgreSQL, RustFS, Cognee, llama.cpp, or their Docker volumes. Force a HostScope rebuild only when its Rust source changed:

```bash
sudo REBUILD_HOSTSCOPE=1 ./START-HERE.sh telemetry-apply
```

## 6. HostScope: use it in three controlled surfaces

HostScope is not a universal telemetry agent. It has three bounded outputs:

1. **OTLP metrics** for time-series dashboards and alerts.
2. **Versioned JSON snapshot** for durable host context and debugging.
3. **Read-only MCP** for an AI client that needs current host facts.

### Run one diagnostic snapshot

```bash
sudo /usr/local/bin/hostscope \
  --profile safe \
  --redaction strict \
  snapshot --pretty
```

### Check collection gaps

```bash
sudo /usr/local/bin/hostscope \
  --profile safe \
  --redaction strict \
  doctor
```

### Send HostScope metrics continuously

The installed systemd service uses:

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:14318
HOSTSCOPE_INTERVAL_SECONDS=15
OTEL_METRIC_EXPORT_INTERVAL=15000
```

Useful production tuning:

```bash
sudoedit /etc/hostscope/hostscope.env
```

```dotenv
# 15 seconds is suitable for a demo. 30-60 seconds is quieter in production.
HOSTSCOPE_INTERVAL_SECONDS=30
OTEL_METRIC_EXPORT_INTERVAL=30000
OTEL_EXPORTER_OTLP_TIMEOUT=10000
RUST_LOG=hostscope=info,hostscope_otel=info
```

Then:

```bash
sudo systemctl restart hostscope.service
```

### Change-only HostScope context events

`hostscope-snapshot-export.timer` periodically captures a strict, safe snapshot. The full latest snapshot stays local at:

```text
/var/lib/hostscope/latest.json
```

Only a small summary event is appended to SigNoz when the snapshot content changes. This preserves searchable configuration changes without sending a large inventory document every 15 seconds.

```bash
sudo systemctl status hostscope-snapshot-export.timer --no-pager
sudo systemctl start hostscope-snapshot-export.service
sudo tail -n 3 /var/log/osai/otel/hostscope-context.jsonl | jq .
```

## 7. Control telemetry volume

Edit `observability/collector/docker-compose.yml` or provide environment values before recreating the agent:

```bash
export OSAI_TRACE_SAMPLING_PERCENTAGE=100   # demo: 100; production example: 10
export HOSTMETRICS_INTERVAL=30s
export DOCKER_STATS_INTERVAL=15s
sudo -E docker compose -f observability/collector/docker-compose.yml up -d --force-recreate
```

Controls already applied:

- Collector receiver is loopback-only.
- Trace sampling is configurable in the Collector.
- Filesystem pseudo-mounts and Docker overlay mounts are excluded from host metrics.
- Docker and host collection intervals are configurable.
- File readers use persistent offsets under `observability/collector/state`.
- Docker logs start at the end instead of replaying all historical logs.
- HostScope context events are change-only.
- HostScope uses strict redaction and a pseudonymous host ID.
- Collector memory is bounded to 512 MiB; the memory limiter is set below that ceiling.

## 8. Generate data and verify it

```bash
cd /opt/osai/OS.rs
sudo ./START-HERE.sh demo
sudo ./START-HERE.sh evidence
```

Direct checks:

```bash
curl -fsS http://127.0.0.1:8080/api/v1/health
curl -fsS http://127.0.0.1:13133/
sudo docker logs --tail 100 osai-signoz-collection-agent
sudo journalctl -u osai-agent.service -u hostscope.service -n 100 --no-pager
```

In SigNoz:

1. Open `http://127.0.0.1:8080`.
2. **Services**: search for `osai-agent`, `osai-supervisor`, and workers.
3. **Traces**: filter `service.namespace = osai`.
4. **Logs**: filter `service.name = osai-agent`, `osai-docker-stack`, or `hostscope-context`.
5. **Metrics / Infrastructure**: compare `service.name = osai-host`, `hostscope-agent`, and `osai-docker-stack`.

Useful metric names:

```text
system.cpu.*
system.memory.*
system.filesystem.*
system.network.*
system.process.*
container.cpu.utilization
container.memory.*
container.network.*
hostscope.system.load_average
hostscope.collection.issue.count
```

## 9. Deployment modes

- `sudo ./START-HERE.sh up onprem` — connected installation; local runtime.
- `sudo ./START-HERE.sh airgap-pack /path/osai-airgap.tar.gz` — export runtime dependencies.
- `sudo ./START-HERE.sh up airgap /path/osai-airgap.tar.gz` — strict offline install on a prepared Linux/Docker host.

See [`OFFLINE-AND-ONPREM.md`](OFFLINE-AND-ONPREM.md).

## 10. Service URLs

| Service | URL |
|---|---|
| OSAI UI/API | `http://127.0.0.1:8000` |
| Cognee API | `http://127.0.0.1:8001` |
| llama.cpp/Qwen | `http://127.0.0.1:8088` |
| SigNoz UI | `http://127.0.0.1:8080` |
| SigNoz MCP | `http://127.0.0.1:18000/mcp` |
| RustFS API | `http://127.0.0.1:9000` |
| RustFS console | `http://127.0.0.1:9001` |
| OSAI local OTLP/HTTP | `http://127.0.0.1:14318` |
| Collection-agent health | `http://127.0.0.1:13133` |

## 11. Operator commands

```bash
sudo ./START-HERE.sh status
sudo ./START-HERE.sh demo
sudo ./START-HERE.sh evidence
sudo ./START-HERE.sh telemetry-apply
sudo ./START-HERE.sh logs
sudo ./START-HERE.sh token
sudo ./START-HERE.sh down
```

Equivalent aliases are available through `./osai local-*`. GCP/OpenTofu commands remain available through `./osai deploy`, `observe`, `demo`, `evidence`, `status`, `tunnel`, and `destroy`.

## 12. What can be deleted

Read [`docs/09-SAFE-DELETION-MATRIX.md`](docs/09-SAFE-DELETION-MATRIX.md) before deleting anything.

The conservative summary:

- Safe for runtime: documentation, blog drafts, CI metadata, test fixtures, and packaging examples.
- Conditional: `hostscope/target` is removable after `/usr/local/bin/hostscope` is installed, but OSAI `target/release/osai-*` binaries must remain because systemd normally executes them in place.
- Do not delete: active `.env` files, model files used by a bind mount, `observability/pours`, Collector state, systemd units, HostScope identity salt, Docker volumes, or `/var/lib/osai`.
- `infra/` is required by `START-HERE.sh up onprem`; deleting it breaks clean reinstall even if the current services continue running.

## 13. Security

No populated credentials, Terraform state, provider cache, GGUF model, active `.env` files, or Rust build targets are included in the source ZIP. Runtime credentials are generated locally and protected through systemd credentials. Keep SigNoz ingestion and collection-agent ports private unless you add TLS and authentication at a reverse proxy.

Read [`SECURITY.md`](SECURITY.md).

## 14. More documentation

- [`docs/08-SIGNOZ-INSTRUMENTATION-AND-HOSTSCOPE.md`](docs/08-SIGNOZ-INSTRUMENTATION-AND-HOSTSCOPE.md)
- [`docs/09-SAFE-DELETION-MATRIX.md`](docs/09-SAFE-DELETION-MATRIX.md)
- [`docs/03-SIGNOZ-SIGNAL-DICTIONARY-AND-QUERY-BUILDER.md`](docs/03-SIGNOZ-SIGNAL-DICTIONARY-AND-QUERY-BUILDER.md)
- [`hostscope/README.md`](hostscope/README.md)
- [`ARCHITECTURE.md`](ARCHITECTURE.md)

> **v2.2 AI policy:** background Cognee ingestion and memory writes are disabled by default. llama.cpp/Qwen generates only for an AI-enabled Ask OSAI request. See `docs/07-UI-ONLY-QWEN-RUNTIME-POLICY.md`.
