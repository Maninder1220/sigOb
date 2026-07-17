# Safe Deletion Matrix

“Can be deleted” depends on the guarantee you want to preserve. A running Linux process can continue after its source file is deleted, but the next restart, upgrade, repair, or disaster recovery may fail. This guide separates those cases.

## 1. Guarantees

| Level | Meaning |
|---|---|
| A — Full project | Install, restart, rebuild, upgrade, air-gap pack, GCP deploy, demo, and troubleshooting still work |
| B — Local operations | Existing machine can restart services and containers, but cannot clean-build every component |
| C — Current processes only | Currently running processes may continue; restart/recreate is not guaranteed |

Use Level A unless disk pressure requires a deliberately minimized runtime.

## 2. Top-level folders

| Path | Level A | Level B | Level C | Explanation |
|---|---|---|---|---|
| `osai-agent/` | Keep | Keep runtime files | Can remove source only | Core Rust source, web UI, compose, model mount, DB initialization, binaries, active env files |
| `observability/` | Keep | Keep | Do not remove | Collector config, SigNoz generated deployment, scripts, state, dashboards, units |
| `hostscope/` | Keep | Source may be removed after install | Source may be removed | `/usr/local/bin/hostscope` and installed units continue, but rebuild/upgrade is lost |
| `infra/` | Keep | May remove if never reinstalling or using GCP | May remove | `START-HERE.sh up onprem` calls `infra/environments/dev/scripts/linux-starters.sh` |
| `scripts/` | Keep | Keep operational scripts | Some may be removed | Demo, repair, air-gap, policy, and incident workflows depend on these scripts |
| `get-osai-os-ready/` | Keep | May remove after successful binary build | May remove | Bootstrap/build helper referenced by the Linux installer; fallback Cargo build may still work |
| `docs/` | Optional | Safe to remove | Safe to remove | Learning material only; `structure` and `queries` commands lose documentation |
| `blog/` | Optional | Safe to remove | Safe to remove | Submission/article content only |
| `.github/` | Optional | Safe to remove | Safe to remove | CI only |

## 3. OSAI agent subtree

| Path | Delete? | Effect |
|---|---|---|
| `osai-agent/target/debug/` | Yes | Debug build cache only |
| `osai-agent/target/release/build`, `.fingerprint`, `deps`, `incremental` | Yes after binaries are copied/kept | Rebuild becomes slower; keep top-level release binaries |
| `osai-agent/target/release/osai-*` | No | Native systemd supervisor and workers depend on these binaries |
| `osai-agent/src/` | Only for an immutable runtime | No rebuild, patch, source audit, or policy script |
| `osai-agent/Cargo.toml`, `Cargo.lock` | Only for an immutable runtime | No reproducible Rust rebuild |
| `osai-agent/web/` | No in source/embedded rebuild mode | UI is embedded/served by the Rust application build; deleting after build may not affect the current binary but breaks rebuild |
| `osai-agent/knowledge/` | No | Runtime knowledge directory and installer/package modes depend on it |
| `osai-agent/models/*.gguf` | No when using `docker-compose.storage.yml` | llama.cpp bind-mounts the model from this directory |
| `osai-agent/models/*.gguf` | Yes only when the model is baked into the active image | Confirm active Compose mode and image first |
| `osai-agent/.env.storage` | No | OSAI token, PostgreSQL, RustFS, and runtime settings |
| `osai-agent/.env.cognee` | No | Cognee/Qwen/memory settings |
| `osai-agent/.env.*.example` | Yes after installation | Templates only; reinstall/repair becomes harder |
| `osai-agent/docker-compose.storage.yml` | No | Needed to recreate, stop, inspect, and air-gap the OSAI containers |
| `osai-agent/docker-compose.model-image.yml` | Yes if never using baked-model mode | Optional alternate deployment mode |
| `osai-agent/docker/` | Only after images are built and permanently retained | Images cannot be rebuilt |
| `osai-agent/storage/postgres-init/` | No for clean DB creation | Existing volume continues, but a fresh PostgreSQL volume will miss schema/databases |
| `osai-agent/docs/` | Yes | Documentation only |
| `osai-agent/packaging/` | Yes if not building RPM/systemd packages | Packaging examples only |
| `osai-agent/scripts/` | Keep | Bucket initialization, model image, storage/memory helper workflows |

## 4. Observability subtree

| Path | Delete? | Effect |
|---|---|---|
| `observability/collector/config.yaml` | No | Collection agent cannot be recreated |
| `observability/collector/docker-compose.yml` | No | Collection agent lifecycle commands fail |
| `observability/collector/state/` | Not while preserving log offsets | Deleting replays or skips logs depending on receiver start settings |
| `observability/pours/` | No on an installed system | Contains the generated SigNoz Compose deployment and backend configs |
| `observability/casting*.yaml` | Keep for regeneration | Foundry cannot regenerate SigNoz without them |
| `observability/casting*.yaml.lock` | Keep | Reproducibility and air-gap generation depend on resolved lock data |
| `observability/systemd/` | Keep in source; installed copies exist in `/etc/systemd/system` | Reinstall/repair source lost |
| `observability/scripts/` | Keep | Install, verify, demo, and HostScope context export workflows |
| `observability/evidence/`, `dashboards/`, `queries/`, `runbooks/` | Optional | Documentation/evidence only |

## 5. HostScope subtree

| Path | Delete? | Effect |
|---|---|---|
| `hostscope/target/` | Yes after `/usr/local/bin/hostscope` is installed | Build cache only |
| `hostscope/crates/` | Only after installation | No rebuild or source-level changes |
| `hostscope/spec/` | Optional for runtime | Removes JSON Schema source |
| `hostscope/docs/`, `README.md`, `ROADMAP.md`, `VALIDATION.md` | Yes | Documentation only |
| `hostscope/deploy/systemd/` | Keep for repair | Installed unit exists in `/etc/systemd/system`, but source reinstall is lost |
| `hostscope/config/` | Templates only | Active configuration is `/etc/hostscope/hostscope.env` |
| `hostscope/Cargo.toml`, `Cargo.lock`, `rust-toolchain.toml` | Keep for rebuild | Reproducible build metadata |
| `hostscope/**/tests/fixtures/` | Yes for a production-only package | Tests no longer run |

## 6. Runtime data outside the repository

Never delete these without a backup and an explicit reset decision:

```text
/etc/hostscope/hostscope.env
/etc/systemd/system/osai-agent.service
/etc/systemd/system/hostscope.service
/etc/systemd/system/hostscope-snapshot-export.service
/etc/systemd/system/hostscope-snapshot-export.timer
/etc/systemd/system/osai-journal-export.service
/var/lib/osai/
/var/lib/hostscope/
/var/log/osai/otel/
```

Docker volumes are the actual durable data:

```bash
docker volume ls
```

Typical critical volumes include:

```text
osai-postgres-data
osai-rustfs-data
osai-cognee-data
SigNoz ClickHouse data
SigNoz Keeper data
SigNoz metastore PostgreSQL data
```

`docker compose down` preserves named volumes. `docker compose down -v`, `docker volume rm`, and `docker system prune --volumes` can destroy persistent data.

## 7. Conservative cleanup commands

### Rust build caches

The OSAI systemd unit normally executes `osai-agent/target/release/osai-all`, so **do not run `cargo clean` in `osai-agent/`** unless you first copy every required release binary elsewhere and update the unit. Safe immediate cleanup is limited to the debug tree:

```bash
rm -rf /opt/osai/OS.rs/osai-agent/target/debug
```

HostScope is installed as `/usr/local/bin/hostscope`, so its Cargo build tree can be removed after confirming that installed binary and service:

```bash
systemctl cat hostscope.service
test -x /usr/local/bin/hostscope
rm -rf /opt/osai/OS.rs/hostscope/target
```

Keep the OSAI release binaries `osai-all`, `osai-agent`, `osai-storage-worker`, `osai-cognee-ingest`, and optional `osai-ask` wherever the active service or air-gap workflow expects them.

### Package caches and unused Docker layers

```bash
docker image prune
sudo journalctl --vacuum-time=14d
```

Avoid `docker system prune -a --volumes` on this project.

### Documentation-only minimized copy

Create a separate runtime copy rather than deleting from the canonical repository:

```bash
rsync -a /opt/osai/OS.rs/ /opt/osai/OS.rs-runtime/ \
  --exclude '.github/' \
  --exclude 'blog/' \
  --exclude 'docs/' \
  --exclude 'hostscope/**/tests/' \
  --exclude 'osai-agent/docs/'
```

Test the copy before replacing anything:

```bash
sudo /opt/osai/OS.rs/START-HERE.sh status
sudo /opt/osai/OS.rs/START-HERE.sh evidence
```

## 8. Before deleting a model

```bash
docker inspect osai-llama --format '{{json .Mounts}}' | jq .
docker inspect osai-llama --format '{{.Config.Image}}'
```

- A mount from `/opt/osai/OS.rs/osai-agent/models` means the GGUF must stay.
- No model mount and an image such as `osai-llama-qwen-with-model:*` means the model may be baked into the image.

## 9. Before deleting source

Confirm installed binaries and service commands:

```bash
systemctl cat osai-agent.service
systemctl cat hostscope.service
readlink -f /usr/local/bin/hostscope
ls -lh /opt/osai/OS.rs/osai-agent/target/release/osai-*
```

Then create a source archive:

```bash
sudo tar -C /opt/osai -czf /root/osai-source-backup-$(date +%F).tar.gz OS.rs
```
