# sdwan-overlay-details-report

Scans a folder of Cisco SD-WAN **admin-tech bundles** from control plane
components (vManage, vSmart, vBond) and prints five summary tables covering
device identity, certificates, WAN interfaces, cluster topology, cluster service
inventory, and interface addressing — all from a single command, no external
dependencies.

---

## Requirements

- Python 3.10 or later (uses the `|` union type hint syntax)
- No third-party packages — stdlib only

---

## Quick start

```bash
# Pass the folder path as an argument
python sdwan-overlay-details-report.py /path/to/admin-techs/

# Or run without arguments — the script will prompt you interactively
python sdwan-overlay-details-report.py
```

### What the script looks for

Place all admin-tech bundles inside a single parent folder.  Each bundle can be
either an **extracted directory** or a **compressed archive**:

```
/path/to/admin-techs/
├── 1.1.1.2-DC-vBond-20260228-200114-admin-tech.tar/    ← extracted directory
├── 1.1.1.3-DC-vsmart-20260228-193743-admin-tech.tar/
├── 1.2.1.1-DR-vManage-20260228-225456-admin-tech.tar/
├── 1.2.1.1-DR-vManage-20260228-225456-admin-tech.tar.gz ← compressed archive
└── ...
```

> The folder or archive name must contain **`admin-tech`** (case-insensitive)
> to be picked up by the scanner.

---

## Output

The script prints five tables to stdout, sorted by personality then system-IP.
Redirect to a file to save the output:

```bash
python sdwan-overlay-details-report.py /path/to/admin-techs/ > results.txt
```

---

### Table 1 — Device Overview

A quick-glance identity table for every device found.

| Column | Source | Description |
|---|---|---|
| System-IP | `config` | Overlay system IP address |
| Personality | `config` / `sysmgrd` fallback | `vmanage`, `vsmart`, or `vbond` |
| Device Model | `config` | Hardware / VM platform identifier |
| SP-Org-Name | `config` | Service Provider organisation name |
| Org-Name | `config` | Customer organisation name |
| Version | `sysmgrd` | Software version string (e.g. `20.6.5.5`) |

> **Personality fallback (firmware ≤ 20.6.x):** older bundles do not include a
> `personality` field in the `config` file.  When the field is absent the script
> reads the `Personality:` line from the `show system status` section of the
> `sysmgrd` file instead.  This applies to vManage, vSmart, and vBond bundles.

```
=== Table 1 – Device Overview ===
+-----------+-------------+--------------+-------------+------------------+---------+
| System-IP | Personality | Device Model | SP-Org-Name | Org-Name         | Version |
+-----------+-------------+--------------+-------------+------------------+---------+
| 1.1.1.2   | vbond       | vedge-cloud  | SP-Acme     | Acme-Corp        | 20.9.3  |
| 1.2.1.1   | vmanage     | vmanage      | SP-Acme     | Acme-Corp        | 20.6.5  |
+-----------+-------------+--------------+-------------+------------------+---------+
```

---

### Table 2 — Certificate & WAN Interfaces

Certificate health and active network interfaces for every device.

| Column | Source | Description |
|---|---|---|
| System-IP | `config` | Overlay system IP |
| Personality | `config` / `sysmgrd` fallback | Device role |
| Cert-Status | `vdaemon` | `Installed`, `Not Installed`, etc. |
| Cert-Validity | `vdaemon` | `Valid` or `Invalid` |
| Cert-Expiry | `vdaemon` | Certificate expiry date |
| Serial-Num | `vdaemon` | Device serial / certificate serial |
| WAN Interfaces | `vdaemon` / CFGMGR | One interface per line — `iface: IP/prefix` |

**WAN Interfaces** column behaviour:
- For **vManage**: populated from the `show control local-properties` WAN table
  in the `vdaemon` file (public IP per interface).
- For **vBond / vSmart**: populated from the `CFGMGR RTM Interface Database`
  section of the `config` file. Interfaces with no configured address (`::/0`)
  are omitted.

```
=== Table 2 – Certificate & WAN Interfaces ===
+-----------+-------------+-------------+---------------+--------------------------+------------------------------------------+-------------------------------+
| System-IP | Personality | Cert-Status | Cert-Validity | Cert-Expiry              | Serial-Num                               | WAN Interfaces                |
+-----------+-------------+-------------+---------------+--------------------------+------------------------------------------+-------------------------------+
| 1.1.1.2   | vbond       | Installed   | Valid         | Jul 29 14:29:11 2026 GMT | 72D813E6F822A6C92C0D09792E9067E3390A2503 | ge0_0: 118.201.99.104/25      |
|           |             |             |               |                          |                                          | system: 1.1.1.2/32            |
|           |             |             |               |                          |                                          | eth0: 10.1.60.133/24          |
|           |             |             |               |                          |                                          | loopback65528: 192.168.0.1/24 |
| 1.2.1.1   | vmanage     | Installed   | Valid         | Mar 01 10:00:00 2027 GMT | AABBCCDDEEFF00112233445566778899AABBCCDD | eth0: 94.97.99.116            |
+-----------+-------------+-------------+---------------+--------------------------+------------------------------------------+-------------------------------+
```

---

### Table 3 — vManage Cluster / Standalone Classification

**vManage devices only.** Determines whether each vManage is part of a cluster
or running standalone, and annotates every cluster peer IP with its owner
system-IP (resolved via the CFGMGR RTM interface data — same source as Table 4).

Each vManage gets one row per cluster peer; the fixed identity columns are only
printed on the first line to keep the output readable.

| Column | Description |
|---|---|
| System-IP | Overlay system IP of this vManage |
| Personality | Always `vmanage` |
| Deployment | `Cluster`, `Standalone`, or `Standalone (SD-AVC)` |
| Mode | `SingleTenant` or `MultiTenant` |
| Cluster-ID | Human-readable cluster name (if present) |
| Node-ID | This node's index within the cluster (`0`, `1`, `2`) |
| Peer IPs | Out-of-band interface IP of every cluster member. `(self)` marks this node. ✓ = admin-tech present in collection + owner system-IP. ✗ = admin-tech missing |
| Detection Notes | Which file and field produced the result |

#### Detection priority

The first source that yields a conclusive answer wins:

| Priority | Source | Logic |
|---|---|---|
| 1 | `opt/data/tech/clusterconfig` → `memberMap` | Entry count > 1 → Cluster |
| 2 | `opt/data/tech/config` → `vmanage-system-ip` field | Present → Cluster |
| 3 | `opt/web-app/etc/deployment` `deploymentmode` + `server_configs.json` hosts | `deploymentmode=Cluster` with multiple IPs → Cluster; single IP → check `sdavc.standalone` |
| 4 | `opt/web-app/etc/deployment` → `tenantmode` only | `MultiTenant` → Cluster; `SingleTenant` → assumed Standalone |

**SD-AVC standalone detection (firmware ≤ 20.6.x):**
When `deploymentmode=Cluster` but `server_configs.json` contains only one IP in
`application-server.hosts`, the node is likely running SD-AVC in standalone mode.
The `Deployment` column will show `Standalone (SD-AVC)` when `services.sdavc.standalone`
is `true` in `server_configs.json`.

**Non-Cluster deployment file display:**
When `deploymentmode` is absent or is not `Cluster`, the raw contents of
`opt/web-app/etc/deployment` are printed below the table for analyst review.

> **How peer IP lookup works:** peer IPs from `clusterconfig` (or
> `server_configs.json` hosts) are the out-of-band interface IPs visible in
> Table 4.  The script builds a `bare-IP → system-IP` map from the CFGMGR RTM
> interface database and looks each peer IP up in that map.  ✓ shows the
> matching system-IP; ✗ means the admin-tech for that node was not collected.

```
=== Table 3 – vManage Cluster / Standalone Classification ===
+-----------+-------------+------------+--------------+------------+---------+---------------------------------------+-----------------------------------------+
| System-IP | Personality | Deployment | Mode         | Cluster-ID | Node-ID | Peer IPs (idx: ip  ✓=...  ✗=...)     | Detection Notes                         |
+-----------+-------------+------------+--------------+------------+---------+---------------------------------------+-----------------------------------------+
| 1.1.20.1  | vmanage     | Cluster    | SingleTenant | N/A        | 0       | [0] 10.206.177.2 (self)               | deploymentmode=Cluster; 3 peer IP(s)    |
|           |             |            |              |            |         | [1] 10.206.177.5 ✓ 1.1.20.4          |                                         |
|           |             |            |              |            |         | [2] 10.206.177.6 ✗ admin-tech missing |                                         |
+-----------+-------------+------------+--------------+------------+---------+---------------------------------------+-----------------------------------------+
| 1.2.1.1   | vmanage     | Standalone | SingleTenant | N/A        | 0       | N/A                                   | tenantmode=SingleTenant (assumed stand…)|
+-----------+-------------+------------+--------------+------------+---------+---------------------------------------+-----------------------------------------+
```

#### Disaster Recovery block

Immediately after Table 3, if any vManage bundle contains the deployment file
(`opt/web-app/etc/deployment`), its raw contents are printed with a header
indicating whether DR is enabled:

```
=== Disaster Recovery configuration FOUND ===

  File: 1.2.1.1-DR-vManage-.../opt/web-app/etc/deployment

  deploymentmode=Cluster
  tenantmode=SingleTenant
  drenabled=true
```

---

### Table 3.5 — Cluster Service Inventory

**Cluster vManage nodes only.** Printed once per unique cluster (deduplicated
by cluster UUID) immediately after the DR block.

Rows are unique IP addresses found across all service entries in
`opt/web-app/etc/server_configs.json`.  Columns are the services that expose at
least one real network endpoint (standalone-only services with no hosts/clients
are silently excluded).  The cell value is `✓ port1,port2` when the IP serves
that service, or `N/A` otherwise.

**Title** reflects tenant mode:
- `SingleTenant` (or unknown) → `Table 3.5 – Cluster Services`
- `MultiTenant` → `Table 3.5 – Multi-Tenant Cluster Services`

| Column | Description |
|---|---|
| IP Address | Cluster node IP, sorted numerically |
| app-server | `application-server` — vManage web/API (ports 8080 / 8443) |
| config-db | `configuration-db` — Neo4j configuration database |
| stats-db | `statistics-db` — Elasticsearch statistics database |
| container-mgr | `container-manager` — SD-AVC container (port 10502) |
| coord-server | `coordination-server` — ZooKeeper (ports 2181 / 2888:3888) |
| messaging | `messaging-server` — NATS (ports 4222 / 6222) |
| sdavc | `sdavc` — SD-AVC service endpoint |
| + others | Any additional services present in the file |

```
=== Table 3.5 – Cluster Services ===
    Source: 1.1.20.1-vmanage-admin-tech.tar  |  System-IP: 1.1.20.1  |  Mode: SingleTenant

+----------------+-------------+-------------+---------------+-----------------+-------------+---------+----------+
| IP Address     | app-server  | config-db   | container-mgr | coord-server    | messaging   | sdavc   | stats-db |
+----------------+-------------+-------------+---------------+-----------------+-------------+---------+----------+
| 10.206.177.2   | ✓ 8080,8443 | ✓ 5000,7687 | ✓ 10502       | ✓ 2181,2888:3888 | ✓ 4222,6222 | ✓ 10502 | ✓ 9300   |
| 10.206.177.5   | ✓ 8080,8443 | ✓ 5000,7687 | ✓ 10502       | ✓ 2181,2888:3888 | ✓ 4222,6222 | ✓ 10502 | ✓ 9300   |
| 10.206.177.6   | ✓ 8080,8443 | ✓ 5000,7687 | ✓ 10502       | ✓ 2181,2888:3888 | ✓ 4222,6222 | ✓ 10502 | ✓ 9300   |
+----------------+-------------+-------------+---------------+-----------------+-------------+---------+----------+
```

---

### Table 4 — CFGMGR RTM Interface Summary

Three sub-tables — one per personality group — showing the IP address assigned
to every interface found in the `CFGMGR RTM Interface Database` section of each
device's `config` file.

- **Columns are dynamic**: the script collects the union of all interface names
  seen across every device in the group.  Devices that lack a particular
  interface show `N/A`.
- **Groups are skipped** if no device of that personality is in the collection.
- Interfaces with no configured address (`::/0`) are displayed as `N/A`.

```
=== Table 4 – CFGMGR RTM Interface Summary (vManage / vSmart / vBond) ===

+-----------+-------------+----------------+----------------+---------------+----------------+
| System-IP | Personality | eth1           | eth2           | docker0       | cbr-vmanage    |
+-----------+-------------+----------------+----------------+---------------+----------------+
| 1.2.1.1   | vmanage     | 10.1.60.229/24 | 10.1.198.51/24 | 172.17.0.1/16 | 169.254.8.1/24 |
+-----------+-------------+----------------+----------------+---------------+----------------+

+-----------+-------------+----------------+
| System-IP | Personality | eth1           |
+-----------+-------------+----------------+
| 1.1.1.3   | vsmart      | 10.1.60.131/24 |
| 1.2.1.3   | vsmart      | 10.1.9.51/24   |
+-----------+-------------+----------------+

+-----------+-------------+--------------------+------------------+-------+-------+----------------+--------------+
| System-IP | Personality | eth0               | ge0_0            | ge0_1 | ge0_2 | loopback65528  | system       |
+-----------+-------------+--------------------+------------------+-------+-------+----------------+--------------+
| 1.1.1.2   | vbond       | 172.28.240.104/24  | 118.201.99.104/25| N/A   | N/A   | 192.168.0.1/24 | 1.1.1.2/32   |
| 1.1.1.4   | vbond       | N/A                | N/A              | N/A   | N/A   | N/A            | N/A          |
+-----------+-------------+--------------------+------------------+-------+-------+----------------+--------------+
```

---

## Files read inside each bundle

| File path (relative to bundle root) | Used for |
|---|---|
| `var/tech/config` | Identity fields, CFGMGR interfaces (vBond, vSmart) |
| `opt/data/tech/config` | Identity fields, CFGMGR interfaces (vManage) |
| `var/tech/vdaemon` | Certificates, WAN interfaces (vBond, vSmart) |
| `opt/data/tech/vdaemon` | Certificates, WAN interfaces (vManage) |
| `var/tech/sysmgrd` | Software version, personality fallback (vBond, vSmart) |
| `opt/data/tech/sysmgrd` | Software version, personality fallback (vManage) |
| `opt/data/tech/clusterconfig` | Cluster topology — primary detection method (vManage) |
| `opt/web-app/etc/deployment` | DR status, deployment mode, tenant mode (vManage) |
| `opt/web-app/etc/server_configs.json` | Cluster peer IPs + service inventory — fallback detection (vManage) |

---

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| A bundle is not picked up | Folder / archive name does not contain `admin-tech` |
| All fields show `N/A` | The `config` file was not found at any candidate path |
| Personality shows `N/A` | Neither `config` nor `sysmgrd` contained a recognisable personality field |
| Version shows `N/A` | The `sysmgrd` file is missing or has no `show system status` section |
| Certificates show `N/A` | The `vdaemon` file is missing from the bundle |
| Table 3 is skipped | No vManage bundles found in the scanned folder |
| Table 3 shows `Unknown` deployment | `clusterconfig`, `vmanage-system-ip`, and `deployment` file all absent |
| Table 3.5 is not printed | No cluster vManage with a readable `server_configs.json` was found |
| Peer IPs all show `✗ admin-tech missing` | Only one vManage admin-tech was collected; the rest of the cluster nodes are absent from the folder |
| `Standalone (SD-AVC)` in Deployment column | `deploymentmode=Cluster` but `server_configs.json` has only one IP and `sdavc.standalone=true` |
