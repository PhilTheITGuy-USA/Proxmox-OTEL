# Proxmox VE → LogicMonitor via OpenTelemetry: Design Spec

Status: draft v2 · Date: 2026-10-22 · Owner: Phillip Green
v2 changes: logs move from an OTel collector to LogicMonitor's native syslog LogSource; LM OTel Collector is the optional metrics gateway; adds repo layout, build and deployment sections.

## 1. Goal
Get Proxmox VE (PVE) metrics and logs into LogicMonitor (LM), using OpenTelemetry for metrics, packaged to LM Community Exchange standards, scaling from a few nodes to multi-cluster estates (thousands of hosts, tens of thousands of guests).

## 2. Decisions
| Question | Decision | Why |
|---|---|---|
| Direct trace forwarding (`/rest/api/v1/traces`)? | **Rejected** | Traces only; PVE emits metrics. |
| LM OTel Collector? | **Yes, as the optional metrics gateway** | v7.0 (OTel 0.154) has otlp receiver; batch, resource, attributes, transform, filter, cumulativetodelta, metricstransform processors; otlphttp exporter. Managed from the portal. |
| Metrics path | **PVE → LM OTLP endpoint directly**; gateway only at scale | LM OTel Metrics accepts OTLP/HTTP protobuf + Bearer; no collector or resource registration needed. |
| Logs path | **Syslog from hosts/VMs/LXCs → LM Collector Syslog LogSource** | Native and recommended by LM; the LM Collector's syslog support has resource mapping, filters and fallback; no custom collector build; the LogSource is a LogicModule so it fits the Exchange. The LM-wrapped OTel Collector has no syslog/journald receivers, and the contrib `logicmonitor` exporter requires a self-managed build. |
| Custom OTel collector build (OCB) for logs | **Dropped** (kept as documented fallback) | Not needed if syslog tests pass. |

## 3. Architecture
```
PVE cluster ──OTLP/HTTP+gzip+Bearer──► [optional LM OTel Collector gateway] ──► LM OTLP metrics endpoint
PVE nodes / VMs / LXCs ──syslog (rsyslog)──► LM Collector (Syslog LogSource) ──► LM Logs
```
Two independent paths; either can be deployed alone.

## 4. Metrics
- One cluster-wide PVE metric server per cluster: `pvesh create /cluster/metrics/server/<id> --type opentelemetry --server <host> --port 443 ...`.
- Official API options (verified from the PVE API schema): `otel-protocol` (http|https, default https), `otel-path` (default `/v1/metrics`), `otel-compression` (gzip|none, default gzip), `otel-headers` and `otel-resource-attributes` (JSON, **base64-encoded, max 1024 chars each**), `otel-timeout` (1–10 s, default 5), `otel-max-body-size` (min 1024, default 10 MB), `otel-verify-ssl` (default on).
- The official admin guide documents none of these. Push interval, metric names and units come from community reports and are **unverified** until Phase 2/3.
- Community-reported families: `proxmox_node_*`, `proxmox_vm_*` (QEMU+LXC; labels node/type/vmid/name), `proxmox_storage_*`; CPU fields are 0–1 ratios.
- Entities are identified by attributes, not LM resources. `cluster` is set via `otel-resource-attributes` so multi-cluster data separates.
- Deliverables: PromQL query set, dashboards, alert templates (node down, storage >85%, guest CPU sustained high).

## 5. Logs
- Receiver: an LM Collector with syslog enabled (agent.conf `lmlogs.syslog.enabled=true`, or "Enable log collection for Syslog and SNMP trap" when adding a collector). Transport is **UDP 514 only** (decision). UDP has no delivery guarantee: messages sent while the collector is unreachable or overloaded are lost, and that is accepted.
- Senders: PVE nodes (rsyslog `imjournal`/journald forward), and any VM or LXC that should be searchable (their own rsyslog).
- Syslog LogSource (type "LM Logs: SysLog"): filters (Application, Facility, Message, Severity), log fields (Application/Facility/Severity attributes, static `cluster` tag), resource mapping by IP (`system.ips`), hostname/FQDN (`system.hostname`) or static/regex.
- Precedence: a LogSource applied to a monitored device beats one applied to the collector; a collector-applied LogSource beats the agent.conf property. Collector version gates: ≥ EA-35.200 apply-to-collector, ≥ 37.300 OR mapping, ≥ 40.700 complex filters. Collector-applied LogSources cannot use Auto-Balanced Collector Groups; only one Syslog LogSource per collector.
- Unmapped logs: configure `lmlogs.logsource.syslog.unmapped.resource` (a backup resource per collector); unmapped logs otherwise appear under `_resource.id=0`.
- Fallbacks if syslog tests fail: logs ingest API (`/rest/log/ingest`, existing resource required, error 4001 otherwise) or the contrib `logicmonitor` exporter via a self-built collector.

## 6. Resource strategy
- Metrics: nothing to create (satisfies the owner's rule that OTEL creates the entities).
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
docs/         runbook, sizing guide
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
- Metrics: apply the metric-server definition once per cluster (script or Ansible); verify with `journalctl -u pvestatd`.
- Gateway (optional): LM OTel Collector installed from the portal wizard (Docker/Linux), one per site, two behind a load balancer for HA.
- Logs: enable syslog on the site's LM Collector, import the LogSource, push the rsyslog snippet to nodes (and chosen guests), then verify mapping.
- Upgrades/rollback: metric server can be disabled with `--disable 1`; remove the rsyslog snippet to stop logs.

## 13. Open risks (resolved by the test plan)
1. OTel Metrics feature flag and exact OTLP endpoint on the portal.
2. PVE payload compatibility with LM (temporality, naming, gzip, 1024-char header cap vs token length).
3. Whether `pvesh` accepts the base64 `otel-headers` / `otel-resource-attributes` value as-is, or expects plain JSON (the API schema says base64; community reports show JSON).
4. UDP syslog loss under load and during collector restarts (UDP only by decision; measure, don't fix).
5. Resource mapping for PVE nodes (existing resources vs backup resource).
6. Series-count and rate limits at scale.
7. Exchange acceptance of non-module companion content.
