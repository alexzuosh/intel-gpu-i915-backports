# Intel i915 GPU Debugger Support: Design & Architecture

**Date:** February 6, 2026  
**Status:** Complete  
**Scope:** Comprehensive debug infrastructure, error capture, hang detection, and userspace debugger protocol

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Overall Debug Architecture](#overall-debug-architecture)
3. [Core Subsystems](#core-subsystems)
4. [Userspace Debugger Protocol](#userspace-debugger-protocol)
5. [Error Capture & Coredumps](#error-capture--coredumps)
6. [GPU Hang Detection & Recovery](#gpu-hang-detection--recovery)
7. [Firmware Debugging (GuC/HuC)](#firmware-debugging-guchuc)
8. [Debug Interfaces (Debugfs/Sysfs)](#debug-interfaces-debugfssysfs)
9. [Key Data Structures](#key-data-structures)
10. [Integration & Performance](#integration--performance)

---

## Executive Summary

The i915 GPU driver implements a **multi-layered debugging infrastructure** designed to support three distinct use cases:

1. **Kernel-side Error Capture** - Automatic snapshot of GPU state when hangs/faults occur
2. **Userspace Debugging** - Event-driven protocol for GPU execution control and monitoring
3. **Performance Analysis** - Sampling and tracing infrastructure for profiling

**Key Design Philosophy:**
- **Asynchronous monitoring**: GPU execution is not synchronous with CPU; debugging is post-mortem or event-driven
- **Low overhead by default**: Debugging disabled/inactive until needed
- **Comprehensive state capture**: Error snapshots include registers, memory, context, and firmware state
- **Pluggable interfaces**: Debugfs, sysfs, ioctls, and event streams for different consumer types

**Diagram: Debug Architecture Overview**

```
┌─────────────────────────────────────────────────────────────┐
│                    Userspace                               │
├─────────────────────────────────────────────────────────────┤
│  Debugger      Event FIFO      Metrics                      │
│  Daemon       (uuid, context,  Stream                       │
│  (gdb-like)    eu_attention)   (OA samples)                 │
└────┬──────────────┬──────────────┬─────────────────────────┘
     │              │              │
┌────▼──────────────▼──────────────▼─────────────────────────┐
│              i915 Driver (Kernel)                           │
├───────────────────────────────────────────────────────────┤
│                                                            │
│  Error State Capture ─┬─→ Debugfs (/i915_error_state)    │
│  (i915_gpu_error.c)   │    [Coredump analysis]            │
│                        │                                  │
│  GPU Hang Detection ──┤─→ Heartbeat (2.5s interval)      │
│  (heartbeat.c)        │    [Periodic health check]        │
│                        │                                  │
│  Userspace Debugger ──┤─→ Event Stream + ACK              │
│  (i915_debugger.c)    │    [Runtime control]              │
│                        │                                  │
│  Firmware Debug ──────┼─→ GuC Logs + Capture             │
│  (intel_guc_log.c)    │    [Firmware-side events]         │
│                        │                                  │
│  Perf/Tracing ────────┴─→ Tracepoints + OA Sampling      │
│  (i915_perf.c)           [Performance monitoring]         │
│                                                            │
└────────────────────────────────────────────────────────────┘
           │
           │  Triggers
           ▼
  ┌─────────────────────┐
  │  Reset Pipeline     │
  │  - Preemption       │
  │  - Engine reset     │
  │  - Full reset       │
  └─────────────────────┘
```

---

## Overall Debug Architecture

### Design Layers

The i915 debugger stack consists of four complementary layers:

#### **Layer 1: Observational (Non-Intrusive)**
- **Debugfs files** - Read-only snapshots of current state
- **Sysfs attributes** - Error counters, reset counts
- **Tracepoints** - Linux ftrace integration
- **GuC logging** - Firmware execution logs
- **Memory dumps** - Compressed VMA captures in error state
- **Effect**: Zero runtime cost when disabled; no GPU stalls

#### **Layer 2: Monitoring (Periodic Checks)**
- **Heartbeat mechanism** - Injects periodic requests to detect hangs
- **Performance OA sampling** - Configurable metrics collection
- **Watchdog timers** - Optional strict enforcement
- **Effect**: ~0.1% overhead per heartbeat interval; tunable

#### **Layer 3: Event-Driven (Reactive)**
- **Error capture** - Triggered on hang/fault/reset
- **Page fault handling** - GPU page fault notifications
- **EU attention tracking** - Compute shader stall events
- **Effect**: High overhead only when errors occur

#### **Layer 4: Control (Interactive)**
- **Userspace debugger protocol** - Full GPU execution control
- **Context gating** - Prevent GuC optimizations during debug
- **ACK mechanism** - Synchronize with GPU execution
- **Effect**: Only active when debugger attached

---

## Core Subsystems

### 1. Error Capture System (`i915_gpu_error.c`)

**Purpose**: Capture comprehensive GPU state when hangs or faults occur.

**Key Components**:
- **`struct i915_gpu_coredump`** - Top-level error snapshot container
  - Device info, platform configuration, timing info
  - Per-GT state (registers, fault data)
  - Per-engine state (active context, requests, VMAs)
  - Compressed memory pages from batch buffers and context objects

- **`struct intel_gt_coredump`** - GT-wide state
  - FORCEWAKE, fault data (EIR, PGTBL_ER, etc.)
  - EU attentions (before, after, resolved bitmasks)
  - Per-engine list for detailed inspection

- **`struct intel_engine_coredump`** - Engine-specific state
  - Active context info (process name, PID, UID)
  - Request sequencing (head, tail, seqno)
  - Engine registers (CCID, IP, ring state)
  - VMA captures (batch, context objects)
  - GuC capture node (if GuC-managed)

**Capture Workflow**:

```
GPU Hang Detected (e.g., via heartbeat timeout)
  │
  ├─→ reset_engine() / intel_gt_reset()
  │   │
  │   └─→ i915_capture_error_state()
  │       │
  │       ├─ Allocate coredump container
  │       ├─ For each GT:
  │       │   ├─ Capture GT registers
  │       │   └─ For each engine:
  │       │       ├─ Capture engine registers
  │       │       ├─ Capture active context (PID, name)
  │       │       ├─ intel_engine_coredump_add_request()
  │       │       │   └─ Get active request seqno
  │       │       ├─ intel_engine_coredump_add_vma()
  │       │       │   ├─ Batch buffer VMA
  │       │       │   ├─ Context object VMA
  │       │       │   └─ Page compression pool
  │       │       └─ GuC capture if available
  │       │
  │       ├─ i915_vma_capture_prepare() - Compression
  │       ├─ Copy VMA pages with optional zlib compression
  │       ├─ i915_vma_capture_finish() - Cleanup
  │       │
  │       └─ i915_error_state_store()
  │           └─ Save to debugfs (first error retained)
  │
  └─→ Perform reset (preemption → engine reset → full reset)
```

**Error State Output** (via `/sys/kernel/debug/dri/*/i915_error_state`):

```
Timestamp: 2024-02-06 10:23:45 (uptime: 123.456s)
Kernel: 6.8.0-rc1 i915 #1
Device: Intel Arc A770 (PCI 0d04:1673)
IOMMU: enabled, DMA mask: 64-bit

Engine: Render (rcs0)
  State: hung, reset pending
  Active Context:
    Process: glxgears (PID 1234, UID 1000)
    HW ID: 0x0001
    Timeline: 1 (seqno: 0x0000001f)
  Active Request: seqno 31, priority normal
  Registers:
    CCID: 0x00010001 (context ID, active)
    IP:   0x7fff0020 (instruction pointer)
    EIP:  0x00000000 (error instruction)
    RING_HEAD: 0x00001230
    RING_TAIL: 0x00001280
    Ring state: 16 pending
  Active VMA: 
    Batch buffer at 0x1000000000 (8192 bytes)
    Context at 0x2000000000 (1 MB, compressed)
    Fault Info: None

Faults:
  TLB Fault at 0x1234567890: invalid translation
  EIR: 0x00000001 (illegal instruction)
  PGTBL_ER: 0x00000002 (page table error)

EU Attentions: [before] 0xFFFF00FF [after] 0x0000FFFF [resolved] 0xFFFFFFFF
  (16 EU threads in render state, attention acquired during reset)
```

**Memory Compression**:
- Uses zlib with ascii85 encoding for debugfs readability
- Scatter-gather list for efficient buffer management
- Optional compile-time configuration (CPTCFG_DRM_I915_COMPRESS_ERROR)
- Tradeoff: ~80% reduction in memory, +5-10ms capture time

---

### 2. Hang Detection System (`intel_engine_heartbeat.c`)

**Purpose**: Proactively detect stalled GPU engines before user-visible hangs.

**Design Philosophy**:
- Regular "heartbeat" requests verify GPU is responsive
- Timeout detection avoids false positives
- Preemption attempt (faster recovery) before full reset
- Configurable sensitivity via `heartbeat_interval_ms` debugfs file

**Heartbeat Mechanism**:

```
┌─────────────────────────────────────────────────┐
│  Engine Heartbeat: 2500ms interval (default)    │
└──────┬──────────────────────────────────────────┘
       │
       ├─ [T=0s] next_heartbeat() schedules delayed work
       │          └─ heartbeat_work() enqueues pulse request
       │
       ├─ [T=0.1s] Pulse request submitted to GPU
       │            └─ GPU executes in background
       │
       ├─ [T=0.5s] Pulse completes, seqno advances
       │            └─ next_heartbeat() reschedules
       │
       ├─ [T=2.5s] No pulse completion (TIMEOUT)
       │            ├─ show_heartbeat() logs diagnostics
       │            │   ├─ Last completed seqno
       │            │   ├─ Last submitted seqno
       │            │   ├─ Ring head/tail
       │            │   ├─ Active request
       │            │   └─ GuC firmware state
       │            │
       │            └─ Attempt preemption (fast path)
       │                ├─ intel_engine_reset_prepare()
       │                │   └─ Timeout wait for preemption
       │                │
       │                └─ If preemption fails → Full reset
       │                    ├─ reset_prepare() quiesces engines
       │                    ├─ __intel_gt_reset() HW reset
       │                    ├─ reset_finish() restores state
       │                    └─ intel_uc_reset_finish() (GuC/HuC)
       │
       └─ [T=5.0s] Engine recovered, heartbeat resumes
                    └─ next_heartbeat() rescheduled

Reset Counters:
  ├─ gt->reset.count++ (full GT reset)
  ├─ engine->reset.count++ (engine-specific reset)
  ├─ i915->gpu_error.reset_count++ (global)
  └─ Exposed via sysfs: /sys/class/drm/card*/gt/gt*/reset_count
```

**Configurable Parameters**:
- `heartbeat_interval_ms` (1000-10000, default 2500) - Detection sensitivity
- `preempt_timeout_ms` (100-10000, default 650) - Preemption patience
- Per-engine via debugfs: `/sys/kernel/debug/dri/*/gt/*/heartbeat_interval_ms`

**Watchdog Timer (Optional)**:
```c
// Separate from heartbeat, stricter enforcement
engine->props.watchdog_interval_ms = 60000;  // 60 second timeout
// If exceeded: safety-critical reset without preemption attempt
```

---

### 3. Userspace Debugger Protocol (`i915_debugger.c`)

**Purpose**: Enable runtime GPU execution control for interactive debugging.

**Architecture**:
- **Event-driven**: Kernel → userspace event FIFO for asynchronous notification
- **Bidirectional**: Userspace acknowledges events or queries state
- **Resource tracking**: UUID-based handle system for managed resources
- **Context gating**: Prevents GuC optimizations when debugging

**Session Lifecycle**:

```
1. Userspace opens debugger session
   └─ ioctl(I915_DEBUGGER_OPEN, client_fd, event_fifo_size, ...)
      ├─ Allocate i915_debugger struct
      ├─ Create kfifo event ring buffer (16-4096 entries)
      └─ Return event fd for polling

2. Initial discovery
   └─ i915_debugger_wait_on_discovery()
      └─ Kernel enqueues current clients/contexts/VMs:
         ├─ CLIENT_CREATE for each DRM client
         ├─ CONTEXT_CREATE for each context
         ├─ VM_CREATE for each address space
         ├─ UUID_CREATE for debuggable resources
         └─ Userspace reads all events before proceeding

3. Runtime events
   └─ As GPU executes:
      ├─ CONTEXT_CREATE/DESTROY (new context submitted)
      ├─ VM_BIND (memory mapping changes)
      ├─ EU_ATTENTION (compute shader stalled)
      ├─ PAGEFAULT (GPU memory fault)
      └─ Context-specific gating prevents GuC fast submission

4. Userspace controls
   ├─ i915_debugger_context_set_param()
   │   └─ Modify engine assignment, VM binding
   ├─ i915_debugger_read() → receives events
   │   └─ struct i915_debugger_event (union of event types)
   └─ Send ACK for synchronization
       └─ Event acknowledging gates further execution

5. Cleanup
   └─ close(event_fd)
      ├─ Restore contexts to normal GuC optimization
      ├─ Drain remaining events
      └─ Release resource handles
```

**Event Types**:

```c
enum i915_debugger_event_type {
    I915_DEBUGGER_EVENT_CLIENT,          // Client creation/destruction
    I915_DEBUGGER_EVENT_CONTEXT,         // Context creation/destruction/param
    I915_DEBUGGER_EVENT_UUID,            // Debuggable resource lifetime
    I915_DEBUGGER_EVENT_VM,              // Address space creation/destruction
    I915_DEBUGGER_EVENT_VM_BIND,         // Memory mapping changes
    I915_DEBUGGER_EVENT_PAGEFAULT,       // GPU page fault with address & fault_type
    I915_DEBUGGER_EVENT_EU_ATTENTION,    // EU thread stall with bitmask
    I915_DEBUGGER_EVENT_CONTEXT_PARAM,   // Context parameter modification
};

struct i915_debugger_event {
    __u32 type;                 // Event type above
    __u32 flags;                // Event flags
    __u64 client_handle;        // DRM client ID
    __u64 timestamp;            // Kernel time snapshot
    union {
        struct i915_debugger_event_pagefault {
            __u64 page_addr;
            __u32 fault_type;   // Read/write/execute fault
            __u64 eu_attention; // EU threads involved
        } pagefault;
        struct i915_debugger_event_eu_attention {
            __u64 eu_attention; // Bitmask of stalled EUs
            __u32 engine_class; // Which engine
        } eu_attention;
        // ... other event unions
    };
};
```

**UUID Resource System**:

```c
// Debugger registers resources for tracking
struct i915_uuid_resource {
    uuid_t uuid;                    // Unique identifier
    struct drm_i915_gem_object *obj; // Associated object
    struct list_head link;          // Linked in driver->uuid_list
};

// Kernel notifies debugger of resource lifecycle:
// UUID_CREATE → debugger learns of allocation
// UUID_DESTROY → debugger removes tracking

// Benefits:
// - Userspace debugger learns all GPU allocations
// - Can correlate fault addresses to source objects
// - Enables symbolic debugging (map addresses → object names)
```

---

### 4. Firmware Debugging (GuC/HuC)

**GuC Logging System** (`intel_guc_log.c`):

**Three-buffer architecture**:
```
┌─────────────────────────────────────────────┐
│            GuC Logging Buffers              │
├─────────────────────────────────────────────┤
│                                             │
│  Crash Buffer (64KB default)                │
│  ├─ OOPS and fatal errors                   │
│  ├─ High-priority firmware faults           │
│  └─ Always captured on GuC error            │
│                                             │
│  Debug Buffer (256KB default)               │
│  ├─ Detailed execution logs                 │
│  ├─ State transitions                       │
│  ├─ Configurable verbosity (level 0-3)     │
│  └─ Disabled by default                     │
│                                             │
│  Capture Buffer (varies)                    │
│  ├─ Firmware-initiated state capture        │
│  ├─ GuC own registers, memory state         │
│  ├─ EU attentions and context data          │
│  └─ Triggered on hang/fault                 │
│                                             │
└─────────────────────────────────────────────┘
```

**Configuration**:
```bash
# Module parameters
insmod i915.ko \
  guc_log_level=2              # 0=none, 1=error, 2=debug, 3=verbose
  guc_log_size_crash=65536     # Crash buffer size
  guc_log_size_debug=262144    # Debug buffer size
  guc_log_size_capture=1048576 # Capture buffer size

# Runtime control (debugfs)
echo 2 > /sys/kernel/debug/dri/*/guc_log_level
cat /sys/kernel/debug/dri/*/guc_log_dump  # View accumulated logs
```

**Relay Channel for Streaming**:
```c
// Real-time log streaming to userspace
intel_guc_log_relay_open()
  ├─ Create relay channel (/sys/kernel/debug/dri/*/guc_log)
  ├─ IRQ handler flushes ring to relay on overflow
  └─ Userspace can read tail pointer

// Overflow detection
intel_guc_check_log_buf_overflow()
  └─ When GuC log wraps: signal userspace reader
```

**GuC Capture System** (`intel_guc_capture.c`):

```
GPU Hang with GuC Submission
  │
  ├─ GuC detects hang or fault
  │  ├─ Context-specific fault
  │  ├─ Memory protection fault
  │  └─ EU thread timeout
  │
  ├─ GuC initiates firmware capture
  │  ├─ Save GuC own registers
  │  ├─ Save EU attentions
  │  ├─ Save process descriptor
  │  └─ Write to capture buffer
  │
  └─ Kernel error capture path
     ├─ intel_guc_capture_process()
     │  └─ Read capture buffer from GuC
     │
     ├─ intel_guc_capture_print_engine_node()
     │  └─ Format output for debugfs/error_state
     │
     └─ Merged with engine coredump
        └─ Available in i915_error_state for analysis
```

**HuC Firmware**:
- Media encoding verification and debugging
- Status inspection via `intel_huc_debugfs_register()`
- Load authentication tracking
- Simpler than GuC (no logging or capture)

---

## Error Capture & Coredumps

### Comprehensive State Capture

The error state snapshot captures multiple levels of GPU state:

**Device-Level**:
```
Device:          Intel Arc A770 (0d04:1673)
Driver version:  6.8.0-rc1 i915 #1
Kernel:          5.15.0-56-generic
System time:     2024-02-06 10:23:45 UTC
Uptime:          123.456s
Boot time:       2024-02-06 10:21:41 UTC
IOMMU:           enabled (DMAR)
DMA mask:        64-bit
RPM status:      awake
```

**GT-Level**:
```
GT0:
  Status:        hung (engine reset pending)
  FORCEWAKE:     0x00000003 (multicore enabled)
  Faults:
    EIR:         0x00000001 (illegal instruction)
    PGTBL_ER:    0x00000002 (page table entry error)
    FAULT_TLB_DATA: 0x1234567890 (faulting address)
  EU Attentions:
    Before:      0xFFFF00FF (16 EUs stalled before reset)
    After:       0x0000FFFF (other 16 EUs stalled after)
    Resolved:    0xFFFFFFFF (all attention cleared by reset)
```

**Per-Engine State**:
```
Engine: Render (RCS0)
  State:         active, hung
  Active Context:
    Process:     glxgears (PID 1234, UID 1000)
    HW ID:       0x00000001
    Timeline:    1
  Active Request:
    Seqno:       31 (0x1f)
    Flags:       0x00000005 (active, signal)
    Priority:    normal (0)
    Created:     2.123s ago
  Registers:
    CCID:        0x00010001 (context active)
    IP:          0x7fff0020 (instruction pointer)
    EIP:         0x00000000 (faulting instruction)
    SR:          0x00000001 (EU status: stalled)
    RING_HEAD:   0x00001230 (head pointer)
    RING_TAIL:   0x00001280 (tail pointer)
    RING_STATUS: 0x00000010 (16 pending entries)
    INSTPM:      0x00000000 (instruction parser state)
    MODE:        0x0000FFFF (render mode)
  VMA Captures:
    Batch:       0x1000000000 (8192 bytes, active)
    Context:     0x2000000000 (1M, compressed to 200KB)
    Indirect:    0x3000000000 (512KB, compressed)
```

**VMA Memory Dumps**:
```
VMA 0x1000000000 (batch, 8192 bytes):
  PTR: 0xffffaa00
  [raw hex and ASCII dump]
  00000000: 48 65 6c 6c 6f 20 47 50 55 21 00 00 00 00 00 00 |Hello GPU!..|
  ...
  
VMA 0x2000000000 (context, compressed):
  [zlib compressed, ascii85 encoded]
  u+VL)2fQdI1X/p0eWP5BJ|/X=KqF1QdI1X/p0eWMYP0~
  ... (1000 lines of base85 data)
```

### Post-Mortem Analysis Tools

**Userspace utilities** (to be provided):
- Parse error state from debugfs
- Decompress and display memory contents
- Decode GPU instructions in batch buffer
- Cross-reference with GL/Vulkan debugging symbols
- Generate stack traces for compute shaders

---

## GPU Hang Detection & Recovery

### Hang Detection Flow

```
Timeline of GPU Hang Detection & Recovery
─────────────────────────────────────────────

T=0ms:      normal execution, requests submit
T=1000ms:   heartbeat_work() enqueues pulse request
T=1100ms:   pulse processes normally, reschedules

T=3500ms:   next heartbeat scheduled
T=3600ms:   heartbeat_work() enqueues new pulse
T=3700ms:   pulse STALLS (GPU doesn't complete)

T=4250ms:   [TIMEOUT: 650ms preemption timeout]
            show_heartbeat() logs:
              Last seqno complete: 1020
              Last seqno submitted: 1021 (hung)
              Ring status: 0x00000010 (16 pending)
              Active request: seqno 1021
              GuC state: active, handling context
            
            Attempt intel_engine_reset_prepare():
              ├─ Try preemption (kick GuC)
              └─ Wait for pulse to complete (~100-200ms)

T=4350ms:   [PREEMPTION FAILED]
            Escalate to intel_engine_reset():
              ├─ reset_prepare()
              │   ├─ Quiesce all engines
              │   ├─ Drain ring buffers
              │   └─ Wait for in-flight work
              │
              ├─ __intel_gt_reset()
              │   ├─ Assert reset signal to GPU
              │   ├─ Wait for reset complete (~10ms)
              │   ├─ Deassert reset
              │   └─ Verify recovery state
              │
              ├─ reset_finish()
              │   ├─ Restore engine state
              │   ├─ Reprogram ring buffers
              │   └─ Restore preemption state
              │
              └─ intel_uc_reset_finish()
                  ├─ Reload GuC firmware
                  ├─ Reinitialize GuC state
                  ├─ Restore context (if safe)
                  └─ Resume submission

T=4550ms:   [RESET COMPLETE]
            ├─ Pulse request retired (completing via reset path)
            ├─ heartbeat_work() reschedules
            ├─ Counter increments:
            │   ├─ engine->reset.count++
            │   ├─ gt->reset.count++
            │   └─ i915->gpu_error.reset_count++ (sysfs)
            │
            └─ Normal operation resumes

T=5000ms+:  Subsequent heartbeats succeed
            └─ Continue monitoring
```

### Reset Strategies (Escalation Path)

```
Reset Escalation
────────────────

1. PREEMPTION (fastest, ~0.5s)
   └─ Attempt to preempt current context
      └─ If hangs stalled on long-running shader
         └─ Success: Context switched, no full reset

2. ENGINE RESET (fast, ~1s)
   └─ Reset specific engine (e.g., RCS0)
      └─ Other engines continue
      └─ If only one engine hung
         └─ Success: Minimal disruption

3. FULL GT RESET (slow, ~2s)
   └─ Reset entire GPU tile
      └─ All engines affected
      └─ Firmware reloaded
      └─ For critical faults affecting multiple engines

4. GLOBAL RESET (slowest, ~5s)
   └─ Reset entire device (multiple GTs)
      └─ Last resort for unrecoverable state
      └─ May require driver restart

Selection:
  ├─ Default: Preemption → Engine → Full
  ├─ GuC managed: GuC initiates (firmware decision)
  └─ Watchdog: Skip preemption, go straight to reset
```

### Platform-Specific Reset Implementations

Different GPU generations use different reset mechanisms:

```c
// Generation-specific reset functions in drivers/gpu/drm/i915/gt/

Gen 2-3:
  └─ i915_do_reset()            // Legacy reset (VGA register)

Gen 3:
  └─ g33_do_reset()             // Brookdale reset (device specific)

Gen 4-5:
  └─ g4x_do_reset()             // Ironlake reset

Gen 6+:
  └─ gen6_hw_domain_reset()     // Multi-domain with RCS/BCS/VCS

Gen 8+:
  └─ gen8_engine_reset_prepare() // Per-engine reset
  └─ gen8_gpu_reset()            // Full reset

Gen 12:
  └─ gen12_reset_engines()       // Multiple GT support
  └─ gen12_reset()               // Full device reset
```

---

## Firmware Debugging (GuC/HuC)

### GuC Logging Integration

**Log Level Control**:
```
Level 0: Disabled (no logging)
  └─ Minimal firmware overhead
  └─ Only critical faults logged

Level 1: Error
  └─ ERROR and fatal events
  └─ Still low overhead (~1-2%)

Level 2: Debug (default with debugging enabled)
  └─ State transitions, command processing
  └─ ~3-5% overhead

Level 3: Verbose
  └─ Detailed state tracking, all function calls
  └─ ~10-15% overhead
  └─ For deep debugging only
```

**Log Dump Workflow**:
```
1. GuC logging enabled during boot
   └─ Three ring buffers allocated (crash/debug/capture)
   └─ GuC firmware writes asynchronously

2. Overflow handling
   └─ relay_init() creates userspace ring buffer
   └─ IRQ handler checks overflow flag
   └─ Flushes to relay if needed

3. Access methods:
   a) Real-time streaming (relay)
      └─ cat /sys/kernel/debug/dri/*/guc_log
   
   b) Batch dump (debugfs)
      └─ cat /sys/kernel/debug/dri/*/guc_log_dump
   
   c) Load failure logs
      └─ cat /sys/kernel/debug/dri/*/guc_load_err_log

4. Error state includes GuC logs
   └─ When GPU hangs: logs captured to error_state
   └─ Available for post-mortem analysis
```

### Capture Buffer Processing

**Flow**:
```
GPU Hang (GuC submission)
  │
  ├─ GuC detects hang/fault
  │  └─ Writes diagnostic data to capture buffer:
  │     ├─ GuC own registers
  │     ├─ Active context descriptor
  │     ├─ EU attention bitmask
  │     ├─ Memory fault info
  │     └─ Status page snapshot
  │
  ├─ Kernel receives hang notification
  │  └─ i915_capture_error_state()
  │
  └─ intel_guc_capture_process()
     ├─ Read capture buffer from shared memory
     ├─ Parse capture nodes (per-context)
     ├─ Format for human inspection
     └─ Merge into engine coredump
```

---

## Debug Interfaces (Debugfs/Sysfs)

### Debugfs Files Organization

```
/sys/kernel/debug/dri/*/
├── i915_capabilities          [Device capabilities]
├── i915_mocs_table            [Memory object caching]
├── i915_gpu_info              [GPU state snapshot]
├── i915_error_state           [Last error (coredump)]
├── i915_wal_dumped_status     [Workaround application]
│
├── gt0/                        [GT0 (tile 0)]
│   ├── freq_up_threshold      [Power up frequency]
│   ├── freq_down_threshold    [Power down frequency]
│   ├── heartbeat_interval_ms  [Hang detection sensitivity]
│   ├── preempt_timeout_ms     [Preemption timeout]
│   ├── engines/
│   │   ├── rcs0/
│   │   │   ├── ring_registers [Ring buffer state]
│   │   │   ├── seqno          [Latest sequence number]
│   │   │   └── preempt_timeout_ms
│   │   ├── bcs0/
│   │   └── vcs0/
│   │
│   └── uc/
│       ├── guc_info           [GuC status]
│       ├── guc_load_err_log   [GuC load failure details]
│       ├── guc_log_level      [Logging verbosity 0-3]
│       ├── guc_log_dump       [Current log content]
│       ├── huc_info           [HuC status]
│       └── huc_load_err_log   [HuC load failure details]
│
├── gt1/                        [GT1 (tile 1, if multi-tile]
│   └── (same structure as gt0)
│
├── perf/                       [Performance monitoring]
│   ├── oa_format_*            [OA metric formats]
│   └── metrics_*              [Performance metrics]
│
└── ...
```

### Sysfs Error Counters

```
/sys/class/drm/card*/gt/gt*/
├── reset_count              [Total GT resets]
└── engine_reset_count       [Engine-specific resets]

/sys/class/drm/card*/
├── error/                   [Error statistics]
│   └── reset_count          [Global reset counter]
```

### Example Debugfs Usage

**Inspect GPU state**:
```bash
# Read current GPU info
cat /sys/kernel/debug/dri/0/i915_gpu_info

# Last error state (if hang occurred)
cat /sys/kernel/debug/dri/0/i915_error_state | head -50

# GuC status
cat /sys/kernel/debug/dri/0/gt0/uc/guc_info

# GuC logs
cat /sys/kernel/debug/dri/0/gt0/uc/guc_log_dump
```

**Modify parameters**:
```bash
# Increase hang detection sensitivity (500ms)
echo 500 > /sys/kernel/debug/dri/0/gt0/heartbeat_interval_ms

# Enable verbose GuC logging
echo 3 > /sys/kernel/debug/dri/0/gt0/uc/guc_log_level

# Change power up threshold (frequency scaling)
echo 80 > /sys/kernel/debug/dri/0/gt0/freq_up_threshold
```

---

## Key Data Structures

### `struct i915_gpu_coredump`

```c
struct i915_gpu_coredump {
    ktime_t time;                     // Capture timestamp
    struct timespec64 boottime;       // System boot time
    struct drm_i915_private *i915;    // Back reference
    
    struct intel_gt_coredump *gt;     // Per-GT state
    
    /* Device info */
    int num_gt;                       // Number of GT tiles
    struct {
        u32 id;                       // Device ID
        u32 revision;                 // Device revision
    } device_info;
    
    /* Platform-specific */
    struct i915_params params;        // Module parameters
    struct intel_device_info info;    // Capabilities
    
    /* Error state */
    u32 reset_count;                  // Reset counter
    u32 suspend_count;                // Suspend counter
    
    /* System state */
    unsigned long uptime;             // Uptime in seconds
    unsigned long boottime;           // Boot time reference
};
```

### `struct intel_engine_coredump`

```c
struct intel_engine_coredump {
    const struct intel_engine_cs *engine;
    
    /* Active context at hang */
    struct {
        char comm[TASK_COMM_LEN];      // Process name (e.g., "glxgears")
        pid_t pid;                      // Process ID
        uid_t uid;                      // User ID
        u32 handle;                     // Context handle
    } ctx;
    
    /* Active request */
    u64 rseqno;                        // Request seqno
    int context_loss_count;            // Context resets
    u32 ccid;                          // Cached context ID
    u32 seqno_relative_to_first;       // Seqno offset
    
    /* Registers */
    struct {
        u32 instpm;                    // Instruction parser
        u32 ring_esr;                  // Ring error status
        u32 ip;                        // Instruction pointer
        u32 eip;                       // Error IP
    } reg;
    
    /* Ring buffer state */
    u32 ring_head;                     // RCS0_HEAD
    u32 ring_tail;                     // RCS0_TAIL
    u32 ring_status;                   // RCS0_STATUS
    
    /* VMAs captured */
    struct intel_vma_coredump *vmas;   // Batch, context objects
};
```

### `struct i915_debugger`

```c
struct i915_debugger {
    struct drm_i915_private *i915;
    
    /* Lifecycle */
    struct kref ref;
    bool enabled;
    
    /* Event stream */
    struct kfifo *event_fifo;          // Circular event buffer
    wait_queue_head_t wq;              // Wake waiters on events
    
    /* Resource tracking */
    struct rb_tree uuids;              // UUID resource handles
    struct list_head signalers;        // Contexts with pending signals
    
    /* ACK synchronization */
    struct rb_tree ack_tree;           // Pending ACKs
    spinlock_t ack_lock;
    
    /* Guarded contexts */
    struct list_head gated_contexts;   // Contexts under debug
    spinlock_t gated_lock;
};
```

---

## Integration & Performance

### Subsystem Integration Points

```
Debugger Support Integration Map
─────────────────────────────────

GPU Execution Path:
  i915_request_submit()
    ├─→ i915_debugger_notify_submit()    [Notify event stream]
    └─→ Check if context gated (debugged)
        └─→ Skip GuC optimization if yes

GPU Completion Path:
  intel_engine_signal_breadcrumbs_irq()
    ├─→ Check EU attention state
    │   └─→ i915_debugger_handle_eu_attention()
    │       └─→ Queue EU_ATTENTION event
    └─→ Normal signaling continues

Error Path:
  reset_engine() / intel_gt_reset()
    ├─→ i915_capture_error_state()
    │   ├─ Capture for all VMs
    │   ├─ Capture GuC logs
    │   └─ Store to i915->gpu_error.first_error
    │
    ├─→ Show diagnostics (show_heartbeat)
    │   └─→ Log to dmesg for system visibility
    │
    └─→ Perform reset
        └─→ Timestamp stored in error state

Memory Fault Path:
  iommu fault handler
    └─→ i915_debugger_handle_page_fault()
        ├─→ Log to error state
        └─→ Queue PAGEFAULT event to debugger
```

### Performance Overhead

**Disabled (default)**:
```
No cost - debug infrastructure inert
```

**Heartbeat only** (active):
```
- Pulse request: ~100μs submission overhead
- Timeout check: ~10μs per interval (2.5s default)
- Net: 0.1% overhead for hang detection
```

**Error capture** (on hang):
```
- Coredump collection: ~50-100ms (varies)
- Compression: ~5-10ms for large VMAs
- Memory usage: 1-10 MB per error state
- Cost: Only on hang (amortized low)
```

**Debugger enabled** (with active session):
```
- Event enqueue: ~1-5μs per submission/fault
- UUID tracking: O(1) lookup in rb_tree
- Context gating: Skips GuC optimization (~5% submission overhead)
- Memory: ~1MB per debugger session + event FIFO
```

**GuC logging** (level 2):
```
- Firmware overhead: 3-5%
- Memory: 64KB crash + 256KB debug + capture buffer
- No I/O overhead (ring buffer writes, no syscalls)
```

**Performance profiling** (OA sampling):
```
- At 100Hz: ~5% GPU overhead
- At 10Hz: ~0.5% GPU overhead
- Memory: Configurable (default ~16MB ring)
- CPU: Negligible (async sample collection)
```

### Optimization Knobs

```bash
# For minimal overhead (production):
heartbeat_interval_ms=5000           # Longer timeout
guc_log_level=-1                      # Disable GuC logging
CPTCFG_DRM_I915_COMPRESS_ERROR=n     # No compression

# For debugging:
heartbeat_interval_ms=1000           # Tighter detection
guc_log_level=2                      # Debug logging enabled
CPTCFG_DRM_I915_COMPRESS_ERROR=y     # Enable compression

# For full diagnostics:
heartbeat_interval_ms=500            # Very sensitive
guc_log_level=3                      # Verbose firmware logs
perf_stream enabled                  # OA profiling
i915_debugger enabled                # Runtime debugging
```

---

## Summary

The i915 GPU driver's debugging infrastructure provides:

1. **Comprehensive error capture** - Complete GPU state snapshots on hang/fault
2. **Proactive hang detection** - Heartbeat mechanism with tunable sensitivity
3. **Firmware visibility** - GuC logging and capture for firmware-side debugging
4. **Runtime control** - Event-driven debugger protocol for interactive debugging
5. **Performance visibility** - OA sampling and tracepoint integration
6. **Minimal default overhead** - Features disabled/tuned by default

This multi-layered approach enables both offline (post-mortem) and online (interactive) debugging of GPU execution, making it suitable for both developer debugging and production diagnostics.

---

## References

**Source Files**:
- `drivers/gpu/drm/i915/i915_gpu_error.c` - Error capture
- `drivers/gpu/drm/i915/i915_debugger.c` - Event debugger
- `drivers/gpu/drm/i915/gt/intel_engine_heartbeat.c` - Hang detection
- `drivers/gpu/drm/i915/gt/uc/intel_guc_log.c` - Firmware logging
- `drivers/gpu/drm/i915/i915_debugfs.c` - Debug interface
- `drivers/gpu/drm/i915/gt/sysfs_gt_errors.c` - Error counters

**Related Documentation**:
- `06-Virtual-Memory.md` - Memory debugging context
- `07-TBB-Task-Scheduling.md` - CPU task monitoring
- `05-Request-Scheduling.md` - Request tracking

