# Proxmox VE → LogicMonitor OTEL: Test Plan (v2)

Lab: 2-node PVE cluster (VM + LXC guests), Docker Desktop. Run phases in order; don't advance until exit criteria are met. Record results in `results.md`. Run on 2026-10-27: phases 0 to 3 and the metrics part of 5 (results marked below). Not run yet: phases 4, 6 and 7.

## Phase 0: Portal prerequisites
1. Confirm the **OpenTelemetry Metrics** feature flag is enabled (More → OTel Metrics visible).
2. Create an API-only LM user with OTLP Metrics manage (and any role needed for LogSource import).
3. More → OTel Metrics → Configure → View connection details; record the OTLP endpoint; generate the bearer token. Check the encoded `otel-headers` value stays under 1024 characters.
4. Identify the LM Collector for the lab; record its version (≥ 40.700 preferred) and that it is not in an Auto-Balanced Collector Group.
Exit: endpoint, token and collector version recorded (not committed).
**Result 2026-10-27:** done. Feature enabled; API-only user and role created (the portal can take an hour or more to show new users and roles in the gateway wizard); endpoint `https://<portalname>.logicmonitor.com/rest/api/v1/metrics`. See `docs/deploy-lm-otel-collector-docker.md` sections 2 and 3.

## Phase 1: Prove the metrics endpoint (no PVE)
Docker `otelcol` with `hostmetrics` → `otlphttp` (Bearer) to the LM endpoint; query the metric with PromQL in Explore.
Exit: metrics visible in LM.
**Result 2026-10-27:** superseded by the gateway build, which proves the endpoint and token with real PVE data (`otlphttp/lm` exporter, protobuf, Bearer). Not run as a separate `hostmetrics` test.

## Phase 2: PVE direct metrics (single node)
1. `pvesh create /cluster/metrics/server/lm-otlp --type opentelemetry --server <portal> --port 443 --otel-protocol https --otel-path <from Phase 0> --otel-compression gzip --otel-verify-ssl 1 --otel-headers <base64 JSON> --otel-resource-attributes <base64 JSON: cluster=lab>`
2. Check `journalctl -u pvestatd` for push errors.
3. Query `{__name__=~"proxmox_.*"}` in Explore.
Exit: node, VM, LXC and storage families arrive with node/vmid/type labels; record the real push interval and any rejection reason.
**Result 2026-10-27:** direct push failed on Create with `415 Unsupported Media Type` (PVE sends OTLP JSON; LM requires protobuf). Phase 2b is now the primary path: PVE → LM OTel Collector gateway (Docker, 4318) → LM. Re-test direct push if PVE adds protobuf.

## Phase 2b: PVE → gateway → LM (primary path)
1. Build the gateway per `docs/deploy-lm-otel-collector-docker.md` (Docker, port 4318, `lm.env`, gateway `lmconfig.yaml`); health check on port 13133.
2. Open the host firewall for TCP 4318 from the PVE subnet; from a node run the `curl` POST test and expect `200`.
3. Add the metric server per `docs/enable-pve-otel-metrics.md` (Server = gateway IP, 4318, HTTP, `/v1/metrics`, no headers, `{"cluster":"lab"}`).
4. Check `journalctl -u pvestatd`, `docker logs`, and Explore.
Exit: node, VM, LXC and storage families arrive.
**Result 2026-10-27:** passed. The `200` test passed, Create succeeded, `pvestatd` showed no OTel errors and `proxmox_*` series appeared in Explore with `proxmox_cluster`, `node`, `type`, `vmid`, `name`, `storage` and `cluster` labels. Real push interval still to measure.

## Phase 3: Data quality
Compare node CPU/memory, guest CPU/memory and storage used against the PVE UI; record unit handling (0–1 ratios) and needed PromQL transforms; add node 2 and confirm `cluster` separation.
Exit: values match the PVE UI within tolerance.
**Result 2026-10-27:** passed for guest CPU, memory, disk and uptime, node CPU and memory, and storage used (within sampling differences of the PVE UI; both nodes seen). Details in `docs/pve-otel-metrics-reference.md`. Open: I/O counter units, node uptime, node CPU time counters, PSI, storage flags, shared storage.

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
Build the dashboard and alert rules; trigger each. Run `lm_dashboards.py check` if a dashboard JSON is packaged.
**Result 2026-10-27 (built, not yet triggered):** dashboard "Proxmox VE OTel" with four widgets, and alert group `Proxmox-VE` with `PVE_Storage_Used_Over_85`, `PVE_Node_Memory_High`, `PVE_Node_CPU_High` and `PVE_Guest_CPU_High`. Steps in `docs/pve-otel-alerts-dashboard-example.md`. To do: trigger each rule (lower a threshold temporarily), and test node-down (absent series) and storage-inactive (`proxmox_storage_active`) alerts.

## Phase 6: Scale and resilience
Estimate series per guest; project to 1k/10k/50k guests; load-test with a synthetic OTLP generator directly and through a gateway; watch for 429s and limits. Failure test: block egress 5 minutes, restore, check the metrics gap; syslog loss during the outage is accepted (UDP only), so record whether the LM Collector buffered anything and confirm forwarding resumes on restore.
Exit: documented thresholds for when to add a gateway and extra collectors.

## Phase 7: Exchange packaging
Capture the syslog LogSource export from the portal; update the skill with its shape; build `dist/`; run both skill checks; confirm companion-content acceptance with lm-exchange@logicmonitor.com; remove secrets; peer review.

## Outputs
`results.md`, LogSource module, rsyslog snippets, pvesh script, gateway config, dashboard JSON, alert templates, runbooks, metrics reference, alerts and dashboard example.
