# REMAINING-STEPS — After deploy, finish the hackathon deliverables

Use this after you can run `/investigate` successfully (`STEP-BY-STEP.md` through Step 5–7).

---

## Already done in this pack (code + prior env work)

- [x] Project design (Track 01 Incident Sentinel)
- [x] Full source: Foundry casting files, demo apps, copilot agent, scripts
- [x] Docs: README, IMPLEMENT-README, blog draft, dashboard/alert specs
- [x] On the original lab: Foundry SigNoz+MCP running, demo+copilot deployed, E2E investigate verified with mock LLM

---

## Remaining for a complete prize submission

Do these on your demo env + accounts. Order recommended.

### 1. Long-lived SigNoz API key
- [ ] Create Service Account API key in SigNoz UI  
- [ ] Store in k8s Secret `incident-sentinel-secrets` (not only short JWT)  
- [ ] Confirm MCP tools work after pod restart  

### 2. Alert → webhook end-to-end
- [ ] Webhook channel → `http://<node>:32080/webhook/signoz`  
- [ ] Alert rule `checkout-error-spike` (and optionally latency)  
- [ ] Prefer channel attached; fire with `./demo/break.sh errors`  
- [ ] Confirm report without calling `/investigate` manually  

### 3. Optional but stronger demo
- [ ] Real `LLM_API_KEY` (OpenAI / Anthropic / Groq)  
- [ ] `SLACK_WEBHOOK_URL` for human-readable reports  
- [ ] Install k8s-infra chart pointing at Foundry `:4318`  

### 4. Dashboards (Track 02-ready artifact)
- [ ] Build **Incident Overview** panels in SigNoz UI from `project/dashboards/incident-overview.json`  
- [ ] Build **Copilot Operations** (tokens, cost, duration, tool errors)  
- [ ] Export real dashboard JSON back into the repo  

### 5. Meta-alert (the “watcher is watched” beat)
- [ ] Create cost-budget alert from `project/alerts/sentinel-cost-budget.json`  
- [ ] Show it in the demo video  

### 6. Demo video (2–3 minutes)
- [ ] Record: break demo → alert/investigate → Slack or stdout report → SigNoz shows `incident-sentinel` investigation trace  
- [ ] Script: `project/docs/DEMO-VIDEO.md`  
- [ ] Upload (YouTube unlisted is fine)  

### 7. Blog
- [ ] Edit `project/blog/hackathon-blog-draft.md` with your real screenshots / gotchas  
- [ ] Publish on **Dev.to / Medium / Substack** (not LinkedIn-only)  
- [ ] Keep Early Win blog separate if you also submit that prize  

### 8. GitHub + form
- [ ] Push this `project/` (or the whole handoff) to a public GitHub repo  
- [ ] Ensure `foundry/casting.yaml` **and** `casting.yaml.lock` are in the repo  
- [ ] Submit WeMakeDevs form: Track 01, repo URL, blog URL, video URL  
- [ ] **Declare AI assistant use** (required)  
- [ ] Optional: social posts tagging @wemakedevs + SigNoz  

### 9. Stretch only
- [ ] kagent packaging (`project/copilot/kagent/`) after core demo is solid  

---

## Separate prize (do not mix stories)

- [ ] Early Win blog due **July 19, 2026** — materials under `~/vicky/signoz-early-win` / `signoz-k8s-early-win`, not this pack’s main narrative  

---

## Definition of “submission ready”

You are ready when:

1. Someone else can clone the GitHub repo and follow `STEP-BY-STEP.md`  
2. Demo video shows alert/investigation + self-telemetry in SigNoz  
3. Blog is published and linked  
4. Form submitted with AI disclosure  
