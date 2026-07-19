# OSAI Signal Ownership Standard

## Non-negotiable rules

1. One signal has one ingestion path.
2. `service.name` identifies a stable component, not a host, request, or container ID.
3. Unique request IDs and trace IDs never become metric attributes.
4. Raw prompts, answers, Cognee memory, and credentials are not exported by default.
5. Errors set span status and also emit a structured log event.
6. Every browser-visible Ask response returns its Trace ID.
7. The Collector is the only gateway from native OSAI telemetry to SigNoz.

## Application signal contract

### Resource attributes

```text
service.name=osai-agent
service.namespace=osai
service.version=<Cargo package version>
deployment.environment=<environment>
telemetry.distro.name=osai-flight-recorder
```

### Ask spans

```text
osai.api.ask
osai.agent.answer
osai.ask.plan
osai.llm.health_check
osai.postgres.connect
osai.postgres.load_latest_scan
osai.ask.fact_pack.build
osai.cognee.recall
osai.ask.prompt.build
gen_ai.chat
```

### Structured log events

```text
osai.ask.received
osai.ask.plan.completed
osai.ask.component.failed
osai.cognee.recall.completed
osai.gen_ai.completion
osai.ask.answer.ready
osai.ask.completed
osai.ask.failed
```

### Application metrics

```text
osai.ask.request.count
osai.ask.error.count
osai.ask.duration
osai.ask.component.duration
osai.gen_ai.token.usage
```

## Host signal contract

```text
hostscope-agent   HostScope metrics
hostscope-context Strict-redaction change events
osai-host         Standard hostmetrics
```

## Container signal contract

```text
osai-docker-stack Docker resource metrics and stdout/stderr logs
```

## Collector health contract

```text
service.name=osai-signoz-collection-agent
metric prefix=otelcol_
```

## Why journald is no longer a SigNoz ingestion path

The Rust process still writes readable logs to journald. However, copying journald into SigNoz as a second path creates duplicate semantics and makes trace/log correlation harder. Direct OTLP logs preserve standard Trace ID and Span ID context. Therefore:

```text
journalctl = local operator fallback
OTLP logs  = SigNoz application log source
```
