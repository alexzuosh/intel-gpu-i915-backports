# Intel i915 GPU Driver - Complete Codebase Documentation

**Documentation Generated:** 2026-02-06

---

## 📑 Documentation Index

This comprehensive documentation covers all major functional components of the Intel i915 GPU driver codebase.

### Core Documentation Files

| # | Component | File | Status |
|---|-----------|------|--------|
| 0 | **Architecture Outline** | [00-OUTLINE.md](./00-OUTLINE.md) | ✅ Complete |
| 1 | **Memory Management** | [01-Memory-Management.md](./01-Memory-Management.md) | ✅ Complete |
| 2 | **GuC Firmware System** | [02-GuC-Firmware.md](./02-GuC-Firmware.md) | ✅ Complete |
| 3 | **Context Management** | [03-Context-Management.md](./03-Context-Management.md) | ✅ Complete |
| 4 | **Power Management** | [04-Power-Management.md](./04-Power-Management.md) | ✅ Complete |
| 5 | **Request Scheduling** | [05-Request-Scheduling.md](./05-Request-Scheduling.md) | ✅ Complete |
| 6 | **Virtual Memory (MMU)** | [06-Virtual-Memory.md](./06-Virtual-Memory.md) | ✅ Complete |
| 6b | **Memory Migration & Eviction** | [06b-Memory-Migration.md](./06b-Memory-Migration.md) | ✅ Complete |
| 7 | **TBB Task Scheduling** | [07-TBB-Task-Scheduling.md](./07-TBB-Task-Scheduling.md) | ✅ Complete |
| 7b | **TBB Implementation Guide** | [07b-TBB-Implementation-Guide.md](./07b-TBB-Implementation-Guide.md) | ✅ Complete |
| 8 | **Interrupt Handling** | [08-Interrupt-Handling.md](./08-Interrupt-Handling.md) | 📝 [Planned] |
| 9 | **Reset & Error Handling** | [09-Reset-Error-Handling.md](./09-Reset-Error-Handling.md) | 📝 [Planned] |
| 10 | **Display Subsystem** | [10-Display-Subsystem.md](./10-Display-Subsystem.md) | 📝 [Planned] |
| 11 | **Protected Execution (PXP)** | [11-Protected-Execution.md](./11-Protected-Execution.md) | 📝 [Planned] |
| 12 | **Performance Monitoring** | [12-Performance-Monitoring.md](./12-Performance-Monitoring.md) | 📝 [Planned] |
| 13 | **Firmware Management** | [13-Firmware-Management.md](./13-Firmware-Management.md) | 📝 [Planned] |
| 14 | **Debugging & Telemetry** | [14-Debugging-Telemetry.md](./14-Debugging-Telemetry.md) | 📝 [Planned] |
| 15 | **SR-IOV Virtualization** | [15-SR-IOV-Virtualization.md](./15-SR-IOV-Virtualization.md) | 📝 [Planned] |

---

## 🎯 Quick Navigation

### By Development Task

**New Driver Developer?**
→ Start with [00-OUTLINE.md](./00-OUTLINE.md) for architecture overview
→ Then read [01-Memory-Management.md](./01-Memory-Management.md) for foundational concepts
→ Follow with [03-Context-Management.md](./03-Context-Management.md) for execution model
→ Study [07-TBB-Task-Scheduling.md](./07-TBB-Task-Scheduling.md) for CPU task framework

**Working on GPU Memory Issues?**
→ [01-Memory-Management.md](./01-Memory-Management.md) - GEM, LMEM, buddy allocator
→ [06-Virtual-Memory.md](./06-Virtual-Memory.md) - Address space, GGTT/PPGTT
→ [06b-Memory-Migration.md](./06b-Memory-Migration.md) - Migration, eviction, coherency

**Working on CPU Task Scheduling?**
→ [07-TBB-Task-Scheduling.md](./07-TBB-Task-Scheduling.md) - Design & architecture
→ [07b-TBB-Implementation-Guide.md](./07b-TBB-Implementation-Guide.md) - Usage patterns & examples

**Debugging GPU Hangs?**
→ [08-Reset-Error-Handling.md](./08-Reset-Error-Handling.md) - Recovery mechanisms
→ [08-Interrupt-Handling.md](./08-Interrupt-Handling.md) - Error detection

**Optimizing GPU Performance?**
→ [04-Power-Management.md](./04-Power-Management.md) - Frequency scaling
→ [05-Request-Scheduling.md](./05-Request-Scheduling.md) - Workload management
→ [07-TBB-Task-Scheduling.md](./07-TBB-Task-Scheduling.md) - CPU task efficiency
→ [12-Performance-Monitoring.md](./12-Performance-Monitoring.md) - Profiling tools

**Working on Display?**
→ [10-Display-Subsystem.md](./10-Display-Subsystem.md) - Display pipeline

**Implementing New Feature?**
→ [02-GuC-Firmware.md](./02-GuC-Firmware.md) - For firmware-driven features
→ [05-Request-Scheduling.md](./05-Request-Scheduling.md) - For execution-related
→ [07b-TBB-Implementation-Guide.md](./07b-TBB-Implementation-Guide.md) - For CPU task scheduling

---

## 📚 Component Relationships

```
┌──────────────────────────────────────────────────────────┐
│                   User Application                        │
│              (Vulkan, OpenGL, Compute)                    │
└─────────────────────┬──────────────────────────────────┘
                      ↓
            ┌─────────────────────┐
            │   DRM/ioctls API    │
            └──────────┬──────────┘
                       ↓
        ┌──────────────────────────────┐
        │  GEM Objects & Memory (01)   │ ← Foundational
        └──────────────┬───────────────┘
                       ↓
        ┌──────────────────────────────┐
        │  Virtual Memory - MMU (06)   │ ← Maps memory
        └──────────────┬───────────────┘
                       ↓
        ┌──────────────────────────────┐
        │  Context Management (03)     │ ← Execution environment
        └──────────────┬───────────────┘
                       ↓
        ┌──────────────────────────────┐
        │  Request Scheduling (05)     │ ← Workload ordering
        └──────────────┬───────────────┘
                       ↓
        ┌──────────────────────────────────────┐
        │      Submission Backends             │
        │  ┌────────────────────────────────┐  │
        │  │ GuC Firmware System (02)       │  │
        │  └────────────────────────────────┘  │
        │              OR                      │
        │  ┌────────────────────────────────┐  │
        │  │ Execlists (CPU-driven)         │  │
        │  └────────────────────────────────┘  │
        └──────────────┬───────────────────────┘
                       ↓
        ┌──────────────────────────────┐
        │   GPU Hardware Execution     │
                       ↓
        ┌──────────────────────────────┐
        │  Interrupt Handling (07)     │ ← Completion tracking
        └──────────────┬───────────────┘
                       ↓
        ┌──────────────────────────────────────┐
        │   Supporting Systems:                 │
        │  - Power Management (04)              │
        │  - Reset & Errors (08)                │
        │  - Performance Monitoring (11)        │
        │  - Debugging (13)                     │
        │  - Display (09)                       │
        │  - Protected Execution (10)           │
        │  - Firmware Mgmt (12)                 │
        │  - Virtualization (14)                │
        └──────────────────────────────────────┘
```

---

## 🔍 Key Concepts Summary

### Memory Management (01)
- **GEM Objects:** User-facing GPU memory abstraction
- **Memory Regions:** Different memory types (system, LMEM, stolen)
- **Buddy Allocator:** Efficient VRAM allocation with fragmentation management
- **Eviction:** Memory pressure handling and object shrinking

### GuC Firmware (02)
- **Unified Controller:** Embedded microcontroller for GPU management
- **Work Queue:** Firmware-driven command submission
- **Communication Transport:** Bi-directional CPU-GPU communication
- **SLPC:** Self-managed frequency scaling

### Context Management (03)
- **User Contexts:** Application isolation and scheduling
- **Hardware Contexts:** GPU-side state representation (LRC)
- **Address Spaces:** Per-context PPGTT for memory isolation
- **Execution State:** Register contexts and context switching

### Power Management (04)
- **RPS:** Dynamic frequency/voltage scaling based on load
- **RC6:** Power gating to reduce idle power consumption
- **SLPC:** GuC-based autonomous power management
- **Runtime PM:** System-level suspend/resume handling

### Request Scheduling (05)
- **GPU Requests:** Individual batch submission units
- **Priority Queue:** Workload prioritization and ordering
- **Scheduler:** Manages request execution order
- **Submission:** Passes requests to GuC or Execlists

### Virtual Memory - MMU (06)
- **GGTT:** Global Graphics Translation Table (shared address space)
- **PPGTT:** Per-Process GTT (per-context address spaces)
- **VMA:** Virtual Memory Address object for binding
- **Page Faulting:** On-demand paging mechanisms

### Interrupt Handling (07)
- **IRQ Processing:** Kernel interrupt handler
- **Fence Completion:** Notification on request completion
- **Breadcrumbs:** Lightweight completion tracking
- **Timestamp Sync:** GPU/CPU time synchronization

### Reset & Error (08)
- **Hang Detection:** Watchdog timer monitoring
- **Error Capture:** Diagnostic data collection
- **Reset Choreography:** Safe GPU reset procedure
- **Recovery:** Post-reset state restoration

---

## 🗂️ Codebase File Organization

```
drivers/gpu/drm/i915/
├── gem/                              # Memory & Execution (01, 05)
│   ├── i915_gem_*.c
│   ├── i915_gem_context.c
│   ├── i915_gem_execbuffer.c
│   └── selftests/
│
├── gt/                               # Graphics Technology (02, 03, 04, 06, 07, 08)
│   ├── intel_context.c               # Context (03)
│   ├── intel_lrc.c                   # Context state (03)
│   ├── intel_rps.c                   # Power (04)
│   ├── intel_rc6.c                   # Power gating (04)
│   ├── intel_ggtt.c                  # Memory (06)
│   ├── intel_ppgtt.c                 # Memory (06)
│   ├── intel_reset.c                 # Error handling (08)
│   ├── intel_engine_heartbeat.c      # Error detection (08)
│   ├── intel_gt_irq.c                # Interrupts (07)
│   ├── uc/                           # Microcontroller (02)
│   │   ├── intel_guc*.c
│   │   ├── intel_huc.c
│   │   └── abi/
│   └── selftests/
│
├── display/                          # Display Subsystem (09)
│   ├── intel_display.c
│   ├── intel_dpll_mgr.c
│   └── selftests/
│
├── pxp/                              # Protected Execution (10)
│   └── intel_pxp*.c
│
├── fabric/                           # Multi-GPU (14)
│   └── intel_fabric*.c
│
├── i915_driver.c                     # Main driver
├── i915_request.c                    # Requests (05)
├── i915_scheduler.c                  # Scheduling (05)
├── i915_irq.c                        # Interrupts (07)
├── i915_gpu_error.c                  # Error handling (08)
├── intel_pm.c                        # Power (04)
├── intel_runtime_pm.c                # Power (04)
├── i915_perf.c                       # Performance (11)
├── i915_pmu.c                        # Monitoring (11)
├── i915_debugger.c                   # Debugging (13)
├── i915_sysfs.c                      # Telemetry (13)
└── i915_sriov.c                      # Virtualization (14)
```

---

## 🔄 Common Code Modification Workflows

### Adding Memory Allocation Feature
1. Study [01-Memory-Management.md](./01-Memory-Management.md)
2. Understand GEM object lifecycle
3. Review buddy allocator in `i915_buddy.c`
4. Implement in `gem/i915_gem_*.c`
5. Add selftests to `gem/selftests/`

### Optimizing Request Scheduling
1. Read [05-Request-Scheduling.md](./05-Request-Scheduling.md)
2. Review priority queue in `i915_scheduler.c`
3. Profile submission path
4. Modify scheduling policy in `i915_scheduler.c`
5. Test with relevant workloads

### Implementing Frequency Scaling Policy
1. Study [04-Power-Management.md](./04-Power-Management.md)
2. Understand RPS state machine in `gt/intel_rps.c`
3. Modify frequency update algorithm
4. Test thermal/performance impact
5. Add sysfs interface for debugging

### Debugging GPU Hang
1. Check [08-Reset-Error-Handling.md](./08-Reset-Error-Handling.md)
2. Review heartbeat logic in `intel_engine_heartbeat.c`
3. Check reset choreography in `gt/intel_reset.c`
4. Enable logging in `i915_gpu_error.c`
5. Analyze error capture data

---

## 📊 Supported GPU Platforms

**Discrete GPUs:**
- Intel Arc A-Series (Alchemist / DG2)
- Intel Data Center GPU Flex
- Intel Data Center GPU Max (PVC)

**Integrated GPUs:**
- Intel Xe-HPG
- Older generations (Skylake, Kaby Lake, etc.)

---

## 🔗 External References

### Kernel Documentation
- `Documentation/gpu/i915.rst` - Upstream kernel documentation
- `Documentation/gpu/i915/` - Detailed component documentation

### Firmware Interface
- `gt/uc/abi/` - GuC/HUC ABI definitions
- `gt/uc/abi/guc_communication_*` - Protocol specifications

### Standards & Specifications
- Intel GPU Architecture Specification
- PCI Express Specification (for power management)
- ACPI Specification (for platform interaction)

---

## 💡 Tips for Code Navigation

### Understanding Request Flow
1. Start at `i915_gem_execbuffer_ioctl()` in `gem/i915_gem_execbuffer.c`
2. Follow to `i915_request_create()` in `i915_request.c`
3. Trace scheduler logic in `i915_scheduler.c`
4. See submission in `intel_guc_submission.c` or `intel_execlists_submission.c`

### Memory Allocation Path
1. User calls `DRM_IOCTL_I915_GEM_CREATE`
2. Reaches `i915_gem_create_ioctl()` in `gem/i915_gem_create.c`
3. Calls `i915_gem_object_create_from_data()`
4. Allocates via `i915_buddy_alloc()` for LMEM or shmem

### Power State Transition
1. Check `intel_pm_get_active()` reference counting
2. Trace through `intel_runtime_pm_get/put()`
3. See RPS updates in `gen11_rps_irq_handler()`
4. Monitor RC6 entry in GT idle path

### Error Recovery
1. Watchdog timer in `intel_engine_heartbeat.c` detects hang
2. Calls `intel_gt_handle_error()` in `i915_gpu_error.c`
3. Triggers reset in `gt/intel_reset.c`
4. Restores state post-reset

---

## 🧪 Testing & Validation

### Unit Tests
Located in `*/selftests/` directories:
- Memory tests: `gem/selftests/`
- Context tests: `gt/selftests/`
- Scheduler tests: `selftest_execlists.c`
- Power tests: `selftest_rc6.c`, `selftest_rps.c`

### Running Tests
```bash
# Enable selftest module
modprobe i915 enable_selftests=1

# Check test results
dmesg | grep -i selftest
```

---

## 📝 Documentation Maintenance

**Last Updated:** 2026-02-06

**Covered Components:**
- ✅ Memory Management (GEM, LMEM, buddy)
- ✅ GuC Firmware System
- ✅ Context Management
- ✅ Power Management
- ✅ Request Scheduling
- 📝 Virtual Memory (in progress)
- 📝 Interrupt Handling (in progress)
- 📝 Reset & Error Handling (in progress)
- 📝 Display Subsystem (planned)
- 📝 Protected Execution (planned)
- 📝 Performance Monitoring (planned)
- 📝 Firmware Management (planned)
- 📝 Debugging & Telemetry (planned)
- 📝 SR-IOV Virtualization (planned)

---

## 🤝 Contributing to Documentation

When adding new documentation:
1. Follow markdown formatting with clear headers
2. Include architecture diagrams using ASCII art
3. Provide code flow examples
4. Link to related components
5. Include references to source files
6. Add to this index

---

## ❓ FAQ

**Q: Where should I start learning this codebase?**
A: Begin with [00-OUTLINE.md](./00-OUTLINE.md), then [01-Memory-Management.md](./01-Memory-Management.md)

**Q: How do GEM and GuC relate?**
A: GEM manages memory; GuC manages execution. See [01](./01-Memory-Management.md) and [02](./02-GuC-Firmware.md)

**Q: What's the difference between GGTT and PPGTT?**
A: GGTT is global (shared); PPGTT is per-process (isolated). See [06-Virtual-Memory.md](./06-Virtual-Memory.md)

**Q: How does preemption work?**
A: Hardware-supported context switching triggered by scheduler. See [05-Request-Scheduling.md](./05-Request-Scheduling.md)

**Q: Where is power management code?**
A: `intel_pm.c`, `intel_runtime_pm.c`, `gt/intel_rps.c`, `gt/intel_rc6.c`. See [04-Power-Management.md](./04-Power-Management.md)

---

## 📞 Support & Questions

For questions about specific components, refer to:
- Kernel mailing list: `dri-devel@lists.freedesktop.org`
- Source code: `drivers/gpu/drm/i915/`
- Issue tracker: Check Intel GPU driver repositories

---

**End of Documentation Index**
