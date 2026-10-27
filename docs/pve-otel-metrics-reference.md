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
| `storage` | storage | PVE storage ID |
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
`total`, `used`, `avail`, `active`, `enabled`, `shared`. Labels: `storage` (the PVE storage ID, for example `local`, `local-lvm`, `media01-usb`) plus the common labels; there is no `type` label. One series per node per storage, so a storage visible on several nodes (shared, or the same ID such as `local` on every node) appears once per node: group by `proxmox_cluster, node, storage` for per-node views, and use `max by (proxmox_cluster, storage)` to avoid double counting shared storage ⚠ (shared behaviour not yet checked).

## 4. Units and scale
| Metric | Observed or expected | Status |
|---|---|---|
Verification on 2026-10-27 compared LM Explore with the PVE Datacenter → Search view (the same moment, within sampling differences).

| Metric | Observed or expected | Status |
|---|---|---|
| `proxmox_vm_cpu` | 0–1 ratio of the guest's allotted CPU; multiply by 100. `nvr01`: graph ≈ 0.12–0.13 versus PVE 13.34% | verified (one guest) |
| `proxmox_vm_mem`, `proxmox_vm_maxmem` | bytes. `nvr01`: mem 1,051,820,032 (1003.1 MiB; PVE 1003.39 MiB), maxmem 4,294,967,296 (4 GiB); `mem / maxmem * 100` = 24.5%, PVE 24.5% | verified |
| `proxmox_vm_disk`, `proxmox_vm_maxdisk` | bytes. `nvr01`: `disk / maxdisk * 100` = 15.905%, PVE 15.9% | verified |
| `proxmox_vm_uptime`, `proxmox_node_uptime` | seconds. `nvr01`: `uptime / 3600` = 202.7 h (8 d 10:42), PVE 8 days 10:41 | verified for guests; node uptime ⚠ |
| `proxmox_node_memory_memused` / `memtotal` | bytes; `* 100` gives percent. Node-A ≈ 59%, Node-B ≈ 48%, PVE 59.1% and 47.9% (lab storage is LVM, not ZFS, so ZFS ARC handling is not tested) | verified |
| `proxmox_node_cpustat_cpu` | 0–1 ratio; `* 100` gives percent. Node-A ≈ 11%, Node-B ≈ 1.3%, PVE 10.6% and 1.3% | verified |
| `proxmox_node_cpustat_used` | **not** a usage ratio (about 3,000,000,000 for Node-A); do not use for CPU % | observed |
| `proxmox_storage_used`, `total`, `avail` | bytes. `local-lvm` Node-A: 116,923,328,731 = 116.92 GB, PVE 116.92 of 362.78 (32.2%); `media01-usb` ≈ 83%, `local` ≈ 11% on Node-A, matching PVE | verified |
| `proxmox_vm_netin_total`, `netout_total`, `diskread_total`, `diskwrite_total` | cumulative counters; `rate(...[5m])` works (`nvr01` netin ≈ 315,000 per second) | rate verified; unit (bytes) ⚠ |
| `proxmox_node_cpustat_*_total` | cumulative CPU time counters (seconds in community lists) | ⚠ |
| `proxmox_storage_active`, `enabled`, `shared` | 0/1 flags | ⚠ |
| PSI `pressure*` | percentage-style values (community) | ⚠ |

## 5. Starter queries (PromQL-compatible; run in Explore)
Queries marked ✔ were run on lab data and matched the PVE UI. Those marked ⚠ are candidates that still need running before they become dashboard widgets or alert rules.
```
# Guest CPU %                                                         ✔
proxmox_vm_cpu * 100

# Guest memory used %                                                 ✔
proxmox_vm_mem / proxmox_vm_maxmem * 100

# Guest disk used %                                                   ✔
proxmox_vm_disk / proxmox_vm_maxdisk * 100

# Guest uptime in hours                                               ✔
proxmox_vm_uptime / 3600

# Guest network in (per second)                                       ✔ rate works
rate(proxmox_vm_netin_total[5m])

# Node CPU %                                                          ✔
proxmox_node_cpustat_cpu * 100

# Node memory used %                                                  ✔
proxmox_node_memory_memused / proxmox_node_memory_memtotal * 100

# Storage used % (labels match, so the division pairs per node and storage)   ✔
proxmox_storage_used / proxmox_storage_total * 100

# Worst node per storage (aggregation works; only for shared storage, because
# non-shared storages with the same ID on several nodes, like `local`, would merge)   ✔
max by (proxmox_cluster, storage) (proxmox_storage_used / proxmox_storage_total * 100)

# Storage inactive                                                    ⚠
proxmox_storage_active == 0

# Guests per node and type                                            ⚠
count by (proxmox_cluster, node, type) (proxmox_vm_uptime)
```
Alert candidates (to build and trigger in test-plan Phase 5): node down (series absent), storage used over 85%, guest CPU sustained high, storage inactive.

Always group or filter by `proxmox_cluster` as well as `vmid` in multi-cluster estates.

## 6. Topology
Not possible from these metrics alone. LM topology maps are built from resources that carry external resource IDs (ERIs), linked by TopologySource LogicModules; OTel metrics create no resources. Options: model PVE with a PropertySource (ERIs) plus a TopologySource that reads the PVE API (guest → node → cluster, guest → storage), or use label-grouped dashboards as a logical view without maps or root-cause suppression. See the design spec.

## 7. Open checks
1. Name counts are confirmed by count queries: node 53, guest 22, storage 6 (QEMU balloon and block-stat names are in addition to the guest 22).
2. Check how shared storage is reported (one series per node?) before building the storage alert.
3. Units still to confirm against the PVE UI: network and disk I/O counters (bytes), node uptime, node CPU time counters, PSI values, and the `active`/`enabled`/`shared` flags.
4. Measure the real push interval (series point spacing in the graph).
5. Re-check names after a PVE upgrade.

## References
- [OpenTelemetry Metrics Overview (LM)](https://www.logicmonitor.com/support/opentelemetry-metrics-overview)
- [Topology Mapping Overview (LM)](https://www.logicmonitor.com/support/topology-mapping-overview)
- [Proxmox forum: questions about OTEL metrics listing](https://forum.proxmox.com/threads/questions-about-otel-metrics-listing.184949/)
- [Virtually Boring: Proxmox 9 observability](https://www.virtuallyboring.com/proxmox-observability/)
- [CardinalHQ: Proxmox OTLP metric server](https://docs.cardinalhq.io/how-to/proxmox-otlp-metric-server)
