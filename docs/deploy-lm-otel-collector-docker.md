# Deploy the LogicMonitor OTel Collector in Docker

Purpose: run the LM OTel Collector as an optional **metrics gateway** between Proxmox VE and LogicMonitor (LM). Skip this runbook if PVE pushes directly to LM (see `enable-pve-otel-metrics.md`).

Status: written from LogicMonitor documentation; **not yet tested in the lab**. Items marked ⚠ are unverified and are checked in test-plan Phase 2b.

## 1. Prerequisites
- A Docker engine on a host the PVE nodes can reach over the network. Tested targets (to confirm): Docker Desktop on Windows (WSL2 backend); the same commands apply to Docker Engine on Linux, Podman (`podman` instead of `docker`), and Rancher Desktop.
- LM portal access with permission to add OpenTelemetry Collectors.
- Outbound HTTPS (443) from the Docker host to `https://<portalname>.logicmonitor.com`.
- Inbound TCP **4318** (OTLP/HTTP) from the PVE nodes to the Docker host.
- An **API-only LM user with minimum privileges** for the collector (the wizard creates an Access ID/Key for it).

## 2. Generate the install command in LM
1. LM portal → **Settings → OpenTelemetry Collectors → Add**.
2. Name the collector (for example `pve-gateway-lab`), choose platform **Docker**, and pick a version (7.0.00 / OTel 0.154 or later recommended; it includes the `otlp` receiver, `batch`, `resource`, `attributes`, `transform`, `filter` and `otlphttp`).
3. **Enable for**: choose *Traces* (the wizard has no metrics-only option ⚠; the traces pipeline is harmless and can be removed in step 4).
4. **Review Configuration**: edit the config per section 4, then **Validate**. Optionally **Download Configuration** (saves `lmconfig.yaml`).
5. **Next: Commands**: select the API-only user, add a note, and copy the **Run** command. The download link in it expires after two hours.

## 3. Environment variables
Required by the container: `LOGICMONITOR_ACCOUNT` (portal name), `LOGICMONITOR_ACCESS_ID`, `LOGICMONITOR_ACCESS_KEY`, `LOGICMONITOR_OTEL_NAME`. Optional: `LOGICMONITOR_OTEL_VERSION` (pin a version), `CONTAINER_ENV=docker`.

Keep the Access ID/Key and the OTLP bearer token out of shell history, compose files in git, and chat. Use an `.env` file (git-ignored in this repo).

## 4. Gateway configuration (`lmconfig.yaml`) ⚠
The wizard's default config exports traces with `lmexporter` and binds the `otlp` receiver to `localhost`. For the gateway, the receiver must listen on all container interfaces, and metrics must go to LM's OTLP metrics endpoint with the bearer token. Starting point, to validate in Phase 2b:

```yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318

processors:
  memory_limiter:
    check_interval: 5s
    limit_mib: 400
  resource:
    attributes:
      - key: cluster
        value: lab            # set per cluster/site
        action: upsert
  batch:
    timeout: 10s

exporters:
  otlphttp/lm:
    endpoint: ${env:LM_OTLP_ENDPOINT}     # from More → OTel Metrics (see enable-pve-otel-metrics.md)
    headers:
      Authorization: Bearer ${env:LM_OTLP_BEARER}
    compression: gzip

extensions:
  health_check:
    endpoint: 0.0.0.0:13133

service:
  extensions: [health_check]
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, resource, batch]
      exporters: [otlphttp/lm]
```
Notes:
- Use the exporter block that LM generates under **More → OTel Metrics → Configure → Configure OTel collector** if it differs from the above.
- Binding to `0.0.0.0` exposes an unauthenticated OTLP port and the health-check port: restrict 4318 to the PVE nodes, and 13133 to the hosts that run health checks, with firewall rules.
- If PVE metrics already carry `cluster` via `otel-resource-attributes`, remove the `resource` processor.
- Also keep the wizard's own `lmextension`/`lmexporter` entries if Validate requires them ⚠.

## 5. Run it
Save the wizard's command, adding the port, config mount and extra variables. Use the quoting for your shell.

The config is mounted at `/lmconfig.yaml` and passed by that absolute path, so it is found whatever the image's working directory is. Compare the image name, tag and arguments with the wizard's own Run command and prefer the wizard's where they differ ⚠.

**Docker Desktop on Windows (PowerShell)**
```powershell
docker run -d --name lmotel --restart unless-stopped `
  --env-file .\lm.env `
  -p 4318:4318 -p 13133:13133 `
  -v "${PWD}\lmconfig.yaml:/lmconfig.yaml" `
  logicmonitor/lmotel --config /lmconfig.yaml
```
`lm.env` (git-ignored) holds the four required variables plus `LM_OTLP_ENDPOINT` and `LM_OTLP_BEARER`, one `NAME=value` per line, no quotes.

**Linux / macOS (bash)**
```bash
docker run -d --name lmotel --restart unless-stopped \
  --env-file ./lm.env \
  -p 4318:4318 -p 13133:13133 \
  -v "$PWD/lmconfig.yaml:/lmconfig.yaml" \
  logicmonitor/lmotel --config /lmconfig.yaml
```

**Docker Compose (any flavor)**
```yaml
services:
  lmotel:
    image: logicmonitor/lmotel
    container_name: lmotel
    restart: unless-stopped
    env_file: ./lm.env
    ports: ["4318:4318", "13133:13133"]
    volumes: ["./lmconfig.yaml:/lmconfig.yaml:ro"]
    command: ["--config", "/lmconfig.yaml"]
```

## 6. Platform notes
- **Docker Desktop on Windows:** published ports bind on the Windows host; PVE nodes use the **host's LAN IP**, not the WSL/container IP. Add inbound Windows Defender Firewall rules: TCP 4318 limited to the PVE subnet, TCP 13133 limited to your monitoring/admin hosts. Mount the config from a path under your user profile (shared with Docker Desktop). Keep Docker Desktop set to start at sign-in, or the gateway stops with the host.
- **Podman / rootless:** ports below 1024 aren't used here; `:Z` may be needed on SELinux hosts for the config mount.
- **Production:** run on a Linux host or VM, pin the image tag (`LOGICMONITOR_OTEL_VERSION`), and run two instances behind a load balancer for HA.

## 7. Verify
1. LM → **Settings → OpenTelemetry Collectors**: the collector appears (name from `LOGICMONITOR_OTEL_NAME`).
2. `docker logs lmotel` shows no export errors.
3. `curl http://<docker-host>:13133/` returns healthy (health_check).
4. Point PVE at `http://<docker-host>:4318/v1/metrics` (see the PVE runbook) and query `{__name__=~"proxmox_.*"}` in LM Explore.

## 8. Troubleshooting
- 401/403 from LM: wrong or regenerated bearer token (regenerating invalidates every sender).
- Connection refused from PVE: container published on loopback only, firewall, or wrong host IP.
- Nothing in LM but no errors: pipeline missing `metrics`, or the OTel Metrics feature flag isn't enabled.
- Large payloads rejected: lower PVE `otel-max-body-size` or tune `batch` size.

## References
- [Installation from LogicMonitor Wizard](https://www.logicmonitor.com/support/opentelemetry-collector-installation-from-logicmonitor-wizard)
- [Container installation parameters](https://www.logicmonitor.com/support/configurations-for-opentelemetry-collector-container-installation)
- [Collector components and versions](https://www.logicmonitor.com/support/opentelemetry-collector-components)
- [Metrics via an existing collector](https://www.logicmonitor.com/support/opentelemetry-metrics-configuration-using-existing-opentelemetry-collector-from-contrib)
