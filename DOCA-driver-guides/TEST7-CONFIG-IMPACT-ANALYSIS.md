# Test 7 -- Configuration Impact Analysis

How `terminationGracePeriodSeconds`, the DTK poll sleep, and `RESTORE_DRIVER_ON_POD_TERMINATION` affect the operator lifecycle in test 7 -- and how different settings compared to test 6 produce different behavior.

---

## 1. `terminationGracePeriodSeconds: 30`

### What It Is

Same Kubernetes pod-level setting as test 6, but set to **30 seconds** instead of 90. The kubelet will SIGKILL the container if it hasn't exited within 30 seconds of receiving SIGTERM.

### Key Difference from Test 6

Test 7 does **not** enable `ENTRYPOINT_DEBUG`, so there is no debug sleep on exit. The termination handler is fast (~instant), meaning 30 seconds is more than sufficient.

### Where It Matters in Test 7

#### Upgrade Termination (09:28:48)

```
09:28:48  SIGTERM received (x3 container instances)
          |
          |  termination_handler():
          |    1. Unset driver readiness    (~instant)
          |    2. RESTORE_DRIVER=false      (~instant)
          |       "Keeping currently loaded Mellanox OFED Driver..."
          |    3. Unmount rootfs            (~instant)
          |    4. Cleanup temp files        (~instant)
          |
          |  exit_entryp(1):
          |    ENTRYPOINT_DEBUG=false -> NO debug sleep
          |    exit(1) immediately
          |
          |  Total time: < 1 second
          |  Well within 30-second grace period
          |
09:29:22  New container starts (25.10 image)
          Time from SIGTERM to new container: ~34 seconds
          (includes Kubernetes pod scheduling overhead)
```

**The 30-second grace period is perfectly adequate** because there is no debug sleep and no driver restore to wait for.

### What Would Happen If ENTRYPOINT_DEBUG Were True

If test 7 had `ENTRYPOINT_DEBUG=true` with `DEBUG_SLEEP_SEC_ON_EXIT=300` (the default) but kept `terminationGracePeriodSeconds=30`:

```
09:28:48  SIGTERM received
          termination_handler completes (~instant)
          exit_entryp -> sleep 300 starts
          |
09:29:18  +30s: kubelet sends SIGKILL!
          Container FORCE KILLED mid-sleep
          |
          The cleanup already completed before the sleep,
          so no functional harm. But the exit is unclean.
          
          With the default DEBUG_SLEEP_SEC_ON_EXIT=300,
          you would need terminationGracePeriodSeconds >= 300.
```

This is why test 6 explicitly set both `DEBUG_SLEEP_SEC_ON_EXIT=90` and `terminationGracePeriodSeconds=90` -- they must be coordinated. Test 7 avoids this complexity by not enabling debug mode at all.

### Actual Impact on Test 7 Timeline

| Event | terminationGracePeriodSeconds effect |
|---|---|
| Install: VF fail at 09:24:42 | No effect (self-exit, not SIGTERM) |
| Upgrade: SIGTERM at 09:28:48 | 30s grace is sufficient (exit < 1s) |
| Upgrade: VF fail at 09:39:00 | No effect (self-exit, not SIGTERM) |

**Net impact**: The 30-second grace period adds essentially **zero wasted time**. The container exits almost instantly on SIGTERM, and the new container starts ~34 seconds later (mostly Kubernetes scheduling overhead).

### Test 6 vs Test 7: Termination Speed

```
TEST 6 (grace=90, debug=true, debug_sleep=90):
  08:27:39  SIGTERM
  08:29:09  exit (90 seconds later)
  08:29:13  new container starts
  OVERHEAD: 94 seconds from SIGTERM to new container

TEST 7 (grace=30, debug=false):
  09:28:48  SIGTERM
  09:28:49  exit (<1 second later)
  09:29:22  new container starts
  OVERHEAD: 34 seconds from SIGTERM to new container

SAVINGS: 60 seconds faster termination in test 7
```

---

## 2. DTK Poll Sleep (`sleep 300` with exponential backoff)

### The Poll Loop Behavior

Identical code to test 6. The MOFED container polls for the DTK build flag with a decaying sleep:

```
Iteration:  1     2     3     4     5     6    7    8    9    10
Sleep (s):  300   150   75    37    18    9    9    9    9    9
Cumulative: 300   450   525   562   580   589  598  607  616  625
```

Maximum total wait before timeout: ~625 seconds (~10.4 minutes).

### Where It Matters in Test 7

#### Install Phase (09:17:20)

```
09:17:20  MOFED container starts
          DTK flag not found
          sleep 300 seconds (first poll)        <-- 5-MINUTE WAIT
          |
09:17:20  DTK sidecar starts building          (parallel)
          |
~09:20    DTK build completes (~2.5 min)
          Flag file created
          |
          |  BUT: container sleeping until 09:17:20 + 300 = 09:22:20
          |
09:22:20  sleep 300 finishes, check flag -> FOUND
          Proceed with RPM install
```

**Wasted time: ~2.5 minutes** -- the DTK build finished around 09:20 but the container slept until 09:22:20.

#### Upgrade Phase (09:29:22)

```
09:29:22  MOFED container starts (new 25.10 image)
          DTK flag not found
          sleep 300 seconds (first poll)        <-- 5-MINUTE WAIT
          |
09:29:22  DTK sidecar starts building          (parallel)
          |
~09:34:02 DTK build completes (~5 min)
          Flag file created
          |
          |  Container sleeping until 09:29:22 + 300 = 09:34:22
          |
09:34:22  sleep 300 finishes, check flag -> FOUND
          Proceed with RPM install
```

**Wasted time: ~20 seconds** -- the 5-minute build nearly filled the 5-minute first poll.

### Impact Comparison

```
              Install              Upgrade
              =======              =======
DTK build:    ~2.5 min             ~5 min
Poll sleep:   5 min (300s)         5 min (300s)
Wasted wait:  ~2.5 min             ~20 sec
```

The initial 300-second sleep is optimized for the worst case (long build). For the MOFED 25.07 install (shorter build), it wastes significant time. For the MOFED 25.10 upgrade (longer build with double compile), it aligns well.

### What a Shorter Poll Would Save

If the initial poll were 30 seconds instead of 300:

```
INSTALL:  Build finishes at ~09:20, detected at ~09:20:30 (30s poll)
          vs. detected at 09:22:20 (300s poll)
          SAVINGS: ~2 minutes

UPGRADE:  Build finishes at ~09:34:02, detected at ~09:34:32 (30s poll)
          vs. detected at ~09:34:22 (300s poll)
          SAVINGS: ~10 seconds (negligible)
```

A shorter first poll with the same exponential backoff would primarily help with fast builds.

---

## 3. `RESTORE_DRIVER_ON_POD_TERMINATION: false`

### What It Achieves in Test 7

The same fundamental benefit as test 6, but with an additional nuance: test 7 starts with a **leftover MOFED 25.10 driver from test 6** because `RESTORE_DRIVER_ON_POD_TERMINATION=false` in test 6 left it on the host.

### The Chain of Consequences

```
TEST 6 ended:
  MOFED 25.10 loaded on host (RESTORE=false, driver stays)
  NicClusterPolicy deleted
  Pods removed
  Host still running MOFED 25.10
     |
     v
TEST 7 begins:
  NicClusterPolicy created with version: 25.07
  MOFED container starts
  Compares: loaded AE60...(25.10) vs installed 1852...(25.07)
  MISMATCH -> must reload (this is a DOWNGRADE)
  openibd restart: unload 25.10, load 25.07
  *** VFs destroyed and recreated ***
```

This shows that `RESTORE_DRIVER_ON_POD_TERMINATION=false` has a **persistent effect beyond the pod's lifetime**. The driver version that ends up on the host is determined by the last successful driver load, not by the container lifecycle.

### During Test 7 Upgrade (09:28:48)

```
WITH RESTORE_DRIVER_ON_POD_TERMINATION=false (actual):
======================================================
09:28:48  SIGTERM (3 instances, from CrashLoopBackOff)
          |
          |  Each instance exits cleanly:
          |    "Keeping currently loaded Mellanox OFED Driver..."
          |    No driver reload during termination
          |    exit in < 1 second
          |
          Host: MOFED 25.07 stays loaded, 50 VFs remain operational
          |
09:29:22  New container (25.10) starts
          Builds new driver, does ONE reload (25.07 -> 25.10)
          ONE disruption window


IF RESTORE_DRIVER_ON_POD_TERMINATION were true:
================================================
09:28:48  SIGTERM
          |
          |  termination_handler:
          |    /usr/sbin/mlnxofedctl --alt-mods force-restart
          |    *** DISRUPTION #1: openibd restart ~2 min ***
          |    *** ALL 50 VFs DESTROYED ***
          |    restore_sriov_config (50 VFs)
          |    *** ~5-10 min to restore 50 VFs ***
          |
          |  BUT terminationGracePeriodSeconds = 30!
          |  Kubelet SIGKILL after 30 seconds!
          |  openibd restart takes ~2 min > 30 seconds!
          |
09:29:18  SIGKILL! Container killed mid-openibd-restart!
          |
          |  HOST IN UNDEFINED STATE:
          |  - mlx5_core may be partially loaded
          |  - VFs may be partially configured
          |  - Blacklist file may still be on host
          |
09:29:22  New container starts into a broken state
          May fail unpredictably
```

**With `terminationGracePeriodSeconds=30` and `RESTORE_DRIVER_ON_POD_TERMINATION=true`, the upgrade would be CATASTROPHIC.** The openibd restart alone takes ~2 minutes, but the kubelet would kill the container after just 30 seconds, leaving the host's NIC driver stack in an inconsistent state.

### The Three Instances of SIGTERM

Test 7 logs show the termination event three times at 09:28:48. This is because the container was in CrashLoopBackOff from the VF restore failure during install. When the NicClusterPolicy version changed, Kubernetes terminated all three pending/running instances. Because `RESTORE_DRIVER=false`, each exits instantly -- no problem. If it were `true`, each would try to reload the driver, causing cascading disruptions.

### What `false` Achieves -- Summary for Test 7

| Benefit | Explanation |
|---|---|
| **Fast termination** | Exit in < 1 second vs. ~7-12 min for driver restore |
| **Compatible with short grace period** | 30s grace works because there's nothing slow to do |
| **No SIGKILL risk** | No long operation that could be interrupted |
| **Single disruption** | Only the NEW container does one driver reload |
| **Predictable host state** | Driver stays loaded as-is, known good state |
| **Handles CrashLoopBackOff** | Multiple terminations are harmless (each is instant) |
| **Cross-test persistence** | MOFED 25.10 from test 6 persisted into test 7 |

---

## Combined Impact: How All Three Settings Interact in Test 7

### The Critical Path During Upgrade

```
TIME    EVENT                                     SETTING THAT GOVERNS IT
======= ========================================= ================================

09:28:48  Old containers receive SIGTERM (x3)      Kubernetes (policy change)
          |
          |  termination_handler():
          |    Skip driver restore                 RESTORE_DRIVER=false
          |    No debug sleep                      ENTRYPOINT_DEBUG=false
          |    exit immediately                    
          |
          |  Kubelet 30s timer starts              terminationGracePeriodSeconds=30
          |  Container exits in < 1 second         (30s never needed)
          |
09:29:22  New container starts (25.10 image)
          |
          |  DTK poll: sleep 300 seconds           entrypoint.sh poll loop
          |  DTK sidecar builds in parallel        sleep_sec=300
          |
~09:34:02 |  DTK build completes (~5 min)
          |
09:34:22  |  Poll expires, flag found
          |
          |  RPM install, driver comparison
          |  srcversion mismatch -> reload
          |
09:34:23  |  /etc/init.d/openibd restart           entrypoint.sh restart_driver()
          |  *** ONLY DISRUPTION WINDOW ***
          |  (~2.5 min)
          |
09:36:55  |  Driver loaded, VF restore begins
          |  VF0-VF15 OK, VF16 FAIL
          |  exit immediately (no debug sleep)     ENTRYPOINT_DEBUG=false
          |
09:39:03  Container restarts
09:39:04  Driver ready (skip reload, succeed)

TOTAL UPGRADE: ~10 min
    - 0s wasted on debug sleep during termination (ENTRYPOINT_DEBUG=false)
    - ~20s wasted on DTK poll (build ~= poll time)
    - 0s wasted on driver restore (RESTORE=false)
    - 0s wasted on debug sleep after VF failure (ENTRYPOINT_DEBUG=false)
    - SINGLE disruption window of ~2.5 min
```

### Test 6 vs Test 7: Configuration Impact Comparison

```
                              TEST 6              TEST 7
                              ======              ======
terminationGracePeriod:       90s                 30s
ENTRYPOINT_DEBUG:             true                false
DEBUG_SLEEP_SEC_ON_EXIT:      90s                 N/A
RESTORE_DRIVER:               false               false

Time from SIGTERM to new:     94s                 34s      (-60s)
Time wasted on debug sleep    
  during termination:         90s                 0s       (-90s)
Time wasted on debug sleep
  after VF failure:           90s                 0s       (-90s)
DTK poll waste (install):     ~2.5 min            ~2.5 min (same)
DTK poll waste (upgrade):     ~11s                ~20s     (same)
Driver restore overhead:      0s (false)          0s (false) (same)
Disruption windows:           1 x ~2 min          1 x ~2.5 min
Total upgrade time:           ~11 min             ~10 min  (-1 min)

VFs restored before failure
  (upgrade):                  1                   16       (better)
```

### Key Takeaway

The three settings work together as a system:

1. **`RESTORE_DRIVER_ON_POD_TERMINATION=false`** is the most critical -- it eliminates double-disruption and makes termination fast enough to work with any grace period.

2. **`terminationGracePeriodSeconds`** must be >= `DEBUG_SLEEP_SEC_ON_EXIT` when `ENTRYPOINT_DEBUG=true`. Test 6 coordinates them at 90s. Test 7 avoids the issue entirely by not enabling debug mode, allowing a minimal 30s grace period.

3. **DTK poll sleep (300s)** affects startup time equally in both tests. It is most impactful when the DTK build is fast (install: ~2.5 min wasted). A shorter initial poll or an inotify/watch mechanism would improve both tests.
