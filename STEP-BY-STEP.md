# STEP-BY-STEP — Deploy Incident Sentinel on another server

Follow these steps **in order**. After each step there is a **Checkpoint** — do not continue until it passes.  
Each step explains *why*, not only *what*.

Working directory after unzip: `incident-sentinel-handoff/project/`

---

## Before you start — what you need

| Need | Why |
|------|-----|
| A Linux VM/server with Docker + Compose v2 | SigNoz installs via **Foundry** (hackathon rule) as Docker Compose |
| ~20 GB free disk, ~4+ GB free RAM | ClickHouse + SigNoz need space |
| Open ports **8080**, **4317**, **4318**, **8000** | UI, OTLP, MCP |
| A Kubernetes cluster **that can reach that server’s IP** on those ports | Demo apps + copilot run in k8s and send telemetry to SigNoz |
| `kubectl` access to that cluster | Deploy demo + agent |
| Optional: LLM API key, Slack webhook | Richer demo; `mock` LLM works without a key |

**Important design rule:** put demo + copilot on a cluster that can talk to the Foundry host. Same machine / same VPC is easiest. Cross-cluster with blocked firewall will fail (we hit that once already).

Set these for your env (examples):

```bash
export FOUNDRY_HOST=10.0.0.50          # IP of the Docker host running SigNoz
export NS=foundry                      # k8s namespace
cd incident-sentinel-handoff/project
```

---

## STEP 1 — Understand the pieces (no install yet)

**What you are building:**

```text
[Demo apps in k8s] --OTLP:4318--> [SigNoz + MCP on Docker]
                                      |
                                      | alert webhook (later)
                                      v
                              [Incident Sentinel copilot in k8s]
                                      |
                         MCP queries + Slack/stdout report
                         + own OTel traces back to SigNoz
```

| Folder | Role |
|--------|------|
| `foundry/` | How to install SigNoz + MCP reproducibly |
| `demo/` | Fake shop that can break on purpose |
| `copilot/` | The AI investigator (`investigator.py` is the brain) |
| `scripts/` | One-shot deploy helpers |

**Checkpoint:** You can explain in one sentence: “apps send telemetry to SigNoz; the copilot uses MCP to investigate and reports back.”

---

## STEP 2 — Install SigNoz + MCP with Foundry

**Why:** Hackathon field requirement — judges may re-run `foundryctl cast -f casting.yaml`. Helm-only SigNoz is not enough for submission.

On the **Docker host**:

```bash
mkdir -p ~/signoz-foundry && cd ~/signoz-foundry
cp /path/to/incident-sentinel-handoff/project/foundry/casting.yaml .

# Try native install first
curl -fsSL https://signoz.io/foundry.sh | bash
# then:
foundryctl cast -f casting.yaml
```

### If `foundryctl` fails with `GLIBC_2.32/2.34 not found` (common on EL8)

**Why:** Binary needs newer glibc than the host. Run foundryctl in a newer container, but drive the **host** Docker daemon:

```bash
cd ~/signoz-foundry
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /usr/bin/docker:/usr/bin/docker:ro \
  -v /usr/libexec/docker/cli-plugins:/usr/libexec/docker/cli-plugins:ro \
  -v "$PWD":/work -w /work \
  debian:bookworm-slim bash -c '
    set -e
    apt-get update -qq && apt-get install -y -qq curl ca-certificates >/dev/null
    ln -sf /usr/libexec/docker/cli-plugins/docker-compose /usr/local/bin/docker-compose
    curl -fsSL https://github.com/SigNoz/foundry/releases/download/v0.2.11/foundry_linux_amd64.tar.gz | tar xz
    ./foundry_linux_amd64/bin/foundryctl cast -f casting.yaml
  '
```

### If ClickHouse Keeper crash-loops (exit 139)

**Why:** Some hosts segfault on `clickhouse-keeper:25.12.5`. Pin an older image:

```bash
cd ~/signoz-foundry/pours/deployment
sed -i 's|clickhouse/clickhouse-keeper:25.12.5|clickhouse/clickhouse-keeper:24.8.14|' compose.yaml
docker compose up -d
docker compose ps
```

Copy the lock file into the project (required for submission):

```bash
cp ~/signoz-foundry/casting.yaml.lock \
  /path/to/incident-sentinel-handoff/project/foundry/
```

**Checkpoint:**

```bash
curl -s http://127.0.0.1:8080/api/v1/health
curl -s http://127.0.0.1:8000/livez
# both should succeed (200 / ok)
```

Open UI: `http://$FOUNDRY_HOST:8080`

---

## STEP 3 — Create the admin account

**Why:** SigNoz rejects meaningful use until an org exists. Do this before trusting data.

Browser: register at the UI, **or**:

```bash
curl -s -X POST http://127.0.0.1:8080/api/v1/register \
  -H 'Content-Type: application/json' \
  -d '{
    "name":"Admin",
    "email":"admin@example.com",
    "password":"ChooseAStrongPassword!",
    "orgName":"IncidentSentinel"
  }'
```

Save `orgId` from the response.

Then create a **Service Account API key** in the UI (Settings → Service Accounts).  
That key becomes `SIGNOZ_API_KEY` for MCP. (Short-lived login JWT also works for tests.)

**Checkpoint:** You can log into the UI. You have an API key string saved somewhere safe (not in git).

---

## STEP 4 — Deploy demo apps + copilot to Kubernetes

**Why:** Demo generates traces/logs; copilot is the investigator. Both need the Foundry host IP as OTLP/MCP endpoint.

```bash
cd /path/to/incident-sentinel-handoff/project
export FOUNDRY_HOST=<YOUR_SIGNOZ_HOST_IP>
export LLM_PROVIDER=mock          # no paid LLM needed for first demo
export SIGNOZ_API_KEY='<your-key>'
# optional:
# export LLM_PROVIDER=openai
# export LLM_API_KEY=sk-...
# export SLACK_WEBHOOK_URL=https://hooks.slack.com/...

chmod +x scripts/*.sh demo/break.sh
./scripts/deploy-k8s.sh
```

What the script does (so you understand):

1. Ensures namespace `foundry` exists  
2. Applies `demo/demo-app.yaml` with `OTLP_HOST_PLACEHOLDER` → `$FOUNDRY_HOST`  
3. Packs `copilot/app` into a ConfigMap and runs it with `python:3.11-slim`  
4. Creates Service **NodePort 32080** for the webhook  

**Checkpoint:**

```bash
kubectl -n foundry get pods
# checkout-api, payment-svc, inventory-svc, incident-sentinel → Running

kubectl -n foundry run netcheck --rm -i --restart=Never --image=curlimages/curl:8.5.0 -- \
  curl -s -o /dev/null -w "%{http_code}\n" --connect-timeout 5 http://$FOUNDRY_HOST:8000/livez
# expect 200
```

---

## STEP 5 — Put the API key into the copilot and smoke-test

```bash
kubectl -n foundry create secret generic incident-sentinel-secrets \
  --from-literal=SIGNOZ_API_KEY="$SIGNOZ_API_KEY" \
  --from-literal=LLM_API_KEY="${LLM_API_KEY:-}" \
  --from-literal=SLACK_WEBHOOK_URL="${SLACK_WEBHOOK_URL:-}" \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl -n foundry rollout restart deploy/incident-sentinel
kubectl -n foundry rollout status deploy/incident-sentinel
```

Run an investigation (uses `investigator.py`):

```bash
SVC=$(kubectl -n foundry get svc incident-sentinel -o jsonpath='{.spec.clusterIP}')

curl -s http://$SVC:8080/healthz

# use a unique ruleId so dedupe (10 min) does not skip
curl -s -X POST http://$SVC:8080/investigate \
  -H 'Content-Type: application/json' \
  -d "{\"alert_name\":\"checkout-error-spike\",\"ruleId\":\"demo-$(date +%s)\",\"labels\":{\"service\":\"checkout-api\"}}" \
  | python3 -m json.tool | head -80
```

**What “good” looks like:**

- `"skipped": false`
- `"report"` with `summary` / `root_cause` / `confidence`
- `"trace_id"` present  
- Pod logs show MCP `HTTP/1.1 200 OK`

**Checkpoint:** JSON report printed. You understand this hit `Investigator.investigate()`.

---

## STEP 6 — Break the demo on purpose

**Why:** So SigNoz has real error spans to investigate.

```bash
./demo/break.sh errors
# wait ~1–2 minutes for CronJob / traffic

kubectl -n foundry port-forward svc/checkout-api 18000:8000
# other terminal:
curl -s -w "\n%{http_code}\n" http://127.0.0.1:18000/checkout
# 502 sometimes = success of fault injection
```

In SigNoz UI: **Traces** → filter `checkout-api` / `has_error`.

Heal when done:

```bash
./demo/break.sh heal
```

**Checkpoint:** You see error traces (or at least services) in SigNoz.

---

## STEP 7 — (Almost done on infra) Wire alert → webhook

**Why:** Full product loop is alert-driven, not only manual `/investigate`.

1. In SigNoz UI → notification channel type **webhook**  
   URL: `http://<K8S_NODE_IP>:32080/webhook/signoz`  
   (must be reachable from the Foundry host)  
2. Create alert for checkout errors (see `alerts/README.md` / `alerts/checkout-error-spike.json`)  
3. Attach the channel  
4. Run `./demo/break.sh errors` and watch copilot logs  

**Checkpoint:** Alert fires → copilot log shows webhook received → report appears.

---

## STEP 8 — Optional polish on this server

| Optional | Command / action |
|----------|------------------|
| Real LLM | Set `LLM_PROVIDER` + `LLM_API_KEY`, restart deploy |
| Slack | Set `SLACK_WEBHOOK_URL` in Secret |
| k8s-infra metrics | `helm install ... -f demo/k8s-infra-values.yaml` (edit endpoint first) |
| Dashboards | Recreate from `dashboards/*.json` in UI, export real JSON |
| Cost meta-alert | See `alerts/sentinel-cost-budget.json` |

---

## After Step 7 — what is left?

Infra for a demo is done. Submission work is listed in **`REMAINING-STEPS.md`**.

---

## Troubleshooting (short)

| Problem | Fix |
|---------|-----|
| foundryctl glibc error | Container workaround in Step 2 |
| Keeper exit 139 | Pin keeper `24.8.14` |
| Pods timeout to MCP/OTLP | Wrong cluster / firewall — deploy closer to Foundry |
| MCP 401 | Refresh API key / Service Account key |
| `/investigate` → `skipped: true` | Same `ruleId` within 10 min — change `ruleId` |
| NodePort curl empty | Use Service ClusterIP instead |

More detail: `project/IMPLEMENT-README.md`.
