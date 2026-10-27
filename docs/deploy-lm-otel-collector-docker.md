# Proxmox VE → LogicMonitor with the LM OTel Collector (Docker): start to finish

Purpose: send Proxmox VE (PVE) OpenTelemetry metrics to LogicMonitor (LM) through an LM OTel Collector running in Docker. The collector is the **metrics gateway** and is required today: PVE sends OTLP **JSON** only, LM's metrics endpoint accepts **protobuf** only (a direct push returns `415 Unsupported Media Type`), and the collector converts between them.

```
PVE cluster ──OTLP/HTTP JSON (port 4318, no token)──► LM OTel Collector (Docker) ──OTLP protobuf + Bearer──► LM portal
```

Status: **verified in the lab on 2026-10-27** (Docker Desktop 4.81 on Windows, collector 7.0.00 / OTel 0.154, PVE 9.x two-node cluster): the collector registered as *Running*, PVE's connection test passed, and `proxmox_node_*` series from both nodes arrived in LM Explore. Not yet verified: value accuracy against the PVE UI, behaviour after a reboot or token rotation, Linux/Podman hosts, load at scale. Items marked ⚠ are unconfirmed.

Placeholders: `<portalname>` is your LM portal name (the part before `.logicmonitor.com`). Never put real tokens, keys or the portal name in git, tickets or chat.

## 1. Prerequisites
- PVE 9.x with **Datacenter → Metric Server → Add → OpenTelemetry**. Check with `pveversion`.
- The LM **OpenTelemetry Metrics** feature enabled on the portal (**More → OTel Metrics** is visible) ⚠ feature flag; ask LM if it's missing.
- A Docker engine on a host the PVE nodes can reach. Tested: Docker Desktop on Windows (WSL2). The same commands apply to Docker Engine on Linux, Podman (`podman` for `docker`) and Rancher Desktop.
- Outbound HTTPS (443) from the Docker host to `https://<portalname>.logicmonitor.com`.
- Inbound TCP 4318 from the PVE nodes to the Docker host.
- Root (or `Sys.Modify` on `/`) on the PVE cluster.

## 2. LogicMonitor: role and API-only user
Create a dedicated, least-privilege user rather than reusing an administrator. The install wizard needs a user that can create an API token, and administrator accounts can't have one.

1. **Settings → Users & Roles → Roles → Add**. Name it for example `otel-collector`. Give it only these permissions (per LM documentation; compare with the OTel Collector wizard docs if the wizard rejects it ⚠):
   - **Settings → Data Ingestion**: Traces and OpenTelemetry Collectors, **Manage**.
   - **Settings → Installed Modules**: **Manage**.
   - **Resources**: all, **Manage**.
   - **OTLP Metrics / OpenTelemetry Metrics**: **Manage** (needed to get the bearer token; querying only needs **Read**).
   - **Allow Creation of API Token**: enabled.
2. **Users → Add** an API-only user (for example `otel-collector-svc`) with that role. Give it a password too: the wizard's user picker may need a normal user that can sign in ⚠.
3. Do **not** use an administrator account for the collector. LM documents that administrator roles can't install it.

Observed in the lab (cause unconfirmed): at first the wizard's user list showed only the two administrator accounts, searching for the API-only user returned *No options*, and the wizard won't continue without a user. Later the same portal began listing the new user. If the list is wrong: re-check the role permissions above, make sure the user is active, sign out and back in, wait and refresh, and if it persists open an LM support case (docs say API-only users should be selectable).

## 3. LogicMonitor: add the collector with the wizard
1. **Settings → OpenTelemetry Collectors → Add**.
2. Name it (lab used `ProxmoxOTEL`; no spaces), platform **Docker**, version **7.0.00** (OTel 0.154) or later.
3. **Enable for**: choose *Traces* (there is no metrics-only choice ⚠). The default traces and host-metrics pipelines can stay; they are harmless.
4. **Review Configuration**: replace the editor content with the config in section 3.1, then **Validate**, then **Download Configuration** (saves `lmconfig.yaml`).
5. **Next: Commands**: pick the user from section 2 and add a note. Copy the wizard's Docker command to read the **Access ID**, **Access Key** and version from it. Do not run it as printed; see section 4 for why.

### 3.1 Gateway config (`lmconfig.yaml`)
This is the wizard default plus an `otlphttp/lm` exporter and a `metrics/pve` pipeline. The `otlp` receiver must listen on `0.0.0.0` so the PVE nodes can connect. YAML is strict: every key at one level must start in the same column, and the editor reports *All mapping items must start at the same column* otherwise.

```yaml
receivers:
  hostmetrics:
    collection_interval: 5m
    scrapers:
      cpu:
      disk:
      filesystem:
        include_virtual_filesystems: true
      memory:
      network:
  prometheus:
    config:
      scrape_configs:
        - job_name: "US-West"
          scrape_interval: 5m
          static_configs:
            - targets: ["0.0.0.0:8888"]
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318
      grpc:

processors:
  batch:
    send_batch_size: 4000
    send_batch_max_size: 4000
    timeout: 1000ms
  memory_limiter:
    check_interval: 60s
    limit_mib: 200
  probabilistic_sampler:
    sampling_percentage: 10
  # keep the wizard's default tail_sampling block here unchanged

exporters:
  lmexporter:
    url: https://${LOGICMONITOR_ACCOUNT}.logicmonitor.com/rest
  file:
    path: ./fileout.json
  otlphttp/lm:
    metrics_endpoint: https://${LOGICMONITOR_ACCOUNT}.logicmonitor.com/rest/api/v1/metrics
    headers:
      Authorization: Bearer ${LM_OTLP_BEARER}
    compression: gzip

extensions:
  lmextension:
    properties:
      system.categories: lmotel
  health_check:
    endpoint: 0.0.0.0:13133

service:
  extensions: [lmextension, health_check]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [probabilistic_sampler, batch]
      exporters: [lmexporter]
    metrics:
      receivers: [prometheus, hostmetrics]
      exporters: [lmexporter]
    metrics/pve:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlphttp/lm]
```
Notes:
- Use `metrics_endpoint` (the full URL). The plain `endpoint` setting appends `/v1/metrics` again and breaks the path.
- The bearer token is read from the environment variable `LM_OTLP_BEARER`; it is never written in this file.
- Binding to `0.0.0.0` exposes an unauthenticated OTLP port. Restrict it with a firewall (section 5).
- Port 4317 (gRPC) is not used by PVE. In this config the collector listens for gRPC on localhost only, so publishing 4317 is optional.

## 4. Docker host: build and run the container
Why not the wizard's command as printed: it has no path in `-v lmconfig.yaml:/lmconfig.yaml`, so Docker treats `lmconfig.yaml` as a **named volume**, not your file; `LM_OTLP_BEARER` isn't set; the health port 13133 isn't published; and secrets sit in shell history. Use an env file and a full path instead.

### 4.1 Get the OTLP bearer token
1. LM → **More → OTel Metrics → Configure → View connection details** (*Connect an OTel Metrics Source*).
2. Note the **OTLP Ingestion Endpoint** (form `https://<portalname>.logicmonitor.com/rest/api/v1/metrics`).
3. Choose the API-only user and copy the **Bearer Token** into a password manager. Generating a new token invalidates the old one everywhere.

### 4.2 Working folder and env file (Windows PowerShell)
Keep the folder outside any git repository.
```powershell
mkdir C:\Users\<you>\lmotel
copy C:\Users\<you>\Downloads\lmconfig.yaml C:\Users\<you>\lmotel\lmconfig.yaml
notepad C:\Users\<you>\lmotel\lm.env
```
Contents of `lm.env` (no quotes, no spaces around `=`; the account is the portal name only):
```
LOGICMONITOR_ACCOUNT=<portalname>
LOGICMONITOR_ACCESS_ID=<from the wizard command>
LOGICMONITOR_ACCESS_KEY=<from the wizard command>
LOGICMONITOR_OTEL_NAME=ProxmoxOTEL
LOGICMONITOR_OTEL_VERSION=7000
LM_OTLP_BEARER=<bearer token from 4.1>
```
Check the file without printing secrets:
```powershell
(Get-Content C:\Users\<you>\lmotel\lm.env) -replace '=.*','='
```
Make sure Notepad didn't save it as `lm.env.txt`.

### 4.3 Run
**Windows (PowerShell, one line)**
```powershell
docker run -d --name lmotel --restart unless-stopped --env-file C:\Users\<you>\lmotel\lm.env -p 4317:4317 -p 4318:4318 -p 13133:13133 -v C:\Users\<you>\lmotel\lmconfig.yaml:/lmconfig.yaml logicmonitor/lmotel:latest --config lmconfig.yaml
```
**Linux / macOS (bash)**
```bash
docker run -d --name lmotel --restart unless-stopped \
  --env-file ./lm.env \
  -p 4317:4317 -p 4318:4318 -p 13133:13133 \
  -v "$PWD/lmconfig.yaml:/lmconfig.yaml" \
  logicmonitor/lmotel:latest --config lmconfig.yaml
```
**Docker Compose (any flavor)**
```yaml
services:
  lmotel:
    image: logicmonitor/lmotel:latest
    container_name: lmotel
    restart: unless-stopped
    env_file: ./lm.env
    ports: ["4317:4317", "4318:4318", "13133:13133"]
    volumes: ["./lmconfig.yaml:/lmconfig.yaml:ro"]
    command: ["--config", "lmconfig.yaml"]
```

### 4.4 Check the container
```powershell
docker ps
docker logs lmotel --tail 40
curl.exe -s http://localhost:13133/
```
Expect `Up`, the log line `Everything is ready. Begin running and processing data.`, and `{"status":"Server available",...}` from the health check. The first lines (`Download Started`) are normal. A stack trace saying `error shutting down fluentbit ... Fluent Bit process is not running` is harmless: log collection isn't enabled.

Then in LM → **Settings → OpenTelemetry Collectors**, the collector (`ProxmoxOTEL`) should show **Running**, version 7.0.00, format *container*.

## 5. Network: reach the gateway from the PVE nodes
1. Find the Docker host's LAN IP. On Windows use the physical adapter, not the Hyper-V/WSL virtual ones:
   ```powershell
   Get-NetIPAddress -AddressFamily IPv4 | Where-Object {$_.IPAddress -notlike '169.*' -and $_.IPAddress -ne '127.0.0.1'} | Select-Object InterfaceAlias, IPAddress
   ```
2. Allow TCP 4318 from the PVE network only (elevated PowerShell; replace the subnet):
   ```powershell
   New-NetFirewallRule -DisplayName "LM OTel gateway 4318 (PVE)" -Direction Inbound -Protocol TCP -LocalPort 4318 -RemoteAddress <pve-subnet>/24 -Action Allow -Profile Any
   ```
   Restrict 13133 the same way if other hosts need the health check.
3. From a PVE node, test the path. PVE's own connection test sends an empty payload like this:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' -X POST -H 'Content-Type: application/json' -d '{}' http://<docker-host-ip>:4318/v1/metrics
   ```
   `200` means the gateway accepts it.

## 6. Proxmox VE: create the metric server
The definition is cluster-wide; one entry covers all nodes in the cluster. In the GUI: **Datacenter → Metric Server → Add → OpenTelemetry**.

| Field | Value |
|---|---|
| Name | `lm-otlp` (no spaces; the name is the entry ID, and a space gives error 501) |
| Server | `<docker-host-ip>` (hostname or IP only; `http://…` gives *invalid address*, 400) |
| Port | `4318` |
| Protocol | `HTTP` |
| Path | `/v1/metrics` |
| Verify SSL | off (plain HTTP to the gateway) |
| HTTP Headers (JSON) | leave empty (the gateway holds the token) |
| Resource Attributes (JSON) | `{"cluster":"lab"}` (unique per cluster; shows up as the `cluster` label in LM) |

The JSON fields (under *Advanced JSON Configuration*) take plain, flat JSON typed as is; no shell commands and no base64. Equivalent CLI (the CLI wants the JSON value **base64-encoded**, max 1024 characters) ⚠ not run in the lab:
```bash
pvesh create /cluster/metrics/server/lm-otlp --type opentelemetry \
  --server <docker-host-ip> --port 4318 --otel-protocol http --otel-path /v1/metrics \
  --otel-compression gzip \
  --otel-resource-attributes "$(printf '%s' '{"cluster":"lab"}' | base64 -w0)"
```
Click **Create**. PVE runs a connection test (an empty POST); success means the gateway answered.

## 7. Verify
1. On a PVE node: `journalctl -u pvestatd -n 20 --no-pager`. No new OTel or HTTP errors is the good sign.
2. `docker logs lmotel --tail 20` shows no export errors (successful exports aren't logged at the default level).
3. In LM → **Explore** (OTel Metrics source, Raw Query), last 15 minutes:
   ```
   {__name__=~"proxmox_.*"}
   ```
   Lab result: `proxmox_node_*` series for both nodes with `cluster=lab`, `node=<name>`, `proxmox_cluster`, and `service_version` labels. Data appeared within a few minutes of creating the metric server.
4. Spot-check CPU, memory and storage values against the PVE UI ⚠ (not yet done; CPU fields may be 0–1 ratios).

## 8. Operate
- **Rotate the bearer token:** generate a new token in LM, edit `LM_OTLP_BEARER` in `lm.env`, then recreate the container (`docker rm -f lmotel`, then run section 4.3 again). A plain `docker restart` does not re-read the env file.
- **Change the config:** edit `lmconfig.yaml`, then `docker restart lmotel`.
- **Pause metrics:** `pvesh set /cluster/metrics/server/lm-otlp --disable 1`; remove with `pvesh delete /cluster/metrics/server/lm-otlp`.
- **Stop the gateway:** `docker stop lmotel` (PVE then logs failed pushes).
- **Add another cluster:** add another metric server on that cluster pointing at the same gateway, with a different `cluster` value.

## 9. Troubleshooting (errors met in the lab)
- **415 Unsupported Media Type** when PVE points straight at LM: expected; PVE sends JSON, LM wants protobuf. Point PVE at the gateway.
- **501 `POST /cluster/metrics/server//…` not implemented:** the Name had a space or was empty.
- **400 `server: invalid format`:** the Server field contained `https://` or a path; use host or IP only.
- **PVE "Invalid JSON format" in the headers/attributes boxes:** shell commands or base64 were pasted; the GUI wants plain JSON.
- **Wizard editor: *All mapping items must start at the same column*:** mixed indentation; fix the spaces so sibling keys line up.
- **Wizard user list shows only administrators / *No options*:** see section 2.
- **Container starts but LM data never arrives:** `LM_OTLP_BEARER` missing from `lm.env`, token regenerated, `metrics/pve` pipeline missing, or the OTel Metrics feature flag isn't enabled.
- **401/403 in `docker logs`:** wrong or rotated bearer token, or the user lacks OTLP Metrics permission.
- **Connection refused / timeout from PVE:** wrong host IP (use the LAN IP, not the WSL address), firewall rule missing, or the container isn't published on 4318.
- **Mount looks empty or the config isn't used:** the `-v` source had no full path (named volume) or the env file is saved as `lm.env.txt`.
- **Payloads rejected as too large:** lower PVE `otel-max-body-size` or tune the `batch` size.

## 10. Platform notes and scale
- **Docker Desktop on Windows:** published ports bind on the Windows host; keep Docker Desktop set to start at sign-in or the gateway stops with the host.
- **Podman / rootless:** `:Z` may be needed on SELinux hosts for the config mount.
- **Production:** run on a Linux host or VM, pin the image tag instead of `latest` (and `LOGICMONITOR_OTEL_VERSION`), run two instances behind a load balancer, restrict 4318 by firewall, and use one gateway per site.

## Security reminders
- The bearer token lives only in `lm.env` on the Docker host. PVE sends no header, so `/etc/pve/status.cfg` holds no secret.
- `lm.env` must be git-ignored and never pasted into chat. If a token is exposed, rotate it.
- The OTLP port is unauthenticated plain HTTP: restrict it to the PVE nodes.

## References
- [Installation from LogicMonitor Wizard](https://www.logicmonitor.com/support/opentelemetry-collector-installation-from-logicmonitor-wizard)
- [Container installation parameters](https://www.logicmonitor.com/support/configurations-for-opentelemetry-collector-container-installation)
- [Collector components and versions](https://www.logicmonitor.com/support/opentelemetry-collector-components)
- [OpenTelemetry Metrics Overview](https://www.logicmonitor.com/support/opentelemetry-metrics-overview)
- [Configure OpenTelemetry Metrics Source Connection](https://www.logicmonitor.com/support/configure-opentelemetry-metrics-source-connection)
- [Proxmox VE API viewer: /cluster/metrics/server](https://pve.proxmox.com/pve-docs/api-viewer/)
- [Proxmox External Metric Server](https://pve.proxmox.com/wiki/External_Metric_Server)
