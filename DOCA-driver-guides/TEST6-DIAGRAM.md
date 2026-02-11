# Test 6 -- Visual Timeline Diagram

## NicClusterPolicy Config

```
ENTRYPOINT_DEBUG=true | DEBUG_SLEEP_SEC_ON_EXIT=90 | terminationGracePeriodSeconds=90
RESTORE_DRIVER_ON_POD_TERMINATION=false | drain.enable=false
Install: Inbox -> MOFED 25.07 | Upgrade: MOFED 25.07 -> 25.10
```

## mlx5_core srcversion Transitions

```
Inbox                    MOFED 25.07                     MOFED 25.10
570B167417792AFC9308A50  1852822C2F9ABA6A93704E4         AE6029416A873D39E95B044
       |                        |                               |
       +--- Install loads ----->+                               |
                                +-------- Upgrade loads ------->+
                                                                |
                                                                +--> Stays after NIC policy delete
                                                                     (RESTORE_DRIVER=false)
```

---

## Phase 1: Install (MOFED 25.07-0.9.7.0)

```
TIME        MOFED CONTAINER                    DTK SIDECAR                      mlx5_core / VFs
07:47       ================================== ================================ =========================
            |                                  |                                |
07:47:25    | Start, poll for DTK flag         |                                | Inbox driver loaded
            | (sleep 300s)                     |                                | 50 VFs present
            |                                  |                                |
07:47:34    |                                  | Start compiling MOFED 25.07    |
            |                                  | - ofed-scripts                 |
            |                                  | - mlnx-tools                   |
            |                                  | - mlnx-ofa_kernel (mlx5_core)  |
            |                                  | - xpmem                        |
            |                                  | - kernel-mft                   |
            |                                  |                                |
07:49:57    |                                  | Build complete (~2.5 min)      |
            |                                  | Touch flag file, exit          |
            |              .  .  .             |                                |
            |          (still sleeping)        |                                |
            |              .  .  .             |                                |
07:52:25    | Detect flag, copy RPMs           |                                |
            | Install RPMs, depmod             |                                |
            |                                  |                                |
07:52:27    | Compare srcversion:              |                                |
            |   modinfo: 1852...(new 25.07)    |                                |
            |   sysfs:   570B...(inbox)        |                                |
            |   MISMATCH -> reload needed      |                                |
            |                                  |                                |
07:52:27    | modprobe tls, psample, macsec    |                                |
            | Write OFED blacklist to host     |                                |
            | /etc/init.d/openibd restart      |                                |
            |                                  |                                |
            | +------------------------------+ |                                |
            | | CRITICAL DISRUPTION WINDOW   | |                                |
07:52:27    | | Unloading HCA driver [OK]    | |                                | mlx5_core UNLOADED
            | |                              | |                                | *** ALL 50 VFs GONE ***
            | |        ~2 minutes            | |                                |
            | |                              | |                                |
07:54:24    | | Loading HCA driver [OK]      | |                                | mlx5_core 25.07 LOADED
            | +------------------------------+ |                                | PF back, no VFs yet
            |                                  |                                |
07:54:24    | Remove blacklist                 |                                |
            | restore_sriov_config:            |                                |
            |   PF enp1s0f0np0 UP             |                                |
            |   echo 50 > sriov_numvfs        |                                | 50 VFs CREATING...
            |                                  |                                | (kernel probing)
            |   (NO BIND_DELAY_SEC here!)     |                                |
            |                                  |                                |
07:54:43    |   Restore VF0 (0000:01:00.3)    |                                |
            |   netdev = "eth0" (wrong name!)  |                                | VF0 netdev not ready
            |   "Cannot find device eth0"      |                                |
            |                                  |                                |
07:54:44    | *** VF RESTORE FAILED ***        |                                |
            |   exit_entryp(1)                 |                                |
            |   ENTRYPOINT_DEBUG=true          |                                |
            |   +---------------------------+  |                                |
            |   | DEBUG SLEEP 90 seconds    |  |                                |
            |   | (container stays alive    |  |                                |
            |   |  for post-mortem debug)   |  |                                |
07:56:14    |   +---------------------------+  |                                |
            |   EXIT trap: remove blacklist    |                                |
            |   exit(1)                        |                                |
            |                                  |                                |
07:56:17    | RESTART (CrashLoopBackOff)       |                                |
            | Driver already loaded (match)    |                                | VFs now fully initialized
            | Skip reload                      |                                |
            | VF restore succeeds              |                                |
            | Set driver ready                 |                                |
            | sleep infinity                   |                                | 50 VFs READY
            ================================== ================================ =========================
```

---

## Phase 2: Upgrade (MOFED 25.07 -> 25.10-1.2.8.0)

```
TIME        MOFED CONTAINER (25.07/25.10)      DTK SIDECAR                      mlx5_core / VFs
08:27       ================================== ================================ =========================
            |                                  |                                |
08:27:39    | SIGTERM received                 |                                | MOFED 25.07 loaded
            |   Terminate event caught         |                                | 50 VFs present
            |   Unset driver readiness         |                                |
            |   RESTORE_DRIVER=false           |                                |
            |   (keep loaded 25.07 driver)     |                                |
            |   Unmount rootfs                 |                                |
            |   +---------------------------+  |                                |
            |   | DEBUG SLEEP 90 seconds    |  |                                |
08:29:09    |   +---------------------------+  |                                |
            |   exit(1)                        |                                |
            |                                  |                                |
08:29:13    | NEW CONTAINER (25.10 image)       |                                |
            | Copy files to DTK shared dir     | Start compiling MOFED 25.10    |
            | Poll for DTK flag (sleep 300s)   | - mlnx-tools 2510              |
            |                                  | - ofed-scripts 25.10           |
            |                                  | - mlnx-ofa_kernel 25.10        |
            |                                  |   (built twice)                |
            |                                  | - xpmem 2510                   |
            |                                  | - kernel-mft 4.34.0            |
            |                                  |   (built twice)                |
            |                                  |                                |
08:34:02    |                                  | Build complete (~5 min)        |
            |                                  | Touch flag file                |
            |                                  |                                |
08:34:13    | Detect flag, copy RPMs           |                                |
            | Install 25.10 RPMs, depmod       |                                |
            |                                  |                                |
08:34:15    | Compare srcversion:              |                                |
            |   modinfo: AE60...(new 25.10)    |                                |
            |   sysfs:   1852...(loaded 25.07) |                                |
            |   MISMATCH -> reload needed      |                                |
            |                                  |                                |
08:34:15    | modprobe tls, psample, macsec    |                                |
            | Write OFED blacklist to host     |                                |
            | /etc/init.d/openibd restart      |                                |
            |                                  |                                |
            | +------------------------------+ |                                |
            | | CRITICAL DISRUPTION WINDOW   | |                                |
08:34:15    | | Unloading HCA driver [OK]    | |                                | mlx5_core UNLOADED
            | |                              | |                                | *** ALL 50 VFs GONE ***
            | |        ~2 minutes            | |                                |
            | |                              | |                                |
08:36:20    | | Loading HCA driver [OK]      | |                                | mlx5_core 25.10 LOADED
            | +------------------------------+ |                                | PF back, no VFs yet
            |                                  |                                |
08:36:20    | Remove blacklist                 |                                |
            | restore_sriov_config:            |                                |
            |   PF enp1s0f0np0 UP             |                                |
            |   echo 50 > sriov_numvfs        |                                | 50 VFs CREATING...
            |   sleep 4 (BIND_DELAY_SEC)      |                                |
            |                                  |                                |
08:36:42    |   Restore VF0 (0000:01:00.3)    |                                | VF0 ready!
            |   netdev=enp1s0f0v0 (correct!)   |                                |
            |   set MAC, unbind, rebind, MTU   |                                |
            |   VF0 SUCCESS                    |                                |
            |                                  |                                |
08:36:48    |   Restore VF1 (0000:01:00.4)    |                                |
            |   /sys/.../net/ not found!       |                                | VF1 netdev NOT READY
            |   "dev argument required"        |                                |
            |                                  |                                |
08:36:48    | *** VF RESTORE FAILED ***        |                                |
            |   exit_entryp(255)               |                                |
            |   +---------------------------+  |                                |
            |   | DEBUG SLEEP 90 seconds    |  |                                |
08:38:18    |   +---------------------------+  |                                |
            |   exit(255)                      |                                |
            |                                  |                                |
08:38:23    | RESTART (CrashLoopBackOff)       |                                |
            | Driver already loaded (match)    |                                | VFs now initialized
            | Skip reload                      |                                |
            | ethtool: mlx5_core 25.10-1.2.2  |                                |
            | Mount rootfs                     |                                |
            | Set driver readiness             |                                |
            | sleep infinity                   |                                | 50 VFs READY
            ================================== ================================ =========================
```

---

## GPU Operator (parallel, independent)

```
TIME        GPU DRIVER CONTAINER               DTK SIDECAR (GPU)
08:38       ================================== ================================
            |                                  |
08:38:28    | Start, wait for DTK              | Building GPU modules
            |                                  |   nvidia.ko (open)
08:38:43    | DTK started                      |   nvidia-uvm.ko
            |                                  |   nvidia-modeset.ko
08:38:58    | Waiting for build...             |   nvidia-peermem.ko
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
07:47  07:50  07:52  07:54  07:56  ...  08:27  08:29  08:34  08:36  08:38
  |      |      |      |      |           |      |      |      |      |
  |      |      |      |      |           |      |      |      |      |
  +------+------+------+------+-----------+------+------+------+------+
  |  DTK |      | RPM  |openibd  VF fail  | TERM |  DTK |RPM   |openibd  READY
  | build|      | inst | restart  +debug  |      | build|inst  |restart
  |      |      |      |  (~2m)   sleep   |      |(~5m)|      | (~2m)
  | ~2.5 |      |      |          (90s)   |      |      |      |
  |  min |      |      |                  |      |      |      |
  |      |      |      |                  |      |      |      +-- VF0 OK
  |      |      |      +-- VF0 FAIL       |      |      |      +-- VF1 FAIL
  |      |      |                         |      |      |          +debug(90s)
  |      |      |      Container restart  |      |      |
  |      |      |      -> succeeds        |      |      |      Container restart
  |      |      |                         |      |      |      -> READY 08:38:23
  |      |      |                         |      |      |
  +------+------+------+------+-----------+------+------+------+------+

  |<---- INSTALL PHASE ---->|              |<--------- UPGRADE PHASE -------->|
  |    ~9 min (to stable)   |              |   ~11 min (terminate to ready)   |
```
