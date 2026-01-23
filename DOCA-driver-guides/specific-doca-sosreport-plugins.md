# Additional Logs Collected by doca-sosreport

This document details the additional plugins and modifications in `doca-sosreport` compared to the upstream `sos` project.

---

## Additional Plugins in doca-sosreport (Not in upstream sos)

### 1. `doca.py` - DOCA Package and Libraries Plugin
Collects NVIDIA DOCA framework debug information:

| Type | Data Collected |
|------|----------------|
| **Commands** | `/opt/mellanox/doca/tools/doca_caps -v` (version) |
| | `/opt/mellanox/doca/tools/doca_caps --list-devs` (devices) |
| | `/opt/mellanox/doca/tools/doca_caps --list-rep-devs` (representor devices) |
| | `/opt/mellanox/doca/tools/doca_caps --list-libs` (libraries) |
| | `/opt/mellanox/doca/tools/doca_caps` (full capabilities) |
| **Scripts** | Executes `/opt/mellanox/doca/tools/sos_script.sh` |
| **Dynamic commands** | Any commands listed in `/var/tmp/sos_commands` |
| **Dynamic logs** | Any log files listed in `/opt/mellanox/doca/scripts/sos_logs` |

---

### 2. `doca_dpf.py` - DOCA Platform Framework Plugin
Collects Kubernetes-based DOCA Platform Framework (DPF) resources:

| Resource Type | Description |
|---------------|-------------|
| `bfb` | BlueField Boot images |
| `dpfoperatorconfig` | DPF operator config |
| `dpu` | DPU (Data Processing Unit) resources |
| `dpudeployment` | DPU deployments |
| `dpuflavor` | DPU flavors/configurations |
| `dpuservice` | DPU services |
| `dpuservicechain` | Service chains on DPU |
| `dpuserviceconfiguration` | Service configurations |
| `dpuservicecredentialrequest` | Credential requests |
| `dpuserviceinterface` | Service interfaces |
| `dpuserviceipam` | IP address management |
| `dpuservicetemplate` | Service templates |
| `dpuset` | DPU sets |
| `servicechain`, `servicechainset` | Service chain resources |
| `serviceinterface`, `serviceinterfaceset` | Interface resources |
| `cidrpool`, `ippool` | NV-IPAM resources |
| `application`, `appproject` | ArgoCD resources |
| `tenantcontrolplane` | Kamaji tenant control planes |

All collected via `kubectl get -o json` per namespace.

---

### 3. `devlink_health.py` - Devlink Health Reporters Plugin
Collects network device health information:

| Commands | Description |
|----------|-------------|
| `devlink health -jp` | List all devices and health reporters (JSON) |
| `devlink health diagnose <dev> reporter <name>` | Diagnostics per reporter |
| `devlink health dump show <dev> reporter <name>` | Dump data per reporter |

---

### 4. `mlx5_core.py` - Mellanox 5th Gen Driver Plugin
Collects mlx5 driver debug information:

| Files Collected | Description |
|-----------------|-------------|
| `/sys/kernel/debug/mlx5/0000:*/*` | All debugfs entries for ConnectX-5/6/7 NICs |

---

### 5. `hbn.py` - Host Based Networking Plugin
Collects HBN (BlueField DPU networking) information:

| Type | Data Collected |
|------|----------------|
| **Config files** | `/etc/mellanox` |
| **Logs** | `/var/log/doca/hbn` |
| | `/var/log/sfc-install.log` |
| **State files** | `/var/lib/dhcp` |
| | `/var/lib/hbn/etc/cumulus` |
| **udev** | `/usr/lib/udev` (HBN udev rules) |
| **Temp files** | `/tmp/sf_devices`, `/tmp/sfr_devices` (renamed scalable function devices) |
| | `/tmp/sfc-activated` |
| | `/tmp/.BR_*`, `/tmp/.ENABLE_*`, `/tmp/.LINK_*` (HBN config interim files) |
| **Journals** | `sfc` unit logs |
| | `sfc-state-propagation` unit logs |
| **Commands** | `mlnx-sf -a show` (scalable functions) |
| | `mlnx-sf -a show --json --pretty` (verbose JSON output) |

---

### 6. `rdma.py` - RDMA Plugin
Collects RDMA (Remote Direct Memory Access) device information:

| Commands | Description |
|----------|-------------|
| `rdma dev show -d` | Device info |
| `rdma link show -d` | Link info |
| `rdma system show -d` | System info |
| `rdma statistic show -d` | Statistics |
| `rdma resource show qp -d` | Queue pairs |
| `rdma resource show cm_id -d` | Connection manager IDs |
| `rdma resource show cq -d` | Completion queues |
| `rdma resource show pd -d` | Protection domains |
| `rdma resource show mr -d` | Memory regions |
| `rdma resource show ctx -d` | Contexts |
| `rdma resource show srq -d` | Shared receive queues |

Uses `/opt/mellanox/iproute2/sbin/rdma` if available.

---

### 7. `pxe.py` - PXE Boot Plugin
Collects PXE provisioning information:

| Platform | Data |
|----------|------|
| **RedHat** | `/usr/sbin/pxeos -l`, `/etc/dhcpd.conf`, optionally `/tftpboot` |
| **Debian/Ubuntu** | `/etc/dhcp/dhcpd.conf`, `/etc/default/tftpd-hpa`, optionally `/var/lib/tftpboot` |

---

## Modified Plugins (Enhanced in doca-sosreport)

### `nvidia.py` - Added GPU topology command

```
nvidia-smi topo -m   # GPU topology matrix
```

### `mellanox_firmware.py` - Enhanced for broader device support

| Change | Description |
|--------|-------------|
| Package detection | Added `mft` package |
| Device detection | Changed from `lspci -d 15b3::0200` to `lspci -d 15b3::` (catches all Mellanox devices, not just network) |
| Commands | Changed `mlxdump pcie_uc` → `mlxdump mstdump` |
| | Changed `mstconfig` → `mlxconfig` |
| | Added device compatibility check via `mcra` |
| | Increased timeout from 30s to 300s |

---

## DOCA-Specific Configuration Files

| File | Purpose |
|------|---------|
| `sos-nvidia.conf` | Full system collection, skipping ssh/flatpack/login plugins |
| `sos-nvdebug.conf` | **Targeted debug** - only runs: `openvswitch, rdma, hbn, infiniband, doca, networking, mlx5_core` |
| `sos-mlx-cloud-verification.conf` | Cloud verification preset |

---

## Summary

**doca-sosreport adds 7 new plugins** focused on:

1. **DOCA SDK** - framework capabilities, devices, libraries
2. **DPU/BlueField** - Kubernetes-based DPF resources, HBN networking, scalable functions
3. **RDMA** - complete RDMA device/resource inventory
4. **Mellanox drivers** - mlx5 debugfs, enhanced firmware data collection
5. **Devlink health** - network device health diagnostics
6. **PXE** - boot provisioning (for DPU deployment scenarios)
