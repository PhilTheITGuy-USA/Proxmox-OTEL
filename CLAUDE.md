# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this folder is

A design-stage project to get Proxmox VE metrics and logs into LogicMonitor via **OpenTelemetry**,
not via LogicModules. There is no code, build or test harness yet -- only two documents:

- `proxmox-otel-spec.md` -- the design: decisions, architecture, open risks (section 9).
- `proxmox-otel-test-plan.md` -- phased lab plan (Phase 0-7) that resolves those risks.

Its own git repository: github.com/PhilTheITGuy-USA/Proxmox-OTEL (private). The name follows the
workspace's `<Product>-<Method>` exception for non-LogicModule projects (see `../CLAUDE.md`).

## Architecture (from the spec)

Three independent paths -- keep them separate when building:

1. **Metrics, direct:** PVE's cluster-wide OpenTelemetry metric server (`pvesh create
   /cluster/metrics/server/<id> --type opentelemetry ...`) pushes OTLP/HTTP + gzip + Bearer straight
   to LM's OTLP metrics endpoint. No collector, no LM resources: series are flat, identified by
   attributes (`node`, `vmid`, `type`, plus a `cluster` resource attribute set per cluster).
2. **Logs, via collector:** node journald -> rsyslog `imjournal`/`omfwd` (TCP, RFC5424) ->
   `otelcol` `syslog` receiver -> `logicmonitor` exporter -> LM log ingest.
3. **Resource bootstrap:** an idempotent Python script (LM REST API) that lists nodes from the PVE
   API and creates one LM resource per PVE node (`system.displayname`, `pve.cluster`, `pve.node`).
   It exists only because LM log ingest needs a pre-existing resource (error 4001, no auto-create).
   Guests never get LM resources.

Optional at scale: one collector gateway per cluster/site in front of LM for batching, attribute
enrichment and cardinality control (`otlphttp` exporter).

## Constraints to respect

- Owner's rule: OTEL should create the monitored entities wherever possible; the bootstrap script is
  the documented exception for logs.
- Known PVE quirks to account for in queries/dashboards: `*_cpu_percent` is a 0-1 ratio; node PSI is
  not emitted. Treat these as unverified until Phase 3 of the test plan confirms them.
- LM log limits: 8 MB/payload, 1 MB/event, +/-3 h timestamp window, 240K req/min.
- Secrets: endpoint and tokens are recorded during Phase 0 but never committed; the Exchange package
  carries placeholders only. Note that PVE stores `otel-headers` (the bearer token) base64-encoded in
  `/etc/pve/status.cfg`, readable by cluster admins.
- Exchange packaging form for non-LogicModule content is an open item (spec section 8) -- don't
  assume the LogicModule packaging conventions apply.

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
