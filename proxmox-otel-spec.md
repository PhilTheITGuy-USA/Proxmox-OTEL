# Proxmox VE → LogicMonitor via OpenTelemetry: Design Spec

Status: draft v3 · Date: 2026-10-27 · Owner: Phillip Green
v3 changes: metrics path verified end to end in the lab on 2026-10-27 (PVE → LM OTel Collector in Docker → LM); sections 2, 4, 6, 10, 12 and 13 updated with verified facts, the example alerts and dashboard, and the topology answer.
v2 changes: logs move from an OTel collector to LogicMonitor's native syslog LogSource; LM OTel Collector is the optional metrics gateway; adds repo layout, build and deployment sections.

## 1. Goal
Get Proxmox VE (PVE) metrics and logs into LogicMonitor (LM), using OpenTelemetry for metrics, packaged to LM Community Exchange standards, scaling from a few nodes to multi-cluster estates (thousands of hosts, tens of thousands of guests).

## 2. Decisions
| Question | Decision | Why |
|---|---|---|
| Direct trace forwarding (`/rest/api/v1/traces`)? | **Rejected** | Traces only; PVE emits metrics. |
| LM OTel Collector? | **Yes, required as the metrics gateway (verified)** | v7.0 (OTel 0.154) has the otlp receiver (accepts JSON and protobuf); batch, resource, attributes, transform, filter, cumulativetodelta, metricstransform processors; otlphttp exporter (sends protobuf). Managed from the portal. |
| Metrics path | **PVE → LM OTel Collector gateway → LM OTLP endpoint** (gateway required) | PVE's plugin sends OTLP **JSON** only; LM's OTLP metrics endpoint takes OTLP HTTP/protobuf and returned `415 Unsupported Media Type` to PVE's connection test (observed 2026-10-27). The collector converts JSON to protobuf. Verified in the lab: the gateway runs in Docker on port 4318 without a token, the `otlphttp/lm` exporter (protobuf, `metrics_endpoint`, Bearer from an env file) delivers to LM, and `proxmox_*` series appear in Metrics Explorer. Direct push becomes possible only if PVE adds protobuf (Proxmox Bugzilla #7592, reported). |
| Logs path | **Syslog from hosts/VMs/LXCs → LM Collector Syslog LogSource** | Native and recommended by LM; the LM Collector's syslog support has resource mapping, filters and fallback; no custom collector build; the LogSource is a LogicModule so it fits the Exchange. The LM-wrapped OTel Collector has no syslog/journald receivers, and the contrib `logicmonitor` exporter requires a self-managed build. |
| Custom OTel collector build (OCB) for logs | **Dropped** (kept as documented fallback) | Not needed if syslog tests pass. |

## 3. Architecture
```
PVE cluster ──OTLP/HTTP+gzip+Bearer──► [LM OTel Collector gateway: JSON→protobuf] ──► LM OTLP metrics endpoint
PVE nodes / VMs / LXCs ──syslog (rsyslog)──► LM Collector (Syslog LogSource) ──► LM Logs
```
Two independent paths; either can be deployed alone.

## 4. Metrics
- One cluster-wide PVE metric server per cluster: `pvesh create /cluster/metrics/server/<id> --type opentelemetry --server <host> --port 443 ...`.
- Official API options (verified from the PVE API schema): `otel-protocol` (http|https, default https), `otel-path` (default `/v1/metrics`), `otel-compression` (gzip|none, default gzip), `otel-headers` and `otel-resource-attributes` (JSON, **base64-encoded, max 1024 chars each**), `otel-timeout` (1–10 s, default 5), `otel-max-body-size` (min 1024, default 10 MB), `otel-verify-ssl` (default on).
- With the gateway the PVE metric server needs no token: Server = gateway IP, port 4318, protocol HTTP, path `/v1/metrics`, Verify SSL off, headers empty, resource attributes `{"cluster":"<name>"}` (see `docs/enable-pve-otel-metrics.md`). The base64 and 1024-character limits matter only for the direct route.
- Metric names, labels and units are now **verified against the PVE UI** and documented in `docs/pve-otel-metrics-reference.md`: node (53 names), guest (22 names plus QEMU-only), storage (6). LM shows names without unit suffixes; counters keep `_total`. CPU values are 0–1 ratios (multiply by 100); memory, disk and storage sizes are bytes; uptime is seconds. PVE adds `proxmox_cluster` and `node` labels itself; `vmid` is unique only within a cluster. Still open: push interval, shared-storage behaviour, I/O counter units, node CPU time counters, PSI values.
- Entities are identified by attributes, not LM resources. `cluster` is set via `otel-resource-attributes` so multi-cluster data separates.
- Deliverables: PromQL query set (in the metrics reference), a dashboard and alert rules. Built in the lab as examples (`docs/pve-otel-alerts-dashboard-example.md`): dashboard "Proxmox VE OTel" (node CPU, node memory, storage used, guest CPU) and alert group `Proxmox-VE` with `PVE_Storage_Used_Over_85` (Critical), `PVE_Node_Memory_High`, `PVE_Node_CPU_High` and `PVE_Guest_CPU_High` (Warning, above 90). OTel Metrics alert rules and dashboard widgets are portal objects created through Metrics Explorer, not LogicModules, so they ship as documented steps and queries. Node-down (absent series) and storage-inactive alerts are untested.
- Topology: not possible from these metrics alone. LM topology maps need resources with external resource IDs linked by a TopologySource; OTel metrics create no resources. Options are a PropertySource plus TopologySource reading the PVE API, or label-grouped dashboards as a logical view. Decision deferred.

## 5. Logs
- Receiver: an LM Collector with syslog enabled (agent.conf `lmlogs.syslog.enabled=true`, or "Enable log collection for Syslog and SNMP trap" when adding a collector). Transport is **UDP 514 only** (decision). UDP has no delivery guarantee: messages sent while the collector is unreachable or overloaded are lost, and that is accepted.
- Senders: PVE nodes (rsyslog `imjournal`/journald forward), and any VM or LXC that should be searchable (their own rsyslog).
- Syslog LogSource (type "LM Logs: SysLog"): filters (Application, Facility, Message, Severity), log fields (Application/Facility/Severity attributes, static `cluster` tag), resource mapping by IP (`system.ips`), hostname/FQDN (`system.hostname`) or static/regex.
- Precedence: a LogSource applied to a monitored device beats one applied to the collector; a collector-applied LogSource beats the agent.conf property. Collector version gates: ≥ EA-35.200 apply-to-collector, ≥ 37.300 OR mapping, ≥ 40.700 complex filters. Collector-applied LogSources cannot use Auto-Balanced Collector Groups; only one Syslog LogSource per collector.
- Unmapped logs: configure `lmlogs.logsource.syslog.unmapped.resource` (a backup resource per collector); unmapped logs otherwise appear under `_resource.id=0`.
- Fallbacks if syslog tests fail: logs ingest API (`/rest/log/ingest`, existing resource required, error 4001 otherwise) or the contrib `logicmonitor` exporter via a self-built collector.

## 6. Resource strategy
- Metrics: nothing to create (satisfies the owner's rule that OTEL creates the entities); metric series are not LM resources, so they have no resource tree entry or topology.
- Logs: map to resources that already exist (for example PVE nodes already monitored in the portal) or fall back to the per-collector backup resource. A bootstrap script that creates one LM resource per PVE node is **deferred**; build it only if the Phase 4 tests show mapping can't be done otherwise. Guests never get LM resources from this project.

## 7. Scalability
- Metrics path is stateless and per-cluster; guest count affects series cardinality only. Gateway adds batching, enrichment and drop/aggregate rules for noisy families (per-disk blockstat).
- Logs: volume is controlled at the source (rsyslog severity/facility rules) and in the LogSource filters; nodes only by default, guests opt-in. Scale by adding LM Collectors (non-ABCG) per site.
- Multi-cluster: one metric-server definition per cluster, distinguished by `cluster`; one gateway and one syslog collector per site.

## 8. Security
- Dedicated API-only LM user, least privilege (OTLP Metrics manage for metrics).
- Bearer token lives in `otel-headers`, base64-stored in `/etc/pve/status.cfg` and readable by cluster admins. Rotation invalidates every sender.
- Syslog is unauthenticated plaintext on the network: restrict by firewall to PVE/guest networks; note in the runbook.
- Secrets never committed; the Exchange package carries placeholders only.

## 9. LM Exchange readiness
- The Syslog LogSource (and any PropertySource, dashboard JSON, alert templates) are LogicModule content and follow Exchange conventions; run `lm_lint_module.py` and `lm_dashboards.py check` over `dist/` before calling anything done.
- Collector config, PVE setup script, rsyslog snippets and the runbook ship as companion documentation. Confirm acceptance of non-module content with lm-exchange@logicmonitor.com.
- Skill gap: `logicmonitor-modules` documents script LogSources only; the syslog LogSource export shape must be captured from a portal export and added to the skill (lesson back into skill, with a lint check and test case).

## 10. Repo layout (Proxmox-OTEL)
```
pve/          metric-server setup script (pvesh), rsyslog snippets
collector/    LM OTel Collector gateway config (+ docker-compose for the lab)
logsource/    Syslog LogSource module source
build/        build.py (assembles dist/)
dist/         built output (linted): module JSON at dist/*.json, dashboards at dist/dashboards/*.json
lm/           dashboards, alert templates, PromQL set
docs/         runbooks (gateway, PVE side, direct-to-portal), metrics reference, alerts and dashboard example, sizing guide
results.md    test-plan results
```

## 11. Build and verification
`build/build.py` assembles `dist/` from `logsource/` (modules → `dist/*.json`) and `lm/` (dashboards → `dist/dashboards/*.json`). Run the two skill checks from the repo root:
```sh
python ../logicmonitor-modules-skill/skill/scripts/lm_lint_module.py dist
python ../logicmonitor-modules-skill/skill/scripts/lm_dashboards.py check dist/dashboards/*.json --modules dist
```
 Record verification per layer (local checks / portal import / live target) in `results.md` only. Lab: 2-node PVE cluster plus Docker Desktop.

## 12. Deployment
- Gateway (required, first): LM OTel Collector from the portal wizard, run in Docker with an env file for the credentials and a gateway `lmconfig.yaml` (see `docs/deploy-lm-otel-collector-docker.md`), one per site, two behind a load balancer for HA (HA untested).
- Metrics: apply the metric-server definition once per cluster, pointing at the gateway (script or Ansible); verify with `journalctl -u pvestatd` and Metrics Explorer.
- Logs: enable syslog on the site's LM Collector, import the LogSource, push the rsyslog snippet to nodes (and chosen guests), then verify mapping.
- Upgrades/rollback: metric server can be disabled with `--disable 1`; remove the rsyslog snippet to stop logs.

## 13. Open risks (resolved by the test plan)
1. Resolved: OTel Metrics is enabled on the lab portal; the endpoint is `https://<portalname>.logicmonitor.com/rest/api/v1/metrics` (protobuf + Bearer).
2. Resolved: PVE's JSON is rejected with 415; the gateway converts it. The 1024-character header cap no longer applies.
3. Whether `pvesh` accepts base64 `otel-resource-attributes` as-is (the GUI takes plain JSON, verified). CLI form untested.
4. UDP syslog loss under load and during collector restarts (UDP only by decision; measure, don't fix).
5. Resource mapping for PVE nodes (existing resources vs backup resource).
6. Series-count and rate limits at scale, and gateway sizing and HA (untested; the lab has 8 guests).
7. Exchange acceptance of non-module companion content.
