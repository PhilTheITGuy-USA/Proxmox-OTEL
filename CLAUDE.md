# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this folder is

A design-stage project to get Proxmox VE metrics and logs into LogicMonitor via **OpenTelemetry**,
not via LogicModules. There is no code, build or test harness yet -- only two documents:

- `proxmox-otel-spec.md` -- the design: decisions, architecture, open risks (section 9).
- `proxmox-otel-test-plan.md` -- phased lab plan (Phase 0-7) that resolves those risks.

Its own git repository: github.com/PhilTheITGuy-USA/Proxmox-OTEL (private). The name follows the
workspace's `<Product>-<Method>` exception for non-LogicModule projects (see `../CLAUDE.md`).

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
  for complex filters; one Syslog LogSource per collector; no Auto-Balanced Collector Groups. Docs
  state UDP 514; TCP is unconfirmed. (Log ingest API limits, if used as fallback: 8 MB/payload,
  1 MB/event, +/-3 h timestamp window.)
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

## Working the test plan

Run phases in order and don't advance until a phase's exit criteria are met. Results go in
`results.md` (not yet created). Planned outputs: `results.md`, collector configs, the bootstrap
script, dashboard JSON, alert templates (node down / storage >85% / guest CPU sustained high), and a
runbook. The lab is a 2-node PVE cluster with VM and LXC guests, with collectors run in Docker Desktop.

Any LogicMonitor REST API, dashboard JSON or PropertySource work here still goes through the
`logicmonitor-modules` skill and its checks, per `../CLAUDE.md`.

## References

Official Proxmox VE sources -- prefer these over community reports when verifying PVE behaviour:

- https://pve.proxmox.com/pve-docs/ -- admin guide and man pages (metric server, `pvesh`)
- https://pve.proxmox.com/wiki/Main_Page -- Proxmox VE wiki
- https://pve.proxmox.com/pve-docs/api-viewer/ -- PVE REST API viewer
