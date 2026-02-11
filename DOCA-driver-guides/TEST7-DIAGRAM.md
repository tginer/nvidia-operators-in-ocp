# Test 7 -- Visual Timeline Diagram

## NicClusterPolicy Config

```
ENTRYPOINT_DEBUG=false (default) | terminationGracePeriodSeconds=30
RESTORE_DRIVER_ON_POD_TERMINATION=false | drain.enable=false
Install: MOFED 25.10 (leftover) -> MOFED 25.07 (downgrade) | Upgrade: MOFED 25.07 -> 25.10
```

## mlx5_core srcversion Transitions

```
Previous MOFED 25.10 (leftover)     MOFED 25.07                     MOFED 25.10
AE6029416A873D39E95B044             1852822C2F9ABA6A93704E4         AE6029416A873D39E95B044
       |                                   |                               |
       +--- Install DOWNGRADES to -------->+                               |
                                           +-------- Upgrade loads ------->+
```

---

## Phase 1: Install (MOFED 25.07-0.9.7.0) -- Downgrade from 25.10

```
TIME        MOFED CONTAINER                    DTK SIDECAR                      mlx5_core / VFs
09:17       ================================== ================================ =========================
            |                                  |                                |
09:17:20    | Start, copy files to DTK         |                                | MOFED 25.10 loaded
            | Poll for DTK flag (sleep 300s)   | Start compiling MOFED 25.07    | (from previous test6)
            |                                  | - ofed-scripts                 | 50 VFs present
            |                                  | - mlnx-tools                   |
            |                                  | - mlnx-ofa_kernel              |
            |                                  | - xpmem, kernel-mft            |
            |                                  |                                |
~09:20      |                                  | Build complete                 |
            |                                  | Touch flag file                |
            |                                  |                                |
~09:22:20   | Detect flag, copy RPMs           |                                |
            | Install 25.07 RPMs, depmod       |                                |
            | Compare srcversion:              |                                |
            |   modinfo: 1852...(25.07)        |                                |
            |   sysfs:   AE60...(25.10!)       |                                |
            |   MISMATCH -> DOWNGRADE          |                                |
            |                                  |                                |
09:22:22    | Write OFED blacklist             |                                |
            | /etc/init.d/openibd restart      |                                |
            |                                  |                                |
            | +------------------------------+ |                                |
            | | CRITICAL DISRUPTION WINDOW   | |                                |
09:22:22    | | Unloading HCA driver [OK]    | |                                | mlx5_core UNLOADED
            | |                              | |                                | *** ALL 50 VFs GONE ***
            | |        ~2 minutes            | |                                |
            | |                              | |                                |
09:24:22    | | Loading HCA driver [OK]      | |                                | mlx5_core 25.07 LOADED
            | +------------------------------+ |                                | PF back, no VFs yet
            |                                  |                                |
09:24:22    | Remove blacklist                 |                                |
            | restore_sriov_config:            |                                |
            |   PF UP, echo 50 > sriov_numvfs |                                | 50 VFs CREATING...
            |                                  |                                |
09:24:41    |   Restore VF0 (0000:01:00.3)    |                                |
            |   netdev = "eth0" (wrong name!)  |                                | VF0 netdev NOT READY
            |   "Cannot find device eth0"      |                                |
            |                                  |                                |
09:24:42    | *** VF RESTORE FAILED ***        |                                |
            |   exit immediately               |                                |
            |   (NO debug sleep -- DEBUG off)  |                                |
            |   EXIT trap: remove blacklist    |                                |
            |                                  |                                |
            | RESTART (CrashLoopBackOff)       |                                |
            | Driver already loaded (match)    |                                | VFs now initialized
            | Skip reload, VFs restored        |                                |
            |                                  |                                |
09:24:45    | Set driver ready                 |                                | 50 VFs READY
            | sleep infinity                   |                                |
            ================================== ================================ =========================
```

---

## Phase 2: Upgrade (MOFED 25.07 -> 25.10-1.2.8.0)

```
TIME        MOFED CONTAINER (25.07/25.10)      DTK SIDECAR                      mlx5_core / VFs
09:28       ================================== ================================ =========================
            |                                  |                                |
09:28:48    | SIGTERM received (x3 instances)  |                                | MOFED 25.07 loaded
            |   Terminate event caught         |                                | 50 VFs present
            |   Unset driver readiness         |                                |
            |   RESTORE_DRIVER=false           |                                |
            |   (keep loaded 25.07 driver)     |                                |
            |   No debug sleep (DEBUG off)     |                                |
            |   exit within 30s grace period   |                                |
            |                                  |                                |
09:29:22    | NEW CONTAINER (25.10 image)       |                                |
            | Copy files to DTK shared dir     | Start compiling MOFED 25.10    |
            | Poll for DTK flag (sleep 300s)   | - mlnx-tools 2510              |
            |                                  | - ofed-scripts 25.10           |
            |                                  | - mlnx-ofa_kernel 25.10        |
            |                                  |   (built twice)                |
            |                                  | - xpmem 2510                   |
            |                                  | - kernel-mft 4.34.0            |
            |                                  |   (built twice)                |
            |                                  |                                |
~09:34:02   |                                  | Build complete (~5 min)        |
            |                                  | Touch flag file                |
            |                                  |                                |
09:34:23    | Detect flag, copy RPMs           |                                |
            | Install 25.10 RPMs, depmod       |                                |
            | Compare: loaded 25.07 != 25.10   |                                |
            |   MISMATCH -> reload needed      |                                |
            |                                  |                                |
09:34:23    | Write OFED blacklist             |                                |
            | /etc/init.d/openibd restart      |                                |
            |                                  |                                |
            | +------------------------------+ |                                |
            | | CRITICAL DISRUPTION WINDOW   | |                                |
09:34:23    | | Unloading HCA driver [OK]    | |                                | mlx5_core UNLOADED
            | |                              | |                                | *** ALL 50 VFs GONE ***
            | |       ~2.5 minutes           | |                                |
            | |                              | |                                |
09:36:55    | | Loading HCA driver [OK]      | |                                | mlx5_core 25.10 LOADED
            | +------------------------------+ |                                | PF back, no VFs yet
            |                                  |                                |
09:36:55    | Remove blacklist                 |                                |
09:36:56    | restore_sriov_config:            |                                |
            |   PF UP, echo 50 > sriov_numvfs |                                | 50 VFs CREATING...
            |   sleep 4 (BIND_DELAY_SEC)      |                                |
            |                                  |                                |
            | +--------------------------------------------------+             |
            | | VF RESTORE LOOP (~6-7 sec each)                  |             |
            | |                                                  |             |
09:37:17    | |  VF0  0000:01:00.3  SUCCESS  enp1s0f0v0         |             |
09:37:24    | |  VF1  0000:01:00.4  SUCCESS                     |             |
09:37:30    | |  VF2  0000:01:00.5  SUCCESS                     |             |
09:37:36    | |  VF3  0000:01:00.6  SUCCESS                     |             |
09:37:43    | |  VF4  0000:01:00.7  SUCCESS                     |             |
09:37:49    | |  VF5  0000:01:01.0  SUCCESS                     |             |
09:37:56    | |  VF6  0000:01:01.1  SUCCESS                     |             |
09:38:02    | |  VF7  0000:01:01.2  SUCCESS                     |             |
09:38:08    | |  VF8  0000:01:01.3  SUCCESS                     |             |
09:38:15    | |  VF9  0000:01:01.4  SUCCESS                     |             |
09:38:21    | |  VF10 0000:01:01.5  SUCCESS                     |             |
09:38:28    | |  VF11 0000:01:01.6  SUCCESS                     |             |
09:38:34    | |  VF12 0000:01:01.7  SUCCESS                     |             |
09:38:41    | |  VF13 0000:01:02.0  SUCCESS                     |             |
09:38:47    | |  VF14 0000:01:02.1  SUCCESS                     |             |
09:38:54    | |  VF15 0000:01:02.2  SUCCESS                     |             |
            | |                                                  |             |
09:39:00    | |  VF16 0000:01:02.3  *** FAIL ***                |             | VF16 netdev NOT READY
            | |  /sys/.../net/ not found                        |             |
            | |  "dev argument required"                        |             |
            | +--------------------------------------------------+             |
            |                                  |                                |
09:39:00    | *** VF RESTORE FAILED ***        |                                |
            |   exit immediately (no debug)    |                                |
            |                                  |                                |
09:39:03    | RESTART (CrashLoopBackOff)       |                                |
            | Driver already loaded (match)    |                                | VFs now initialized
            | ethtool: 25.10-1.2.2            |                                |
            | Mount rootfs                     |                                |
            | Set driver readiness             |                                |
            |                                  |                                |
09:39:04    | sleep infinity                   |                                | 50 VFs READY
            ================================== ================================ =========================
```

---

## GPU Operator (parallel, independent)

```
TIME        GPU DRIVER CONTAINER               DTK SIDECAR (GPU)
09:24       ================================== ================================
            |                                  |
09:24:49    | Start, wait for DTK              | Building GPU modules
            |                                  |   nvidia.ko (open)
09:25:04    | DTK started                      |   nvidia-uvm.ko
            |                                  |   nvidia-modeset.ko
09:25:19    | Waiting for build...             |   nvidia-peermem.ko
            |                                  |
            | Copy modules from shared vol     | Build complete
            | Load: nvidia, nvidia-uvm,        |
            |   nvidia-modeset, nvidia-peermem |
            | Detect Mellanox @ 0000:01:00.0   |
            |   -> load nvidia-peermem         |
            | Start nvidia-persistenced        |
            | Mount rootfs (SELinux)            |
            | "Done, now waiting for signal"   |
            ================================== ================================
```

---

## Overall Timeline Summary

```
09:17  09:20  09:22  09:24  09:25  ...  09:28  09:29  09:34  09:37  09:39
  |      |      |      |      |           |      |      |      |      |
  +------+------+------+------+-----------+------+------+------+------+
  |  DTK |      |openibd| VF   | Ready    | TERM |  DTK |openibd| VF   | READY
  | build|      |restart| fail | (restart)|      | build|restart| loop |
  |      |      | (~2m) | exit |          |      |(~5m) |(~2.5m)| 16ok |
  |      |      |       | imm  |          |      |      |       | VF16 |
  |      |      |       |      |          |      |      |       | fail |
  |      |      |       +------+          |      |      |       | exit |
  |      |      |       VF0 fail          |      |      |       | imm  |
  |      |      |       no debug sleep    |      |      |       |      |
  |      |      |                         |      |      |       |      |
  +------+------+------+------+-----------+------+------+------+------+

  |<--- INSTALL PHASE --->|                |<-------- UPGRADE PHASE -------->|
  |  ~7.5 min to stable   |                |  ~10 min (terminate to ready)   |
```

---

## Side-by-Side: Test 6 vs Test 7 Upgrade

```
TEST 6 UPGRADE                              TEST 7 UPGRADE
==============                              ==============

08:27:39  SIGTERM + 90s debug sleep         09:28:48  SIGTERM (x3) immediate exit
08:29:13  New container starts              09:29:22  New container starts
08:29:13  DTK build starts                  09:29:22  DTK build starts
08:34:02  DTK build done (5 min)            09:34:02  DTK build done (5 min)
08:34:15  openibd restart                   09:34:23  openibd restart
          |                                           |
          |  *** VFs GONE (~2 min) ***                |  *** VFs GONE (~2.5 min) ***
          |                                           |
08:36:20  openibd done                      09:36:55  openibd done
08:36:42  VF0 restore: OK                   09:37:17  VF0 restore: OK
08:36:48  VF1 restore: FAIL                 09:37:24  VF1 restore: OK
          |                                 09:37:30  VF2 restore: OK
          |  90s debug sleep                09:37:36  VF3 restore: OK
          |                                    ...       ...
08:38:18  Container restart                 09:38:54  VF15 restore: OK
08:38:23  READY (driver 25.10-1.2.2)        09:39:00  VF16 restore: FAIL
                                                      |  immediate exit
          1 VF restored before failure      09:39:03  Container restart
                                            09:39:04  READY (driver 25.10-1.2.2)

                                                      16 VFs restored before failure

Total: ~11 min                              Total: ~10 min
```
