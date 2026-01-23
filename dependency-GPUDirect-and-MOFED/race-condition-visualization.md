# Race Condition Visualization
## GPU Operator vs Network Operator Timing After Kernel Upgrade
This analysis is performed with Cursor Claude-4.5-Sonnet

```
Time          Network Operator                          GPU Operator                         State
═══════════════════════════════════════════════════════════════════════════════════════════════════
11:51:06      ┌─────────────────────┐                   ┌─────────────────────┐              
              │ mofed-container     │                   │ k8s-driver-manager  │              BOTH
              │ - Unset ready flag  │                   │ - Shutdown clients  │              START
              └─────────────────────┘                   └─────────────────────┘              
                       │                                          │                           
11:51:13      ┌─────────────────────┐                           │                           
              │ DTK build starts    │                           │                           
              │ (compilation prep)  │                           │                           
              └─────────────────────┘                           │                           
                       │                                          │                           
11:51:23      ┌─────────────────────┐                           │                           
              │ MOFED Compilation   │                           │                           NETWORK
              │ v25.07-0.9.7.0      │                           │                           COMPILING
              │                     │                           │                           
              │     [████░░░░░]     │                  ┌─────────────────────┐              
11:51:41      │                     │                  │ Waiting for MOFED   │              
              │                     │                  │ to be installed     │              
              │                     │                  └─────────────────────┘              
              │     [████████░]     │                           │                           
11:53:43      │ Compilation DONE ✓  │                           │                           
              └─────────────────────┘                           │                           
                       │                                          │                           
              ┌─────────────────────┐                  ┌─────────────────────┐              
              │ Waiting for DTK     │                  │ DTK: Compiling      │              GPU
              │ to finish           │                  │ NVIDIA drivers      │              COMPILING
              │                     │                  │ - Links MOFED src   │              
11:54-11:56   │                     │                  │ - Builds nvidia.ko  │              
              │                     │                  │ - Builds peermem.ko │              
              └─────────────────────┘                  └─────────────────────┘              
                       │                                          │                           
11:56:08      ┌─────────────────────┐                           │                           
              │ MOFED Installation  │                           │                           RACE
              │ - Install RPMs      │                  ┌─────────────────────┐              CONDITION
              │ - Unload old HCA    │                  │ nvidia-installer    │              WINDOW
              │ - Load new HCA      │                  │ running...          │              
              │     [███░░░]        │                  └─────────────────────┘              
11:57:55      │ HCA Loading DONE ✓  │                           │                           ↓
              └─────────────────────┘                  ┌─────────────────────┐              
                       │                                │ nvidia-driver-ctr   │              
              ┌─────────────────────┐                  │ Loading modules:    │              
              │ Restoring SR-IOV    │                  │  ✓ nvidia.ko       │              
              │ Restoring VF config │                  │  ✓ nvidia-uvm.ko   │              
11:58:39      │ Set .driver-ready✓  │                  │  ✓ nvidia-modeset  │              
              │ mlx5_core loaded    │                  │  ❌ nvidia-peermem  │              
              └─────────────────────┘                  │    ERROR: Invalid   │              FAILURE!
                       │                                │    argument         │              
                       │                                └─────────────────────┘              
              ┌─────────────────────┐                           │                           
11:58:40+     │ [✓] Running 2/2     │                  ┌─────────────────────┐              
              │ MOFED Ready         │                  │ CrashLoopBackOff    │              STUCK
              └─────────────────────┘                  │ 64 restarts         │              STATE
                                                       │ 7 pods blocked      │              
                                                       └─────────────────────┘              
                                                                                             
═══════════════════════════════════════════════════════════════════════════════════════════════════

KEY INSIGHT: The 2-minute gap (11:56:08 → 11:58:39) when MOFED is transitioning is when 
             the GPU Operator attempts to load nvidia_peermem, causing the failure.

CRITICAL FILES:
  /run/mellanox/drivers/.driver-ready          ← Created at 11:58:39 (too late!)
  /run/nvidia/validations/.driver-ctr-ready    ← Never created (driver crashed)
  /run/nvidia/validations/toolkit-ready        ← Never created (waiting on driver)

SOLUTION: GPU Operator must WAIT for Network Operator before loading nvidia_peermem
```

## Detailed Failure Sequence

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         WHAT SHOULD HAVE HAPPENED                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Network Operator                              GPU Operator                 │
│  ─────────────────                             ──────────────                │
│                                                                              │
│  1. Compile MOFED ✓                            1. Wait for MOFED            │
│  2. Install MOFED ✓                            2. Compile NVIDIA drivers    │
│  3. Load mlx5_core ✓                           3. Wait for .driver-ready ✓  │
│  4. Create .driver-ready ✓   ───────────────>  4. Load nvidia_peermem ✓     │
│                                                                              │
│                          (Sequential, Safe)                                  │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                         WHAT ACTUALLY HAPPENED                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Network Operator                              GPU Operator                 │
│  ─────────────────                             ──────────────                │
│                                                                              │
│  1. Compile MOFED ✓        (parallel)          1. Compile NVIDIA drivers ✓  │
│  2. Install MOFED...       <─────┐             2. Load nvidia.ko ✓          │
│  3. Load mlx5_core...            │             3. Load nvidia-peermem ❌     │
│  4. Create .driver-ready ✓       └──(MISSED)──    └─> ERROR: Invalid arg   │
│                                                    └─> Crash & Restart loop  │
│                                                                              │
│                   (Race Condition, nvidia-driver-ctr has no wait logic)     │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Container-Level View

```
GPU Operator Driver Pod: nvidia-driver-daemonset-418.94.202510230424-0-564xm
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Init Container: k8s-driver-manager                                        │
│  ┌─────────────────────────────────────────────────────────────┐          │
│  │ ✓ HAS wait logic for MOFED                                  │          │
│  │ ✓ Logs: "Waiting for MOFED to be installed" (11:51:41)     │          │
│  │ ✓ Completes: "Driver uninstallation completed" (11:51:41)  │          │
│  └─────────────────────────────────────────────────────────────┘          │
│                          │                                                  │
│                          ▼                                                  │
│  Main Containers (run in parallel):                                        │
│  ┌───────────────────────────────────┬────────────────────────────────┐   │
│  │ [1] nvidia-driver-ctr             │ [2] nvidia-peermem-ctr         │   │
│  ├───────────────────────────────────┼────────────────────────────────┤   │
│  │ ❌ NO wait logic for MOFED         │ ✓ HAS wait logic for MOFED     │   │
│  │ Immediately loads modules:        │ Waits for:                     │   │
│  │  ✓ nvidia.ko                      │  - .driver-ready               │   │
│  │  ✓ nvidia-uvm.ko                  │  - .driver-ctr-ready           │   │
│  │  ✓ nvidia-modeset.ko              │ Never runs because [1] crashes │   │
│  │  ❌ nvidia-peermem.ko (CRASHES)    │                                │   │
│  └───────────────────────────────────┴────────────────────────────────┘   │
│  ┌───────────────────────────────────┬────────────────────────────────┐   │
│  │ [3] nvidia-gdrcopy-ctr            │ [4] openshift-driver-toolkit   │   │
│  ├───────────────────────────────────┼────────────────────────────────┤   │
│  │ Waits for [1]                     │ ✓ Compilation successful       │   │
│  │ Never starts                      │ ✓ Creates precompiled package  │   │
│  └───────────────────────────────────┴────────────────────────────────┘   │
│                                                                             │
│  Pod Status: 1/4 Running, CrashLoopBackOff                                 │
└────────────────────────────────────────────────────────────────────────────┘

THE BUG: nvidia-driver-ctr (container #1) should check for .driver-ready before
         loading nvidia-peermem, just like nvidia-peermem-ctr (container #2) does.
```

## Files Being Checked (or not checked)

```
Host Filesystem at /run/
├─ mellanox/
│  └─ drivers/
│     ├─ .driver-ready              ← Created by Network Operator at 11:58:39
│     │                                ✓ Checked by: nvidia-peermem-ctr
│     │                                ❌ NOT checked by: nvidia-driver-ctr
│     └─ usr/src/ofa_kernel/         ← MOFED headers (available at 11:53:43)
│
└─ nvidia/
   └─ validations/
      ├─ .driver-ctr-ready           ← Should be created by nvidia-driver-ctr
      │                                 ❌ Never created (driver crashes)
      └─ toolkit-ready                ← Should be created by container-toolkit
                                        ❌ Never created (waiting on driver)
```

