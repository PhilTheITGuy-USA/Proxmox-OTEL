# Example alerts and dashboard for Proxmox VE OTel metrics in LogicMonitor

Purpose: a worked example for end users. It builds one alert rule and one dashboard from the PVE metrics that arrive through the LM OTel Collector gateway (see `deploy-lm-otel-collector-docker.md`; metric names and scales are in `pve-otel-metrics-reference.md`). Built and checked in the lab on 2026-10-27. Replace `<portalname>` with your portal.

Where: **More → OTel Metrics** has two tabs, **Metrics Explorer** and **Alert Configurations**.

## 1. Naming rules (the portal rejects anything else)
- Alert group and rule names: **no spaces, no hyphens in rule names.** Lab results: group `Proxmox-VE` was accepted, `Proxmox VE` was not; rule `PVE_Storage_Used_Over_85` was accepted, a hyphenated rule name was not. Use underscores in rule names.
- Dashboard and widget names may contain spaces.

## 2. Alert rule: storage used over 85%
1. Metrics Explorer → **Raw Query** → enter `proxmox_storage_used / proxmox_storage_total * 100` → **Run query**. Confirm the values match the PVE UI (lab: `local-lvm` 32%, `media01-usb` about 83%).
2. Click **Create Alert Rule** above the graph. Or open **Alert Configurations**, create an **Alert Group** (`Proxmox-VE`), then add a rule inside it.
3. Rule name `PVE_Storage_Used_Over_85`, query as above, condition **above 85**, severity **Critical**.
4. Save, then confirm the rule appears under the group in **Alert Configurations**.

Other rules to build the same way (run each query first, then set the threshold):

| Rule | Query | Condition | Severity |
|---|---|---|---|
| `PVE_Node_Memory_High` | `proxmox_node_memory_memused / proxmox_node_memory_memtotal * 100` | above 90 | Warning |
| `PVE_Node_CPU_High` | `proxmox_node_cpustat_cpu * 100` | above 90 | Warning |
| `PVE_Guest_CPU_High` | `proxmox_vm_cpu * 100` | above 90 | Warning |

Not yet verified: storage-inactive (`proxmox_storage_active`) and node-down (absent series) alerts.

## 3. Dashboard: "Proxmox VE OTel"
Widgets are added from Metrics Explorer; the dashboard itself is created beforehand under **Dashboards** (an empty one is fine).

Widgets built (query, then widget name):
| Widget name | Query |
|---|---|
| Node CPU percent | `proxmox_node_cpustat_cpu * 100` |
| Node memory percent | `proxmox_node_memory_memused / proxmox_node_memory_memtotal * 100` |
| Storage used percent | `proxmox_storage_used / proxmox_storage_total * 100` |
| Guest CPU percent | `proxmox_vm_cpu * 100` |

Steps for each widget:
1. Metrics Explorer → **Raw Query** → enter the query → **Run query**.
2. Click **Add to Dashboard** (above the graph, next to **Create Alert Rule**).
3. Type the **Widget name**.
4. Click **Add record** (the **+**). In the new row choose the **Dashboard Group** first, then the **Dashboard**.
5. Click the row's **save (disk) icon**, not the trash icon beside it. The **Save** button at the bottom stays greyed out until the row is confirmed.
6. Click **Save**, open the dashboard and check the widget appears.

## 4. Quirks to expect
- The **Dashboard** list says "No options" until a **Dashboard Group** is chosen.
- Widgets show an `ms` y-axis label. That is a default, not the unit; these queries return percent.
- Explore's table view rounds to whole numbers; use the graph.
- Page layout shifts after a query runs, so click by element rather than fixed screen position if you script this.
- Guest series are one per guest and per node; use `proxmox_cluster` plus `vmid` when you filter in multi-cluster estates.
