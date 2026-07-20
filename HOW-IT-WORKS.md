# HOW-IT-WORKS — Architecture and `investigator.py`

## Big picture

```text
Users / CronJob
    → checkout-api → payment-svc → inventory-svc
         │  (OpenTelemetry traces + logs)
         ▼
    SigNoz Collector :4318  →  ClickHouse  →  SigNoz UI
         ▲
         │ MCP queries (HTTP :8000)
         │
    Incident Sentinel (FastAPI)
         │
         ├─ POST /webhook/signoz   ← SigNoz alert channel
         ├─ POST /investigate      ← manual demo
         ├─ LLM (mock or real) decides tool vs conclude
         ├─ MCP tools: search traces/logs, list services, …
         └─ OTLP out again → same SigNoz (self-observability)
```

## Main files

| File | Job |
|------|-----|
| `copilot/app/main.py` | HTTP API: healthz, webhook, investigate |
| `copilot/app/investigator.py` | Agent loop: dedupe → LLM → MCP tools → report |
| `copilot/app/mcp_client.py` | Calls SigNoz MCP `tools/call` |
| `copilot/app/llm.py` | Mock / OpenAI / Anthropic / Groq + system prompt |
| `copilot/app/telemetry.py` | OTel traces + metrics setup |
| `copilot/app/report.py` | Slack Block Kit or stdout |
| `copilot/app/config.py` | Env vars |

## What `investigator.py` does (simple)

1. Read alert JSON → decide `rule_id` / `alert_name`  
2. Skip if same rule investigated recently (default 10 minutes) or too many in flight  
3. Start span `sentinel.investigate`  
4. Ask LLM what to do (JSON: `tool` or `conclude`)  
5. If tool → call matching MCP function, feed result back to LLM  
6. If conclude → build report → `post_report()`  
7. Record token/cost metrics; return JSON including `trace_id`  

## Tools allowed

- `signoz_list_alerts`  
- `signoz_get_alert_history`  
- `signoz_list_services`  
- `signoz_search_traces`  
- `signoz_get_trace_details`  
- `signoz_search_logs`  
- `signoz_query_metrics`  

## Mock mode

If `LLM_PROVIDER=mock` or no `LLM_API_KEY`, the mock still requests MCP tools then concludes — so you can demo without paying for an LLM.
