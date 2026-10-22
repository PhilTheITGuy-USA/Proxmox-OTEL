# Proxmox VE → LogicMonitor OTEL: Test Plan

Lab: 2-node PVE cluster (VM + LXC guests), Docker Desktop. Run phases in order; do not advance until exit criteria are met. Record results in `results.md`.

## Phase 0: Portal prerequisites
1. Confirm the **OpenTelemetry Metrics** feature flag is enabled (More → OTel Metrics visible).
2. Create an API-only LM user with: OTLP Metrics manage, Logs manage, device manage.
3. More → OTel Metrics → Configure → View connection details. Record the OTLP endpoint; generate the bearer token.
4. Create LM REST API token (for bootstrap) and confirm Logs is enabled.
Exit: endpoint and tokens recorded (not committed to files).

## Phase 1: Prove the endpoint (no PVE)
1. In Docker Desktop, run `otelcol-contrib` with an `otlphttp` exporter to the LM endpoint (Authorization: Bearer) and a `hostmetrics` receiver.
2. In LM, query the metric with PromQL in Explore.
Exit: metrics visible in LM. Validates endpoint, token, and permissions.

## Phase 2: PVE direct metrics (single node)
1. On node 1: `pvesh create /cluster/metrics/server/lm-otlp --type opentelemetry --server <portal>.logicmonitor.com --port 443 --otel-protocol https --otel-path <path from Phase 0> --otel-compression gzip --otel-verify-ssl 1 --otel-headers '<json or base64, see risk 3>' --otel-resource-attributes '<json: cluster=lab>'`
2. Check `journalctl -u pvestatd` for push errors.
3. In LM Explore, query `{__name__=~"proxmox_.*"}`.
Exit: node, VM, LXC and storage families arriving with `node/vmid/type` labels. Record any rejection reason.
Fallback if direct fails: Phase 2b, PVE → local collector (Docker, port 4318) → LM `otlphttp`.

## Phase 3: Data quality
1. Compare 5 metrics (node CPU, node memory, guest CPU, guest memory, storage used) against the PVE UI.
2. Confirm CPU ratio handling (0–1) and units; record needed PromQL transforms.
3. Add second node; confirm both report and the `cluster` attribute separates labs.
Exit: values match the PVE UI within tolerance.

## Phase 4: Logs
1. Docker: collector with `syslog` receiver (TCP 514) and `logicmonitor` exporter (`LOGICMONITOR_ACCOUNT` set, bearer auth, `resource_mapping_op: OR`).
2. Run the bootstrap script (dry-run first) to create one LM resource per node.
3. Configure rsyslog `imjournal` + `omfwd` on both nodes pointing at the collector.
4. Generate events (`logger`, restart `pveproxy`); confirm in LM Logs mapped to the right resource.
5. Negative test: a node with no LM resource returns 4001; confirm retries do not loop forever.
6. Compare with a collector `journald` receiver installed on a node.
Exit: logs from both nodes land on their resources; failure behaviour documented.

## Phase 5: Dashboard and alerts
Build the dashboard and 3 alert rules (node down/metric absent, storage >85%, guest CPU sustained high). Trigger each deliberately.

## Phase 6: Scale and resilience
1. Estimate series per guest from Phase 2; project to 1k/10k/50k guests.
2. Load test: Docker-based synthetic OTLP generator (telemetrygen) at projected volume, directly and via a gateway collector; watch for 429s, latency, and LM limits.
3. Failure tests: block egress 5 minutes, then restore; confirm queue/retry and no data loss for logs.
Exit: documented thresholds for when to add a gateway collector.

## Phase 7: Exchange packaging
Confirm accepted format with lm-exchange@logicmonitor.com; assemble package; remove secrets; peer review; submit.

## Outputs
`results.md`, final collector configs, bootstrap script, dashboard JSON, alert templates, runbook.
