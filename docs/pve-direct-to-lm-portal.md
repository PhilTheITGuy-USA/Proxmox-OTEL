# Send Proxmox VE metrics directly to your LogicMonitor portal

Purpose: configure PVE's OpenTelemetry metric server to push straight to the LogicMonitor (LM) portal, with no collector or gateway. Portal used in this runbook: `https://<portalname>.logicmonitor.com/`.

Use this when the customer has a small number of clusters. For many clusters or very large guest counts, see `deploy-lm-otel-collector-docker.md` (gateway) and the decision notes in `proxmox-otel-spec.md`.

Status: written from LogicMonitor documentation and the official PVE API schema; **not yet tested in the lab**. Items marked ⚠ are unverified and are confirmed in test-plan Phase 0/2. Nothing in this document contains a real token.

## 1. Prerequisites
- Proxmox VE 9.x with the OpenTelemetry metric server type. Check on a node: `pveversion`, and confirm Datacenter → Metric Server → Add offers **OpenTelemetry**.
- LM portal `<portalname>` with the **OpenTelemetry Metrics** feature enabled (feature flag) ⚠. If **More → OTel Metrics** is missing from the menu, ask LM support/your account team to enable it.
- An LM user with **Manage** permission for OpenTelemetry Metrics (Settings role permissions). Use a dedicated API-only user with minimum rights.
- Network: every PVE node must reach `<portalname>.logicmonitor.com` on TCP 443 (PVE sends from the node that runs the stats daemon; allow all nodes). Proxy use is not documented for this feature ⚠.
- Root (or `Sys.Modify` on `/`) on the cluster to add a metric server.

## 2. Find the OTLP endpoint and create the bearer token (LM portal)
1. Sign in to `https://<portalname>.logicmonitor.com/`.
2. Go to **More → OTel Metrics → Configure → View connection details** (page title: *Connect an OTel Metrics Source*).
3. Copy the **OTLP Ingestion Endpoint**. Note its scheme, host, port and path (used in section 4). The host is expected to be on your portal domain ⚠; use exactly what the page shows.
4. In **User**, choose the dedicated API-only user.
5. Copy the **Bearer Token** into a password manager. Anyone with it can send telemetry to your portal; don't paste it into chat, tickets or git.
6. Protocol: **OTLP HTTP/protobuf**. Header: `Authorization: Bearer <token>`.

Rotation: generating a new token invalidates the old one on every cluster at once. Plan the change window (section 8).

## 3. Build the `otel-headers` value
PVE takes headers as **JSON, base64-encoded, max 1024 characters** (official API schema). Build it on a PVE node without leaving the token in shell history:

```bash
read -rs LM_TOKEN                                   # paste the token, press Enter (no echo)
OTEL_HEADERS=$(printf '%s' "{\"Authorization\":\"Bearer $LM_TOKEN\"}" | base64 -w0)
unset LM_TOKEN
echo ${#OTEL_HEADERS}                               # must be 1024 or less ⚠
```
Resource attributes label this cluster in LM (also base64 JSON, max 1024 characters):
```bash
OTEL_ATTRS=$(printf '%s' '{"cluster":"<cluster-name>"}' | base64 -w0)
```
Use a unique `cluster` value per cluster. If `OTEL_HEADERS` is longer than 1024 characters, direct push can't carry your token; use the gateway runbook instead.

## 4. Split the endpoint into PVE fields

| PVE field | Value for `<portalname>` |
|---|---|
| `--server` | host from the endpoint (DNS name) ⚠ |
| `--port` | port from the endpoint (443 if none shown) |
| `--otel-protocol` | `https` |
| `--otel-path` | path from the endpoint (PVE default is `/v1/metrics`) ⚠ |
| `--otel-compression` | `gzip` |
| `--otel-verify-ssl` | `1` (keep certificate verification on) |
| `--otel-timeout` | `5` (default; 1–10 allowed) |

## 5. Add the metric server in Proxmox VE
The definition is cluster-wide: one entry covers all nodes in the cluster.

**CLI (any node)**
```bash
pvesh create /cluster/metrics/server/lm-otlp \
  --type opentelemetry \
  --server <host from endpoint> --port 443 \
  --otel-protocol https --otel-path <path from endpoint> \
  --otel-compression gzip --otel-verify-ssl 1 \
  --otel-headers "$OTEL_HEADERS" \
  --otel-resource-attributes "$OTEL_ATTRS"
```
The CLI is expected to take the base64 strings as-is ⚠ (community posts show other forms). If it rejects them, tell us the exact error so we can adjust the runbook.

**GUI**
1. Datacenter → **Metric Server → Add → OpenTelemetry**.
2. Fill **Name** (`lm-otlp`), **Server**, **Port**, **Protocol**, **Path**; keep **Verify SSL** on.
3. Add headers/resource attributes if the form offers them; otherwise add them afterwards with `pvesh set` (section 7).
4. Click **Create**.

The config is stored in `/etc/pve/status.cfg`. The base64 headers, including the bearer token, are readable by cluster admins.

## 6. Verify
1. On a node: `journalctl -u pvestatd -n 50 --no-pager`. HTTP 401/403 means a token problem, 404 a wrong path, TLS errors a certificate or CA problem.
2. In the LM portal: **Explore**, query `{__name__=~"proxmox_.*"}` (PromQL-compatible). Expect `proxmox_node_*`, `proxmox_vm_*` and `proxmox_storage_*` with `node`, `vmid`, `type` and your `cluster` label. Metric names and push interval are community-reported ⚠.
3. Compare node CPU, memory and one guest against the PVE UI (CPU fields may be 0–1 ratios).
4. Optional auth check from a node (a 401/403 means the token or user is wrong; any other status means authentication passed):
```bash
read -rs LM_TOKEN
curl -sS -o /dev/null -w '%{http_code}\n' -X POST \
  -H "Authorization: Bearer $LM_TOKEN" -H 'Content-Type: application/x-protobuf' \
  --data-binary '' '<full OTLP endpoint>' ; unset LM_TOKEN
```

## 7. Roll out to more clusters, change, pause, remove
- **Pilot first:** configure one cluster, wait for data, check the dashboards, then repeat per cluster with a unique server id and `cluster` label.
- **Rotate the token** (all clusters, same window): generate the new token in LM, rebuild `OTEL_HEADERS`, then on each cluster:
```bash
pvesh set /cluster/metrics/server/lm-otlp --otel-headers "$OTEL_HEADERS"
```
- **Pause / remove:**
```bash
pvesh get    /cluster/metrics/server                       # list
pvesh set    /cluster/metrics/server/lm-otlp --disable 1   # pause
pvesh delete /cluster/metrics/server/lm-otlp               # remove
```

## 8. Troubleshooting
- `401/403`: wrong or regenerated token, or the user lacks OTel Metrics permission.
- `404`: wrong path; copy it exactly from **View connection details**.
- Nothing in LM and no errors: feature flag not enabled, or the node can't reach port 443 (`curl -I https://<portalname>.logicmonitor.com/`).
- Timeouts: raise `--otel-timeout` (max 10) or reduce body size with `--otel-max-body-size`.
- Headers rejected for length: token too long for the 1024-character field; use a gateway.

## References
- [Configure OpenTelemetry Metrics Source Connection](https://www.logicmonitor.com/support/configure-opentelemetry-metrics-source-connection)
- [OpenTelemetry Metrics Overview](https://www.logicmonitor.com/support/opentelemetry-metrics-overview)
- [Proxmox VE API viewer: /cluster/metrics/server](https://pve.proxmox.com/pve-docs/api-viewer/)
- [Proxmox External Metric Server](https://pve.proxmox.com/wiki/External_Metric_Server)
