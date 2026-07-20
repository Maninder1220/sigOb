# START HERE — Incident Sentinel Handoff Pack

This folder is a **complete, portable copy** of the Incident Sentinel hackathon project.

Anyone new should read files in this order:

1. **This file** (`START-HERE.md`) — what you have and how to use the pack  
2. **`STEP-BY-STEP.md`** — deploy on a **new** server with understanding at each step  
3. **`REMAINING-STEPS.md`** — what is still left after deploy (submission deliverables)  
4. **`project/README.md`** — product story + current status from the original build  
5. **`project/IMPLEMENT-README.md`** — deeper manual checks / troubleshooting  

---

## What is in this pack

```text
incident-sentinel-handoff/
├── START-HERE.md          ← you are here
├── STEP-BY-STEP.md        ← deploy guide for a new person / new server
├── REMAINING-STEPS.md     ← unfinished submission work
├── HOW-IT-WORKS.md        ← simple architecture + investigator.py flow
└── project/               ← full source code + docs
    ├── foundry/           ← casting.yaml + casting.yaml.lock (SigNoz install)
    ├── demo/              ← checkout → payment → inventory + break.sh
    ├── copilot/           ← Incident Sentinel agent (FastAPI + MCP + OTel)
    ├── dashboards/        ← dashboard specs
    ├── alerts/            ← alert specs
    ├── scripts/           ← deploy-k8s.sh, refresh-api-key.sh
    ├── blog/              ← blog draft
    ├── docs/              ← detailed notes
    ├── README.md
    └── IMPLEMENT-README.md
```

---

## What this project does (one paragraph)

**Incident Sentinel** watches SigNoz alerts, investigates them using the **SigNoz MCP** tools (traces/logs/metrics), writes a root-cause report (Slack or stdout), and sends **its own** OpenTelemetry traces (LLM steps, tool calls, token/cost) back into the same SigNoz — so the AI SRE helper is itself observable.

Hackathon: [Agents of SigNoz](https://www.wemakedevs.org/hackathons/signoz) — Track 01 primary.

---

## Quick path

| Goal | Open |
|------|------|
| Deploy on another server | `STEP-BY-STEP.md` |
| Understand the code loop | `HOW-IT-WORKS.md` |
| See what’s left for the prize | `REMAINING-STEPS.md` |
| Smoke-test an already-running stack | `project/IMPLEMENT-README.md` Part A |

---

## Unzip and go

```bash
unzip incident-sentinel-handoff.zip
cd incident-sentinel-handoff
# read STEP-BY-STEP.md, then work from project/
cd project
```

---

## AI declaration

This project was built with AI coding assistants (Cursor). Declare that on the hackathon submission form.
