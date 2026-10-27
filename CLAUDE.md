# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this folder is

A design-stage project to get Proxmox VE metrics and logs into LogicMonitor via **OpenTelemetry**,
with the Syslog LogSource as its one LogicModule. There is no code, build or test harness yet --
only documents:

- `proxmox-otel-spec.md` -- the design (v2): decisions, architecture, repo layout (section 10),
  build (11), open risks (13).
- `proxmox-otel-test-plan.md` -- phased lab plan (Phase 0-7) that resolves those risks.
- `docs/` -- runbooks: `enable-pve-otel-metrics.md` (PVE metric server, LM endpoint and token),
  `pve-direct-to-lm-portal.md` (end-to-end direct push, no gateway) and
  `deploy-lm-otel-collector-docker.md` (optional metrics gateway). All untested; their unverified
  steps are marked with a warning sign and tied to a test-plan phase.

Its own git repository: github.com/PhilTheITGuy-USA/Proxmox-OTEL (private). The name follows the
workspace's `<Product>-<Method>` exception for non-LogicModule projects (see `../CLAUDE.md`).

Before every push, run `python ../lab-scrub/check.py` from the repo root (workspace rule). Runbooks
name the portal as `<account>.logicmonitor.com`, never the real portal; real lab details go in a
gitignored `CLAUDE.local.md`.

## Architecture (from the spec, v2)

Two independent paths -- keep them separate when building:

1. **Metrics, direct:** PVE's cluster-wide OpenTelemetry metric server (`pvesh create
   /cluster/metrics/server/<id> --type opentelemetry ...`) pushes OTLP/HTTP + gzip + Bearer straight
   to LM's OTLP metrics endpoint. No collector, no LM resources: series are flat, identified by
   attributes (`node`, `vmid`, `type`, plus a `cluster` resource attribute set per cluster).
   Optional at scale: the LM OTel Collector as a per-site gateway (otlp receiver -> batch/resource/
   transform -> otlphttp) for batching, enrichment and cardinality control.
2. **Logs, native syslog:** PVE nodes (and chosen VMs/LXCs) forward syslog with rsyslog to an LM
   Collector with syslog enabled; a Syslog LogSource handles filters, log fields and resource mapping
   (IP/hostname). Unmapped logs fall back to the collector's backup resource. No custom OTel
   collector build. The contrib `logicmonitor` exporter and the log ingest API are documented
   fallbacks only.

A bootstrap script that creates LM resources per PVE node is **deferred** -- build it only if
Phase 4 shows mapping can't be done with existing resources or the backup resource.

## Constraints to respect

- Owner's rule: OTEL should create the monitored entities wherever possible (metrics need no resources;
  logs map to existing resources or the backup resource).
- Known PVE quirks to account for in queries/dashboards: `*_cpu_percent` is a 0-1 ratio; node PSI is
  not emitted. Treat these as unverified until Phase 3 of the test plan confirms them.
- Syslog LogSource: collector >= EA-35.200 to apply to a collector, >= 37.300 for OR mapping, >= 40.700
  for complex filters; one Syslog LogSource per collector; no Auto-Balanced Collector Groups.
- Syslog is **UDP 514 only** (owner's decision). Loss while the collector is unreachable or
  overloaded is accepted -- measure it, don't design around it with TCP or queues.
- Log ingest API limits, if it is used as the fallback: 8 MB/payload, 1 MB/event, +/-3 h timestamp
  window.
- PVE `otel-headers` / `otel-resource-attributes` are base64 JSON, max 1024 chars each (official API
  schema); the admin guide itself documents no OTel options, so push interval and metric names stay
  unverified until Phase 2/3.
- Secrets: endpoint and tokens are recorded during Phase 0 but never committed; the Exchange package
  carries placeholders only. Note that PVE stores `otel-headers` (the bearer token) base64-encoded in
  `/etc/pve/status.cfg`, readable by cluster admins.
- The Syslog LogSource is a LogicModule and follows the module conventions (lint `dist/`); other
  content ships as companion docs. Acceptance of non-module content is an open item (spec section 9).
  The `logicmonitor-modules` skill documents script LogSources only -- capture the syslog LogSource
  export from a portal and add it to the skill.

## Build and checks (planned -- `build/build.py` doesn't exist yet)

`build/build.py` assembles `dist/` from `logsource/` (module JSON -> `dist/*.json`) and `lm/`
(dashboards -> `dist/dashboards/*.json`). From the repo root:

```sh
python ../logicmonitor-modules-skill/skill/scripts/lm_lint_module.py dist
python ../logicmonitor-modules-skill/skill/scripts/lm_dashboards.py check dist/dashboards/*.json --modules dist
```

`--modules` globs only the top level of a directory, so module JSON must sit directly in `dist/`.

## Working the test plan

Run phases in order and don't advance until a phase's exit criteria are met. Results go in
`results.md` (not yet created), which is the only place verification status is recorded. Planned
outputs: the Syslog LogSource module, rsyslog snippets, the `pvesh` metric-server script, the gateway
config, dashboard JSON, alert templates (node down / storage >85% / guest CPU sustained high), and
the runbooks. The lab is a 2-node PVE cluster with VM and LXC guests, with collectors run in Docker
Desktop.

Any Syslog LogSource, LogicMonitor REST API, dashboard JSON or PropertySource work here still goes
through the `logicmonitor-modules` skill and its checks, per `../CLAUDE.md`.

## References

Official Proxmox VE sources -- prefer these over community reports when verifying PVE behaviour:

- https://pve.proxmox.com/pve-docs/ -- admin guide and man pages (metric server, `pvesh`)
- https://pve.proxmox.com/wiki/Main_Page -- Proxmox VE wiki
- https://pve.proxmox.com/pve-docs/api-viewer/ -- PVE REST API viewer
