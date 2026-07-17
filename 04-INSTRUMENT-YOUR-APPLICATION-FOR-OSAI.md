# “Instrument Your Application” for the OSAI Stack

## 1. The page is a routing catalogue

SigNoz lists JavaScript, Python, Java, Rust, C++, mobile, NGINX, eBPF, and platform integrations because each runtime needs a different way to create telemetry. You do **not** install all of them into one process.

For OSAI:

| Component | Instrumentation path |
|---|---|
| Rust Axum API and workers | Rust OpenTelemetry SDK plus manual `tracing` spans |
| Cognee FastAPI | Python/FastAPI automatic instrumentation plus manual Cognee stages |
| llama.cpp | HTTP/client spans plus custom log-to-metric bridge; optional eBPF/OBI |
| Browser UI | Browser/frontend OpenTelemetry if added |
| PostgreSQL | Rust/Python database client spans and database metrics |
| RustFS | HTTP/S3 client spans and object-store attributes |
| Linux host | HostScope and Collector `hostmetrics` receiver |
| Docker | Collector `docker_stats` and container log receiver |
| journald | Collector filelog pipeline fed by the journal bridge |

## 2. Automatic versus manual instrumentation

Automatic instrumentation gives technical boundaries:

```text
POST /api/v1/remember took 70 seconds
```

Manual instrumentation gives application meaning:

```text
cognee.chunk       2 seconds
cognee.embed      12 seconds
cognee.cognify    52 seconds
cognee.persist     4 seconds
```

Professional systems use both.

## 3. Current state in this ZIP

### Rust

`osai-agent/src/telemetry.rs` initializes an OTLP/HTTP trace exporter and sets service resources. Instrumented Rust functions create native spans. Logs remain `tracing` output collected from journald.

Current strength:

- native traces;
- stable service names;
- Collector export;
- graceful provider shutdown.

Current gap:

- the Rust application does not yet expose a complete production metric set through an OpenTelemetry MeterProvider;
- native OTel logs are not yet the primary path.

### Demo harness

`observability/scripts/generate-demo-telemetry.sh` manually builds OTLP JSON. It exists to guarantee that you can learn trace-log correlation and metric queries even before every application instrumentation path is complete.

### Host and Docker

`observability/collector/config.yaml` creates standard host and Docker signals through receivers. This is instrumentation without modifying OSAI source code.

### Cognee

The current container is API-compatible, but full FastAPI/Cognee stage instrumentation is a future enhancement. At present, Cognee container logs and OSAI client spans provide visibility.

### llama.cpp

llama.cpp prints model timings. A production telemetry bridge should parse these into structured logs and model metrics. Generic HTTP spans alone are insufficient for token-level observability.

## 4. Resource identity

Every service should set:

```text
service.namespace=osai
service.name=<logical component>
service.version=<build version>
deployment.environment.name=onprem|airgap|dev
```

`service.name` must describe a logical service, not a unique request or host.

Good:

```text
osai-agent
osai-cognee
osai-llama-bridge
```

Bad:

```text
osai-agent-NAVJYOT-20260717
```

Use `service.instance.id` or `host.name` for the instance.

## 5. Context propagation

A distributed trace works only if context crosses boundaries.

```text
Browser
  → Rust API
    → Cognee HTTP request
      → later background pipeline
```

Synchronous HTTP uses W3C `traceparent`. Background outbox work should store trace context or create a linked trace.

## 6. Recommended next instrumentation work

1. Add a Rust MeterProvider and real OSAI counters/histograms.
2. Add structured `error.type` classifications to failed spans and logs.
3. Add automatic FastAPI instrumentation to Cognee.
4. Add manual Cognee pipeline spans.
5. Build a llama.cpp timing parser/bridge.
6. Add browser fetch tracing and Web Vitals only after backend signals are stable.
7. Add tail sampling only when volume makes it necessary.

## 7. Privacy

Record metadata, not secrets or unrestricted content.

Capture:

```text
model name
token counts
duration
operation
result
retry count
queue depth
```

Avoid by default:

```text
API keys
full prompts
full model responses
authorization headers
private command output
complete environment variables
```
