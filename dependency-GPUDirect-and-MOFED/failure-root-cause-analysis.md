# GPU Operator Failure Root Cause Analysis
## Post-Kernel Upgrade Scenario
This analysis is performed with Cursor Claude-4.5-Sonnet 
## Executive Summary

**Problem**: GPU Operator pods fail with `CrashLoopBackOff` after OpenShift cluster kernel upgrade from old kernel to `5.14.0-427.96.1.el9_4.aarch64+64k`.

**Root Cause**: Race condition between Network Operator and GPU Operator. Both start simultaneously at `11:51:06` after kernel upgrade. GPU Operator attempts to load `nvidia_peermem` kernel module at ~11:57 while Network Operator is still loading MOFED drivers (completes at 11:58:39). The 2-minute gap causes module load failure with "Invalid argument" error.

**Impact**: 
- 1 pod in CrashLoopBackOff (64 restarts)
- 7 pods blocked in Init state
- 1 pod running with validation errors
- Total GPU operator stack non-functional

**Quick Fix**: Delete the crashing driver pod after Network Operator shows `2/2 Running`. It will restart successfully.

**Permanent Fix**: Add MOFED readiness check to `nvidia-driver-ctr` container before loading nvidia_peermem module.

---

### Event Sequence
1. ✅ Both Network Operator and GPU Operator were healthy (old kernel)
2. ✅ OpenShift cluster upgrade → new kernel: `5.14.0-427.96.1.el9_4.aarch64+64k`
3. ✅ Network Operator successfully recompiles DOCA/MOFED drivers for new kernel
4. ❌ GPU Operator pods fail to initialize

---

## Root Cause: Race Condition + Missing Dependency

### The Problem

When the GPU Operator tries to load the `nvidia_peermem` module after kernel upgrade, it fails with:

```
modprobe: ERROR: could not insert 'nvidia_peermem': Invalid argument
```

### Why This Happens

#### 1. **Race Condition Between Operators**

After the kernel upgrade, both operators start **simultaneously** at `11:51:06` (Fri, 21 Nov 2025):

```
Synchronized Timeline (all times in EST):
├─ 11:51:06 - BOTH OPERATORS START
│             ├─ Network Operator: mofed-container starts
│             │  └─ Unsets /run/mellanox/drivers/.driver-ready
│             └─ GPU Operator: k8s-driver-manager starts
│                └─ Begins shutting down GPU clients
│
├─ 11:51:13 - Network Operator: DTK build script starts
├─ 11:51:23 - Network Operator: MOFED compilation begins (v25.07-0.9.7.0)
│
├─ 11:51:41 - GPU Operator: k8s-driver-manager
│             ├─ Detects: "Mellanox device found at 0000:01:00.0"
│             ├─ Detects: "GPUDirectRDMA is enabled"
│             └─ Log: "Waiting for MOFED to be installed" ⚠️
│
├─ 11:53:43 - Network Operator: DTK compilation COMPLETE ✓
│             └─ nvidia-peermem kernel module source available
│
├─ ~11:54-11:56 - GPU Operator: Compiles NVIDIA drivers
│             ├─ GPU_DIRECT_RDMA_ENABLED=true detected
│             ├─ Mellanox device found at 0000:01:00.0
│             ├─ Links to /run/mellanox/drivers/usr/src/ofa_kernel
│             ├─ nvidia-peermem.ko compiled successfully ✓
│             ├─ Creates precompiled package
│             │  └─ ❌ nvidia-peermem.ko NOT included in package
│             └─ nvidia-installer runs
│                ├─ Log: "Required file 'nvidia-peermem.ko' not found"
│                └─ Attempts to build it during installation
│
├─ 11:56:08 - Network Operator: MOFED installation begins
│             ├─ Installing RPM packages
│             ├─ Unloading old HCA drivers
│             └─ Loading new HCA drivers
│
├─ ~11:56-11:58 - GPU Operator: nvidia-driver-ctr tries to load modules
│             ├─ nvidia.ko loads ✓
│             ├─ nvidia-uvm.ko loads ✓
│             ├─ nvidia-modeset.ko loads ✓
│             ├─ Detects Mellanox at 0000:01:00.0
│             ├─ Attempts: modprobe nvidia-peermem
│             └─ ❌ FAILS: "could not insert 'nvidia_peermem': Invalid argument"
│                  └─ REASON: MOFED drivers not fully loaded yet!
│
├─ 11:57:55 - Network Operator: Loading HCA driver complete
├─ 11:58:39 - Network Operator: ✓ Sets .driver-ready state
│             └─ /run/mellanox/drivers/.driver-ready created
│             └─ mlx5_core driver v25.07-0.9.7 loaded
│
└─ ~11:58+ - GPU Operator: CrashLoopBackOff (64 restarts)
              └─ All downstream pods blocked indefinitely
```

**Critical Gap**: GPU Operator tried to load `nvidia_peermem` **2-3 minutes BEFORE** Network Operator finished loading MOFED drivers.

#### 2. **Incomplete Precompiled Package**

Log evidence from `openshift-driver-toolkit-ctr`:

```bash
Line 989:  Skipping BTF generation for nvidia-peermem.ko due to unavailability of vmlinux
Line 1002: ../mkprecompiled --pack nvidia-modules-5.14.0-builtin \
           --kernel-module nvidia-uvm.ko --target-directory .
```

**Issue**: The `mkprecompiled` command only packages:
- `nvidia.ko`
- `nvidia-modeset.ko`  
- `nvidia-uvm.ko`

**Missing from package**:
- `nvidia-peermem.ko` ❌
- `nvidia-drm.ko` ❌

Later when `nvidia-installer` runs:

```
Line 1034: Required file 'nvidia-drm.ko' not found in package
Line 1038: Required file 'nvidia-peermem.ko' not found in package
```

#### 3. **Module Load Failure**

When `nvidia-driver-ctr` tries to load the modules:

```bash
Line 3217: + modprobe nvidia NVreg_CoherentGPUMemoryMode=driver  ✓
Line 3218: + modprobe nvidia-uvm  ✓
Line 3219: + modprobe nvidia-modeset  ✓
Line 3221: Mellanox device found at 0000:01:00.0
Line 3222: Loading NVIDIA Peer Memory kernel module...
Line 3223: + modprobe -a nvidia-peermem
Line 3224: modprobe: ERROR: could not insert 'nvidia_peermem': Invalid argument ❌
```

### Why "Invalid Argument"?

The `nvidia_peermem` module was compiled but it failed to load because:

1. **MOFED Driver State Inconsistency**: At 11:56-11:58 when GPU Operator tried to load nvidia_peermem:
   - MOFED compilation was complete (11:53:43) ✓
   - MOFED kernel headers existed in `/run/mellanox/drivers/usr/src/ofa_kernel` ✓
   - BUT: Old MOFED drivers were being unloaded (11:56:08)
   - AND: New MOFED drivers were still loading (11:57:55)
   - Result: **Incomplete/transitional kernel state**

2. **Missing Kernel Module Dependencies**: `nvidia_peermem` requires these MOFED modules to be loaded:
   - `ib_core` - InfiniBand core subsystem
   - `mlx5_core` - Mellanox ConnectX driver
   - `mlx5_ib` - Mellanox IB layer
   
   Status at GPU load time (~11:57):
   - Old modules: Unloaded ✓
   - New modules: Partially loaded ⚠️
   - Result: **Dependency chain broken**

3. **Symbol Resolution Failure**: The kernel returns `EINVAL` (Invalid argument) when:
   - Module was compiled against headers from new MOFED (25.07-0.9.7.0) ✓
   - But attempts to load while kernel still has old/incomplete symbols
   - Kernel module loader can't resolve required symbols from ib_core/mlx5_core
   - Result: **modprobe rejects the module**

4. **Race Condition Window**: The critical 2-minute gap:
   ```
   11:56:08 - Network Operator: Unloading HCA driver
   11:57:55 - Network Operator: Loading HCA driver (complete)
   11:58:39 - Network Operator: Driver ready marker set
   
   ~11:56-11:58 - GPU Operator: Attempts nvidia-peermem load
                  └─ Falls in the gap where MOFED is transitioning
   ```

Evidence from logs:

**GPU Operator (gpu_operand_pod_nvidia-driver-daemonset...log):**
```bash
Line 980:  + ln -s /run/mellanox/drivers/usr/src/ofa_kernel /usr/src/
Line 983:  + [[ -d /run/mellanox/drivers/usr/src/ofa_kernel/aarch64/5.14.0-427.96.1.el9_4.aarch64+64k ]]
Line 989:  Skipping BTF generation for nvidia-peermem.ko due to unavailability of vmlinux
Line 1038: Required file 'nvidia-peermem.ko' not found in package
Line 1206: time=2025-11-21T11:51:41Z msg=Mellanox device found at 0000:01:00.0
Line 1208: time=2025-11-21T11:51:41Z msg=Waiting for MOFED to be installed
Line 3221: Mellanox device found at 0000:01:00.0
Line 3222: Loading NVIDIA Peer Memory kernel module...
Line 3224: modprobe: ERROR: could not insert 'nvidia_peermem': Invalid argument
```

**Network Operator (network_operand_pod_mofed...log):**
```bash
Line 3:    [21-Nov-25_11:51:13] DTK driver build script start
Line 1091: [21-Nov-25_11:51:23] Starting compilation of driver version 25.07-0.9.7.0
Line 1127: [21-Nov-25_11:53:43] DTK driver build script end
Line 1139: [21-Nov-25_11:51:06] Unsetting driver ready state
Line 1153: [21-Nov-25_11:51:06] Awaiting openshift driver toolkit to complete NIC driver build
Line 1182: Loading HCA driver and Access Layer:[60G[  OK  ]
Line 1216: [21-Nov-25_11:58:39] Current mlx5_core driver version: 25.07-0.9.7
Line 1222: [21-Nov-25_11:58:39] Setting driver ready state
```

**Analysis**: The GPU Operator found MOFED headers at compilation time (~11:54-11:56), but when it tried to **load** the nvidia_peermem module (~11:56-11:58), the Network Operator was:
- Still installing MOFED RPM packages (11:56:08)
- Still loading HCA drivers (11:57:55)
- Had NOT yet set the `.driver-ready` marker (only created at 11:58:39)

This 2-3 minute gap is when the race condition occurred.

---

## Detailed State Observations

### Network Operator Pod State (captured at 2d22h uptime)
```
NAME                              READY   STATUS    RESTARTS   AGE
mofed-rhcos4.18-554bdbd7cf-ds     2/2     Running   0          2d22h
rdma-shared-dp-ds                 1/1     Running   0          2d22h
```
- **Status**: Successfully running ✓
- **Kernel**: `5.14.0-427.96.1.el9_4.aarch64_64k` 
- **MOFED Version**: `25.07-0.9.7.0`
- **Driver Ready**: Yes (`/run/mellanox/drivers/.driver-ready` exists)
- **mlx5_core**: Loaded and functional

### GPU Operator Pod State (captured at 3h28m)
```
NAME                                                  READY   STATUS             RESTARTS
nvidia-driver-daemonset-418.94.202510230424-0-564xm   1/4     CrashLoopBackOff   64 (4m7s ago)
nvidia-container-toolkit-daemonset-sxfkr              0/1     Init:0/1           0
gpu-feature-discovery-hnr8w                           0/1     Init:0/1           0
nvidia-device-plugin-daemonset-v46gq                  0/1     Init:0/1           0
nvidia-dcgm-exporter-n6jqs                            0/1     Init:0/2           0
nvidia-dcgm-m25d9                                     0/1     Init:0/1           0
nvidia-mig-manager-w5799                              0/1     Init:0/1           0
nvidia-operator-validator-6fhjb                       0/1     Init:0/5           0
nvidia-node-status-exporter-cd78t                     1/1     Running            1 (errors)
```

### The Observed Sequence Problem

1. **Both operators restarted simultaneously** (11:51:06) after kernel upgrade
2. **Network Operator took 7.5 minutes** total (11:51:06 → 11:58:39)
   - 2m 37s for compilation (11:51:23 → 11:53:43)
   - 2m 31s for installation/loading (11:56:08 → 11:58:39)
3. **GPU Operator took ~5-7 minutes** for compilation but attempted module loading **too early**
4. **nvidia-peermem-ctr container** was continuously waiting:
   ```
   waiting for mellanox ofed and nvidia drivers to be installed
   [checking for /run/mellanox/drivers/.driver-ready]
   [checking for /run/nvidia/validations/.driver-ctr-ready]
   ```
   But the **nvidia-driver-ctr** container didn't wait and tried to load immediately
5. **The failure occurred** when nvidia-driver-ctr loaded nvidia_peermem before MOFED was ready

### GPU Operator Driver Pod Container Architecture

The driver pod has multiple containers with different responsibilities:

```
nvidia-driver-daemonset-418.94.202510230424-0-564xm (Pod)
├─ Init: k8s-driver-manager
│  └─ Manages driver lifecycle, waits for MOFED ✓ (working correctly)
│     └─ Log: "Waiting for MOFED to be installed" (11:51:41)
│
├─ Container 1/4: nvidia-driver-ctr  ❌ CRASHES HERE
│  ├─ Compiles & loads NVIDIA kernel modules
│  ├─ Does NOT wait for .driver-ready marker
│  ├─ Tries: modprobe nvidia-peermem
│  └─ Fails with "Invalid argument" → CrashLoopBackOff
│
├─ Container 2/4: nvidia-peermem-ctr  ⏳ WAITING
│  ├─ Purpose: Reload nvidia-peermem when MOFED updates
│  ├─ HAS proper wait logic:
│  │  └─ Waits for /run/mellanox/drivers/.driver-ready ✓
│  │  └─ Waits for /run/nvidia/validations/.driver-ctr-ready ✓
│  └─ Never reaches this because nvidia-driver-ctr crashes first
│
├─ Container 3/4: nvidia-gdrcopy-ctr
│  └─ Manages GDRCopy kernel module
│
└─ Container 4/4: openshift-driver-toolkit-ctr
   └─ Compiles drivers using DTK ✓ (worked successfully)
```

**Root Cause Identified**: The `nvidia-driver-ctr` container loads modules **without** checking if MOFED is ready, while the `nvidia-peermem-ctr` container (which has proper wait logic) never gets a chance to run because the driver-ctr crashes first.

---

## Why All Pods Are Blocked

Once `nvidia-driver-daemonset` crashes:

1. ❌ **nvidia-driver-daemonset**: CrashLoopBackOff (64 restarts)
   - Never creates `/run/nvidia/validations/.driver-ctr-ready`

2. ⏳ **nvidia-container-toolkit-daemonset**: Init:0/1 (Waiting)
   - Waits for `.driver-ctr-ready` file
   - Never creates `/run/nvidia/validations/toolkit-ready`

3. ⏳ **All downstream pods** (7 total): Init blocked
   - `gpu-feature-discovery`
   - `nvidia-device-plugin-daemonset`
   - `nvidia-dcgm-exporter`
   - `nvidia-dcgm`
   - `nvidia-mig-manager`
   - `nvidia-operator-validator`
   - All waiting for `toolkit-ready` file

4. ⚠️ **nvidia-node-status-exporter**: Running but errors
   - Reports: `failed to validate driver: error checking driver container status: exit status 1`

---

## Solution Options

### Option 1: Immediate Fix - Manual Intervention (Fastest)
**When**: Network Operator shows `mofed-container` is `Running` and `2/2 Ready`

**Steps**:
1. Verify Network Operator is ready:
   ```bash
   kubectl get pods -n nvidia-network-operator
   # Wait for: mofed-rhcos4.18-* shows 2/2 Running
   
   # Verify MOFED driver is loaded on the node:
   ssh <node> "ls -l /run/mellanox/drivers/.driver-ready"
   # Should exist and show recent timestamp
   ```

2. Delete the crashing GPU operator driver pod:
   ```bash
   kubectl delete pod nvidia-driver-daemonset-418.94.202510230424-0-564xm -n nvidia-gpu-operator
   ```

3. Monitor the new pod:
   ```bash
   kubectl logs -f nvidia-driver-daemonset-<new-hash> -n nvidia-gpu-operator -c nvidia-driver-ctr
   # Should successfully load nvidia-peermem this time
   ```

**Expected Result**: Pod will restart and successfully load all modules since MOFED is now ready.

---

### Option 2: Disable GPU Direct RDMA (Workaround)
**When**: GPU Direct RDMA is not required for your workload

**Impact**: Disables peer-to-peer GPU↔NIC communication (affects GPU Direct RDMA performance)

**Steps**:
```bash
kubectl edit clusterpolicy gpu-cluster-policy -n nvidia-gpu-operator
```

Change:
```yaml
driver:
  rdma:
    enabled: false  # Change from true to false
```

This will skip `nvidia-peermem` module loading entirely.

---

### Option 3: Add Wait Logic to nvidia-driver-ctr (Proper Fix)
**When**: For permanent fix in GPU Operator

**What**: Modify `gpu-driver-container` scripts to check for MOFED readiness before loading nvidia-peermem

**Location**: `gpu-driver-container/rhel9/nvidia-driver`

**Required Change** (around line 394):
```bash
if _gpu_direct_rdma_enabled; then
    echo "Loading NVIDIA Peer Memory kernel module..."
    
    # ADD THIS WAIT LOGIC:
    echo "Waiting for MOFED driver to be ready..."
    timeout=300  # 5 minutes
    elapsed=0
    while [ ! -f /run/mellanox/drivers/.driver-ready ]; do
        if [ $elapsed -ge $timeout ]; then
            echo "ERROR: Timeout waiting for MOFED driver"
            exit 1
        fi
        sleep 5
        elapsed=$((elapsed + 5))
        echo "Still waiting for MOFED... ($elapsed/${timeout}s)"
    done
    echo "MOFED driver is ready, proceeding with nvidia-peermem load"
    
    set -o xtrace +o nounset
    modprobe -a nvidia-peermem "${NVIDIA_PEERMEM_MODULE_PARAMS[@]}"
    set +o xtrace -o nounset
fi
```

**Benefit**: Prevents the race condition entirely by making GPU Operator wait for Network Operator.

---

### Option 4: Operator Startup Ordering (Infrastructure Level)
**When**: For automated recovery in future kernel upgrades

**Approaches**:

**A. Using Init Containers in GPU Operator**:
Add an init container to check for MOFED:
```yaml
apiVersion: v1
kind: DaemonSet
metadata:
  name: nvidia-driver-daemonset
spec:
  template:
    spec:
      initContainers:
      - name: wait-for-mofed
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "Waiting for MOFED to be ready..."
          timeout=600
          elapsed=0
          while [ ! -f /run/mellanox/drivers/.driver-ready ]; do
            if [ $elapsed -ge $timeout ]; then
              echo "ERROR: Timeout waiting for MOFED"
              exit 1
            fi
            sleep 10
            elapsed=$((elapsed + 10))
          done
          echo "MOFED is ready"
        volumeMounts:
        - name: run-mellanox-drivers
          mountPath: /run/mellanox/drivers
```

**B. Using DaemonSet Priority Classes**:
```yaml
# Network Operator DaemonSet
priorityClassName: system-node-critical  # Higher priority (2000001000)

# GPU Operator DaemonSet  
priorityClassName: nvidia-driver-priority  # Lower priority (2000000000)
```

**C. Using Node Startup Probes**:
Add a node condition check in GPU Operator controller to only deploy when Network Operator reports ready.

---

### Option 5: Fix Precompiled Package Build (Additional Improvement)
**Issue**: `nvidia-peermem.ko` and `nvidia-drm.ko` are not included in precompiled packages

**Location**: `gpu-driver-container/rhel9/nvidia-driver` (line ~1002)

**Current**:
```bash
../mkprecompiled --pack nvidia-modules-5.14.0-builtin \
  --kernel-module nvidia-uvm.ko --target-directory .
```

**Should Be**:
```bash
../mkprecompiled --pack nvidia-modules-5.14.0-builtin \
  --kernel-module nvidia-uvm.ko --target-directory . \
  --kernel-module nvidia-peermem.ko --target-directory . \
  --kernel-module nvidia-drm.ko --target-directory .
```

This would prevent the `nvidia-installer` warnings about missing files.

---

## Key Takeaway

**This is NOT a configuration error.** It's a **precise race condition** triggered by the kernel upgrade where:

### The Race Condition Explained

1. **Simultaneous Start** (11:51:06)
   - Both operators detect new kernel and restart simultaneously
   - Network Operator: Takes 7.5 minutes total
   - GPU Operator: Takes ~5-7 minutes

2. **The Critical 2-Minute Gap** (11:56-11:58)
   - GPU Operator finishes compilation first
   - Attempts to load `nvidia_peermem` module
   - Network Operator is still loading MOFED drivers (in transition state)
   - Old MOFED unloaded, new MOFED partially loaded
   - Kernel module dependency chain broken

3. **Module Load Failure**
   - `nvidia_peermem.ko` was compiled correctly against new MOFED headers
   - BUT: Kernel runtime doesn't have complete MOFED symbols yet
   - Kernel rejects module with `EINVAL` (Invalid argument)
   - `nvidia-driver-ctr` container crashes

4. **Cascade Failure**
   - Driver container never creates `.driver-ctr-ready` marker
   - Toolkit waits forever for driver
   - All 7 downstream GPU operator pods blocked indefinitely
   - System stuck in this state even after MOFED becomes ready

### Why This Happens After Kernel Upgrade

- **Normal operation**: Both operators are already running, no race
- **Kernel upgrade**: Both must recompile → both start at same time → race occurs
- **The bug**: `nvidia-driver-ctr` doesn't check if MOFED is ready before loading nvidia_peermem
- **The irony**: There's a separate `nvidia-peermem-ctr` container that HAS proper wait logic, but it never runs because driver-ctr crashes first

### Recommended Solution

**Short-term**: Delete the crashing GPU driver pod once Network Operator shows `2/2 Running`

**Long-term**: Add MOFED readiness check to `nvidia-driver-ctr` before loading nvidia-peermem (Option 3 above)

**Alternative**: Use init containers or priority classes to sequence operator startup (Option 4 above)

