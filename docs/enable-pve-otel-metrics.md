# Enable OpenTelemetry metrics in Proxmox VE → LogicMonitor

Purpose: turn on PVE's OpenTelemetry metric server, find LogicMonitor's OTLP endpoint, and create the bearer token.

Status: written from LogicMonitor docs and the official PVE API schema; **not yet tested in the lab**. Items marked ⚠ are verified in test-plan Phase 0/2.

## 1. Prerequisites
- Proxmox VE 9.x with the OpenTelemetry metric server type (Datacenter → Metric Server → Add shows "OpenTelemetry"). Check: `pveversion`.
- LM portal with the **OpenTelemetry Metrics** feature enabled (feature flag) ⚠.
- An LM user with **Manage** permission for OpenTelemetry Metrics (Settings role permissions). Query users need **Read**.
- Network path: PVE nodes → LM (`https://<portalname>.logicmonitor.com`, 443) or → your gateway (see `deploy-lm-otel-collector-docker.md`).

## 2. Find the OTLP endpoint and create the bearer token (LM portal)
1. Sign in to LM and go to **More → OTel Metrics**.
2. Select **Configure → View connection details** (title: *Connect an OTel Metrics Source*).
3. Copy the **OTLP Ingestion Endpoint**. Record its scheme, host, port and path separately (section 4).
4. In **User**, choose the LM user that authenticates ingestion (the list depends on your permissions). Use a dedicated API-only user with minimum rights.
5. Copy the **Bearer Token**. Treat it like a password: anyone holding it can send telemetry to your portal. Store it in a password manager, not in chat, tickets or git.
6. Protocol to use: **OTLP HTTP/protobuf**. Header: `Authorization: Bearer <token>`.

Rotation: generating a new token invalidates the old one; update every sender (all PVE clusters and gateways) afterward.

## 3. Build the `otel-headers` value
PVE takes headers as **JSON, base64-encoded, max 1024 characters** (official API schema).

Linux/macOS (or a PVE node):
```bash
printf '%s' '{"Authorization":"Bearer <TOKEN>"}' | base64 -w0     # macOS: base64 without -w0
```
Windows PowerShell:
```powershell
[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes('{"Authorization":"Bearer <TOKEN>"}'))
```
Check the result is ≤ 1024 characters ⚠ (a very long token may not fit; if so, use a gateway).

Optional resource attributes (also base64 JSON, ≤ 1024 chars), for example to label the cluster:
```bash
printf '%s' '{"cluster":"lab"}' | base64 -w0
```

## 4. Split the endpoint into PVE fields
Example (placeholder): `https://<host>:<port>/<path>`

| PVE field | Value |
|---|---|
| `--server` | host (DNS name or IP) |
| `--port` | port (443 if none shown) |
| `--otel-protocol` | `https` (or `http` for an internal gateway) |
| `--otel-path` | path (default `/v1/metrics`; use exactly what LM shows) ⚠ |
| `--otel-compression` | `gzip` |

For a gateway: `--server <docker-host-ip> --port 4318 --otel-protocol http --otel-path /v1/metrics`, and omit the bearer header (the gateway holds the token).

## 5. Add the metric server in Proxmox VE
The definition is cluster-wide: one entry covers all nodes.

**GUI**
1. Datacenter → **Metric Server → Add → OpenTelemetry**.
2. Fill **Name** (for example `lm-otlp`), **Server**, **Port**, **Protocol**, **Path**, and enable **Verify SSL** for LM.
3. Add the headers and resource attributes if the form offers them; otherwise use the CLI. ⚠
4. Click **Create**.

**CLI (any node)**
```bash
pvesh create /cluster/metrics/server/lm-otlp \
  --type opentelemetry \
  --server <host> --port 443 \
  --otel-protocol https --otel-path <path> \
  --otel-compression gzip \
  --otel-verify-ssl 1 \
  --otel-headers '<base64 from section 3>' \
  --otel-resource-attributes '<base64 from section 3>'
```
Defaults if omitted: timeout 5 s (1–10), max body 10,000,000 bytes, gzip, path `/v1/metrics`. The CLI accepts the base64 string as-is ⚠ (the community reports plain-JSON forms; confirm in Phase 2).

The config is stored in `/etc/pve/status.cfg`; the base64 headers (including the bearer token) are readable by cluster admins.

## 6. Verify
1. On a node: `journalctl -u pvestatd -n 50 --no-pager` — look for HTTP errors (401/403 = token, 404 = path, TLS errors = verify-ssl/CA).
2. In LM: **Explore**, query `{__name__=~"proxmox_.*"}` (PromQL-compatible). Expect `proxmox_node_*`, `proxmox_vm_*`, `proxmox_storage_*` with `node`, `vmid`, `type` labels. Metric names and push interval are community-reported ⚠.
3. Spot-check a few values against the PVE UI (CPU fields may be 0–1 ratios).

## 7. Change, pause, remove
```bash
pvesh get    /cluster/metrics/server                     # list
pvesh set    /cluster/metrics/server/lm-otlp --disable 1 # pause
pvesh set    /cluster/metrics/server/lm-otlp --otel-headers '<new base64>'   # rotate token
pvesh delete /cluster/metrics/server/lm-otlp             # remove
```

## References
- [OpenTelemetry Metrics Overview](https://www.logicmonitor.com/support/opentelemetry-metrics-overview)
- [Configure OpenTelemetry Metrics Source Connection](https://www.logicmonitor.com/support/configure-opentelemetry-metrics-source-connection)
- [Proxmox VE API viewer: /cluster/metrics/server](https://pve.proxmox.com/pve-docs/api-viewer/)
- [Proxmox External Metric Server](https://pve.proxmox.com/wiki/External_Metric_Server)
