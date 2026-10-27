# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this folder is

A project to get Proxmox VE metrics and logs into LogicMonitor, metrics via **OpenTelemetry** and
logs via native syslog, with the Syslog LogSource as its one LogicModule. There is still no code,
build or test harness -- only documents. The metrics path is built and verified in the lab
(2026-10-27); the logs path is not started.

- `proxmox-otel-spec.md` -- the design (v3): decisions (section 2), metrics (4), logs (5), repo
  layout (10), build (11), deployment (12), open risks (13).
- `proxmox-otel-test-plan.md` -- phased lab plan (Phase 0-7). Phases 0-3 and the metrics half of 5
  have run; 4 (syslog), 6 (scale) and 7 (Exchange packaging) have not.
- `docs/` -- end-user runbooks and references:
  - `deploy-lm-otel-collector-docker.md` -- the metrics gateway, start to finish (verified; build it first).
  - `enable-pve-otel-metrics.md` -- the PVE metric server pointed at the gateway (GUI verified, CLI not).
  - `pve-otel-metrics-reference.md` -- metric names, labels, units and starter PromQL as seen in LM.
  - `pve-otel-alerts-dashboard-example.md` -- the lab's alert rules and dashboard, with portal naming rules.
  - `pve-direct-to-lm-portal.md` -- direct push; **does not work today**, kept for when PVE adds protobuf.

Unverified steps in the docs are marked ⚠ and tied to a test-plan phase; keep that convention, and
update a doc's `Status:` line when a lab run confirms or breaks it.

Its own git repository: github.com/PhilTheITGuy-USA/Proxmox-OTEL (private). The name follows the
workspace's `<Product>-<Method>` exception for non-LogicModule projects (see `../CLAUDE.md`).

Before every push, run `python ../lab-scrub/check.py` from the repo root (workspace rule). Docs
name the portal as `<portalname>.logicmonitor.com`, never the real portal -- `check.py` blocks any
`lm*.logicmonitor.com` host. Real lab details go in the gitignored `CLAUDE.local.md`.

## Architecture

Two independent paths; either can be deployed alone. Keep them separate when building.

1. **Metrics: PVE -> LM OTel Collector gateway -> LM OTLP endpoint.** The gateway is required:
   PVE's plugin sends OTLP **JSON** only, LM's `/rest/api/v1/metrics` takes protobuf + Bearer and
   returns `415` to JSON (Proxmox Bugzilla #7592). One cluster-wide PVE metric server per cluster
   pushes plain HTTP to the gateway on 4318 with **no token and no headers**, and a `cluster`
   resource attribute typed as plain JSON in the GUI. The gateway (LM OTel Collector 7.0 in Docker,
   one per site) holds the Bearer token in an env file and exports with `otlphttp/lm`
   (`metrics_endpoint`); its pipeline is `metrics/pve: otlp -> memory_limiter, batch -> otlphttp/lm`
   (config in the gateway runbook, section 3.1).
   Series are flat metrics identified by labels, not LM resources: no resource tree, no topology.
   Alert rules and dashboard widgets are **portal objects made in Metrics Explorer**, not
   LogicModules, so they ship as documented steps and queries and are not linted.
2. **Logs: rsyslog -> LM Collector Syslog LogSource.** PVE nodes (and opted-in VMs/LXCs) forward
   syslog over UDP 514 to an LM Collector with syslog enabled; the LogSource does filters, log
   fields and resource mapping (IP/hostname). Unmapped logs go to the collector's backup resource.
   The log ingest API and the contrib `logicmonitor` exporter are documented fallbacks only.

A bootstrap script that creates LM resources per PVE node is **deferred** -- build it only if
Phase 4 shows mapping can't be done with existing resources or the backup resource. Topology is
also deferred (spec 4: would need a PropertySource + TopologySource reading the PVE API).

## Facts to respect in queries, alerts and dashboards

Verified in the lab unless marked (details and the open list in `docs/pve-otel-metrics-reference.md`):

- LM shows metric names **without unit suffixes** (`proxmox_node_uptime`, not `..._seconds`);
  counters keep `_total`. CPU values are **0-1 ratios** (`* 100`); sizes are bytes; uptime seconds.
- Labels: PVE itself sets `proxmox_cluster` and `node`; our resource attribute adds `cluster`.
  `vmid` is unique only within a cluster, and a migrating guest changes `node` -- aggregate by
  `proxmox_cluster` + `vmid`, never by `node`.
- Node PSI is not sent (guests only). Still open: push interval, I/O counter units, node CPU time
  counters, shared-storage behaviour, node-down (absent series) and storage-inactive alerts.
- Explore's table rounds to whole numbers (a 0.12 ratio shows `0`) and its `ms` y-axis label is a
  default, not a unit.
- Portal naming: alert group names take no spaces; alert rule names take no spaces or hyphens
  (use underscores). Dashboard and widget names may have spaces.

## Constraints

- Owner's rule: OTEL should create the monitored entities wherever possible (metrics need no
  resources; logs map to existing resources or the backup resource).
- Syslog is **UDP 514 only** (owner's decision). Loss while the collector is unreachable or
  overloaded is accepted -- measure it, don't design around it with TCP or queues.
- Syslog LogSource: collector >= EA-35.200 to apply to a collector, >= 37.300 for OR mapping,
  >= 40.700 for complex filters; one Syslog LogSource per collector; no Auto-Balanced Collector Groups.
- Log ingest API limits, if the fallback is used: 8 MB/payload, 1 MB/event, +/-3 h timestamp window.
- Direct route only: PVE `otel-headers` / `otel-resource-attributes` are base64 JSON, max 1024 chars
  each on the CLI (the GUI takes plain JSON), and PVE stores the token base64 in
  `/etc/pve/status.cfg`, readable by cluster admins. The gateway route avoids both.
- The Syslog LogSource is a LogicModule and follows the module conventions (lint `dist/`); other
  content ships as companion docs (Exchange acceptance open, spec 9). The `logicmonitor-modules`
  skill documents script LogSources only -- capture the syslog LogSource export from a portal and
  add it to the skill.

## Build and checks (planned -- `build/build.py` and the code folders don't exist yet)

Planned layout (spec 10): `pve/` (pvesh script, rsyslog snippets), `collector/` (gateway config,
compose), `logsource/`, `lm/`, `build/`, `dist/`, `results.md`. `build/build.py` assembles `dist/`
from `logsource/` (module JSON -> `dist/*.json`) and `lm/` (dashboards -> `dist/dashboards/*.json`).
From the repo root:

```sh
python ../logicmonitor-modules-skill/skill/scripts/lm_lint_module.py dist
python ../logicmonitor-modules-skill/skill/scripts/lm_dashboards.py check dist/dashboards/*.json --modules dist
```

`--modules` globs only the top level of a directory, so module JSON must sit directly in `dist/`.

## Working the test plan

Run phases in order and don't advance until a phase's exit criteria are met. `results.md` is meant
to be the single place verification status is recorded, but it doesn't exist yet: today each run is
recorded as a **Result** line under its phase in the test plan, mirrored in the docs' `Status:`
lines. Next up is Phase 4 (syslog), plus triggering the Phase 5 alert rules. The lab is a 2-node
PVE 9.x cluster with VM and LXC guests; collectors run in Docker Desktop on Windows.

Any Syslog LogSource, LogicMonitor REST API, dashboard JSON or PropertySource work here still goes
through the `logicmonitor-modules` skill and its checks, per `../CLAUDE.md`.

## References

Official Proxmox VE sources -- prefer these over community reports when verifying PVE behaviour:

- https://pve.proxmox.com/pve-docs/ -- admin guide and man pages (metric server, `pvesh`)
- https://pve.proxmox.com/wiki/Main_Page -- Proxmox VE wiki
- https://pve.proxmox.com/pve-docs/api-viewer/ -- PVE REST API viewer
