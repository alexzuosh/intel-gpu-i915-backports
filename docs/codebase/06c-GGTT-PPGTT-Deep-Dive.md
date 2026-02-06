# GGTT vs PPGTT - Comprehensive Deep Dive

**Document:** i915 GPU Driver Virtual Memory System  
**Topic:** Graphics Translation Tables - Detailed Comparison  
**Related Files:** `intel_ggtt.c`, `intel_ppgtt.c`, `intel_gtt.h`  
**Kernel Version:** 6.8+

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [GGTT: Global Graphics Translation Table](#ggtt-global-graphics-translation-table)
3. [PPGTT: Per-Process Graphics Translation Table](#ppgtt-per-process-graphics-translation-table)
4. [GGTT vs PPGTT Comparison](#ggtt-vs-ppgtt-comparison)
5. [Architecture Integration](#architecture-integration)
6. [Use Cases & Decision Tree](#use-cases--decision-tree)
7. [Advanced Scenarios](#advanced-scenarios)
8. [Performance Implications](#performance-implications)
9. [Hardware Constraints](#hardware-constraints)
10. [Debugging GGTT/PPGTT Issues](#debugging-ggttppgtt-issues)

---

## Executive Summary

### Two Translation Table Systems

The i915 driver uses **two parallel GPU address translation systems** for different purposes:

```
                    GPU Memory Management
                            │
        ┌───────────────────┴───────────────────┐
        │                                       │
    ┌───▼─────┐                         ┌──────▼──────┐
    │  GGTT   │                         │  PPGTT      │
    │ Global  │                         │  Per-Process│
    └────┬────┘                         └──────┬──────┘
         │                                     │
    For i915 internals:                    For applications:
    • Kernel resources                     • User contexts
    • System operations                    • Per-application isolation
    • All contexts share                   • Private memory spaces
    • 256MB-2GB total                      • 48-bit per-context
    • Single flat namespace                • Separate namespaces
```

### Quick Decision Matrix

```
Use GGTT when:
  ✓ GPU firmware needs access
  ✓ Shared resources (HW status pages)
  ✓ Kernel memory objects
  ✓ Display framebuffers
  ✓ Scratch buffers
  
Use PPGTT when:
  ✓ Application memory
  ✓ Need isolation between contexts
  ✓ Large address space needed
  ✓ Multiple applications running
  ✓ Want security/protection
```

---

## GGTT: Global Graphics Translation Table

### Purpose & Philosophy

GGTT is the **GPU's global, system-wide address space** visible to all GPU contexts and hardware units.

```
┌─────────────────────────────────────────┐
│         GPU Memory from View:           │
│                                         │
│  GGTT Virtual Address Space             │
│  ┌─────────────────────────────────┐   │
│  │                                 │   │
│  │ [Firmware Code] [DMA Objects]   │   │
│  │ [Status Pages] [Scratch]        │   │
│  │ [Display Buffers] [Work Queues] │   │
│  │ [System Resources]              │   │
│  │                                 │   │
│  └─────────────────────────────────┘   │
│  All contexts share these mappings      │
└─────────────────────────────────────────┘
```

### GGTT Characteristics

```c
struct i915_ggtt {
    struct i915_address_space base;
    
    // Physical characteristics
    size_t total_size;          // 256MB to 2GB (hardware-dependent)
    u32 mappable_end;           // Limit for CPU access
    
    // Status pages (firmware communication)
    struct i915_page_table *status_page;
    dma_addr_t status_page_dma;
    
    // VM operations
    struct vm_ops vm_ops;
    
    // Initialization
    void (*setup)(struct i915_ggtt *ggtt);
    int (*probe)(struct i915_ggtt *ggtt);
};
```

### Key Features

#### 1. Single Global Namespace

```
GGTT Address Space:
┌─────────────────────────────────────────┐
│ 0x0     ┌──────────────────────────┐   │
│         │  Firmware Objects        │   │
│ 0x200K  ├──────────────────────────┤   │
│         │  Status Pages            │   │
│ 0x400K  ├──────────────────────────┤   │
│         │  Display Framebuffers    │   │
│ 0x100M  ├──────────────────────────┤   │
│         │  System Buffers          │   │
│ 0x200M  ├──────────────────────────┤   │
│         │  User Objects (pinned)   │   │
│ MAX_END └──────────────────────────┘   │
│ (256M-2GB total)                       │
└─────────────────────────────────────────┘

All GPU units see same virtual → physical mapping
```

#### 2. Limited Size

```
GGTT Size by Hardware Generation:

                    Size        Use in Practice
Gen2 (i915):        8 MB        Legacy systems
Gen3-4:             32-128 MB   Older platforms
Gen5+:              512 MB      Standard modern
Gen7+:              2 GB        Latest GPUs

Constraint: Can't grow beyond hardware TLB size
```

#### 3. Always Active

```
GPU Context Switching:

┌─ Context A  ─┐         ┌─ Context B  ─┐
│ PPGTT A      │         │ PPGTT B      │
│ (private)    │         │ (private)    │
└──────────────┘         └──────────────┘
      │                         │
      └─────────┬───────────────┘
                │
        ┌───────▼────────┐
        │ GGTT (always)  │  ← Shared, stays same
        │ (shared)       │
        └────────────────┘

GGTT mappings never change during context switch
PPGTT mappings change (or are per-context anyway)
```

### GGTT Use Cases

#### 1. Firmware Communication

```c
// GPU firmware needs to read/write shared state
// Must use GGTT because firmware runs in all contexts

struct i915_status_page {
    u32 *page_addr;                 // CPU virtual
    dma_addr_t dma;                 // Physical
    u32 ggtt_offset;                // GGTT virtual
};

// Firmware reads GPU status via GGTT mapping
// Driver writes request via GGTT mapping
```

**Example:** GPU completion status, interrupt handling

#### 2. Display Engine

```c
// Display needs consistent framebuffer access
// Cannot use per-context PPGTT (display is not a context)

struct intel_framebuffer {
    struct drm_framebuffer base;
    struct drm_i915_gem_object *obj;    // GPU memory
    
    // Bound in GGTT for display engine access
    u32 ggtt_offset;                    // GGTT virtual address
};
```

**Example:** Scanout buffers visible to display controller

#### 3. Kernel Internal Operations

```c
// Operations that need GPU access outside user contexts

// Memory migration (CPU ↔ VRAM copy)
struct drm_i915_gem_object *copy_batch;
// Bound in GGTT so kernel code can use it

// Error capture (collecting GPU state)
struct i915_gpu_error_state *error_state;
// Scratch buffers in GGTT for state collection

// Register updates (batch commands)
struct batch_command_buffer batch;
// GGTT-resident for kernel submission
```

#### 4. Shared System Resources

```c
// Hardware needs access to:

// 1. Ring buffers (if not PPGTT-based)
struct intel_ring {
    // Queues work for GPU
    // All contexts see same queue
};

// 2. Semaphore and fence objects
struct intel_semaphore {
    dma_addr_t ggtt_offset;  // Shared location
};

// 3. Scratch page (error recovery)
struct drm_i915_private {
    struct i915_page_table *scratch_page;
    // Referenced by all page tables
};
```

#### 5. Legacy/Compatibility

```c
// DRM Gem objects in GGTT (pre-PPGTT systems)

if (!ppgtt_supported) {
    // Old GPU, must use GGTT for all memory
    bind_to_ggtt(obj);
} else {
    // Modern GPU, use PPGTT for apps, GGTT for kernel
    bind_to_ppgtt(obj);
    bind_to_ggtt(kernel_obj);
}
```

### GGTT Advantages

```
✓ Always accessible to all GPU units
✓ No address space switching needed
✓ Firmware can access without context change
✓ Display engine doesn't need GPU context
✓ Simpler for kernel operations
✓ Guaranteed visibility across all work
```

### GGTT Disadvantages

```
✗ Limited size (256MB-2GB)
✗ Contention with user objects
✗ No isolation between applications
✗ No protection boundaries
✗ Can fragment quickly
✗ Shared namespace leads to collisions
```

---

## PPGTT: Per-Process Graphics Translation Table

### Purpose & Philosophy

PPGTT is the **per-context private address space** where each GPU execution context has its own isolated virtual address space.

```
┌──────────────────────────────────────────────┐
│    GPU With Multiple User Applications       │
│                                              │
│ Application A          Application B         │
│ ┌─────────────┐       ┌─────────────┐       │
│ │  PPGTT A    │       │  PPGTT B    │       │
│ │ 0x0-0xFFF   │       │ 0x0-0xFFF   │       │
│ │             │       │             │       │
│ │ [App Memory]│       │ [App Memory]│       │
│ │ [Scratch]   │       │ [Scratch]   │       │
│ │ [Buffers]   │       │ [Buffers]   │       │
│ │             │       │             │       │
│ └─────────────┘       └─────────────┘       │
│       Isolated              Isolated         │
│       Namespaces            Namespaces       │
│                                              │
│ When context A runs: Use PPGTT A's mappings │
│ When context B runs: Use PPGTT B's mappings │
│ GGTT always used for firmware/kernel work   │
└──────────────────────────────────────────────┘
```

### PPGTT Characteristics

```c
struct i915_ppgtt {
    struct i915_address_space base;
    
    // Structure of page tables
    struct i915_pml4 *pml4;         // Top-level directory (4-level)
    // or
    struct i915_pdp *pdp;           // Top-level (3-level)
    
    // Size and limits
    u64 total_size;                 // 48-bit address space (256TB)
    
    // Ownership
    struct i915_gem_context *owner; // Which context uses this
    
    // VM operations
    struct vm_ops vm_ops;
};
```

### Key Features

#### 1. Per-Context Isolation

```
Context A                   Context B
┌──────────┐               ┌──────────┐
│ PPGTT A  │               │ PPGTT B  │
│          │               │          │
│ VAddr    │ Physical      │ VAddr    │ Physical
│ 0x1000   │ → Frame 100   │ 0x1000   │ → Frame 200
│ 0x2000   │ → Frame 101   │ 0x2000   │ → Frame 201
│ 0x3000   │ → [unmapped]  │ 0x3000   │ → Frame 202
│          │               │          │
└──────────┘               └──────────┘

Both can use 0x1000, but point to different physical pages
Complete isolation - App A cannot see App B's memory
```

#### 2. Large Address Space

```
PPGTT Address Space:

48-bit virtual address space
= 256 TB per context

Can map:
  • All system RAM
  • All VRAM
  • Large GPU resources
  • Sparse allocations

vs GGTT (256MB-2GB): Very limited
```

#### 3. Context-Specific

```
GPU Context Management:

┌─────────────────────────┐
│  GPU with 2 Contexts    │
├─────────────────────────┤
│                         │
│  Logical Context 0      │
│  ├─ PPGTT 0             │  ← Per-context
│  ├─ HW Status Page      │     page table
│  └─ Execution State     │
│                         │
│  Logical Context 1      │
│  ├─ PPGTT 1             │  ← Different PPGTT
│  ├─ HW Status Page      │     same GPU HW
│  └─ Execution State     │
│                         │
│  GGTT (shared)          │  ← Always active
│                         │
└─────────────────────────┘

When switching contexts:
  1. Save Context 0 state
  2. Switch PPGTT register to Context 1's PPGTT base
  3. Load Context 1 state
  4. Continue execution
  
GGTT mapping stays same (doesn't need reload)
```

#### 4. Hierarchical Structure

```
PPGTT 4-Level Hierarchy:

┌─────────────────────────────────────────┐
│  Virtual Address 0x123456789ABC         │
└──────────────┬──────────────────────────┘
               │
        [Extract fields]
        ┌──────┬──────┬──────┬──────┐
        │ PML4 │ PDP  │  PD  │  PT  │  PT Entry
        │ Idx  │ Idx  │ Idx  │ Idx  │
        │  9   │  9   │  9   │  9   │   12    
        └──┬───┴──┬───┴──┬───┴──┬───┘
           │      │      │      │
      ┌────▼──┐┌──▼──┐┌─▼──┐┌──▼────┐
      │ PML4  ││ PDP ││ PD ││ PT    │
      │  [0]  ││[512]││[0]││[1024] │
      ├───────┤└─────┘└───┘└───────┘
      │ ...   │         │
      │ [511] │         └─→ Physical Frame Address
      └───────┘             + Cache Bits
                            + Flags (R/W/X)

Depth: 4 levels (supports 48-bit addresses)
Per-level: 512 entries (9-bit indices)
Per-entry: 64 bits (address + metadata)
```

### PPGTT Use Cases

#### 1. User Application Memory

```c
// Every application has its own PPGTT

struct drm_i915_gem_context {
    struct i915_ppgtt *ppgtt;  // Exclusive PPGTT for this context
};

// Userspace ioctl:
gem_context_create(device);
  ↓
Creates new PPGTT
Each app gets isolated address space
All app memory mapped in its PPGTT
```

**Example:** Vulkan app buffer bindings, OpenGL textures

#### 2. Memory Isolation & Security

```c
// Different applications cannot access each other's memory

Context A                    Context B
┌────────────┐              ┌────────────┐
│ PPGTT A    │              │ PPGTT B    │
│ Maps:      │              │ Maps:      │
│ • App A    │              │ • App B    │
│   buffers  │              │   buffers  │
│ • App A    │              │ • App B    │
│   textures │              │   textures │
│            │              │            │
│ Cannot see │              │ Cannot see │
│ PPGTT B!   │              │ PPGTT A!   │
└────────────┘              └────────────┘

Isolation enforced by GPU hardware
Even if malicious app tries:
  mov %cr3, %eax      ← Cannot read other PPGTT base
  jump(other_ppgtt)   ← GPU mode prevents this

Security benefit: Untrusted app can't read other app's data
```

#### 3. Large Working Sets

```c
// Applications with large memory requirements

PPGTT Advantages:
  48-bit address space = 256 TB
  
  • Machine learning: 100GB+ model weights
  • Scientific computing: Massive arrays
  • Video processing: 4K/8K framebuffers
  • Game engines: Entire game world

GGTT Limitation:
  256MB-2GB global space
  
  • Cannot fit large apps
  • Contention with other resources
  • Fragmentation issues
  
Solution: Use PPGTT for app memory, GGTT for kernel resources
```

#### 4. Compute Workloads

```c
// Compute jobs need private memory spaces

Kernel 1 (PPGTT A):
  • Private scratch buffers
  • Work group data
  • Results buffer
  
Kernel 2 (PPGTT B):
  • Different private scratch
  • Different work group data
  • Different results
  
Both can run in parallel:
  • GPU core 0: Kernel 1 (PPGTT A)
  • GPU core 1: Kernel 2 (PPGTT B)
  
No interference or corruption
```

#### 5. Context Switching

```c
// GPU context switch is efficient with PPGTT

Context A → Context B:

1. GPU completes current instruction
2. Save Context A state (LRC)
3. Update PPGTT register to Context B's PPGTT base
4. Load Context B state (LRC)
5. Resume execution
6. GPU automatically uses Context B's PPGTT for VA translation

Cost: Just PPGTT base register update
No need to flush all page tables
TLB keeps helpful entries (tagged with ASID if supported)
```

### PPGTT Advantages

```
✓ Large address space (48-bit, 256TB)
✓ Complete isolation between contexts
✓ No fragmentation (dedicated per context)
✓ Security boundary (untrusted apps)
✓ Efficient context switching (just base register)
✓ No contention with other apps
✓ Sparse address space support
✓ Clean namespace for applications
```

### PPGTT Disadvantages

```
✗ Per-context overhead (memory, management)
✗ Context switch cost (page table state)
✗ More complex page table structures
✗ TLB misses on context switch (potentially)
✗ Requires context switch choreography
✗ Kernel cannot easily access PPGTT memory
✗ Requires GPU context state management
```

---

## GGTT vs PPGTT Comparison

### Side-by-Side Comparison

| Aspect | GGTT | PPGTT |
|--------|------|-------|
| **Visibility** | Global (all contexts) | Per-context (isolated) |
| **Size** | 256MB-2GB | 48-bit (256TB) |
| **Namespace** | Single global | Separate per context |
| **Use Case** | Kernel resources | Application memory |
| **Security** | None | Complete isolation |
| **Contention** | High (shared) | None (private) |
| **Address Spaces** | 1 (global) | 1 per context |
| **Switching** | Never changes | On context switch |
| **Fragmentation** | Likely | Minimal |
| **Access Method** | All GPU units | Only in context |
| **Firmware Access** | ✓ Direct | ✗ Indirect (via GGTT) |
| **Page Levels** | 1-2 levels | 4 levels |
| **TLB Entries** | Global | Per-context |

### Quick Reference Table

```
Question: Where should I map this memory?

┌─────────────────────────────────────────────────────────┐
│ Is it kernel/firmware internal memory?                  │
│   YES → Use GGTT (always visible)                      │
│   NO  → Continue...                                    │
├─────────────────────────────────────────────────────────┤
│ Is it for display engine (scanout)?                    │
│   YES → Use GGTT (display not a GPU context)           │
│   NO  → Continue...                                    │
├─────────────────────────────────────────────────────────┤
│ Is it GPU firmware code/data?                          │
│   YES → Use GGTT (firmware sees all contexts)          │
│   NO  → Continue...                                    │
├─────────────────────────────────────────────────────────┤
│ Is it shared resource (work queue, semaphore)?         │
│   YES → Use GGTT (all contexts need access)            │
│   NO  → Continue...                                    │
├─────────────────────────────────────────────────────────┤
│ Is it user application memory?                         │
│   YES → Use PPGTT (isolation, large space)             │
│   NO  → Use GGTT (default for i915 internals)          │
└─────────────────────────────────────────────────────────┘
```

### Mapping Examples

```
┌─────────────────────────────────────────────────────┐
│ Typical i915 Memory Layout with Both Tables         │
└─────────────────────────────────────────────────────┘

GGTT Virtual Address Space:
┌────────────────────────────┐
│ 0x0                        │
│  ├─ Firmware Code (4MB)    │  In GGTT
│  ├─ Status Pages (2MB)     │  (visible to all contexts)
│  ├─ Display FB (64MB)      │
│  ├─ Work Queues (16MB)     │
│  ├─ Scratch Buffers (4MB)  │
│  └─ System Objects         │
│                            │
│ Total: ~100-200MB used     │
│        ~50-100MB available │
│ (out of 256MB-2GB total)   │
└────────────────────────────┘

Per-Context PPGTT Virtual Address Space (App's view):
┌────────────────────────────┐
│ 0x0                        │
│  ├─ App Textures           │  In PPGTT A
│  ├─ App Buffers            │  (private to this context)
│  ├─ Scratch Space          │
│  ├─ Stack Memory           │
│  └─ Large GPU Allocations  │
│                            │
│ Total: Variable per app    │
│        (unlimited, 256TB)  │
└────────────────────────────┘

Another Context's PPGTT:
┌────────────────────────────┐
│ 0x0                        │
│  ├─ Different Textures     │  In PPGTT B
│  ├─ Different Buffers      │  (isolated from app A)
│  └─ ...                    │
│                            │
│ Completely separate        │
│ namespace from PPGTT A     │
└────────────────────────────┘
```

---

## Architecture Integration

### GGTT/PPGTT in i915 Code Structure

```
┌────────────────────────────────────────┐
│      i915 GPU Driver Structure         │
├────────────────────────────────────────┤
│                                        │
│  struct drm_i915_private               │
│  {                                     │
│    struct i915_ggtt *ggtt;             │  ← Global GGTT
│                                        │
│    struct {                            │
│      struct i915_context *active;      │  ← Current context
│    } contexts;                         │
│  }                                     │
│                                        │
│  struct i915_gem_context               │
│  {                                     │
│    struct i915_ppgtt *ppgtt;           │  ← Context's PPGTT
│    struct intel_engine_cs *engines;    │
│    ...                                 │
│  }                                     │
│                                        │
└────────────────────────────────────────┘
```

### VMA Binding to Both Tables

```c
struct i915_vma {
    struct drm_i915_gem_object *obj;
    struct i915_address_space *vm;  // Which address space?
    
    // Bound in GGTT?
    bool bound_to_ggtt;
    u32 ggtt_offset;
    
    // Bound in PPGTT?
    bool bound_to_ppgtt;
    struct i915_ppgtt_vma *ppgtt_vma;
};

// Typical object lifetime:
drm_i915_gem_object obj;
  ├─ When used by GPU:
  │  ├─ Bind to PPGTT (app context)
  │  └─ Bind to GGTT if also needed by kernel
  └─ When used by kernel:
     └─ Bind to GGTT only
```

### Context Switch Choreography

```
GPU Context Switch (A → B):

T1: Current state
    ├─ GPU running Context A
    ├─ PPGTT register points to PPGTT_A
    ├─ All VA translations use PPGTT_A
    └─ All GGTT mappings visible

T2: Request switch
    └─ Software requests context switch

T3: GPU stop and save
    ├─ GPU finishes current instruction
    ├─ Save Context A state (HW registers, LRC)
    ├─ Flush TLB (to clear context A entries)
    └─ GPU now idle

T4: Switch PPGTT
    ├─ Write PPGTT_B base address to GPU register
    ├─ GPU now translates VA using PPGTT_B
    └─ GGTT still same (always active)

T5: Load and resume
    ├─ Load Context B state (HW registers, LRC)
    ├─ Resume execution
    ├─ GPU running Context B
    ├─ PPGTT register points to PPGTT_B
    └─ All VA translations use PPGTT_B

TLB Optimization:
  • Some GPUs tag TLB entries with context ID
  • Old TLB entries not invalidated (just tagged wrong)
  • Reduces TLB flush overhead
```

---

## Use Cases & Decision Tree

### Decision: When to Use GGTT

```
Use GGTT when ANY of these are true:

1. Firmware/GPU access needed by ALL contexts
   ├─ GPU firmware code/data
   ├─ Status pages (HW communication)
   ├─ Common scratch buffers
   └─ Interrupt handling state

2. Not part of a GPU context
   ├─ Display engine (not a GPU context)
   ├─ DMA operations (kernel-driven)
   ├─ Memory management work
   └─ Register update batches

3. Must be visible across context switches
   ├─ Shared semaphores
   ├─ Global work queues
   ├─ System resource pointers
   └─ Error capture state

4. Legacy/pre-PPGTT systems
   ├─ Old GPUs without PPGTT
   ├─ Backward compatibility
   ├─ Older kernel versions
   └─ Limited address space
```

### Decision: When to Use PPGTT

```
Use PPGTT when ANY of these are true:

1. It's user application memory
   ├─ Buffers created by app
   ├─ Textures
   ├─ Render targets
   ├─ Compute workgroup data
   └─ App-specific allocations

2. Isolation required
   ├─ Different security domains
   ├─ Untrusted applications
   ├─ Multiple users on system
   ├─ Sandboxed execution
   └─ Resource limits needed

3. Large working set
   ├─ >256MB allocations
   ├─ Multiple large buffers
   ├─ Video processing (4K+)
   ├─ Scientific computing
   └─ Game engines

4. Context-specific resources
   ├─ Per-context scratch space
   ├─ Application-private state
   ├─ Thread-local storage
   ├─ Per-context work queues
   └─ Context-specific caches

5. Modern systems
   ├─ GPU with PPGTT support (Gen7+)
   ├─ Security-conscious design
   ├─ Multi-application scenarios
   └─ Large memory systems
```

### Real-World Allocation Examples

#### Example 1: Vulkan Application

```c
// Vulkan buffer creation

struct vulkan_buffer {
    size_t size;
    void *cpu_ptr;
    uint64_t gpu_va;  // GPU virtual address
};

// Creation flow:
vkCreateBuffer(device, &create_info, &buffer)
  ↓
// Get PPGTT for this context
ppgtt = device_context->ppgtt
  ↓
// Allocate physical pages
pages = allocate_pages(size)
  ↓
// Map in context's PPGTT
gpu_va = ppgtt->map(pages)
  ↓
// Return to application
buffer.gpu_va = gpu_va  // Application sees this VA
// Only valid in this context's PPGTT
```

**Result:**
- Buffer in PPGTT (context-specific)
- Not in GGTT (app doesn't need kernel access)
- Large size supported (48-bit address space)
- Isolated from other apps

#### Example 2: GPU Status Page

```c
// GPU status page (HW communication)

struct i915_status_page {
    void *page_addr;        // CPU virtual
    dma_addr_t page_dma;    // Physical
};

// Setup:
allocate_page()
  ↓
// Map in GGTT (not PPGTT)
ggtt_offset = ggtt->map(page_dma)
  ↓
// Tell GPU via register
GPU_REGISTER_STATUS_PAGE = ggtt_offset
  ↓
// Now firmware can access via GGTT
// All contexts see same status page
```

**Result:**
- Status page in GGTT (always visible)
- Firmware can read/write (all contexts)
- Small size (1 page, fits in GGTT easily)
- Shared resource (no isolation needed)

#### Example 3: Display Framebuffer

```c
// Display framebuffer (scanned by display controller)

struct intel_framebuffer {
    struct drm_i915_gem_object *obj;
    uint32_t ggtt_offset;  // Where to scan from
};

// Setup:
find_or_create_framebuffer(width, height, format)
  ↓
// Allocate GPU memory
gpu_obj = gem_object_create(size)
  ↓
// Map in GGTT (display not a GPU context)
ggtt_offset = ggtt->map(gpu_obj->pages)
  ↓
// Tell display controller to scan from GGTT
display_register_primary_plane = ggtt_offset
  ↓
// Display now scans GGTT address during hsync/vsync
```

**Result:**
- Framebuffer in GGTT (display engine access)
- Not in any PPGTT (display not a GPU context)
- Shared (all processes see same scanout)
- Specific size (resolution-dependent)

#### Example 4: GPU Memory Migration

```c
// Move memory from System RAM to VRAM

struct migration_context {
    struct drm_i915_gem_object *obj;  // To move
    
    // Temporary work objects (GGTT)
    struct batch_buffer *copy_batch;   // Copy command batch
    struct drm_i915_gem_object *scratch;  // Scratch space
};

// Migration:
migrate_to_vram(obj)
  ↓
// 1. Create copy batch in GGTT
copy_batch = create_batch_buffer();
ggtt_map(copy_batch);  // Kernel needs access
  ↓
// 2. Bind source in source context's PPGTT
ppgtt_map(obj, source_ppgtt);
  ↓
// 3. Allocate destination in VRAM
vram_frames = allocate_vram(obj->size);
  ↓
// 4. Bind destination in GGTT (temp)
ggtt_map_vram(vram_frames);
  ↓
// 5. Copy (GPU reads source VA, writes dest VA)
// Uses both PPGTT and GGTT in copy commands
  ↓
// 6. Rebind in PPGTT to point to VRAM
ppgtt_update(obj, ppgtt, vram_frames);
  ↓
// 7. Cleanup GGTT scratch
ggtt_unmap(copy_batch);
ggtt_unmap(vram_frames);
```

**Result:**
- Copy batch in GGTT (kernel-driven)
- Source in PPGTT (app memory, original location)
- Destination in GGTT (temporary) then PPGTT (final)
- Complex use of both address spaces

---

## Advanced Scenarios

### Scenario 1: Shared Memory Between Contexts

```
Context A              Context B
┌──────────┐          ┌──────────┐
│ PPGTT A  │          │ PPGTT B  │
│          │          │          │
│ VA 0x1000│──┐       │ VA 0x1000│──┐
│          │  │       │          │  │
└──────────┘  │       └──────────┘  │
              │                     │
              └─────────┬───────────┘
                        │
                ┌───────▼────────┐
                │ Physical Frame │
                │ (shared buffer)│
                └────────────────┘

Same physical frame in both PPGTT!
Different VA (0x1000 in both)
But same physical memory

Implementation:
  1. Create shared object
  2. Allocate physical frames
  3. Map in PPGTT A (VA0 → frames)
  4. Map in PPGTT B (VA1 → frames)
  5. Both contexts can write
  
Synchronization:
  Requires explicit sync (fences, semaphores)
  OS doesn't prevent concurrent writes
  Applications responsible for coordination
```

### Scenario 2: Mixed GGTT and PPGTT Workload

```
GPU Execution:

Kernel-driven work:
  ├─ Status page (GGTT) - read GPU status
  ├─ Copy batch (GGTT) - for memory copy
  └─ Scratch (GGTT) - error capture
  
Application work (Context A):
  ├─ User textures (PPGTT A)
  ├─ User buffers (PPGTT A)
  └─ User render targets (PPGTT A)
  
Application work (Context B):
  ├─ Different textures (PPGTT B)
  ├─ Different buffers (PPGTT B)
  └─ Different render targets (PPGTT B)

Execution timeline:
  T1: App A submits work
      ├─ Use PPGTT A for VA translation
      ├─ Use GGTT for status page reads (HW)
      └─ GPU executes
  
  T2: GPU context switch
      ├─ Switch PPGTT register
      ├─ Flush TLB (or tag entries)
      └─ GGTT still visible (no switch needed)
  
  T3: App B submits work
      ├─ Use PPGTT B for VA translation
      ├─ Use GGTT for status page reads (HW)
      └─ GPU executes
```

### Scenario 3: Sparse Address Space

```
PPGTT sparse mapping:

┌─────────────────────────────────────┐
│ Virtual Address Space (48-bit)      │
├─────────────────────────────────────┤
│                                     │
│ 0x0     ┌─ Mapped (heap)         │
│         │                        │
│ 0x100M  ├─ Unmapped (gap)        │
│         │                        │
│ 0x200M  ┌─ Mapped (stack)        │
│         │                        │
│ 0x300M  ├─ Unmapped (gap)        │
│         │                        │
│ ...     ├─ ...                   │
│         │                        │
│ 0x1TB   ├─ Mapped (texture cache)│
│         │                        │
│ Rest    └─ Unmapped              │
│                                   │
│ Total: 256TB addressable          │
│ Physically: ~100MB used           │
│                                   │
└─────────────────────────────────────┘

GGTT cannot do this:
  256MB total size
  Must map contiguously (mostly)
  High fragmentation risk
  
PPGTT supports:
  Arbitrary sparse mappings
  Only used memory costs physical space
  Large address space, minimal phys RAM
```

---

## Performance Implications

### TLB (Translation Lookaside Buffer) Impact

```
Scenario: GPU rendering 1000x1000 textures

Cache A (GGTT):
  ├─ GGTT TLB entries: ~128-256 entries
  ├─ All entries for GGTT (small, compact)
  ├─ High hit rate if GGTT stays resident
  └─ Cost of miss: 100+ GPU cycles (page walk)

Cache B (PPGTT):
  ├─ Per-context TLB entries: ~256-512
  ├─ Entries for PPGTT A mappings
  ├─ On context switch: entries invalidated/tagged wrong
  ├─ Causing TLB misses initially
  └─ Cost of miss: 100+ GPU cycles
  
Performance:
  GGTT: Stable TLB hit rate
  PPGTT: TLB thrashing on context switch
  
Optimization: ASID (Address Space ID)
  ├─ GPU tags TLB entries with ASID
  ├─ Old ASID entries kept (not invalidated)
  ├─ Switch ASID, skip TLB flush
  ├─ Reduces TLB misses
  └─ Modern GPUs (Gen8+): ASID support
```

### Memory Fragmentation

```
GGTT fragmentation (256MB total):

Initial: [Free  256MB]

After allocation:
  Allocation 1:  64MB → [Used 64MB] [Free 192MB]
  Allocation 2:  32MB → [Used 64MB] [Used 32MB] [Free 160MB]
  Allocation 3:  64MB → [Used 64MB] [Used 32MB] [Used 64MB] [Free 96MB]
  Dealloc 2:     32MB → [Used 64MB] [Free 32MB] [Used 64MB] [Free 96MB]
  Allocation 4:  48MB → [Used 64MB] [Free 32MB] [Used 64MB] [Used 48MB] [Free 48MB]

Result:
  • Fragmented address space
  • Cannot allocate 96MB even though 96MB free (non-contiguous)
  • Compaction/defragmentation needed
  • Limits maximum allocation size

PPGTT (48-bit, 256TB per context):
  ├─ Huge address space
  ├─ Fragmentation unlikely
  ├─ Sparse allocation support
  └─ No contiguity requirement
```

### Context Switch Overhead

```
Measuring context switch cost:

Lightweight switch:
  1. GPU finishes current instruction (0-1000 cycles)
  2. Swap PPGTT base register (1-10 cycles)
  3. Load context state (10-100 cycles)
  4. Resume execution (1 cycle)
  
  Total: ~1000 cycles (1-2 microseconds)

With TLB flush:
  1. GPU finishes (0-1000 cycles)
  2. Flush all TLB entries (100-1000 cycles)
  3. Swap PPGTT base register (1-10 cycles)
  4. Load context state (10-100 cycles)
  5. Resume execution (1 cycle)
  6. TLB misses now: 100+ cycles each (first ~100 mem accesses)
  
  Total: ~2000+ cycles + workload-dependent

With ASID (no TLB flush):
  1. GPU finishes (0-1000 cycles)
  2. Swap PPGTT base + ASID register (1-10 cycles)
  3. Load context state (10-100 cycles)
  4. Resume execution (1 cycle)
  5. TLB entries kept (same ASID) - high hit rate
  
  Total: ~1000 cycles (same as lightweight)

Result:
  Without ASID: Context switch = ~2-4 microseconds
  With ASID: Context switch = ~1-2 microseconds
```

---

## Hardware Constraints

### GPU Generation Support

```
GGTT Support:
  Gen2 (i915):    8 MB     ← Very limited
  Gen3-4:         32-128 MB
  Gen5+:          512 MB   ← Common
  Gen7+:          1-2 GB   ← Modern

PPGTT Support:
  Gen4 and earlier: No PPGTT
  Gen5:            Experimental
  Gen6:            Optional
  Gen7+:           Required (context switching)
  
  Levels (page table depth):
  Gen6-7:          2-level PPGTT
  Gen8+:           4-level PPGTT (48-bit addressing)
```

### Hardware Limits

```
GGTT Limits:
  ├─ Max size: Determined by hardware TLB size
  │           Generally: 256MB-2GB
  ├─ Max entry size: Hardware register width (32 or 64-bit)
  ├─ Max PTE format: Version-specific
  └─ Max concurrent mappings: TLB size

PPGTT Limits:
  ├─ Max address space: 48-bit (256TB) on modern GPUs
  ├─ Max page levels: 4 (for 48-bit addresses)
  ├─ Max page table size: Depends on page size (4KB typical)
  ├─ Max per-context: 256TB per GPU context
  └─ Concurrent contexts: ~32-1024 (hardware-dependent)
```

### Register Layout

```
GGTT Mapping Register:

┌─────────────────────────────────────┐
│  GGTT PTE (Page Table Entry)        │
├─────────────────────────────────────┤
│ Bits 63-12: Physical Address (40 bits)
│ Bits 11-8:  Cache Control
│ Bit 7:      Snoop Control
│ Bit 6:      (reserved)
│ Bit 5:      Valid
│ Bit 4:      Writeable
│ Bit 3-0:    (reserved)
└─────────────────────────────────────┘

PPGTT PTE (similar but context-specific):

┌─────────────────────────────────────┐
│  PPGTT PTE (Page Table Entry)       │
├─────────────────────────────────────┤
│ Bits 63-12: Physical Address (40 bits)
│ Bits 11-8:  Cache Control
│ Bit 7:      Snoop Control
│ Bit 6:      Dirty (write-tracking)
│ Bit 5:      Valid
│ Bit 4:      Writeable
│ Bit 3:      Execute (for code)
│ Bit 2-0:    (reserved)
└─────────────────────────────────────┘
```

---

## Debugging GGTT/PPGTT Issues

### Common Issues & Diagnosis

#### Issue 1: GGTT Exhaustion

```
Symptom:
  └─ GPU operation fails: "No space in GGTT"
  └─ Memory allocation errors
  
Cause:
  ├─ Too many GGTT mappings
  ├─ GGTT not cleaned up
  ├─ Memory leaks (unbound VMAs)
  └─ Fragmentation (cannot allocate contiguously)
  
Diagnosis:
  # Check GGTT usage
  cat /sys/kernel/debug/dri/0/i915_ggtt_bindings
  
  # View detailed GGTT map
  cat /sys/kernel/debug/dri/0/i915_gem_objects
  
  # Check fragmentation
  cat /sys/kernel/debug/dri/0/i915_memory_regions
  
Fix:
  ├─ Unbind unused objects
  ├─ Defragment GGTT
  ├─ Reduce peak GGTT usage
  ├─ Use PPGTT for large objects
  └─ Increase GGTT if possible (BIOS setting)
```

#### Issue 2: PPGTT Context Corruption

```
Symptom:
  └─ GPU hang or wrong results
  └─ Memory corruption in app
  └─ Random crashes
  
Cause:
  ├─ Page table walk error
  ├─ Incorrect PTE setup
  ├─ Dangling PPGTT references
  ├─ Context switch issue
  └─ TLB invalidation problem
  
Diagnosis:
  # Check context state
  cat /sys/kernel/debug/dri/0/i915_contexts
  
  # View PPGTT for context N
  cat /sys/kernel/debug/dri/0/i915_ppgtt_N
  
  # Enable GGTT/PPGTT tracing
  echo 1 > /sys/kernel/debug/dri/0/i915_gem_debug
  
  # Check HW error logs
  cat /proc/i915_error_state
  
Fix:
  ├─ Verify PTE programming
  ├─ Check page table alignment
  ├─ Ensure valid physical addresses
  ├─ Test context switch logic
  └─ Validate TLB invalidation
```

#### Issue 3: TLB Thrashing

```
Symptom:
  └─ GPU performance degradation after context switch
  └─ High TLB miss rate
  └─ Reduced frame rate
  
Cause:
  ├─ TLB flushed on context switch
  ├─ Large working set (many unique VA)
  ├─ No ASID support (older GPU)
  ├─ Tiny TLB (older GPU)
  └─ Frequent context switches
  
Diagnosis:
  # Monitor TLB miss rate
  perf stat -e gpu_mem/tlb_miss/ ./workload
  
  # Check GPU generation (ASID support?)
  cat /proc/cpuinfo | grep "gpu"
  
  # Profile context switches
  perf record -e gpu_ctx_switch -g ./workload
  perf report
  
Fix:
  ├─ Reduce working set (minimize VA)
  ├─ Use sequential access patterns
  ├─ Group related data (improve locality)
  ├─ Reduce context switch frequency
  ├─ Enable ASID if available (GPU driver setting)
  └─ Upgrade GPU hardware
```

### Debug Commands

```bash
# View GGTT bindings
cat /sys/kernel/debug/dri/0/i915_ggtt_bindings

# View all gem objects
cat /sys/kernel/debug/dri/0/i915_gem_objects

# View contexts
cat /sys/kernel/debug/dri/0/i915_contexts

# View error state
cat /proc/i915_error_state

# Enable detailed logging
echo 0xFF > /sys/module/drm_i915/parameters/debug

# Monitor in real-time
watch -n1 'cat /sys/kernel/debug/dri/0/i915_ggtt_bindings | tail -20'

# Analyze GPU hangs
dmesg | grep -i "gpu hang"
```

### Performance Monitoring

```bash
# Monitor context switches
perf stat -e gpu_ctx_switch,gpu_ctx_flush,tlb_miss ./workload

# Profile GGTT/PPGTT operations
perf record -e gpu_mem,vm_ops -g ./workload
perf report

# Measure context switch overhead
./intel_gpu_top  # Real-time GPU metrics

# Trace page table modifications
perf trace -e "gpu_vm:*" ./workload
```

---

## Summary Table

```
┌──────────────────┬─────────────────────┬──────────────────────┐
│ Characteristic   │ GGTT                │ PPGTT                │
├──────────────────┼─────────────────────┼──────────────────────┤
│ Scope            │ Global (all GPU)    │ Per-context (private)│
│ Size             │ 256MB-2GB           │ 48-bit (256TB)       │
│ Visibility       │ All contexts see    │ Only owner context   │
│ Primary Use      │ Kernel/firmware     │ Applications         │
│ Security         │ None (shared)       │ Complete isolation   │
│ Fragmentation    │ High risk           │ Minimal              │
│ Context Switch   │ No action needed    │ Base register change │
│ Contention       │ High (shared)       │ None (private)       │
│ Example          │ Status pages        │ App textures/buffers │
│ Creation         │ During driver init  │ Per GPU context      │
│ Lifetime         │ Driver lifetime     │ Context lifetime     │
└──────────────────┴─────────────────────┴──────────────────────┘
```

---

**End of GGTT/PPGTT Deep Dive Document**
