# OSAI Flight Recorder Learning Path

This edition keeps the runnable v2 stack and adds a structured path for learning the repository and SigNoz professionally.

## Start here

1. Run `sudo ./START-HERE.sh status` and identify every running process and container.
2. Read [`docs/01-FOLDER-AND-FILE-MAP.md`](docs/01-FOLDER-AND-FILE-MAP.md).
3. Read [`docs/02-COMMANDS-AND-EXPECTED-RESULTS.md`](docs/02-COMMANDS-AND-EXPECTED-RESULTS.md).
4. Read [`docs/03-SIGNOZ-SIGNAL-DICTIONARY-AND-QUERY-BUILDER.md`](docs/03-SIGNOZ-SIGNAL-DICTIONARY-AND-QUERY-BUILDER.md).
5. Read [`docs/04-INSTRUMENT-YOUR-APPLICATION-FOR-OSAI.md`](docs/04-INSTRUMENT-YOUR-APPLICATION-FOR-OSAI.md).
6. Complete [`docs/05-SIGNOZ-HANDS-ON-LABS.md`](docs/05-SIGNOZ-HANDS-ON-LABS.md).

## Helpful commands

```bash
./START-HERE.sh structure
./START-HERE.sh queries
sudo ./START-HERE.sh lab
```

## Mental model

```text
Application behavior
    ↓ creates spans, metrics, and logs
OpenTelemetry SDKs / receivers
    ↓ send OTLP
OSAI Collector on 127.0.0.1:14318
    ↓ forwards OTLP
SigNoz ingester on 4317/4318
    ↓ persists
ClickHouse
    ↓ queries and visualizes
SigNoz UI on 8080
```

## Do not memorize field names blindly

Always ask four questions:

1. **Which signal?** Trace, metric, or log?
2. **Who created it?** Rust code, demo harness, Collector receiver, Docker parser, or HostScope?
3. **Is it a metric name or an attribute?** `system.memory.usage` is a metric; `host.name` is an attribute.
4. **Does the field actually exist in the selected time range?** Use SigNoz autocomplete and inspect one record.
