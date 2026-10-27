# Proxmox VE OTel metrics in LogicMonitor: reference

Purpose: what Proxmox VE (PVE) sends over OTLP, as seen in LogicMonitor (LM), and how to query it. Captured from the lab on 2026-10-27 (PVE 9.2.2 and 9.2.3, two nodes, 1 QEMU guest and 7 LXC guests) through the LM OTel Collector gateway (see `deploy-lm-otel-collector-docker.md`).

Status: names and labels below were read from LM Explore. Items marked ⚠ are unverified. Proxmox publishes no official metric list (a Proxmox staff reply on the forum says the values are kernel metrics forwarded as they are), so treat names as observed behaviour that can change between PVE versions.

## 1. How the data looks in LM
- Metrics are **flat**: a metric name plus attributes (labels). They are not LM resources, so they don't appear in the resource tree and have no topology (see section 6).
- LM shows names **without unit suffixes**. Community write-ups list `proxmox_node_uptime_seconds`; LM shows `proxmox_node_uptime`. Counters keep `_total`. Units therefore have to be documented per metric (section 4).
- Explore's table view **rounds values to whole numbers**: a CPU ratio of 0.12 shows as `0`. Use the line graph, or multiply (`* 100`).
- The y-axis label `ms` in Explore is a default, not the real unit.
- Display caps: Explore's list of metric names appears capped at 100 entries, and the table shows 10 rows per page. Narrow queries or page through to see everything.

## 2. Labels
| Label | Where | Meaning |
|---|---|---|
| `node` | node, guest | PVE node name (the node a guest currently runs on) |
| `proxmox_node` | node, guest | same as `node` |
| `proxmox_cluster` | all | PVE cluster name, set by PVE itself (so clusters separate without our own attribute) |
| `cluster` | all | our resource attribute from the PVE metric server (`{"cluster":"lab"}`); use it for site or environment naming |
| `type` | guest | `qemu` or `lxc` |
| `vmid` | guest | guest ID; **unique only within a cluster**, so always pair with `proxmox_cluster` |
| `name` | guest | guest name |
| `service_name` | all | `proxmox-ve` |
| `service_version` | all | PVE version per node, for example `9.2.3/<build>`; nodes can differ during upgrades |

A guest that migrates changes its `node` label, which starts a new series; queries should aggregate by `proxmox_cluster` and `vmid`, not by `node`.

## 3. Metric families
### 3.1 Node (`proxmox_node_*`, label `node`)
Seen in the lab (53 names, confirmed by a count query):
- **CPU** `cpustat_*`: `cpu`, `cpus` (core count), `used`, `sum`, `total`, `avg1`, `avg5`, `avg15`, and the time counters `user_total`, `system_total`, `nice_total`, `idle_total`, `iowait_total`, `wait_total`, `irq_total`, `softirq_total`, `steal_total`, `guest_total`, `guest_nice`.
- **Memory** `memory_*`: `memtotal`, `memused`, `memfree`, `memavailable`, `memshared`, `swaptotal`, `swapused`, `swapfree`, `arcsize`, `arcmax`, `arcmin` (ZFS ARC).
- **Network:** `network_receive_total`, `network_transmit_total`.
- **Uptime:** `uptime`.
- **Root filesystem** `blockstat_*`: `blocks`, `bfree`, `bavail`, `used`, `per`, `fper`, `files`, `ffree`, `favail`, `fused`, plus `user_*` and `su_*` variants.
- Not sent: node-level pressure (PSI); only guests have PSI.
- Known naming quirks (community, ⚠ verify): `cpustat_cpu` versus `cpustat_used`, and `cpustat_sum`, which a Proxmox staff reply calls a leftover value never written; inode metrics are counts despite their names.

### 3.2 Guests (`proxmox_vm_*`, labels `node`, `type`, `vmid`, `name`; QEMU and LXC)
22 names, one series per guest (8 in the lab):
- **CPU:** `cpu`, `cpus`
- **Memory:** `mem`, `maxmem`, `swap`, `maxswap` (LXC only in the lab, 7 series)
- **Disk:** `disk`, `maxdisk`, `diskread_total`, `diskwrite_total`
- **Network:** `netin_total`, `netout_total`
- **Pressure (PSI):** `pressurecpusome`, `pressurecpufull`, `pressureiosome`, `pressureiofull`, `pressurememorysome`, `pressurememoryfull`
- **Uptime:** `uptime`
- **QEMU only (1 series each in the lab):** `balloon`, `freemem`, `memhost`, `ballooninfo_*` (actual, max_mem, total_mem, free_mem, last_update, page faults, swap in/out), and `blockstat_<disk>_*` per attached disk (for example `efidisk0`, `pflash0`): read/write operations, bytes, merged, timing, failed and invalid operation counters.

### 3.3 Storage (`proxmox_storage_*`)
`total`, `used`, `avail`, `active`, `enabled`, `shared`. Storage label names ⚠ not yet captured; check before writing alert filters.

## 4. Units and scale
| Metric | Observed or expected | Status |
|---|---|---|
| `proxmox_vm_cpu` | 0–1 ratio of the guest's allotted CPU. Lab: graph ≈ 0.12–0.13 versus the PVE UI 13.34% for the same guest at the same time. Multiply by 100 for percent. | verified (one guest, one moment) |
| `proxmox_vm_mem`, `maxmem`, `disk`, `maxdisk` | bytes | ⚠ confirm against the PVE UI |
| `proxmox_vm_netin_total`, `netout_total`, `diskread_total`, `diskwrite_total` | cumulative byte counters; use `rate()` | ⚠ |
| `proxmox_vm_uptime`, `proxmox_node_uptime` | seconds | ⚠ |
| `proxmox_node_cpustat_*_total` | cumulative CPU time counters (seconds in community lists) | ⚠ |
| `proxmox_node_memory_*` | bytes | ⚠ |
| `proxmox_storage_*` sizes | bytes | ⚠ |
| `proxmox_storage_active`, `enabled`, `shared` | 0/1 flags | ⚠ |
| PSI `pressure*` | percentage-style values (community) | ⚠ |

## 5. Starter queries (PromQL-compatible; run in Explore)
Only the first has been confirmed on lab data. The rest are candidates to run and check against the PVE UI before they become dashboard widgets or alert rules.
```
# Guest CPU % (verified scale)
proxmox_vm_cpu * 100

# Guest CPU % by cluster and guest
avg by (proxmox_cluster, vmid, name) (proxmox_vm_cpu) * 100          # ⚠

# Guest memory used %
proxmox_vm_mem / proxmox_vm_maxmem * 100                             # ⚠

# Guest network throughput (bytes/s)
rate(proxmox_vm_netin_total[5m])                                     # ⚠

# Storage used %
proxmox_storage_used / proxmox_storage_total * 100                   # ⚠

# Storage inactive
proxmox_storage_active == 0                                          # ⚠

# Node memory used % (check how memused treats ZFS ARC first)
proxmox_node_memory_memused / proxmox_node_memory_memtotal * 100     # ⚠

# Guests per node and type
count by (proxmox_cluster, node, type) (proxmox_vm_uptime)           # ⚠
```
Alert candidates (to build and trigger in test-plan Phase 5): node down (series absent), storage used over 85%, guest CPU sustained high, storage inactive.

Always group or filter by `proxmox_cluster` as well as `vmid` in multi-cluster estates.

## 6. Topology
Not possible from these metrics alone. LM topology maps are built from resources that carry external resource IDs (ERIs), linked by TopologySource LogicModules; OTel metrics create no resources. Options: model PVE with a PropertySource (ERIs) plus a TopologySource that reads the PVE API (guest → node → cluster, guest → storage), or use label-grouped dashboards as a logical view without maps or root-cause suppression. See the design spec.

## 7. Open checks
1. Count names to confirm completeness: node is confirmed (53). Still to check with `count(count by (__name__) ({__name__=~"proxmox_storage_.*"}))` (6 seen).
2. Capture storage labels.
3. Confirm units per section 4 against the PVE UI (memory, disk, network, uptime, node CPU).
4. Measure the real push interval (series point spacing in the graph).
5. Re-check names after a PVE upgrade.

## References
- [OpenTelemetry Metrics Overview (LM)](https://www.logicmonitor.com/support/opentelemetry-metrics-overview)
- [Topology Mapping Overview (LM)](https://www.logicmonitor.com/support/topology-mapping-overview)
- [Proxmox forum: questions about OTEL metrics listing](https://forum.proxmox.com/threads/questions-about-otel-metrics-listing.184949/)
- [Virtually Boring: Proxmox 9 observability](https://www.virtuallyboring.com/proxmox-observability/)
- [CardinalHQ: Proxmox OTLP metric server](https://docs.cardinalhq.io/how-to/proxmox-otlp-metric-server)
