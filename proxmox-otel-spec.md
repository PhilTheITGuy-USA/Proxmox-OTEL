# Proxmox VE → LogicMonitor via OpenTelemetry: Design Spec

Status: draft for review · Date: 2026-10-22 · Owner: Phillip Green

## 1. Goal
Get metrics and logs from Proxmox VE (PVE) into LogicMonitor (LM) using OpenTelemetry, packaged to LM Community Exchange standards, and scalable from a small cluster to multi-cluster estates with thousands of hosts and tens of thousands of guests.

Constraint from owner: OTEL should create the monitored entities unless that is not possible.

## 2. Decision: collector vs. direct forwarding
| Option | Verdict |
|---|---|
| 2) Direct trace forwarding (`/rest/api/v1/traces`) | **Rejected.** Traces only. PVE emits metrics, not traces, and direct mode bypasses collector processors. |
| 1) LM OTel Collector / any contrib collector | **Used for logs, optional for metrics.** |
| 3) (New) PVE → LM OTLP metrics endpoint directly | **Preferred for metrics.** LM's OTel Metrics feature accepts OTLP/HTTP protobuf + Bearer token with no collector. PVE's OpenTelemetry metric server supports custom headers, path, gzip, https. |

Rationale: fewer moving parts for metrics; the collector is retained where it is needed (logs) and as an optional gateway at scale.

## 3. Architecture
```
PVE nodes ──(OTLP/HTTP, gzip, Bearer)──────────────► LM OTLP metrics endpoint
PVE nodes ──(journald/rsyslog, TCP, RFC5424)─► OTel Collector ─(logicmonitor exporter)─► LM /rest/log/ingest
Bootstrap script ──(LM REST API)──► creates one LM resource per PVE node (for log mapping)
```
Optional at scale: a collector gateway per cluster receives PVE OTLP, adds resource attributes (cluster, site), batches, and forwards to LM (`otlphttp` exporter).

## 4. Metrics
- Config: Datacenter → Metric Server → Add → OpenTelemetry (cluster-wide, one definition covers all nodes). CLI: `pvesh create /cluster/metrics/server/<id> --type opentelemetry ...` with `--otel-*` options (protocol, path, compression, headers, resource-attributes, timeout, max-body-size, verify-ssl).
- PVE metric families: `proxmox_node_*`, `proxmox_vm_*` (QEMU and LXC, labels node/type/vmid/name), `proxmox_storage_*`.
- Known quirks (from community reports, to verify): `*_cpu_percent` is a 0–1 ratio; node-level PSI not emitted.
- Resources: LM OTel metrics are flat (name + attributes) and need no resource registration. Entities are identified by attributes (`node`, `vmid`, `type`, `cluster`). Set `cluster` via `otel-resource-attributes` so multi-cluster data separates cleanly.
- Deliverables: PromQL query set, dashboard(s), and alert rule templates (node down, storage >85%, guest CPU/memory saturation).

## 5. Logs
- Source: node journald (pve-cluster/pmxcfs, corosync, pvedaemon, pveproxy, pvestatd, ceph, ha). Guest-internal logs are out of scope (need in-guest agents).
- Transport: rsyslog `imjournal` → `omfwd` TCP (RFC5424) → collector `syslog` receiver → `logicmonitor` exporter. Alternative to test: collector `journald` receiver installed on each node.
- Mapping: LM log ingest requires an **existing** resource (error 4001 otherwise; no auto-create). Map with `_lm.resourceId` (e.g. `system.displayname`); exporter option `logs.resource_mapping_op: OR` tolerates attribute ordering.
- Limits to respect: 8 MB/payload, 1 MB/event, ±3 h timestamp window, 240K req/min.
- Resource bootstrap: idempotent script (Python, LM REST API, bearer/LMv1) creating one resource per PVE node with `system.displayname`, `pve.cluster`, `pve.node` properties. Lists nodes from the PVE API. Run on schedule or via PVE hook for node changes.

## 6. Scalability
- Metrics path is stateless and per-cluster; no per-guest LM objects, so guest count affects only series cardinality.
- Cardinality control: collector gateway can drop or aggregate noisy families (per-disk blockstat) before LM. Document per-guest series estimate and LM limits (to confirm).
- Logs: only nodes need LM resources (hundreds to low thousands), not guests.
- Reliability: collector queues (`sending_queue`, retry), rsyslog disk-assisted queue, gzip on all hops.
- Multi-cluster: one metric-server definition per cluster, distinguished by `cluster` resource attribute; one collector gateway per site.

## 7. Security
- LM bearer token: dedicated API-only user with least privilege (OTLP Metrics manage; Logs manage). Store in PVE `otel-headers` (PVE stores it base64-encoded in `/etc/pve/status.cfg`, readable by cluster admins; document this).
- TLS verification on. Rotation procedure documented (regenerating the token invalidates all senders).
- No secrets in the Exchange package; placeholders only.

## 8. LM Exchange readiness
The Exchange hosts LogicModules; an OTEL integration is not a LogicModule. Package as: (a) README/runbook, (b) collector config, (c) bootstrap script (will undergo LM security review if it ships as script content), (d) dashboard JSON and alert templates, (e) optionally a PropertySource that tags PVE resources. **Open item:** confirm accepted form for non-module content with lm-exchange@logicmonitor.com and the Exchange getting-started guide before building the package.

## 9. Risks / unknowns (resolved by the test plan)
1. OTel Metrics feature flag enabled on the portal? Exact OTLP endpoint and path.
2. Does PVE's payload (temporality, gzip, naming) ingest cleanly into LM?
3. `otel-headers` input format on the CLI (JSON vs base64).
4. Can a resource be created without a collector assignment for log mapping?
5. LM series/cardinality and rate limits for metrics at scale.
6. Whether gateway or direct is better past a size threshold.
