# OSAI Flight Recorder Folder and File Map

This document explains where every part of the stack lives and what responsibility it owns. The final appendix lists every packaged file, including documentation, lock files, schemas, and test fixtures.

## 1. Top-level structure

```text
OSAI-Flight-Recorder/
├── START-HERE.sh          Local/on-prem/air-gap lifecycle entry point
├── osai                   GCP/OpenTofu deployment operator
├── osai-agent/            Main Rust application and local support stack
├── hostscope/             Independent read-only Linux host collector
├── observability/         SigNoz, Collector, queries, alerts, and evidence
├── infra/                 OpenTofu GCP modules and bootstrap scripts
├── scripts/               Cross-component demos, incidents, and air-gap tools
├── get-osai-os-ready/     Older OS/Rust bootstrap tooling retained for reference
├── docs/                  Learning-edition documentation
├── blog/                  Hackathon/blog draft
└── *.md                   Architecture, security, compliance, and runbooks
```

## 2. Runtime responsibility map

| Component | Responsibility | Main files |
|---|---|---|
| `START-HERE.sh` | Installs and controls the complete local stack | `START-HERE.sh`, `scripts/*` |
| OSAI Rust agent | UI/API, scans, reasoning, history, actions | `osai-agent/src/main.rs`, `src/*.rs` |
| OSAI supervisor | Starts API, storage worker, Cognee worker | `osai-agent/src/bin/osai-all.rs` |
| Storage worker | Writes scans to PostgreSQL and RustFS | `src/bin/osai-storage-worker.rs` |
| Cognee ingest worker | Delivers queued memory to Cognee | `src/bin/osai-cognee-ingest.rs` |
| PostgreSQL | OSAI history plus a separate Cognee database | `docker-compose.storage.yml`, `storage/postgres-init/*` |
| RustFS | Raw scan and Markdown evidence objects | `docker-compose.storage.yml` |
| Cognee | Local long-term memory and graph processing | `docker/cognee/Dockerfile`, `.env.cognee` |
| llama.cpp/Qwen | Local inference endpoint | `docker/llama*`, `models/` |
| HostScope | Read-only host facts and host metrics | `hostscope/crates/*` |
| OSAI Collector | Receives/enriches/forwards telemetry | `observability/collector/*` |
| SigNoz | UI, queries, alerts, MCP, telemetry storage | `observability/casting*.yaml` and generated `pours/` |
| GCP/OpenTofu | Creates a remote VM and bootstraps OSAI | `infra/`, `osai` |

## 3. Request flow

```text
Browser
  → Rust Axum API
      → Linux scanner
      → PostgreSQL
      → RustFS
      → Cognee outbox
      → Cognee/FastAPI
          → llama.cpp/Qwen
```

## 4. Telemetry flow

```text
Rust spans + HostScope metrics + hostmetrics + Docker metrics/logs + journald
  → OSAI Collector :14318
  → SigNoz ingester :4317/:4318
  → ClickHouse
  → SigNoz UI :8080
```

## 5. Important generated or runtime-only paths

These are intentionally absent from the ZIP and appear after installation:

```text
/opt/osai/OS.rs/                         Installed source
/var/lib/osai/operator-token             OSAI API token
/var/lib/osai/signoz.done                Successful observability install marker
/var/log/osai/otel/osai-agent.json       Journald bridge output
observability/pours/                      Foundry-generated SigNoz Compose deployment
osai-agent/.env.storage                   Local storage/runtime configuration
osai-agent/.env.cognee                    Local Cognee/LLM configuration
osai-agent/models/*.gguf                  Local Qwen model
osai-agent/target/                        Compiled Rust binaries
hostscope/target/                         Compiled HostScope artifacts
```

## 6. File catalogue

The catalogue below explains every file in the package. Files under `hostscope/.../tests/fixtures` are synthetic Linux `/proc` and `/sys` data used only by automated tests; they are not runtime configuration.

### Top-level

| File | Purpose |
|---|---|
| `.gitignore` | Prevents local secrets, generated state, build artifacts, and caches from being committed or included in builds. |
| `AI-DISCLOSURE.md` | Records how AI assistance was used in project preparation. |
| `ARCHITECTURE.md` | High-level application and telemetry architecture plus port ownership. |
| `HACKATHON-COMPLIANCE.md` | Pre-event work and eligibility disclosure guidance. |
| `IMPROVEMENTS.md` | Known gaps and future improvements. |
| `INTEGRATION-MANIFEST.md` | Lists preserved OSAI/HostScope functionality and integration additions. |
| `LEARNING-PATH.md` | Recommended reading and hands-on sequence for this annotated edition. |
| `LICENSE` | Packaged project file used by the component represented by its directory. |
| `OFFLINE-AND-ONPREM.md` | Defines connected self-hosting versus strict air-gapped installation. |
| `PACKAGE-MANIFEST.txt` | Packaging inventory used to verify included files. |
| `PRIZE-RUNBOOK.md` | Hackathon demonstration and evidence checklist. |
| `QUICKSTART.md` | Minimal connected and air-gap installation instructions. |
| `README.md` | Primary project overview, supported modes, service URLs, commands, and security notes. |
| `SECURITY.md` | Secrets, loopback exposure, privilege, and telemetry privacy guidance. |
| `START-HERE.sh` | Primary local lifecycle command for on-premises and air-gap operation. |
| `THIRD_PARTY-NOTICES.md` | Third-party software and license acknowledgements. |
| `VALIDATION-REPORT.md` | Static validation status and limitations of the packaging environment. |
| `credentials.env.example` | Safe template for cloud/bootstrap credentials; real values must remain untracked. |
| `docs-OSAI-SigNoz-All-Tracks-Strategy.md` | Complete Track 1 strategy with Track 2 depth and Track 3 bridge design. |
| `osai` | OpenTofu/GCP deployment, observation, tunnel, and destroy operator. |

### .github

| File | Purpose |
|---|---|
| `.github/workflows/ci.yml` | GitHub Actions continuous-integration workflow. |

### blog

| File | Purpose |
|---|---|
| `blog/OSAI-SIGNOZ-EARLY-WIN-DRAFT.md` | Draft article describing the SigNoz installation and learning outcome. |

### get-osai-os-ready

| File | Purpose |
|---|---|
| `get-osai-os-ready/Layers/Cargo.toml` | Rust workspace or crate manifest defining package metadata, dependencies, and build features. |
| `get-osai-os-ready/Layers/src/main.rs` | Legacy OS and Rust preparation tooling retained from the original working OSAI project. |
| `get-osai-os-ready/Layers/startersv.sh` | Legacy OS and Rust preparation tooling retained from the original working OSAI project. |
| `get-osai-os-ready/linux-lets-rust-now.sh` | Legacy OS and Rust preparation tooling retained from the original working OSAI project. |
| `get-osai-os-ready/macos-lets-rust-now.sh.sh` | Legacy OS and Rust preparation tooling retained from the original working OSAI project. |

### hostscope

| File | Purpose |
|---|---|
| `hostscope/.github/workflows/ci.yml` | HostScope workspace metadata, policy, documentation, or build configuration. |
| `hostscope/.gitignore` | Prevents local secrets, generated state, build artifacts, and caches from being committed or included in builds. |
| `hostscope/Cargo.lock` | Generated Rust dependency lock for reproducible builds; normally changed by Cargo, not by hand. |
| `hostscope/Cargo.toml` | Rust workspace or crate manifest defining package metadata, dependencies, and build features. |
| `hostscope/LICENSE` | HostScope workspace metadata, policy, documentation, or build configuration. |
| `hostscope/README.md` | HostScope workspace metadata, policy, documentation, or build configuration. |
| `hostscope/ROADMAP.md` | HostScope workspace metadata, policy, documentation, or build configuration. |
| `hostscope/SPECIFICATION.md` | HostScope workspace metadata, policy, documentation, or build configuration. |
| `hostscope/VALIDATION.md` | HostScope workspace metadata, policy, documentation, or build configuration. |
| `hostscope/config/hostscope.env.example` | HostScope environment or MCP client configuration example. |
| `hostscope/config/mcp-client.example.json` | HostScope environment or MCP client configuration example. |
| `hostscope/crates/hostscope-core/Cargo.toml` | Rust workspace or crate manifest defining package metadata, dependencies, and build features. |
| `hostscope/crates/hostscope-core/src/command.rs` | Allowlisted bounded command runner. |
| `hostscope/crates/hostscope-core/src/error.rs` | Structured probe errors. |
| `hostscope/crates/hostscope-core/src/lib.rs` | Public core library boundary. |
| `hostscope/crates/hostscope-core/src/linux.rs` | Linux `/proc`, `/sys`, filesystem, service, and security collector. |
| `hostscope/crates/hostscope-core/src/policy.rs` | Collection profiles, redaction, timeouts, and limits. |
| `hostscope/crates/hostscope-core/src/schema.rs` | Vendor-neutral snapshot data model. |
| `hostscope/crates/hostscope-core/tests/fixtures.rs` | HostScope workspace metadata, policy, documentation, or build configuration. |
| `hostscope/crates/hostscope-core/tests/fixtures/alma/etc/machine-id` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/alma/etc/os-release` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/alma/proc/1/cgroup` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/alma/proc/1/comm` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/alma/proc/99/stat` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/alma/proc/cpuinfo` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/alma/proc/loadavg` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/alma/proc/meminfo` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/alma/proc/self/status` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/alma/proc/sys/crypto/fips_enabled` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/alma/proc/sys/kernel/hostname` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/alma/proc/sys/kernel/osrelease` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/alma/proc/sys/kernel/ostype` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/alma/proc/uptime` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/alma/sys/class/net/ens4/address` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/alma/sys/class/net/ens4/mtu` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/alma/sys/class/net/ens4/operstate` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/alma/sys/class/net/ens4/speed` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/alma/sys/fs/selinux/enforce` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/alma/sys/kernel/security/lockdown` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/etc/machine-id` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/etc/os-release` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/proc/1/cgroup` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/proc/1/comm` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/proc/123/stat` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/proc/cpuinfo` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/proc/loadavg` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/proc/meminfo` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/proc/self/status` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/proc/sys/crypto/fips_enabled` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/proc/sys/kernel/dmesg_restrict` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/proc/sys/kernel/hostname` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/proc/sys/kernel/kptr_restrict` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/proc/sys/kernel/osrelease` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/proc/sys/kernel/ostype` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/proc/sys/kernel/unprivileged_bpf_disabled` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/proc/uptime` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/sys/class/net/eth0/address` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/sys/class/net/eth0/mtu` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/sys/class/net/eth0/operstate` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/sys/class/net/eth0/speed` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/sys/class/net/eth0/statistics/rx_bytes` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/sys/class/net/eth0/statistics/tx_bytes` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/sys/kernel/security/lockdown` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-core/tests/fixtures/ubuntu/sys/module/apparmor/parameters/enabled` | Synthetic Linux filesystem value used by HostScope parser tests; never used in production. |
| `hostscope/crates/hostscope-otel/Cargo.toml` | Rust workspace or crate manifest defining package metadata, dependencies, and build features. |
| `hostscope/crates/hostscope-otel/src/lib.rs` | Converts HostScope snapshots into OpenTelemetry metrics and exports them through OTLP. |
| `hostscope/crates/hostscope/Cargo.toml` | Rust workspace or crate manifest defining package metadata, dependencies, and build features. |
| `hostscope/crates/hostscope/src/main.rs` | HostScope CLI and watch-mode entry point. |
| `hostscope/crates/hostscope/src/mcp.rs` | Read-only MCP interface over HostScope data. |
| `hostscope/deny.toml` | HostScope workspace metadata, policy, documentation, or build configuration. |
| `hostscope/deploy/otel-collector/collector-signoz.yaml` | HostScope deployment configuration for systemd or a Collector. |
| `hostscope/deploy/systemd/hostscope.service` | systemd unit defining how the named service starts, restarts, and is sandboxed. |
| `hostscope/docs/ARCHITECTURE.md` | HostScope design, schema, security, comparison, or integration documentation. |
| `hostscope/docs/COMPARISON.md` | HostScope design, schema, security, comparison, or integration documentation. |
| `hostscope/docs/OSAI-INTEGRATION.md` | HostScope design, schema, security, comparison, or integration documentation. |
| `hostscope/docs/SCHEMA.md` | HostScope design, schema, security, comparison, or integration documentation. |
| `hostscope/docs/SECURITY.md` | HostScope design, schema, security, comparison, or integration documentation. |
| `hostscope/docs/STANDARDIZATION.md` | HostScope design, schema, security, comparison, or integration documentation. |
| `hostscope/rust-toolchain.toml` | HostScope workspace metadata, policy, documentation, or build configuration. |
| `hostscope/rustfmt.toml` | HostScope workspace metadata, policy, documentation, or build configuration. |
| `hostscope/scripts/install.sh` | HostScope install, smoke-test, or uninstall helper. |
| `hostscope/scripts/smoke-test.sh` | HostScope install, smoke-test, or uninstall helper. |
| `hostscope/scripts/uninstall.sh` | HostScope install, smoke-test, or uninstall helper. |
| `hostscope/spec/hostscope-snapshot-v1alpha1.schema.json` | JSON Schema defining the portable HostScope snapshot contract. |

### infra

| File | Purpose |
|---|---|
| `infra/.gitignore` | Prevents local secrets, generated state, build artifacts, and caches from being committed or included in builds. |
| `infra/README.md` | Project documentation. |
| `infra/environments/dev/.terraform.lock.hcl` | Generated OpenTofu provider selection lock; update through `tofu init -upgrade` deliberately. |
| `infra/environments/dev/main.tf` | Development GCP environment configuration, bootstrap, input, or output file. |
| `infra/environments/dev/outputs.tf` | Development GCP environment configuration, bootstrap, input, or output file. |
| `infra/environments/dev/providers.tf` | Development GCP environment configuration, bootstrap, input, or output file. |
| `infra/environments/dev/scripts/linux-starters.sh` | Development GCP environment configuration, bootstrap, input, or output file. |
| `infra/environments/dev/scripts/macos-starters.sh` | Development GCP environment configuration, bootstrap, input, or output file. |
| `infra/environments/dev/terraform.tfvars.example` | Development GCP environment configuration, bootstrap, input, or output file. |
| `infra/environments/dev/variables.tf` | Development GCP environment configuration, bootstrap, input, or output file. |
| `infra/environments/dev/versions.tf` | Development GCP environment configuration, bootstrap, input, or output file. |
| `infra/modules/compute/main.tf` | Reusable OpenTofu module file for compute resources. |
| `infra/modules/compute/outputs.tf` | Reusable OpenTofu module file for compute resources. |
| `infra/modules/compute/variables.tf` | Reusable OpenTofu module file for compute resources. |
| `infra/modules/iam/main.tf` | Reusable OpenTofu module file for iam resources. |
| `infra/modules/iam/output.tf` | Reusable OpenTofu module file for iam resources. |
| `infra/modules/iam/variable.tf` | Reusable OpenTofu module file for iam resources. |
| `infra/modules/network/main.tf` | Reusable OpenTofu module file for network resources. |
| `infra/modules/network/outputs.tf` | Reusable OpenTofu module file for network resources. |
| `infra/modules/network/variables.tf` | Reusable OpenTofu module file for network resources. |

### observability

| File | Purpose |
|---|---|
| `observability/.gitignore` | Prevents local secrets, generated state, build artifacts, and caches from being committed or included in builds. |
| `observability/README.md` | Project documentation. |
| `observability/alerts/RECOMMENDED-ALERTS.md` | Recommended alert conditions and operational rationale for OSAI telemetry. |
| `observability/casting.selinux.yaml` | SigNoz Foundry source or lock file used to generate the self-hosted deployment. |
| `observability/casting.yaml` | SigNoz Foundry source or lock file used to generate the self-hosted deployment. |
| `observability/casting.yaml.lock` | Foundry resolution lock; regenerated for the target environment during installation. |
| `observability/collector/config.yaml` | OpenTelemetry Collector configuration or Compose runtime for ingesting OSAI-side signals. |
| `observability/collector/docker-compose.yml` | OpenTelemetry Collector configuration or Compose runtime for ingesting OSAI-side signals. |
| `observability/dashboards/README.md` | Dashboard construction notes and panel recipes. |
| `observability/evidence/SCREENSHOT-CHECKLIST.md` | Screenshot and evidence checklist for proving telemetry behavior. |
| `observability/mcp/README.md` | SigNoz MCP configuration and client setup documentation. |
| `observability/mcp/client-config.example.json` | SigNoz MCP configuration and client setup documentation. |
| `observability/queries/QUERY-CATALOG.md` | Copyable SigNoz filters and query recipes. |
| `observability/runbooks/COGNEE-INCIDENT.md` | Incident-response runbook for the named dependency or failure mode. |
| `observability/scripts/generate-demo-telemetry.sh` | Operational script for installing, verifying, generating, or locking the SigNoz observability stack. |
| `observability/scripts/install-signoz.sh` | Operational script for installing, verifying, generating, or locking the SigNoz observability stack. |
| `observability/scripts/refresh-foundry-lock.sh` | Operational script for installing, verifying, generating, or locking the SigNoz observability stack. |
| `observability/scripts/verify.sh` | Operational script for installing, verifying, generating, or locking the SigNoz observability stack. |
| `observability/systemd/osai-journal-export.service` | systemd unit defining how the named service starts, restarts, and is sandboxed. |

### osai-agent

| File | Purpose |
|---|---|
| `osai-agent/.dockerignore` | Prevents local secrets, generated state, build artifacts, and caches from being committed or included in builds. |
| `osai-agent/.env.cognee.example` | Safe configuration template copied to an untracked runtime file. |
| `osai-agent/.env.storage.example` | Safe configuration template copied to an untracked runtime file. |
| `osai-agent/.gitignore` | Prevents local secrets, generated state, build artifacts, and caches from being committed or included in builds. |
| `osai-agent/Cargo.lock` | Generated Rust dependency lock for reproducible builds; normally changed by Cargo, not by hand. |
| `osai-agent/Cargo.toml` | Rust workspace or crate manifest defining package metadata, dependencies, and build features. |
| `osai-agent/LICENSE` | Packaged project file used by the component represented by its directory. |
| `osai-agent/README.md` | Project documentation. |
| `osai-agent/config/osai-agent.toml` | Packaged project file used by the component represented by its directory. |
| `osai-agent/docker-compose.model-image.yml` | Docker Compose definition for the local model image or OSAI support services. |
| `osai-agent/docker-compose.storage.yml` | Docker Compose definition for the local model image or OSAI support services. |
| `osai-agent/docker/cognee/Dockerfile` | Container image build recipe for the component represented by its directory. |
| `osai-agent/docker/llama-model/Dockerfile` | Container image build recipe for the component represented by its directory. |
| `osai-agent/docker/llama-model/Dockerfile.dockerignore` | Container image build recipe for the component represented by its directory. |
| `osai-agent/docker/llama/Dockerfile` | Container image build recipe for the component represented by its directory. |
| `osai-agent/docs/ai-toggle-operator-ui.md` | Detailed design or operator documentation for one OSAI subsystem. |
| `osai-agent/docs/ask-osai-rust-insights.md` | Detailed design or operator documentation for one OSAI subsystem. |
| `osai-agent/docs/clean-slate-runbook.md` | Detailed design or operator documentation for one OSAI subsystem. |
| `osai-agent/docs/cognee-cloud-production.md` | Detailed design or operator documentation for one OSAI subsystem. |
| `osai-agent/docs/cognee-memory-lifecycle.md` | Detailed design or operator documentation for one OSAI subsystem. |
| `osai-agent/docs/full-server-askables.md` | Detailed design or operator documentation for one OSAI subsystem. |
| `osai-agent/docs/inference-reasoning-layer-health.md` | Detailed design or operator documentation for one OSAI subsystem. |
| `osai-agent/docs/intent-planner-factpack-builder.md` | Detailed design or operator documentation for one OSAI subsystem. |
| `osai-agent/docs/llama-qwen-cognee-rust-architecture.md` | Detailed design or operator documentation for one OSAI subsystem. |
| `osai-agent/docs/phase18-ai-memory-production.md` | Detailed design or operator documentation for one OSAI subsystem. |
| `osai-agent/docs/phase3-storage-cognee.md` | Detailed design or operator documentation for one OSAI subsystem. |
| `osai-agent/docs/qwen3-gguf-loading-footprint.md` | Detailed design or operator documentation for one OSAI subsystem. |
| `osai-agent/docs/rust-storage-worker.md` | Detailed design or operator documentation for one OSAI subsystem. |
| `osai-agent/docs/rust-to-cognee-to-qwen-data-flow.md` | Detailed design or operator documentation for one OSAI subsystem. |
| `osai-agent/docs/understanding-model-metadata-like-a-pro.md` | Detailed design or operator documentation for one OSAI subsystem. |
| `osai-agent/knowledge/00_agent_identity.md` | Static operator knowledge/runbook content loaded into OSAI reasoning context. |
| `osai-agent/knowledge/01_server_profile.md` | Static operator knowledge/runbook content loaded into OSAI reasoning context. |
| `osai-agent/knowledge/02_allowed_commands.md` | Static operator knowledge/runbook content loaded into OSAI reasoning context. |
| `osai-agent/knowledge/03_guardrails.md` | Static operator knowledge/runbook content loaded into OSAI reasoning context. |
| `osai-agent/knowledge/04_kubernetes_runbook.md` | Static operator knowledge/runbook content loaded into OSAI reasoning context. |
| `osai-agent/knowledge/05_linux_runbook.md` | Static operator knowledge/runbook content loaded into OSAI reasoning context. |
| `osai-agent/knowledge/06_gitlab_incidents.md` | Static operator knowledge/runbook content loaded into OSAI reasoning context. |
| `osai-agent/knowledge/07_troubleshooting_patterns.md` | Static operator knowledge/runbook content loaded into OSAI reasoning context. |
| `osai-agent/knowledge/08_etc.md` | Static operator knowledge/runbook content loaded into OSAI reasoning context. |
| `osai-agent/knowledge/08_response_format.md` | Static operator knowledge/runbook content loaded into OSAI reasoning context. |
| `osai-agent/knowledge/09_inference_reasoning_guidance.md` | Static operator knowledge/runbook content loaded into OSAI reasoning context. |
| `osai-agent/models/.gitkeep` | Model-directory placeholder or instructions; GGUF binaries are downloaded or bundled separately. |
| `osai-agent/models/README.md` | Model-directory placeholder or instructions; GGUF binaries are downloaded or bundled separately. |
| `osai-agent/packaging/rpm/osai-agent.spec` | RPM or systemd packaging definition for installing OSAI as an OS service. |
| `osai-agent/packaging/systemd/osai-agent.env` | RPM or systemd packaging definition for installing OSAI as an OS service. |
| `osai-agent/packaging/systemd/osai-agent.service` | systemd unit defining how the named service starts, restarts, and is sandboxed. |
| `osai-agent/scripts/ask-local-qwen.sh` | Developer/operator helper for model, memory, storage, or package workflows. |
| `osai-agent/scripts/build-llama-model-image.sh` | Developer/operator helper for model, memory, storage, or package workflows. |
| `osai-agent/scripts/build-rpm.sh` | Developer/operator helper for model, memory, storage, or package workflows. |
| `osai-agent/scripts/ensure-rustfs-bucket.sh` | Developer/operator helper for model, memory, storage, or package workflows. |
| `osai-agent/scripts/run-cognee-ingest.sh` | Developer/operator helper for model, memory, storage, or package workflows. |
| `osai-agent/scripts/run-memory-worker.sh` | Developer/operator helper for model, memory, storage, or package workflows. |
| `osai-agent/scripts/setup-storage-venv.sh` | Developer/operator helper for model, memory, storage, or package workflows. |
| `osai-agent/src/actions.rs` | Guarded command proposal, approval, execution, and audit logic. |
| `osai-agent/src/ask.rs` | Ask OSAI orchestration across facts, memory, and local inference. |
| `osai-agent/src/ask_plan.rs` | Structured plan for answering an operator question. |
| `osai-agent/src/bin/osai-all.rs` | Supervisor binary that runs the API and background workers together. |
| `osai-agent/src/bin/osai-ask.rs` | CLI client for asking the local Qwen-backed OSAI reasoning path. |
| `osai-agent/src/bin/osai-cognee-ingest.rs` | Background worker that sends queued memory records to Cognee. |
| `osai-agent/src/bin/osai-storage-worker.rs` | Periodic worker that persists scans and evidence to PostgreSQL/RustFS. |
| `osai-agent/src/cognee_lifecycle.rs` | Cognee remember/recall/improve/forget lifecycle helpers. |
| `osai-agent/src/collector/mod.rs` | Collector module boundary and public exports. |
| `osai-agent/src/collector/models.rs` | Typed scan and system-data structures. |
| `osai-agent/src/collector/ports.rs` | Network-port parsing and collection helpers. |
| `osai-agent/src/collector/scanner.rs` | Linux host scanner implementation. |
| `osai-agent/src/fact_pack.rs` | Builds bounded factual context for reasoning. |
| `osai-agent/src/history.rs` | Local scan history persistence and retrieval. |
| `osai-agent/src/intent.rs` | Classifies user intent and selects an operation. |
| `osai-agent/src/knowledge.rs` | Loads static Markdown knowledge and runbooks. |
| `osai-agent/src/lib.rs` | Library module exports shared by all OSAI binaries. |
| `osai-agent/src/main.rs` | Axum UI/API entry point and HTTP route wiring. |
| `osai-agent/src/plugins/gitlab.rs` | Optional integration plugin for gitlab. |
| `osai-agent/src/plugins/kubernetes.rs` | Optional integration plugin for kubernetes. |
| `osai-agent/src/plugins/mod.rs` | Optional integration plugin for mod. |
| `osai-agent/src/reasoning.rs` | Combines deterministic rules and optional Qwen reasoning. |
| `osai-agent/src/rules.rs` | Health and severity rule evaluation. |
| `osai-agent/src/telemetry.rs` | Shared Rust OpenTelemetry trace initialization and resources. |
| `osai-agent/storage/migrations/003-rename-minio-columns.sql` | Database migration retained for upgrading an older OSAI schema. |
| `osai-agent/storage/postgres-init/001-create-databases.sh` | One-time PostgreSQL initialization script or OSAI schema definition. |
| `osai-agent/storage/postgres-init/002-osai-schema.sql` | One-time PostgreSQL initialization script or OSAI schema definition. |
| `osai-agent/web/app.css` | Embedded OSAI browser UI asset. |
| `osai-agent/web/app.js` | Embedded OSAI browser UI asset. |
| `osai-agent/web/favicon.svg` | Embedded OSAI browser UI asset. |
| `osai-agent/web/index.html` | Embedded OSAI browser UI asset. |
| `osai-agent/web/osai-3d.js` | Embedded OSAI browser UI asset. |
| `osai-agent/web/vendor/THREE-LICENSE.txt` | Embedded OSAI browser UI asset. |

### scripts

| File | Purpose |
|---|---|
| `scripts/demo-all.sh` | Cross-component operational script for demos, incidents, verification, or offline packaging. |
| `scripts/inject-cognee-failure.sh` | Cross-component operational script for demos, incidents, verification, or offline packaging. |
| `scripts/install-airgap.sh` | Cross-component operational script for demos, incidents, verification, or offline packaging. |
| `scripts/prepare-airgap.sh` | Cross-component operational script for demos, incidents, verification, or offline packaging. |
| `scripts/restore-cognee.sh` | Cross-component operational script for demos, incidents, verification, or offline packaging. |
| `scripts/static-validate.py` | Cross-component operational script for demos, incidents, verification, or offline packaging. |
| `scripts/verify-cognee-api.sh` | Cross-component operational script for demos, incidents, verification, or offline packaging. |

