# SigNoz Signal Dictionary and Query Builder Guide

## 1. First separate four concepts

| Concept | Example | Meaning |
|---|---|---|
| Metric name | `osai.demo.request.duration` | The numerical instrument selected from the Metrics dropdown |
| Resource attribute | `service.name`, `host.name` | Identity of the service/host producing the signal |
| Data-point/span/log attribute | `url.path`, `error.type` | Context attached to one measurement, span, or log |
| Log top-level field | `severity_text`, `body`, `trace_id` | Standard field in the OpenTelemetry log record |

A query fails when a metric name is treated as an attribute, when a logs-only field is used in Metrics Explorer, or when a field was never emitted.

## 2. Where the names in this project come from

### `osai.demo.request.duration`

- **Type:** Custom demo metric.
- **Created in:** `observability/scripts/generate-demo-telemetry.sh`.
- **Unit:** milliseconds.
- **Current instrument:** Gauge containing one observed endpoint duration per request.
- **Purpose:** Training, screenshots, and deterministic dashboard examples.
- **Production note:** A production request-duration metric should normally be a histogram, or duration can be derived from traces.

Attributes attached to each point:

```text
http.request.method
url.path
http.response.status_code
```

### `osai.demo.http.error`

- **Type:** Custom demo metric.
- **Created in:** the same demo harness.
- **Value:** `1` when the harness receives HTTP 4xx/5xx or cannot connect; otherwise `0`.
- **Purpose:** A deterministic error counter-like training signal.
- **Production note:** It is implemented as a gauge for simplicity. A real production error total should use a counter or be derived from trace status/HTTP response attributes.

Additional attribute:

```text
osai.demo.expected_error=true|false
```

### `service.name`

- **Type:** Resource attribute.
- **Meaning:** Logical service that produced telemetry.
- **Created by:** Rust SDK resources, Collector resource processors, HostScope, or the demo harness.

Values in this project include:

```text
osai-agent
osai-observability-harness
osai-host
osai-docker-stack
hostscope-agent
```

### `severity` / `severity_text` / `severity_number`

OpenTelemetry logs have two normalized severity fields:

```text
severity_text    Original level string such as INFO, WARN, ERROR
severity_number  Normalized numeric severity
```

In current SigNoz log filters, prefer autocomplete and commonly use:

```text
severity_text = 'ERROR'
severity_text IN ('WARN', 'ERROR')
```

The UI may label a quick-filter category simply as **Severity**, while the stored/query field is `severity_text`.

### `host.name`

- **Type:** Resource attribute.
- **Meaning:** Host that emitted or was associated with telemetry.
- **Created by:** Collector resource detection, HostScope, or the demo harness.
- **Example:** `NAVJYOT`.

### `container.name`

- **Type:** Container resource/log attribute.
- **Meaning:** Docker container associated with a metric or log.
- **Created by:** Docker stats receiver or container log parser.
- **Examples:** `osai-llama`, `osai-cognee`.

Do not assume every Docker record exposes exactly the same attribute spelling. Open one record and use autocomplete/quick actions.

### `error.type`

- **Type:** Standard error-classification attribute.
- **Meaning:** Predictable, low-cardinality error class such as `timeout`, `dns_error`, `404`, or a canonical exception type.
- **Important:** It exists only when the instrumentation explicitly records it. A plain text WARN log does not automatically gain `error.type`.

### `system.cpu.*`

This is a metric namespace, not one complete metric. Common metrics include:

```text
system.cpu.time
system.cpu.utilization
system.cpu.logical.count
```

### `system.memory.*`

Common metrics:

```text
system.memory.usage
system.memory.utilization
system.paging.usage
system.paging.operations
```

### `system.disk.*`

Common metrics:

```text
system.disk.io
system.disk.io_time
system.disk.operation_time
system.disk.operations
```

### `system.filesystem.*`

Common metrics:

```text
system.filesystem.usage
system.filesystem.utilization
system.filesystem.limit
```

Typical attributes:

```text
system.device
system.filesystem.mountpoint
system.filesystem.type
system.filesystem.state
```

### `system.network.*`

Common metrics:

```text
system.network.io
system.network.packets
system.network.errors
system.network.dropped
```

Typical attributes:

```text
network.interface.name
network.io.direction
```

### `process.*` versus `system.process.*`

```text
process.*         One specific OS process
system.process.*  Aggregate process counts for the whole host
```

Common individual-process metrics:

```text
process.cpu.time
process.cpu.utilization
process.memory.usage
process.disk.io
process.network.io
process.thread.count
process.uptime
```

Common aggregate metric:

```text
system.process.count
```

## 3. How metrics are created

There are three creation paths in this project.

### Application code

A Rust/Python SDK creates an instrument and records values:

```text
Meter creates histogram/counter/gauge
  → code records value plus attributes
  → SDK aggregates
  → OTLP exporter sends it
```

### Demo harness

The shell script builds OTLP JSON manually and sends it to `/v1/metrics`.

### Collector receivers

The `hostmetrics` and `docker_stats` receivers read the operating system and Docker Engine and create standard metrics automatically. You do not write Rust code for these metrics.

## 4. Query Builder mental model

### Logs Explorer

1. Choose the time range.
2. Filter records.
3. Select List, Time Series, or Table.
4. Optionally aggregate and group.

Examples:

```text
service.name = 'osai-agent'
service.name = 'osai-agent' AND severity_text IN ('WARN', 'ERROR')
service.name = 'osai-docker-stack' AND body ILIKE '%cancel task%'
service.name = 'osai-observability-harness' AND trace_id EXISTS
error.type EXISTS
```

Use single quotes around string values.

### Traces Explorer

Filter spans:

```text
service.name = 'osai-agent'
service.name = 'osai-observability-harness'
name = 'osai.scan.execute'
name LIKE '%snapshot%'
osai.demo.expected_error = true
error.type EXISTS
```

Use quick filters for duration and status when possible.

### Metrics Explorer

Metrics queries begin by selecting a metric name from the dropdown. Then add attribute filters, aggregation, and group-by.

#### Demo latency

```text
Metric: osai.demo.request.duration
Temporal aggregation: Average or Latest
Space aggregation: Average, Maximum, or P95 if available for the selected type
Group by: url.path
Filter: service.name = 'osai-observability-harness'
```

Because the demo instrument is a gauge, percentile behavior may differ from a true histogram. For professional latency analysis, use trace duration or a proper histogram.

#### Demo errors

```text
Metric: osai.demo.http.error
Temporal aggregation: Sum
Space aggregation: Sum
Group by: url.path
Filter: osai.demo.expected_error = true
```

#### Host memory

```text
Metric: system.memory.utilization
Aggregation: Average or Maximum
Filter: host.name = 'NAVJYOT'
Group by: system.memory.state (only when useful)
```

#### Disk throughput

```text
Metric: system.disk.io
Function/temporal operation: Rate
Filter: host.name = 'NAVJYOT'
Group by: system.device, disk.io.direction
```

Counters such as I/O bytes should usually be converted to a rate to show bytes per second.

#### Network throughput

```text
Metric: system.network.io
Function/temporal operation: Rate
Filter: host.name = 'NAVJYOT'
Group by: network.interface.name, network.io.direction
```

#### Process memory

```text
Metric: process.memory.usage
Aggregation: Maximum
Filter: host.name = 'NAVJYOT'
Group by: process.executable.name or process.command
```

Use only attributes shown by autocomplete; process attribute availability depends on host permissions and Collector version.

## 5. Why SigNoz sometimes “keeps showing” the builder

The Query Builder is the normal primary interface. It needs you to choose:

```text
What data?      Metric or signal
Which records?  Filter
How combined?   Aggregation
How separated?  Group By
How displayed?  Time series, table, value, bar, etc.
```

A blank query is not an error. It is an empty question waiting for those choices.

## 6. Troubleshooting a query

1. Confirm the time range includes fresh telemetry.
2. Use autocomplete rather than guessing field names.
3. Open one span/log and inspect its JSON/attributes.
4. Start with one simple filter.
5. Add conditions one at a time.
6. Put string values in single quotes.
7. Confirm you are querying the correct signal type.
8. For metrics, select the metric first; do not type a metric name as a filter attribute.
9. If a field is absent, fix instrumentation or use a field that exists.

## 7. Training fields versus production fields

Use `osai.demo.*` to learn and demonstrate the pipeline. Do not confuse them with complete production observability.

Production evolution should add metrics such as:

```text
osai.scan.duration            Histogram
osai.cognee.outbox.depth      Gauge
osai.cognee.delivery.duration Histogram
osai.cognee.delivery.failures Counter
osai.llm.inference.duration   Histogram
osai.llm.tokens               Counter
osai.llm.tokens_per_second    Histogram or gauge distribution
```

Where standard OpenTelemetry semantic metrics exist, prefer them over custom duplicates.
