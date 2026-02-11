# Test 6 -- Configuration Impact Analysis

How `terminationGracePeriodSeconds`, the DTK poll sleep, and `RESTORE_DRIVER_ON_POD_TERMINATION` affect the operator lifecycle in test 6.

---

## 1. `terminationGracePeriodSeconds: 90`

### What It Is

This is a Kubernetes pod-level setting propagated from `NicClusterPolicy.spec.ofedDriver.terminationGracePeriodSeconds`. It tells the kubelet: "after sending SIGTERM to this pod's containers, wait up to 90 seconds for them to exit gracefully before sending SIGKILL."

### Where It Matters in Test 6

Test 6 sets `ENTRYPOINT_DEBUG=true` and `DEBUG_SLEEP_SEC_ON_EXIT=90`. When the entrypoint script fails or is terminated, it sleeps 90 seconds for post-mortem debugging. This means the container needs the full 90 seconds of grace period to complete its shutdown cleanly.

#### During Upgrade -- Old Container Termination (08:27:39)

```
08:27:39  SIGTERM received
          |
          |  termination_handler() runs:
          |    1. Unset driver readiness          (~instant)
          |    2. Skip driver restore (false)     (~instant)
          |    3. Unmount rootfs                  (~instant)
          |    4. Cleanup temp files              (~instant)
          |
          |  exit_entryp(1) called:
          |    ENTRYPOINT_DEBUG=true, so:
          |    sleep 90  <-- DEBUG_SLEEP_SEC_ON_EXIT
          |    |
          |    |  This sleep is INSIDE the terminationGracePeriod
          |    |  If terminationGracePeriodSeconds < 90, kubelet
          |    |  would SIGKILL the container before sleep finishes
          |    |
08:29:09  |  sleep 90 finishes
          |  exit(1)
          |
08:29:13  New container starts (25.10 image)
```

**The 90-second grace period exactly matches the 90-second debug sleep.** This is by design in the NicClusterPolicy -- the operator who configured test 6 set both values to 90 to ensure the debug sleep can complete fully without being killed.

#### During VF Restore Failures (07:54:44 and 08:36:48)

When VF restore fails, `exit_entryp` is called with the debug sleep. But this is not a Kubernetes-initiated termination -- it is the container's own exit. The `terminationGracePeriodSeconds` does not apply here because the container is not being terminated by the kubelet. The container simply exits after the sleep, and Kubernetes restarts it via the pod's `restartPolicy`.

However, there is an important interaction: if Kubernetes sends SIGTERM to the pod _while_ the debug sleep is running (e.g., during a CrashLoopBackOff restart), the 90-second grace period ensures the termination handler has time to run and the container can exit cleanly.

### What Would Happen with a Shorter Grace Period

If `terminationGracePeriodSeconds` were 30 (like test 7):

```
08:27:39  SIGTERM received
          termination_handler runs (~instant)
          exit_entryp -> sleep 90 starts
          |
08:28:09  +30s: kubelet sends SIGKILL
          Container force-killed mid-sleep!
          Cleanup may be incomplete:
          - Blacklist file might remain on host
          - Rootfs mount might not be properly cleaned
```

The container would be killed 60 seconds before its debug sleep finishes. In this case the cleanup happens to complete before the sleep (the sleep is the _last_ thing before exit), so functionally it would be okay. But the shell would not exit cleanly -- it would be killed.

### Actual Impact on Test 6 Timeline

| Event | terminationGracePeriodSeconds effect |
|---|---|
| Install: VF fail at 07:54:44 | No effect (self-exit, not SIGTERM) |
| Upgrade: SIGTERM at 08:27:39 | 90s grace allows full debug sleep to complete before exit |
| Upgrade: VF fail at 08:36:48 | No effect (self-exit, not SIGTERM) |

**Net impact**: The 90-second grace period adds **~90 seconds** to the upgrade timeline between old container termination and new container start. The old container occupies the pod for 90 seconds doing nothing useful (just sleeping for debug purposes) before the new image can start.

---

## 2. DTK Poll Sleep (`sleep 300` with exponential backoff)

### What It Is

When the MOFED container starts on OpenShift, it relies on the DTK sidecar container to compile the kernel modules. The main container polls for a flag file (`dtk_done_compile_<version>`) indicating the build is complete. The polling loop in `entrypoint.sh`:

```bash
sleep_sec=300           # Start with 300-second (5-minute) poll interval
total_retries=10        # Maximum 10 retries
total_sleep_sec=0

while [ ! -f "${DTK_OCP_DONE_COMPILE_FLAG}" ] && [ ${total_retries} -gt 0 ]; do
    timestamp_print "Awaiting openshift driver toolkit to complete NIC driver build, next query in ${sleep_sec} sec"
    sleep ${sleep_sec}
    let total_sleep_sec+=sleep_sec
    if [ $sleep_sec -gt 10 ]; then
        sleep_sec=$((sleep_sec/2))    # Halve the interval each iteration
    fi
    total_retries=$((total_retries-1))
done
```

The sleep intervals form a decaying series: **300, 150, 75, 37, 18, 9, 9, 9, 9, 9** seconds (total max: ~625 seconds / ~10.4 minutes before timeout).

### Where It Matters in Test 6

#### Install Phase (07:47:25)

```
07:47:25  MOFED container starts
          DTK flag not found
          sleep 300 seconds (first poll)        <-- 5-MINUTE WAIT
          |
07:47:34  DTK sidecar starts building          (parallel)
          |
07:49:57  DTK build completes (~2.5 min)
          Flag file created
          |
          |  BUT the MOFED container is still sleeping!
          |  It started a 300-second sleep at 07:47:25
          |  Won't wake up until 07:47:25 + 300 = 07:52:25
          |
07:52:25  sleep 300 finishes, check flag -> FOUND
          Proceed with RPM install
```

**Wasted time**: The DTK build finished at 07:49:57, but the container didn't check until 07:52:25. That is **2 minutes and 28 seconds wasted** waiting for the first poll to expire.

#### Upgrade Phase (08:29:13)

```
08:29:13  MOFED container starts (new 25.10 image)
          DTK flag not found
          sleep 300 seconds (first poll)        <-- 5-MINUTE WAIT
          |
08:29:13  DTK sidecar starts building          (parallel)
          |
08:34:02  DTK build completes (~5 min)
          Flag file created
          |
          |  The MOFED container sleep started at 08:29:13
          |  First poll expiry: 08:29:13 + 300 = 08:34:13
          |
08:34:13  sleep 300 finishes, check flag -> FOUND
          Proceed with RPM install
```

**Wasted time**: DTK finished at 08:34:02, container checked at 08:34:13. Only **11 seconds wasted** here because the 5-minute build almost exactly matched the 5-minute first poll interval.

### Impact on Total Timeline

```
                    Install                 Upgrade
                    =======                 =======
DTK build time:     ~2.5 min                ~5 min
Poll sleep:         5 min (300s)            5 min (300s)
Wasted wait:        2 min 28 sec            11 sec
```

For the install, a shorter initial poll (e.g., 30 seconds) would have saved ~2.5 minutes. For the upgrade, the 300-second poll happened to align well with the build time.

### Why 300 Seconds?

The initial 300-second poll is conservative -- it assumes the DTK build could take several minutes (which it does on aarch64). The exponential backoff ensures that if the first poll misses, subsequent checks happen faster (150s, 75s, 37s, etc.). However, the first poll is always the longest wait, and for fast builds it adds significant unnecessary delay.

### Interaction with terminationGracePeriodSeconds

The DTK poll sleep runs during container startup, not during termination, so `terminationGracePeriodSeconds` does not directly affect it. However, if Kubernetes terminates the pod during the DTK wait (e.g., due to a policy change), the SIGTERM will interrupt the sleep and trigger `terminate_event`, which runs the termination handler. The 90-second grace period then governs how long the handler has to clean up.

---

## 3. `RESTORE_DRIVER_ON_POD_TERMINATION: false`

### What It Is

This environment variable controls what happens to the `mlx5_core` driver on the host when the MOFED container is terminated (SIGTERM). The logic in `termination_handler()`:

```bash
if [[ "${RESTORE_DRIVER_ON_POD_TERMINATION}" != false && "${new_driver_loaded}" == true ]]; then
    # RESTORE = true: reload the original inbox/host driver
    /usr/sbin/mlnxofedctl --alt-mods force-restart
    restore_sriov_config
else
    # RESTORE = false: keep the MOFED driver loaded on the host
    timestamp_print "Keeping currently loaded Mellanox OFED Driver..."
fi
```

### What `false` Achieves

Setting `RESTORE_DRIVER_ON_POD_TERMINATION=false` means:

1. **The MOFED driver stays loaded on the host** after the container exits
2. **No driver reload during termination** -- avoids the disruptive openibd restart
3. **No VF destruction during termination** -- the 50 VFs remain operational
4. **Faster container shutdown** -- the termination handler only needs to unset readiness, unmount rootfs, and clean up temp files (~instant)

### Where It Matters in Test 6

#### Upgrade Termination (08:27:39)

```
WITH RESTORE_DRIVER_ON_POD_TERMINATION=false (actual test 6):
=============================================================
08:27:39  SIGTERM
          Unset driver readiness              (~instant)
          "Keeping currently loaded OFED..."  (~instant)
          Unmount rootfs                      (~instant)
          Clean temp files                    (~instant)
          Debug sleep 90s
08:29:09  exit(1)

          Host state: MOFED 25.07 mlx5_core still loaded
                      50 VFs still present and operational
                      Network traffic uninterrupted

          New container starts and loads MOFED 25.10
          (driver reload happens in the NEW container, not during termination)


IF RESTORE_DRIVER_ON_POD_TERMINATION were true (hypothetical):
==============================================================
08:27:39  SIGTERM
          Unset driver readiness              (~instant)
          "Restoring Mellanox OFED Driver..."
          /usr/sbin/mlnxofedctl --alt-mods force-restart
          |
          | *** DISRUPTION: mlx5_core UNLOADED ***
          | *** ALL 50 VFs DESTROYED ***
          | ~2 minutes for openibd restart
          |
          | Then restore_sriov_config()
          | Recreate 50 VFs and configure each
          | ~5-10 minutes for 50 VFs
          |
          | TOTAL: ~7-12 minutes of disruption
          | DURING WHICH the old container is still running
          | and the new container cannot start yet
          |
~08:40    exit(1)   (if it even fits within terminationGracePeriodSeconds!)

          Host state: Inbox driver loaded
                      50 VFs recreated with inbox driver

          New container starts, IMMEDIATELY needs to reload MOFED 25.10
          ANOTHER ~2 minute openibd restart + VF restore
          DOUBLE DISRUPTION!
```

#### The Problem with `true`: Double Disruption + Grace Period Exhaustion

If `RESTORE_DRIVER_ON_POD_TERMINATION=true`:

1. **Termination handler takes ~7-12 minutes** (openibd restart + 50 VF restore)
2. But `terminationGracePeriodSeconds=90` -- kubelet would SIGKILL after 90 seconds!
3. The openibd restart (~2 min) alone exceeds 90 seconds
4. Container gets killed mid-driver-reload, leaving the host in an **undefined state**
5. Then the new container starts and has to reload again -- **double disruption**

Even with a much larger grace period (e.g., 900s), you would get:
- First disruption: old container restores inbox driver (~7-12 min of VF downtime)
- Second disruption: new container loads new MOFED driver (~2 min + VF restore)

### What `false` Achieves in the Full Timeline

```
               RESTORE=false (test 6)       RESTORE=true (hypothetical)
               =====================        ===========================

TERMINATION    Fast (~instant + debug)       Slow (~7-12 min)
               VFs stay up                   VFs destroyed & recreated twice
               Driver stays loaded           Driver reloaded to inbox

NETWORK        No disruption during          DISRUPTION #1: ~7-12 min
IMPACT         termination                   (during old container shutdown)

NEW CONTAINER  Only 1 driver reload needed   DISRUPTION #2: ~2 min + VF restore
               (new MOFED replaces old)      (new container reloads again)

TOTAL          1 disruption window (~2 min)  2 disruption windows (~14-24 min)
DISRUPTION
```

### After NIC Policy Deletion

Test 6 also captured what happens after the NicClusterPolicy is deleted:

```
mlx5_core srcversion after deletion: AE6029416A873D39E95B044 (MOFED 25.10)
```

Because `RESTORE_DRIVER_ON_POD_TERMINATION=false`, the MOFED 25.10 driver **remains loaded on the host permanently** even after the operator and all its pods are removed. The host continues running with the NVIDIA MOFED driver instead of the original inbox driver. This is the intended behavior -- the driver is a kernel-level component that persists across container lifecycle. To get back to the inbox driver, you would need to either:
- Set `RESTORE_DRIVER_ON_POD_TERMINATION=true` before deleting the policy
- Manually run `/usr/sbin/mlnxofedctl --alt-mods force-restart` on the host
- Reboot the node

---

## Combined Impact: How All Three Settings Interact

### The Critical Path During Upgrade

```
TIME    EVENT                                     SETTING THAT GOVERNS IT
======= ========================================= ================================

08:27:39  Old container receives SIGTERM           Kubernetes (policy change)
          |
          |  termination_handler():
          |    Skip driver restore                 RESTORE_DRIVER=false
          |    (saves ~7-12 min of disruption)     (~instant cleanup instead)
          |
          |  exit_entryp -> debug sleep 90s        DEBUG_SLEEP_SEC_ON_EXIT=90
          |  Kubelet waits patiently               terminationGracePeriodSeconds=90
          |                                        (must be >= debug sleep!)
08:29:09  |  Container exits cleanly
          |
08:29:13  New container starts (25.10 image)
          |
          |  DTK poll: sleep 300 seconds           entrypoint.sh poll loop
          |  (first poll, waiting for build)        sleep_sec=300
          |                                        
          |  DTK sidecar builds in parallel        (~5 min)
          |
08:34:13  |  Poll expires, flag found
          |
          |  RPM install, driver comparison
          |  srcversion mismatch -> reload
          |
08:34:15  |  /etc/init.d/openibd restart           entrypoint.sh restart_driver()
          |  *** ONLY DISRUPTION WINDOW ***
          |  (~2 min)
          |
08:36:20  |  Driver loaded, VF restore begins
          |  VF0 OK, VF1 FAIL
          |  exit_entryp -> debug sleep 90s        DEBUG_SLEEP_SEC_ON_EXIT=90
          |
08:38:18  Container restarts
08:38:23  Driver ready (skip reload, succeed)

TOTAL UPGRADE: ~11 min
    - 90s wasted on debug sleep during termination
    - ~0s wasted on DTK poll (build ~= poll time)
    - 0s wasted on driver restore (RESTORE=false)
    - ~90s wasted on debug sleep after VF failure
    - SINGLE disruption window of ~2 min
```

### Optimization Opportunities

| Setting | Current Value | Impact | Could Save |
|---|---|---|---|
| `terminationGracePeriodSeconds` | 90s | Allows debug sleep to complete during SIGTERM | N/A (needed for debug mode) |
| `DEBUG_SLEEP_SEC_ON_EXIT` / `ENTRYPOINT_DEBUG` | 90s / true | Adds 90s on every failure + termination | ~3 min (90s termination + 90s VF failure) |
| DTK poll `sleep_sec=300` | 300s initial | First poll may overshoot build completion | ~2.5 min on install (less on upgrade) |
| `RESTORE_DRIVER_ON_POD_TERMINATION` | false | Avoids double-disruption during upgrade | ~12-20 min of additional disruption |

**Most impactful setting**: `RESTORE_DRIVER_ON_POD_TERMINATION=false` saves the most time and avoids the most disruption. Without it, every upgrade would cause two full driver reload + VF restore cycles instead of one.
