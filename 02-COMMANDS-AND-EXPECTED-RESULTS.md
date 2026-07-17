# Commands and Expected Results

## 1. `sudo ./START-HERE.sh up onprem`

### What it does

1. Re-executes as root while preserving approved installer environment variables.
2. Verifies Linux and systemd.
3. Packages a clean copy of the source.
4. Generates a local OSAI operator token and local service credentials.
5. Stops an older OSAI runtime without deleting named Docker volumes.
6. Installs the source under `/opt/osai/OS.rs`.
7. Runs the Linux bootstrap to install/build OSAI and start PostgreSQL, RustFS, llama.cpp, and Cognee.
8. Verifies that Cognee exposes `POST /api/v1/remember`.
9. Installs Foundry and generates the correct standard or SELinux SigNoz deployment.
10. Starts the OSAI Collector, HostScope, journald bridge, and SigNoz MCP.
11. Runs strict evidence checks.

### What to expect

The first connected run can take substantial time because it may download packages, Docker images, Rust crates, Cognee, llama.cpp source, embedding assets, and a GGUF model.

Successful endpoints:

```text
OSAI UI/API       http://127.0.0.1:8000
Cognee API        http://127.0.0.1:8001
llama.cpp/Qwen    http://127.0.0.1:8088
SigNoz UI         http://127.0.0.1:8080
SigNoz MCP        http://127.0.0.1:18000/mcp
RustFS API        http://127.0.0.1:9000
RustFS console    http://127.0.0.1:9001
```

## 2. `sudo ./START-HERE.sh status`

Shows:

- installation mode;
- systemd state for OSAI, HostScope, and journald bridge;
- OSAI support containers;
- Collector container;
- every SigNoz container, including exited one-shot jobs.

Expected healthy core:

```text
osai-agent.service                         active
hostscope.service                          active
osai-journal-export.service                active
osai-postgres                              healthy
osai-llama                                 healthy
osai-cognee                                healthy
signoz-telemetrykeeper-clickhousekeeper-0  healthy
signoz-telemetrystore-clickhouse-0-0       healthy
signoz-metastore-postgres-0                healthy
signoz-signoz-0                            healthy
signoz-ingester-1                          running
```

The migrator may exit with code `0`; it is a one-time job.

## 3. `sudo ./START-HERE.sh demo`

Creates two types of evidence:

1. Three real authenticated OSAI scan requests, producing native Rust spans.
2. Six deterministic demo rounds, each containing health, authenticated snapshot, authenticated scan, and expected unauthenticated snapshot requests.

It creates these custom **training metrics**:

```text
osai.demo.request.duration
osai.demo.http.error
```

It also creates matching traces and logs with shared trace/span IDs.

## 4. `sudo ./START-HERE.sh evidence`

Performs strict health checks. A PASS means the plumbing is working, not that every application behavior is ideal. For example, repeated Cognee processing or a poorly tuned model can still be an application-level problem.

## 5. `sudo ./START-HERE.sh logs`

Prints recent systemd logs for OSAI, HostScope, and the journal bridge, then the Collector log tail. Use it to identify export errors, application warnings, and service restarts.

## 6. `sudo ./START-HERE.sh token`

Prints the OSAI API operator token from `/var/lib/osai/operator-token`.

This token authenticates OSAI API calls. It is **not** a SigNoz service-account/API key.

## 7. `sudo ./START-HERE.sh down`

Stops services and containers but preserves named Docker volumes and `/var/lib/osai` state. It does not perform a destructive clean reset.

## 8. `sudo ./START-HERE.sh airgap-pack /path/bundle.tar.gz`

Runs on a connected, fully working machine and exports all dependencies for an offline machine of the same CPU architecture.

## 9. `sudo ./START-HERE.sh up airgap /path/bundle.tar.gz`

Installs without downloading packages, images, code, or models. Docker Engine, Compose, systemd, and basic tools must already exist on the target.

## 10. Learning commands

```bash
./START-HERE.sh structure   # print the folder/file learning map
./START-HERE.sh queries     # print the signal and Query Builder guide
sudo ./START-HERE.sh lab    # generate one guided telemetry lab
```
