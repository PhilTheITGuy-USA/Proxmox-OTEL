# Enable OpenTelemetry metrics in Proxmox VE (PVE side)

Purpose: point a PVE cluster's OpenTelemetry metric server at the LM OTel Collector gateway. This is the PVE half of the procedure; build the gateway first with `deploy-lm-otel-collector-docker.md`.

Status: the GUI steps in section 3 were verified in the lab on 2026-10-27 (PVE 9.2.2 and 9.2.3, two nodes). The CLI form in section 4 is written from the PVE API schema and is **not yet tested** (marked ⚠).

## Why a gateway
PVE's OTLP plugin sends **OTLP JSON** only. LM's OTLP metrics endpoint accepts **protobuf with a Bearer token**, so a direct push fails with `415 Unsupported Media Type` (observed 2026-10-27). The gateway accepts PVE's JSON with no token, converts it to protobuf and forwards it to LM, where it holds the token. PVE therefore never needs the LM token, which also avoids PVE's 1024-character header limit. The direct route is kept in `pve-direct-to-lm-portal.md` for when PVE adds protobuf (Proxmox Bugzilla #7592).

```
PVE metric server ──OTLP/HTTP JSON :4318──► LM OTel Collector (Docker) ──protobuf + Bearer──► LM
```

## 1. Prerequisites
- Proxmox VE 9.x with the OpenTelemetry metric server type. Check: `pveversion`, and Datacenter → Metric Server → Add offers **OpenTelemetry**.
- The gateway is running and reachable from every PVE node on its LAN IP, TCP 4318 (see the gateway runbook, sections 4 and 5, for the container and the firewall rule).
- Root (or `Sys.Modify` on `/`) on the cluster.

## 2. Check the path from a PVE node
From any node, with `<gateway-ip>` the Docker host's LAN IP:
```bash
curl -s -o /dev/null -w "%{http_code}\n" -X POST -H "Content-Type: application/json" -d '{}' http://<gateway-ip>:4318/v1/metrics
```
Expect `200`. Anything else is a network, firewall or gateway problem; fix it before continuing.

## 3. Add the metric server (GUI)
The definition is cluster-wide: one entry covers every node.
1. Datacenter → **Metric Server → Add → OpenTelemetry**.
2. Fill the fields:

| Field | Value |
|---|---|
| Name | `lm-otlp` (no spaces) |
| Server | gateway host name or IP only (no scheme, no port) |
| Port | `4318` |
| Protocol | `HTTP` |
| Path | `/v1/metrics` |
| Verify SSL | off (the gateway listens on plain HTTP) |

3. Expand **Advanced JSON Configuration**. Leave **HTTP Headers** empty. In **Resource Attributes (JSON)** type `{"cluster":"<cluster-name>"}` as plain JSON, typed as is: no shell commands, no base64. This `cluster` value is the label to use for site or environment names in LM.
4. Click **Create**.

PVE adds its own `proxmox_cluster` and `node` labels, so clusters stay separate even without `cluster`, but set it anyway for readable names.

## 4. Add the metric server (CLI) ⚠
```bash
pvesh create /cluster/metrics/server/lm-otlp \
  --type opentelemetry \
  --server <gateway-ip> --port 4318 \
  --otel-protocol http --otel-path /v1/metrics \
  --otel-verify-ssl 0 \
  --otel-resource-attributes "$(printf '%s' '{"cluster":"<cluster-name>"}' | base64 -w0)"
```
The API schema says `otel-resource-attributes` is base64-encoded JSON (max 1024 characters). Defaults: compression gzip, timeout 5 s (1 to 10), max body 10,000,000 bytes. Confirm the CLI accepts this form before relying on it in automation. The config is stored in `/etc/pve/status.cfg`.

## 5. Verify
1. On a node: `journalctl -u pvestatd -n 50 --no-pager`. No OTel errors means the push is working (lab result: none).
2. On the gateway: `docker logs --tail 50 lmotel` shows the receiver working without export errors.
3. In LM: **More → OTel Metrics → Metrics Explorer**, run `proxmox_node_uptime`. Series appear after the first push (the push interval is not yet measured; see the open checks in the metrics reference).
4. Spot-check values against the PVE UI with the queries in `pve-otel-metrics-reference.md` (CPU values are 0 to 1 ratios, so multiply by 100).

## 6. Change, pause, remove
```bash
pvesh get    /cluster/metrics/server                       # list
pvesh set    /cluster/metrics/server/lm-otlp --disable 1   # pause
pvesh set    /cluster/metrics/server/lm-otlp --server <new-gateway-ip>   # move to another gateway
pvesh delete /cluster/metrics/server/lm-otlp               # remove
```
Rotating the LM bearer token changes only the gateway's `lm.env`; nothing changes on PVE.

## 7. Troubleshooting
| Symptom | Cause and fix |
|---|---|
| Create fails with `415` | The Server field points at the LM portal, not the gateway. Point it at the gateway. |
| Create fails or times out | Gateway not reachable: check the container, the Windows or host firewall rule for TCP 4318, and the `curl` test in section 2. |
| Name rejected | Names cannot contain spaces. Use `lm-otlp`. |
| No `proxmox_*` series in LM | Check `docker logs lmotel` for 401 (bad token in `lm.env`) or 404 (wrong exporter endpoint), and wait for the first push. |
| `cluster` label missing | Resource Attributes was empty or invalid JSON; edit the metric server and re-enter it. |

## References
- `deploy-lm-otel-collector-docker.md` (gateway, token, firewall)
- `pve-otel-metrics-reference.md` (metric names, labels, units, queries)
- [Proxmox External Metric Server](https://pve.proxmox.com/wiki/External_Metric_Server)
- [Proxmox VE API viewer: /cluster/metrics/server](https://pve.proxmox.com/pve-docs/api-viewer/)
- [OpenTelemetry Metrics Overview (LM)](https://www.logicmonitor.com/support/opentelemetry-metrics-overview)
