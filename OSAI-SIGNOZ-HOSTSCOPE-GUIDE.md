# SigNoz Instrumentation and HostScope Operations

This guide describes the exact telemetry contract for OSAI Flight Recorder. It is written for the current hybrid runtime: Rust and HostScope are native systemd services; databases, memory, inference, the OpenTelemetry collection agent, and SigNoz are containers.

## 1. The four observability layers

| Layer | Question answered | OSAI implementation |
|---|---|---|
| Traces | Which operation called which dependency, and where was time spent? | Rust spans exported over OTLP/HTTP |
| Logs | What event or error message was emitted? | journald and Docker JSON logs collected by `filelog` |
| Metrics | How much resource was used and how did it change? | HostScope, `hostmetrics`, and `docker_stats` |
| Context | What is this machine and what configuration/security facts were observed? | HostScope JSON snapshot and read-only MCP |

A complete monitoring design needs all four. Trying to store everything as logs creates expensive, hard-to-query data. Trying to store everything as metrics loses event detail and causality.

## Apply this design to an existing running installation

After copying the updated source into `/opt/osai/OS.rs`, run:

```bash
cd /opt/osai/OS.rs
sudo ./START-HERE.sh telemetry-apply
```

The command updates only the OSAI collection agent, journald bridge, HostScope service/timer, and OSAI OTLP reconnect. It preserves the existing SigNoz backend and all named Docker volumes.

## 2. Network topology

```text
Native OSAI and HostScope
    OTLP/HTTP http://127.0.0.1:14318
                |
                v
osai-signoz-collection-agent (host network)
    receiver 127.0.0.1:14318
    exporter 127.0.0.1:4317 (OTLP/gRPC, insecure local transport)
                |
                v
signoz-ingester-1 :4317/:4318
                |
                v
ClickHouse -> SigNoz UI
```

Because the collection agent uses `network_mode: host`, both addresses refer to the Linux host namespace. The receiver is bound to loopback so other machines cannot submit telemetry to it.

## 3. Rust OSAI traces

`osai-agent/src/telemetry.rs` initializes one trace provider per binary. Named spans are already present in API, scanning, storage, Cognee, PostgreSQL, RustFS, and Qwen code paths.

Environment used by the installed systemd unit:

```dotenv
OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:14318
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_DEPLOYMENT_ENVIRONMENT=hackathon
```

The current code deliberately exports only traces. Structured Rust logs remain on stderr/stdout, enter journald, and are collected once by the Collector.

### Validation

```bash
sudo systemctl restart osai-agent.service
curl -fsS http://127.0.0.1:8000/api/health
sudo journalctl -u osai-agent.service -n 100 --no-pager
sudo docker logs --tail 100 osai-signoz-collection-agent
```

## 4. Instrumenting another application container

A bridge-network container cannot reach the host's loopback through its own `127.0.0.1`. Add the Linux host gateway:

```yaml
services:
  my-service:
    extra_hosts:
      - "host.docker.internal:host-gateway"
    environment:
      OTEL_EXPORTER_OTLP_ENDPOINT: http://host.docker.internal:14318
      OTEL_EXPORTER_OTLP_PROTOCOL: http/protobuf
      OTEL_SERVICE_NAME: my-service
      OTEL_RESOURCE_ATTRIBUTES: service.namespace=osai,deployment.environment.name=development
```

For OTLP/HTTP, use the base endpoint. The SDK appends `/v1/traces`, `/v1/metrics`, or `/v1/logs` when supported.

### When the container has no OpenTelemetry SDK

Use what the component actually exposes:

- stdout/stderr: collect as logs.
- Docker Engine statistics: collect as container metrics.
- Prometheus endpoint: add a Prometheus receiver.
- HTTP dependency called by OSAI: observe it from the OSAI client span.
- No metrics/tracing interface: do not invent fake telemetry; use logs, synthetic checks, or a wrapper/proxy.

This applies to `llama.cpp`: its Docker metrics and logs are visible, while OSAI's `gen_ai.chat` and health-check spans provide request-side latency and errors.

## 5. HostScope operating model

### Safe profile

Use `--profile safe --redaction strict` for continuous operation. It limits command execution to fixed read-only probes and avoids exposing raw machine identifiers.

```bash
hostscope --profile safe --redaction strict doctor
hostscope --profile safe --redaction strict snapshot --pretty
```

### OTLP metrics

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:14318
export HOSTSCOPE_INTERVAL_SECONDS=15
hostscope --profile safe --redaction strict watch
```

HostScope publishes:

```text
system.uptime
system.cpu.logical.count
system.cpu.physical.count
system.memory.usage
system.memory.utilization
system.paging.usage
system.filesystem.usage
system.filesystem.utilization
system.network.io
system.network.packet.count
system.network.packet.dropped
system.network.errors
system.process.count
hostscope.system.load_average
hostscope.collection.issue.count
```

### JSON context

The timer writes the current full document to:

```text
/var/lib/hostscope/latest.json
```

A changed-content hash prevents the same host context from being emitted repeatedly. A compact event is appended to:

```text
/var/log/osai/otel/hostscope-context.jsonl
```

The Collector parses this file and assigns `service.name=hostscope-context`.

### MCP

HostScope MCP is read-only and stdio-based. It exposes:

```text
hostscope://snapshot/latest
hostscope://schema/v1alpha1
hostscope://policy/effective
host_snapshot(section?, pretty?)
host_health_summary()
```

Do not expose the stdio MCP process directly on a public TCP port.

## 6. Why HostScope and Collector hostmetrics both exist

They are separate series with separate resource identities:

- `osai-host`: standard Collector hostmetrics, optimized for infrastructure dashboards.
- `hostscope-agent`: metrics derived from a versioned host snapshot plus explicit collection-quality information.

They may represent similar quantities, but they are not merged silently. Dashboards must filter by `service.name` so the operator chooses the intended source.

Recommended use:

- Infrastructure dashboard: `service.name = osai-host`.
- HostScope quality/context dashboard: `service.name = hostscope-agent`.
- Cross-check during development: show both to detect namespace/permission mistakes.

## 7. Data-volume controls

| Control | Default | Production starting point |
|---|---:|---:|
| Trace sampling | 100% | 10-25%, then increase for errors/critical paths |
| Host metrics interval | 30 s | 30-60 s |
| Docker stats interval | 15 s | 30 s |
| HostScope metrics interval | 15 s | 30-60 s |
| HostScope context event | On content change | Keep change-only |
| Docker log start position | End | Keep end unless backfill is required |

Apply Collector controls:

```bash
export OSAI_TRACE_SAMPLING_PERCENTAGE=10
export HOSTMETRICS_INTERVAL=60s
export DOCKER_STATS_INTERVAL=30s
sudo -E docker compose \
  -f observability/collector/docker-compose.yml \
  up -d --force-recreate
```

## 8. SigNoz views to create

### OSAI request health

- Trace service: `osai-agent`
- Group by: span name
- Charts: request count, p50/p95/p99 duration, error rate

### Dependency health

Span names:

```text
osai.postgres.connect
osai.postgres.load_latest_scan
osai.rustfs.put_object
osai.cognee.recall
osai.cognee.delivery_cycle
gen_ai.chat
osai.llm.health_check
```

### Host dashboard

Filter `service.name = osai-host` and chart CPU, memory, filesystem, network, paging, and process metrics.

### HostScope quality dashboard

Filter `service.name = hostscope-agent` and chart:

```text
hostscope.collection.issue.count
hostscope.system.load_average
system.memory.utilization
system.filesystem.utilization
system.network.errors
system.network.packet.dropped
```

### Container dashboard

Filter/group by Docker Compose labels when available:

```text
docker.compose.project
docker.compose.service
container.name
container.id
```

## 9. Troubleshooting decision tree

### No OSAI traces

```bash
sudo systemctl show osai-agent.service -p Environment
ss -ltnp | grep 14318
sudo journalctl -u osai-agent.service -n 100 --no-pager
sudo docker logs --tail 100 osai-signoz-collection-agent
```

### HostScope export failure

```bash
sudo systemctl status hostscope.service --no-pager
sudo journalctl -u hostscope.service -n 100 --no-pager
curl -fsS http://127.0.0.1:13133/
sudo /usr/local/bin/hostscope --profile safe --redaction strict watch --once --endpoint http://127.0.0.1:14318
```

### Logs absent

```bash
sudo systemctl status osai-journal-export.service --no-pager
sudo test -s /var/log/osai/otel/osai-agent.json
sudo ls -l /var/lib/docker/containers/*/*-json.log | head
sudo docker logs --tail 100 osai-signoz-collection-agent
```

### Container endpoint confusion

From a bridge container:

```bash
getent hosts host.docker.internal
curl -v http://host.docker.internal:14318/
```

An HTTP 404/405 is sufficient to prove TCP reachability; OTLP requires a protobuf POST to the signal-specific path.
