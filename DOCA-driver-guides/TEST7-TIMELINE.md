# Test 7 -- Operator Install & Upgrade Timeline

## Environment

- **Platform**: OpenShift 4.18 on RHEL 9.4 (aarch64+64k page size kernel)
- **Kernel**: `5.14.0-570.73.1.el9_6.aarch64+64k`
- **NIC**: Mellanox ConnectX at PCI `0000:01:00.0` (PF: `enp1s0f0np0`) with **50 VFs** configured
- **GPU**: NVIDIA GPU driver v580.105.08 (open kernel modules, with vgpu-util present)
- **NicClusterPolicy settings**:
  - `ENTRYPOINT_DEBUG`: **not set** (defaults to false -- no debug trace in logs)
  - `RESTORE_DRIVER_ON_POD_TERMINATION: false`
  - `terminationGracePeriodSeconds: 30` (shorter than test6's 90s)
  - `drain.enable: false`
  - `autoUpgrade: true`

### Key Difference from Test 6

Test 7 does **not** enable `ENTRYPOINT_DEBUG`. This means:
- Logs are much shorter (no shell `set -x` trace)
- `DEBUG_SLEEP_SEC_ON_EXIT` does not apply (debug sleep only runs when `ENTRYPOINT_DEBUG=true`)
- On failure, the container exits immediately without the 90-second debug sleep
- The `terminationGracePeriodSeconds` is 30s (vs. 90s in test6)

## mlx5_core srcversion Tracking

| Checkpoint | srcversion | Notes |
|---|---|---|
| Before install | `AE6029416A873D39E95B044` | This is the MOFED 25.10 driver from a previous test. The note says "corresponds to last compiled driver with network operator 25.10 because we are disabling the restore_driver" |
| After install (MOFED 25.07) | `1852822C2F9ABA6A93704E4` | MLNX_OFED 25.07-0.9.7.0 mlx5_core loaded |
| Before upgrade | `1852822C2F9ABA6A93704E4` | Same as after install, note: "corresponds to previous compiled 25.7. mlx5_core" |
| After upgrade (MOFED 25.10) | `AE6029416A873D39E95B044` | MLNX_OFED 25.10-1.2.8.0 mlx5_core loaded |

> **Important context**: The node had previously run test 6 with MOFED 25.10 loaded. Because `RESTORE_DRIVER_ON_POD_TERMINATION=false`, the 25.10 driver was still loaded on the host when test 7 began. Test 7 starts by installing MOFED 25.07 (a downgrade), then upgrades back to 25.10.

---

## Phase 1: Initial Operator Install (MOFED 25.07-0.9.7.0)

### 09:17:20 -- MOFED Container Starts

The mofed-container starts and:
1. Copies required files to DTK shared directory
2. Begins polling for DTK build completion

### 09:17:20 -- DTK Build & Wait

- Container begins waiting for `dtk_done_compile_25_07_0_9_7_0`
- DTK sidecar compiles MLNX_OFED 25.07 packages (same packages as test6)
- First poll sleeps 300 seconds

### ~09:22:20 -- DTK Build Complete, RPM Install

After the DTK build completes (between 09:17 and 09:22):
1. RPMs copied and installed
2. Driver version comparison detects mismatch (loaded: `AE6029416A873D39E95B044` from previous MOFED 25.10, new RPMs: `1852822C2F9ABA6A93704E4` for MOFED 25.07)
3. **This is a downgrade** -- the loaded driver is newer than the one being installed

### 09:22:22 -- Driver Restart (openibd)

1. OFED blacklist applied to host
2. `/etc/init.d/openibd restart` executed
3. HCA driver unloaded and reloaded with MOFED 25.07 modules

### 09:22:22 -- 09:24:22 -- openibd Restart (~2 minutes)

- **"Unloading HCA driver: [OK]"**
- **"Loading HCA driver and Access Layer: [OK]"**
- Same ~2 minute window as test6 where all VFs are destroyed

### 09:24:22 -- Blacklist Removed, VF Restore Begins

1. Blacklist removed from host
2. `restore_sriov_config` starts for PF `0000:01:00.0` with 50 VFs
3. PF set up, `echo 50 >> sriov_numvfs`

### 09:24:41 -- VF0 Restore FAILURE (Same Bug as Test 6)

- VF0 (`0000:01:00.3`): netdev resolves to `eth0` instead of the expected name
- **"Cannot find device eth0"**
- `ip link set eth0 address 0a:d1:94:74:c9:2d` fails with exit code 1

### 09:24:42 -- Immediate Exit (No Debug Sleep)

Because `ENTRYPOINT_DEBUG` is not enabled:
- Container exits immediately (no 90-second debug sleep)
- Blacklist cleanup runs via EXIT trap
- Container enters CrashLoopBackOff

### Recovery via Restart

On restart, the MOFED 25.07 driver is already loaded, so no reload is needed. The container successfully sets driver readiness and enters stable state.

### 09:24:45 -- Driver Ready

- **"Setting driver ready state"** -- driver readiness file created
- **"NVIDIA driver container exec end, sleeping"** -- container stable

---

## Phase 2: Operator Upgrade (MOFED 25.07 -> 25.10-1.2.8.0)

The NicClusterPolicy `version` is changed to `25.10-1.2.8.0`.

### 09:28:48 -- Old MOFED Container Terminated (3 instances)

The logs show the termination event repeated 3 times, suggesting the container was restarted multiple times (likely due to CrashLoopBackOff from the initial install). Each termination:

1. **"Terminate event caught"**
2. **"Unsetting driver ready state"**
3. **"Keeping currently loaded Mellanox OFED Driver..."** (RESTORE_DRIVER_ON_POD_TERMINATION=false)

> With `terminationGracePeriodSeconds: 30` (vs. test6's 90s), the container has only 30 seconds to complete its shutdown sequence. Since there's no debug sleep and no driver restore, this is sufficient.

### 09:29:22 -- New MOFED Container Starts (25.10-1.2.8.0)

1. Files copied to DTK shared directory
2. Begins polling for DTK build completion (`dtk_done_compile_25_10_1_2_8_0`)

### 09:29:22 -- 09:34:22 -- DTK Compiles MOFED 25.10 (~5 minutes)

Same build process as test6's upgrade:
1. mlnx-tools 2510.0.14
2. ofed-scripts 25.10
3. mlnx-ofa_kernel 25.10 (built twice)
4. xpmem 2510.0.16
5. kernel-mft 4.34.0 (built twice)

### 09:34:23 -- RPM Install & Driver Reload

1. New 25.10 RPMs installed
2. Driver version comparison: loaded `1852822C2F9ABA6A93704E4` (25.07) vs new `AE6029416A873D39E95B044` (25.10)
3. Reload required

### 09:34:23 -- Driver Restart

1. OFED blacklist applied
2. `/etc/init.d/openibd restart`

### 09:34:23 -- 09:36:55 -- openibd Restart (~2.5 minutes)

- HCA driver unloaded and reloaded with MOFED 25.10 modules
- All VFs destroyed during this window

### 09:36:55 -- 09:37:17 -- VF Restore Begins

1. Blacklist removed
2. `restore_sriov_config` starts
3. PF up, 50 VFs created

### 09:37:17 -- 09:39:00 -- VF Restoration (Partial Success)

Unlike test6 which failed on VF1, test7 successfully restores **16 VFs** (VF0 through VF15) before failing:

| Time | VF PCI Address | Status |
|---|---|---|
| 09:37:17 | 0000:01:00.3 (VF0) | SUCCESS |
| 09:37:24 | 0000:01:00.4 (VF1) | SUCCESS |
| 09:37:30 | 0000:01:00.5 (VF2) | SUCCESS |
| 09:37:36 | 0000:01:00.6 (VF3) | SUCCESS |
| 09:37:43 | 0000:01:00.7 (VF4) | SUCCESS |
| 09:37:49 | 0000:01:01.0 (VF5) | SUCCESS |
| 09:37:56 | 0000:01:01.1 (VF6) | SUCCESS |
| 09:38:02 | 0000:01:01.2 (VF7) | SUCCESS |
| 09:38:08 | 0000:01:01.3 (VF8) | SUCCESS |
| 09:38:15 | 0000:01:01.4 (VF9) | SUCCESS |
| 09:38:21 | 0000:01:01.5 (VF10) | SUCCESS |
| 09:38:28 | 0000:01:01.6 (VF11) | SUCCESS |
| 09:38:34 | 0000:01:01.7 (VF12) | SUCCESS |
| 09:38:41 | 0000:01:02.0 (VF13) | SUCCESS |
| 09:38:47 | 0000:01:02.1 (VF14) | SUCCESS |
| 09:38:54 | 0000:01:02.2 (VF15) | SUCCESS |
| **09:39:00** | **0000:01:02.3 (VF16)** | **FAIL** |

Each VF takes approximately 6-7 seconds to restore (unbind, set MAC, rebind, set MTU, set admin state, 4s BIND_DELAY_SEC).

### 09:39:00 -- VF16 Restore FAILURE

- `ls /sys/bus/pci/devices/0000:01:02.3/net/` returns "No such file or directory"
- VF16's netdev is not yet available
- `ip link set address ee:59:6f:98:b9:5c` fails with "Not enough information: dev argument is required"
- Exit code 255

### 09:39:00 -- Recovery (No Debug Sleep)

Since `ENTRYPOINT_DEBUG` is not set, the container exits immediately (no 90-second sleep).

### 09:39:03 -- Container Restarts, Succeeds

On the restart:
1. Driver already loaded -- no reload needed
2. Reports driver version via ethtool: **`25.10-1.2.2`**
3. Mounts rootfs
4. **Sets driver readiness**

### 09:39:04 -- Stable State

Container enters `sleep infinity` -- upgrade complete.

---

## GPU Driver (parallel operation)

Same as test6: NVIDIA GPU driver v580.105.08 installed via DTK sidecar, loads open kernel modules, detects Mellanox device for nvidia-peermem, starts persistenced, mounts rootfs.

---

## Comparison: Test 6 vs Test 7

| Aspect | Test 6 | Test 7 |
|---|---|---|
| ENTRYPOINT_DEBUG | `true` | `false` (default) |
| DEBUG_SLEEP_SEC_ON_EXIT | 90s | N/A (not triggered) |
| terminationGracePeriodSeconds | 90s | 30s |
| Starting mlx5_core | Inbox `570B...` | Previous MOFED 25.10 `AE60...` |
| Install direction | Inbox -> 25.07 (upgrade) | 25.10 -> 25.07 (downgrade) |
| Upgrade direction | 25.07 -> 25.10 | 25.07 -> 25.10 |
| VFs restored before failure (install) | 0 (VF0 failed) | 0 (VF0 failed, same bug) |
| VFs restored before failure (upgrade) | 1 (VF0 ok, VF1 failed) | 16 (VF0-VF15 ok, VF16 failed) |
| Recovery time after VF failure | 90s (debug sleep) | Immediate |
| Total upgrade time | ~11 min | ~10 min |

### Why More VFs Succeeded in Test 7 Upgrade

Test 7's upgrade restored 16 VFs successfully vs. test6's 1 VF. This is likely because:
1. Each VF restoration takes ~6-7 seconds (including the 4s BIND_DELAY_SEC)
2. By the time VF16 is being restored (~103s after VF creation), the kernel has had more time to probe and initialize netdevs
3. However, VF16 at PCI `0000:01:02.3` still wasn't ready, suggesting the VF probe is not sequential by PCI address and some VFs take longer than others

### Key Takeaway: The VF Netdev Timing Bug

Both tests demonstrate the same fundamental issue: the `restore_sriov_config` function does not wait for each VF's netdev to appear before attempting to configure it. The `BIND_DELAY_SEC` (4 seconds after VF creation) is a single global delay and is insufficient for 50 VFs on this platform. A per-VF readiness check (polling for the netdev's existence before proceeding) would fix this issue.
