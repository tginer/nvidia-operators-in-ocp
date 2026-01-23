# NVIDIA GPU Operator Pod Dependency Diagram
This analysis is performed with Cursor Claude-4.5-Sonnet

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  1. nvidia-driver-daemonset                                          │  │
│  │     Status: CrashLoopBackOff (64 restarts)                           │  │
│  │     ❌ ERROR: nvidia_peermem module load failure                      │  │
│  │     Missing: /run/nvidia/validations/.driver-ctr-ready               │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                  │                                          │
│                                  │ Creates: .driver-ctr-ready               │
│                                  ▼                                          │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  2. nvidia-container-toolkit-daemonset                               │  │
│  │     Status: Init:0/1 (Pending)                                       │  │
│  │     ⏳ WAITING: for .driver-ctr-ready file                            │  │
│  │     Missing: /run/nvidia/validations/toolkit-ready                   │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                  │                                          │
│                                  │ Creates: toolkit-ready                   │
│                                  ▼                                          │
│         ┌────────────────────────┴────────────────────────┐                │
│         │                        │                        │                │
│         │                        │                        │                │
│         ▼                        ▼                        ▼                │
│  ┌──────────────┐        ┌──────────────┐        ┌──────────────┐         │
│  │ 3. gpu-      │        │ 5. nvidia-   │        │ 6. nvidia-   │         │
│  │    feature-  │        │    device-   │        │    dcgm-     │         │
│  │    discovery │        │    plugin-   │        │    exporter  │         │
│  │              │        │    daemonset │        │              │         │
│  │ Init:0/1     │        │ Init:0/1     │        │ Init:0/2     │         │
│  │ ⏳ WAITING    │        │ ⏳ WAITING    │        │ ⏳ WAITING    │         │
│  └──────────────┘        └──────────────┘        └──────────────┘         │
│         │                        │                        │                │
│         │                        │                        │                │
│         ▼                        ▼                        ▼                │
│  ┌──────────────┐        ┌──────────────┐        ┌──────────────┐         │
│  │ 4. nvidia-   │        │ 7. nvidia-   │        │ 8. nvidia-   │         │
│  │    dcgm      │        │    mig-      │        │    operator- │         │
│  │              │        │    manager   │        │    validator │         │
│  │ Init:0/1     │        │ Init:0/1     │        │ Init:0/5     │         │
│  │ ⏳ WAITING    │        │ ⏳ WAITING    │        │ ⏳ WAITING    │         │
│  └──────────────┘        └──────────────┘        └──────────────┘         │
│                                                           │                │
│                                                           │ Validates all  │
│                                                           ▼                │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  9. nvidia-node-status-exporter                                      │  │
│  │     Status: Running (1/1) but with errors                            │  │
│  │     ⚠️  ERROR: failed to validate driver (continuous failures)        │  │
│  │     Impact: Cannot report accurate GPU status                        │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

Legend:
  ❌ FAILED     - Pod crashed/failing
  ⏳ WAITING    - Pod waiting on dependency
  ⚠️  ERROR     - Pod running but with errors
  
Dependency Chain:
  ═══════════╗
             ║ BLOCKED by nvidia_peermem module failure
             ║
             ╠══► nvidia-driver-daemonset (FAILS)
             ║      │
             ║      └──► nvidia-container-toolkit-daemonset (WAITS)
             ║             │
             ║             ├──► gpu-feature-discovery (WAITS)
             ║             ├──► nvidia-device-plugin-daemonset (WAITS)
             ║             ├──► nvidia-dcgm-exporter (WAITS)
             ║             ├──► nvidia-dcgm (WAITS)
             ║             ├──► nvidia-mig-manager (WAITS)
             ║             └──► nvidia-operator-validator (WAITS)
             ║
             ╚══► nvidia-node-status-exporter (RUNS but ERRORS)
```

## Summary

**Root Cause**: The `nvidia_peermem` kernel module fails to load with "Invalid argument" error.

**Cascade Impact**: 
- 1 pod in **CrashLoopBackOff** (nvidia-driver-daemonset)
- 7 pods **blocked** waiting for dependencies (toolkit, feature-discovery, device-plugin, dcgm, dcgm-exporter, mig-manager, validator)
- 1 pod **running with errors** (node-status-exporter)

**Fix Required**: Resolve the nvidia_peermem module loading issue to unblock the entire GPU operator stack.

