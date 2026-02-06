# Intel i915 GPU Driver - Codebase Architecture Outline

## 📋 Overview

The Intel i915 graphics driver is a comprehensive kernel driver for Intel discrete and integrated GPUs. This documentation provides a structured analysis of all major functional components.

**Supported GPUs:**
- Intel Arc A-Series Graphics (Alchemist - DG2)
- Intel Data Center GPU Flex Series
- Intel Data Center GPU Max Series (PVC)

---

## 🏗️ Core Functional Components

### 1. **Memory Management System**
   - **Location:** `gem/`, `drivers/gpu/drm/i915/i915_*memory*`
   - **Key Files:**
     - `gem/i915_gem_object.c` - Core GEM object management
     - `gem/i915_gem_lmem.c` - Local memory management
     - `gem/i915_gem_pages.c` - Page allocation and management
     - `gem/i915_gem_shrinker.c` - Memory pressure handling
     - `intel_memory_region.c` - Memory region abstractions
     - `i915_buddy.c` - Buddy allocator for VRAM
   - **Responsibilities:**
     - GEM object creation, management, and lifecycle
     - Local memory (LMEM) allocation and management
     - System memory vs VRAM management
     - Memory pressure and eviction policies

### 2. **Graphics Unified Controller (GuC)**
   - **Location:** `gt/uc/`
   - **Key Files:**
     - `intel_guc.c` / `intel_guc.h` - Main GuC driver
     - `intel_guc_ct.c` - Communication transport (CT) protocol
     - `intel_guc_submission.c` - Command submission via GuC
     - `intel_guc_ads.c` - Address descriptor structure
     - `intel_guc_log.c` - Firmware logging
     - `intel_guc_capture.c` - Error capture and diagnostics
   - **Responsibilities:**
     - Firmware loading and initialization
     - GuC-based command submission
     - Bi-directional communication with firmware
     - Workload submission and scheduling

### 3. **Context Management**
   - **Location:** `gem/`, `gt/`
   - **Key Files:**
     - `gem/i915_gem_context.c` - User-facing context API
     - `gt/intel_context.c` - Hardware context structures
     - `gt/intel_lrc.c` - Logical Ring Context (LRC)
     - `gt/intel_engine_cs.c` - Engine command stream setup
   - **Responsibilities:**
     - User context creation and management
     - Hardware context initialization
     - Per-context state and address spaces
     - SSEU (Slice/Sub-slice/EU) configuration

### 4. **Power Management**
   - **Location:** `intel_pm.c`, `intel_runtime_pm.c`, `gt/`
   - **Key Files:**
     - `intel_pm.c` - Global power management policies
     - `intel_runtime_pm.c` - Runtime PM framework
     - `gt/intel_rps.c` - Render Performance State (RPS)
     - `gt/intel_rc6.c` - RC6 power gating
     - `gt/intel_gt_pm.c` - GT-level power management
   - **Responsibilities:**
     - Runtime power state management
     - Frequency scaling (RPS/SLPC)
     - RC6 power gating and idle states
     - Display and GT power well management

### 5. **Request Scheduling & Execution**
   - **Location:** `gt/`, `gem/`
   - **Key Files:**
     - `i915_request.c` - GPU request/command batches
     - `i915_scheduler.c` - Request scheduler
     - `gt/intel_engine_cs.c` - Engine command stream
     - `gt/intel_execlists_submission.c` - Execlists submission
     - `gt/intel_guc_submission.c` - GuC-based submission
     - `i915_gem_execbuffer.c` - User-level batch execution
   - **Responsibilities:**
     - Request creation and management
     - Scheduling policies (priority, preemption)
     - Command submission mechanisms
     - Fence and completion tracking

### 6. **Virtual Memory Management (MMU)**
   - **Location:** `gt/`, `gem/`
   - **Key Files:**
     - `i915_vma.c` - Virtual Memory Address objects
     - `gt/intel_ggtt.c` - Global Graphics Translation Table
     - `gt/intel_ppgtt.c` - Per-Process Graphics Translation Table
     - `gt/gen8_ppgtt.c` - Gen8+ PPGTT implementation
     - `gt/intel_pagefault.c` - Page fault handling
   - **Responsibilities:**
     - Address space management (GGTT/PPGTT)
     - VMA binding and unbinding
     - Page table management
     - Page fault handling and recoveryVA mapping/unmapping

### 7. **Interrupt & Event Handling**
   - **Location:** `i915_irq.c`, `gt/`
   - **Key Files:**
     - `i915_irq.c` - Main interrupt handling
     - `gt/intel_gt_irq.c` - GT-level interrupts
     - `i915_active.c` - Active reference tracking
     - `intel_breadcrumbs.c` - Fence completion notification
   - **Responsibilities:**
     - CPU interrupt handling
     - GT interrupt routing and ACK
     - Fence completion detection
     - Timestamp synchronization

### 8. **Reset & Error Handling**
   - **Location:** `gt/`, `i915_gpu_error.c`
   - **Key Files:**
     - `gt/intel_reset.c` - GPU reset logic
     - `i915_gpu_error.c` - Error capture and diagnostics
     - `gt/intel_engine_heartbeat.c` - Hang detection
     - `gt/intel_reset.c` - Reset orchestration
   - **Responsibilities:**
     - GPU hang detection
     - Error state capture and analysis
     - Reset choreography and recovery
     - Per-engine and full GPU reset

### 9. **Display Subsystem**
   - **Location:** `display/`
   - **Key Files:**
     - `display/intel_display.c` - Main display driver
     - `display/intel_dpll_mgr.c` - DPLL management
     - `display/intel_cdclk.c` - Core Display Clock
     - `display/intel_psr.c` - Panel Self Refresh
     - `display/intel_dmc.c` - Display Microcontroller Firmware
   - **Responsibilities:**
     - Display output configuration
     - Connector and encoder management
     - Clock and power well control for display
     - Low-power display modes (PSR, DRRS)

### 10. **Protected Execution (PXP)**
   - **Location:** `pxp/`
   - **Key Files:**
     - `intel_pxp.c` - Main PXP driver
     - `intel_pxp_cmd.c` - PXP commands
     - `intel_pxp_tee.c` - TEE (Trusted Execution Environment) integration
     - `intel_pxp_pm.c` - PXP power management
   - **Responsibilities:**
     - Secure content protection
     - TEE communication
     - Key management
     - Protected memory handling

### 11. **Performance Monitoring & Profiling**
   - **Location:** `i915_perf.c`, `i915_pmu.c`
   - **Key Files:**
     - `i915_perf.c` - OA (Observation Architecture) sampling
     - `i915_pmu.c` - Performance Monitoring Unit
     - `i915_perf_stall_cntr.c` - Stall counter analysis
   - **Responsibilities:**
     - OA metrics collection
     - Performance counter monitoring
     - Performance data export to userspace

### 12. **Firmware & Microcontroller Management**
   - **Location:** `gt/uc/`
   - **Key Files:**
     - `intel_uc.c` - Unified microcontroller framework
     - `intel_uc_fw.c` - Firmware loading and validation
     - `intel_huc.c` - HUC (Hardware Utility Controller)
     - `intel_gsc_fw.c` - GSC (Graphics Security Controller)
   - **Responsibilities:**
     - Firmware parsing and loading
     - Microcontroller initialization
     - Firmware versioning and validation

### 13. **Debugging & Telemetry**
   - **Location:** `i915_debugger.c`, `i915_debugfs.c`
   - **Key Files:**
     - `i915_debugger.c` - Live GPU debugging
     - `i915_debugfs.c` - Debug filesystem interface
     - `i915_sysfs.c` - Sysfs attributes
     - `pvc_fatal_error_dump.c` - Fatal error diagnostics
   - **Responsibilities:**
     - In-kernel GPU debugging support
     - Runtime diagnostics and error capture
     - Debug information export
     - Performance analysis tools

### 14. **Memory Protection/SR-IOV**
   - **Location:** `i915_sriov.c`, `gt/iov/`
   - **Key Files:**
     - `i915_sriov.c` - SR-IOV virtualization support
     - `gt/iov/` - IOV-specific implementations
   - **Responsibilities:**
     - Virtual Function (VF) management
     - Resource partitioning
     - VF driver communication

### 15. **DMA & Buffer Management**
   - **Location:** `gem/`, `drivers/dma-buf/`
   - **Key Files:**
     - `gem/i915_gem_dmabuf.c` - DMA buffer export
     - `drivers/dma-buf/dma-buf.c` - Kernel DMA-BUF framework
   - **Responsibilities:**
     - Cross-device memory sharing via DMA-BUF
     - Buffer lifecycle management
     - Sync point handling

### 16. **Compute & Multi-GPU (Fabric)**
   - **Location:** `fabric/`
   - **Key Files:**
     - Fabric interconnect support for data center GPUs
   - **Responsibilities:**
     - Multi-GPU communication
     - Fabric topology management
     - High-speed inter-GPU links

---

## 📂 File Organization Structure

```
drivers/gpu/drm/i915/
├── gem/                          # Graphics Execution Memory
│   ├── i915_gem_*.c             # GEM object management
│   ├── i915_gem_context.c       # User context API
│   ├── i915_gem_execbuffer.c    # Batch execution
│   └── selftests/               # Unit tests
├── gt/                           # Graphics Technology
│   ├── intel_engine_cs.c        # Engine setup
│   ├── intel_context.c          # Hardware contexts
│   ├── intel_gtt.c              # Translation tables
│   ├── intel_rps.c              # Performance scaling
│   ├── intel_rc6.c              # Power gating
│   ├── intel_reset.c            # Reset logic
│   ├── uc/                      # Microcontroller (GuC, HUC, GSC)
│   ├── iov/                     # SR-IOV support
│   └── selftests/               # GT unit tests
├── display/                      # Display subsystem
│   ├── intel_display.c
│   ├── intel_dpll_mgr.c
│   └── selftests/
├── pxp/                         # Protected Execution
│   ├── intel_pxp.c
│   └── intel_pxp_*.c
├── fabric/                      # Multi-GPU fabric
├── i915_driver.c                # Main driver initialization
├── i915_irq.c                   # Interrupt handling
├── i915_request.c               # GPU requests
├── i915_scheduler.c             # Request scheduling
├── intel_pm.c                   # Power management
├── i915_debugger.c              # Live debugging
└── i915_perf.c                  # Performance monitoring
```

---

## 🔄 High-Level Data Flow

```
User Application
      ↓
Userspace Driver API (libdrm_intel)
      ↓
Ioctl Interface (i915_drm.h)
      ↓
[GEM Object Creation] → [Memory Allocation] → [Virtual Mapping]
      ↓
[Context Creation] → [Address Space Setup]
      ↓
[Batch Execution] → [Request Creation] → [Scheduling] → [Submission]
      ↓
[GuC/Execlists Submission]
      ↓
Hardware GPU Execution
      ↓
[Interrupt/Fence Completion]
      ↓
User Notification
```

---

## 📖 Documentation Index

1. [01-Memory-Management.md](./01-Memory-Management.md) - GEM, LMEM, buddy allocator
2. [02-GuC-Firmware.md](./02-GuC-Firmware.md) - GuC communication and submission
3. [03-Context-Management.md](./03-Context-Management.md) - Hardware context lifecycle
4. [04-Power-Management.md](./04-Power-Management.md) - RPS, RC6, frequency scaling
5. [05-Request-Scheduling.md](./05-Request-Scheduling.md) - Scheduling and execution
6. [06-Virtual-Memory.md](./06-Virtual-Memory.md) - MMU, GGTT, PPGTT
7. [07-Interrupt-Handling.md](./07-Interrupt-Handling.md) - IRQ and completion
8. [08-Reset-Error-Handling.md](./08-Reset-Error-Handling.md) - Recovery mechanisms
9. [09-Display-Subsystem.md](./09-Display-Subsystem.md) - Display and output
10. [10-Protected-Execution.md](./10-Protected-Execution.md) - PXP and security
11. [11-Performance-Monitoring.md](./11-Performance-Monitoring.md) - PMU and profiling
12. [12-Firmware-Management.md](./12-Firmware-Management.md) - UC, GuC, HUC, GSC
13. [13-Debugging-Telemetry.md](./13-Debugging-Telemetry.md) - Debugging and analysis
14. [14-SR-IOV-Virtualization.md](./14-SR-IOV-Virtualization.md) - Virtualization support

---

## 🎯 Key Architectural Patterns

### 1. **Split Hardware Model**
- Discrete components: GuC, HUC, GSC, Display MC
- Each with own firmware, state machines, and communication protocols

### 2. **Layered Submission Architecture**
- User API → GEM → Request → Scheduler → Submission (Execlists/GuC)
- Multiple submission backends for different GPU generations

### 3. **Modular Memory Management**
- GEM objects abstract hardware memory
- Virtual Memory (VMA) abstracts address space mappings
- TTMs (Translation Tables) handle actual HW mappings

### 4. **Power Management Hierarchy**
- Global PM → Runtime PM → GT PM → Engine PM
- Device-wide power states cascade to components

### 5. **Error Recovery Strategy**
- Watchdog/heartbeat monitors GPU liveness
- Per-engine or full-system reset paths
- Error capture for diagnostics

---

## 🛠️ Development Workflow

### Common Modification Areas:

| Task | Files |
|------|-------|
| Add new submission feature | `i915_request.c`, `intel_guc_submission.c` |
| Modify memory management | `gem/i915_gem_*.c`, `i915_buddy.c` |
| Change power policy | `intel_pm.c`, `gt/intel_rps.c` |
| Add debugging feature | `i915_debugger.c`, `i915_debugfs.c` |
| Fix GPU hang | `gt/intel_reset.c`, `i915_gpu_error.c` |
| Performance optimization | `i915_scheduler.c`, `intel_breadcrumbs.c` |

---

## 📚 Related Resources

- **Kernel Documentation:** `Documentation/gpu/i915.rst`
- **Firmware ABI:** `gt/uc/abi/` - GuC/HUC interface specifications
- **Selftests:** `*/selftests/` - Unit tests for validation
- **Display:** Follows DRM/KMS framework standards

---

## 🔗 Cross-Component Dependencies

```
GEM Objects
    ↓
Virtual Memory (VMA) + Address Spaces
    ↓
Contexts + Engines
    ↓
Requests + Scheduling
    ↓
Submission (GuC/Execlists)
    ↓
Hardware Execution
    ↓
Interrupts + Completion
    ↓
Error Recovery + Reset
```

**Next:** Start with [01-Memory-Management.md](./01-Memory-Management.md) for foundational concepts.
