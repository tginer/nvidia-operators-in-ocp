# Test 6 -- Operator Install & Upgrade Timeline

## Environment

- **Platform**: OpenShift 4.18 on RHEL 9.4 (aarch64+64k page size kernel)
- **Kernel**: `5.14.0-570.73.1.el9_6.aarch64+64k`
- **NIC**: Mellanox ConnectX at PCI `0000:01:00.0` (PF: `enp1s0f0np0`) with **50 VFs** configured
- **GPU**: NVIDIA GPU driver v580.105.08 (open kernel modules)
- **NicClusterPolicy settings**:
  - `ENTRYPOINT_DEBUG: true`
  - `DEBUG_SLEEP_SEC_ON_EXIT: 90`
  - `RESTORE_DRIVER_ON_POD_TERMINATION: false`
  - `terminationGracePeriodSeconds: 90`
  - `drain.enable: false` (no node drain during driver operations)
  - `autoUpgrade: true`

## mlx5_core srcversion Tracking

| Checkpoint | srcversion | Meaning |
|---|---|---|
| Before install (inbox driver) | `570B167417792AFC9308A50` | Stock RHEL inbox mlx5_core |
| After install (MOFED 25.07) | `1852822C2F9ABA6A93704E4` | MLNX_OFED 25.07-0.9.7.0 mlx5_core |
| Before upgrade | `1852822C2F9ABA6A93704E4` | Same as after install |
| After upgrade (MOFED 25.10) | `AE6029416A873D39E95B044` | MLNX_OFED 25.10-1.2.8.0 mlx5_core |
| Before uninstall | `AE6029416A873D39E95B044` | Same as after upgrade |
| After NIC policy deletion | `AE6029416A873D39E95B044` | Driver remains loaded (RESTORE_DRIVER_ON_POD_TERMINATION=false) |

---

## Phase 1: Initial Operator Install (MOFED 25.07-0.9.7.0)

### 07:47:25 -- MOFED Container Starts, Waits for DTK Build

The `mofed-container` (DOCA driver build entrypoint) starts and detects that the OpenShift Driver Toolkit (DTK) sidecar has not yet finished compiling the NIC drivers. It enters a polling loop:

- Polls every 300 seconds for the DTK completion flag file `dtk_done_compile_25_07_0_9_7_0`
- First poll at `07:47:25`: flag not found, sleeps 300s

### 07:47:34 -- DTK Sidecar Compiles MOFED 25.07 (parallel)

The `openshift-driver-toolkit-ctr` sidecar container starts compiling the MLNX_OFED 25.07-0.9.7.0 driver packages from source RPMs. The build includes:

1. **ofed-scripts** 25.07
2. **mlnx-tools** 25.07
3. **mlnx-ofa_kernel** 25.07 (the core `mlx5_core` kernel module) -- compiled with `--with-mlx5-ipsec`
4. **xpmem** 2.7.4
5. **kernel-mft** 4.33.0

### 07:49:57 -- DTK Build Completes

DTK touches `dtk_done_compile_25_07_0_9_7_0` flag file and exits. Total build time: ~2.5 minutes.

### 07:52:25 -- MOFED Container Detects Build Completion

On the next poll cycle, the mofed-container finds the DTK completion flag. It proceeds to:

1. **Copy built RPMs** from the DTK shared volume to `/tmp/nvidia_nic_driver_11-02-2026_07-47-25/`
2. **Prepare kernel modules directory**: `mkdir /lib/modules/5.14.0-570.73.1.el9_6.aarch64+64k`, create `modules.order` and `modules.builtin`
3. **Install RPMs**: `rpm -ivh --replacepkgs --nodeps` for all 12 built RPMs
4. **Run depmod**: `depmod 5.14.0-570.73.1.el9_6.aarch64+64k`

### 07:52:27 -- Driver Version Comparison & Reload

The `load_driver` function compares the installed (`modinfo`) vs. loaded (`/sys/module/mlx5_core/srcversion`) driver versions:

- **modinfo** (new, from RPMs): `1852822C2F9ABA6A93704E4`
- **sysfs** (currently loaded inbox): `570B167417792AFC9308A50`
- **Result**: srcversion differs -- driver reload required

### 07:52:27 -- Driver Restart Sequence (`restart_driver`)

1. Load dependency modules: `modprobe tls`, `modprobe psample`, `modprobe macsec`
2. Write OFED module blacklist to host (`/host/etc/modprobe.d/blacklist-ofed-modules.conf`) to prevent inbox driver from reloading
3. Execute **`/etc/init.d/openibd restart`** -- this unloads the old HCA driver and loads the new one

### 07:52:27 -- 07:54:24 -- openibd Restart (~2 minutes)

The openibd restart is the critical window:
- **"Unloading HCA driver: [OK]"** -- all mlx5_core, mlx5_ib, ib_core etc. are unloaded. VFs disappear.
- **"Loading HCA driver and Access Layer: [OK]"** -- new MOFED 25.07 modules loaded. PF comes back but VFs are gone.

This takes approximately **2 minutes** on this aarch64+64k system.

### 07:54:24 -- Remove Blacklist & Restore SR-IOV Config

1. Remove the OFED modules blacklist file from host
2. Begin `restore_sriov_config`:
   - Scans 54 Mellanox devices (2 PFs + 50 VFs from before + 2 PF ports)
   - Finds PF `0000:01:00.0` (`enp1s0f0np0`) with `pf_numvfs=50`
   - Sets PF link up: `ip link set dev enp1s0f0np0 up`
   - Writes VF count to sysfs: `echo 50 >> /sys/bus/pci/devices/0000:01:00.0/sriov_numvfs`

### 07:54:43 -- VF Restore FAILURE (Bug)

The script tries to restore VF0 (`0000:01:00.3`):
- VF netdev name resolves to `eth0` (kernel assigned a different name than expected `enp1s0f0v0`)
- **ERROR**: `ip link set eth0 address 1a:df:a7:11:c6:46` fails with "Cannot find device eth0"
- The VF netdev was not yet fully initialized when the script tried to access it (race condition / timing issue with `BIND_DELAY_SEC`)

### 07:54:44 -- Debug Sleep & Exit

- `exit_entryp 1` triggered
- Because `ENTRYPOINT_DEBUG=true`, the container sleeps for `DEBUG_SLEEP_SEC_ON_EXIT=90` seconds before exiting
- At `07:56:14` the blacklist cleanup runs (EXIT trap), then the container restarts via CrashLoopBackOff

### 07:56:17 -- Container Restarts (Second Attempt, same MOFED 25.07)

The container restarts. This time:
- The inbox driver is NOT loaded (the new MOFED 25.07 driver is already loaded from the previous attempt)
- The DTK build flag is already present, so no compilation needed
- The srcversion comparison shows the loaded driver **already matches** the installed one
- **This means the driver does NOT need to be reloaded -- it just needs VF restoration**

> **Note**: The logs for this second restart are not captured separately, but the container eventually enters `sleep infinity` successfully on a subsequent restart (visible in the upgrade phase starting point).

---

## Phase 2: Operator Upgrade (MOFED 25.07 -> 25.10-1.2.8.0)

The NicClusterPolicy `version` was changed to `25.10-1.2.8.0`, triggering an upgrade.

### 08:27:39 -- Old MOFED Container Terminated

Kubernetes sends SIGTERM to the running mofed-container (MOFED 25.07):
1. **Terminate event caught**
2. **Unset driver readiness**: removes `/run/mellanox/drivers/.driver-ready`
3. **RESTORE_DRIVER_ON_POD_TERMINATION=false**: keeps the currently loaded MOFED 25.07 driver
4. **Unmount rootfs**: unmounts `/run/mellanox/drivers`
5. **Cleanup temp files**: removes driver packages directory
6. **Debug sleep**: 90 seconds before exit (ENTRYPOINT_DEBUG=true)

### 08:29:13 -- New MOFED Container Starts (25.10-1.2.8.0)

The new container image for MOFED 25.10-1.2.8.0 starts:
1. Copies files to DTK shared directory
2. Waits for DTK to compile the new driver version
3. Polls for `dtk_done_compile_25_10_1_2_8_0`

### 08:29:13 -- 08:34:02 -- DTK Compiles MOFED 25.10 (~5 minutes)

The DTK sidecar builds MLNX_OFED 25.10-1.2.8.0 packages:
1. **mlnx-tools** 2510.0.14
2. **ofed-scripts** 25.10
3. **mlnx-ofa_kernel** 25.10 (built twice -- normal for this version, second build overwrites)
4. **xpmem** 2510.0.16
5. **kernel-mft** 4.34.0 (also built twice)

At `08:34:02`, DTK touches the completion flag.

### 08:34:13 -- MOFED Container Installs New RPMs

1. Copies built RPMs from DTK shared volume
2. Installs 12 new RPMs via `rpm -ivh --replacepkgs --nodeps`
3. Runs `depmod`

### 08:34:15 -- Driver Version Comparison & Reload

- **modinfo** (new 25.10): `AE6029416A873D39E95B044`
- **sysfs** (loaded 25.07): `1852822C2F9ABA6A93704E4`
- **Result**: srcversion differs -- driver reload required

### 08:34:15 -- Driver Restart Sequence

Same as initial install:
1. Load dependency modules (tls, psample, macsec)
2. Write OFED blacklist to host
3. Execute `/etc/init.d/openibd restart`

### 08:34:15 -- 08:36:20 -- openibd Restart (~2 minutes)

- HCA driver unloaded and reloaded with MOFED 25.10 modules
- **This is the critical disruption window** -- all VFs disappear during this time

### 08:36:20 -- VF Restoration (Successful for VF0 this time)

1. Remove blacklist, begin `restore_sriov_config`
2. PF `enp1s0f0np0` set up, 50 VFs created via sysfs
3. **4-second BIND_DELAY_SEC sleep** after VF creation (this was missing in the install phase!)
4. VF0 (`0000:01:00.3`) restores successfully:
   - Netdev resolves to `enp1s0f0v0` (correct name this time)
   - MAC set, admin MAC set via PF
   - VF unbound from mlx5_core, rebound
   - MTU set to 1500, link set down

### 08:36:48 -- VF1 Restore FAILURE

VF1 (`0000:01:00.4`) fails:
- `ls /sys/bus/pci/devices/0000:01:00.4/net/` returns "No such file or directory"
- The VF netdev is not yet available (same timing bug, affects VF1 instead of VF0)
- `ip link set address a6:e5:8b:ee:2c:85` fails with "Not enough information: dev argument is required"
- **Exit code 255**, debug sleep 90 seconds

### 08:38:18 -- Container Restarts (After Debug Sleep)

The container restarts again. On this restart:
- The MOFED 25.10 driver is already loaded (srcversion matches)
- No driver reload needed
- The entrypoint goes straight to:

### 08:38:23 -- Successful Completion

1. Detects loaded driver version via ethtool: **`25.10-1.2.2`** (MOFED 25.10)
2. Confirms srcversion: `AE6029416A873D39E95B044`
3. Mounts rootfs at `/run/mellanox/drivers/`
4. **Sets driver readiness**: `touch /run/mellanox/drivers/.driver-ready`
5. Enters `sleep infinity` -- container is now stable

### Ping Connectivity

The `ping.log` shows **1244 consecutive successful pings** to `16.0.2.2` with **zero packet loss**. The ping was running on a different interface (`enp1s0f1np1` / the second PF or an RDMA shared device) that was not affected by the VF restoration bug. Latencies range from 0.002ms to 0.071ms.

---

## GPU Driver (parallel operation)

The NVIDIA GPU driver (v580.105.08) installed via the GPU operator runs in parallel:
- Waits for the DTK sidecar to build GPU kernel modules
- Copies precompiled modules from shared volume
- Loads nvidia, nvidia-uvm, nvidia-modeset, nvidia-peermem
- Detects Mellanox device at `0000:01:00.0` and loads nvidia-peermem for GPUDirect RDMA
- Starts nvidia-persistenced
- Mounts driver rootfs with SELinux context

---

## Key Observations & Issues

### BUG: VF Netdev Not Ready During Restore

Both during install and upgrade, the `restore_sriov_config` function fails because VF netdevs are not yet initialized when the script tries to configure them:
- **Install**: VF0 gets name `eth0` instead of expected `enp1s0f0v0`, then device not found
- **Upgrade**: VF0 succeeds (4s BIND_DELAY_SEC helps), but VF1's `/sys/bus/pci/devices/0000:01:00.4/net/` does not exist yet

The root cause is a race condition: after writing `sriov_numvfs` and the BIND_DELAY_SEC sleep, the VFs are being probed by the kernel but not all have their netdev interfaces ready. With 50 VFs on aarch64+64k, the probe time is substantial.

### Recovery Behavior

The container uses CrashLoopBackOff recovery effectively:
1. First attempt: driver loads, VF restore fails, container exits after debug sleep
2. Second attempt: driver already loaded (skip reload), VF restore succeeds because VFs are now fully initialized
3. Container enters stable `sleep infinity` state

### Timing Summary

| Event | Time | Duration |
|---|---|---|
| **Install**: DTK build start | 07:47:34 | -- |
| **Install**: DTK build complete | 07:49:57 | ~2.5 min |
| **Install**: openibd restart | 07:52:27 - 07:54:24 | ~2 min |
| **Install**: VF restore fail + debug sleep | 07:54:44 - 07:56:14 | 90s |
| **Upgrade**: Old container termination | 08:27:39 | -- |
| **Upgrade**: DTK build start | 08:29:13 | -- |
| **Upgrade**: DTK build complete | 08:34:02 | ~5 min |
| **Upgrade**: openibd restart | 08:34:15 - 08:36:20 | ~2 min |
| **Upgrade**: VF restore fail + debug sleep | 08:36:48 - 08:38:18 | 90s |
| **Upgrade**: Final success (driver ready) | 08:38:23 | -- |
| **Total upgrade time** (termination to ready) | 08:27:39 - 08:38:23 | **~11 min** |
