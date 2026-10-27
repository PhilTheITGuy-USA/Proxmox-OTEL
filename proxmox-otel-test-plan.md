# Proxmox VE → LogicMonitor OTEL: Test Plan (v2)

Lab: 2-node PVE cluster (VM + LXC guests), Docker Desktop. Run phases in order; don't advance until exit criteria are met. Record results in `results.md`. Nothing here has been run yet.

## Phase 0: Portal prerequisites
1. Confirm the **OpenTelemetry Metrics** feature flag is enabled (More → OTel Metrics visible).
2. Create an API-only LM user with OTLP Metrics manage (and any role needed for LogSource import).
3. More → OTel Metrics → Configure → View connection details; record the OTLP endpoint; generate the bearer token. Check the encoded `otel-headers` value stays under 1024 characters.
4. Identify the LM Collector for the lab; record its version (≥ 40.700 preferred) and that it is not in an Auto-Balanced Collector Group.
Exit: endpoint, token and collector version recorded (not committed).

## Phase 1: Prove the metrics endpoint (no PVE)
Docker `otelcol` with `hostmetrics` → `otlphttp` (Bearer) to the LM endpoint; query the metric with PromQL in Explore.
Exit: metrics visible in LM.

## Phase 2: PVE direct metrics (single node)
1. `pvesh create /cluster/metrics/server/lm-otlp --type opentelemetry --server <portal> --port 443 --otel-protocol https --otel-path <from Phase 0> --otel-compression gzip --otel-verify-ssl 1 --otel-headers <base64 JSON> --otel-resource-attributes <base64 JSON: cluster=lab>`
2. Check `journalctl -u pvestatd` for push errors.
3. Query `{__name__=~"proxmox_.*"}` in Explore.
Exit: node, VM, LXC and storage families arrive with node/vmid/type labels; record the real push interval and any rejection reason.
**Result 2026-10-27:** direct push failed on Create with `415 Unsupported Media Type` (PVE sends OTLP JSON; LM requires protobuf). Phase 2b is now the primary path: PVE → LM OTel Collector gateway (Docker, 4318) → LM. Re-test direct push if PVE adds protobuf.

## Phase 3: Data quality
Compare node CPU/memory, guest CPU/memory and storage used against the PVE UI; record unit handling (0–1 ratios) and needed PromQL transforms; add node 2 and confirm `cluster` separation.
Exit: values match the PVE UI within tolerance.

## Phase 4: Logs via syslog
1. On the lab LM Collector enable syslog (`lmlogs.syslog.enabled=true`); confirm it listens on UDP 514 (UDP only by design).
2. On each PVE node, add an rsyslog UDP forward (journal → collector). Do the same on one VM and one LXC.
3. Set the backup resource (`lmlogs.logsource.syslog.unmapped.resource`).
4. Generate events (`logger`, restart `pveproxy`); check LM Logs.
5. Create the Syslog LogSource (collector-applied) with IP/hostname mapping; test a mapped node, an unmapped guest and the backup resource.
6. Test filters (severity/facility) and log fields (`cluster` tag).
7. Negative/load tests: burst of messages over UDP, collector restart, large messages; record loss.
Exit: logs from nodes and chosen guests land on the right resource (or the backup resource); loss behaviour documented. If mapping is inadequate, evaluate the deferred bootstrap script.

## Phase 5: Dashboard and alerts
Build the dashboard and three alert rules (metric absent/node down, storage >85%, guest CPU sustained high); trigger each. Run `lm_dashboards.py check`.

## Phase 6: Scale and resilience
Estimate series per guest; project to 1k/10k/50k guests; load-test with a synthetic OTLP generator directly and through a gateway; watch for 429s and limits. Failure test: block egress 5 minutes, restore, check the metrics gap; syslog loss during the outage is accepted (UDP only), so record whether the LM Collector buffered anything and confirm forwarding resumes on restore.
Exit: documented thresholds for when to add a gateway and extra collectors.

## Phase 7: Exchange packaging
Capture the syslog LogSource export from the portal; update the skill with its shape; build `dist/`; run both skill checks; confirm companion-content acceptance with lm-exchange@logicmonitor.com; remove secrets; peer review.

## Outputs
`results.md`, LogSource module, rsyslog snippets, pvesh script, gateway config, dashboard JSON, alert templates, runbook.
