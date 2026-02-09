# User-Space Interface and UAPI

**Document ID:** 15 | **Status:** Complete | **Last Updated:** February 2026

---

## Executive Summary

The user-space API (UAPI) is the fundamental interface between applications and the i915 GPU driver. This document covers GEM objects, execbuf submissions, contexts, and synchronization primitives including the high-performance User Fence mechanism.

### Key Topics
- **GEM API:** Memory object creation and management
- **Execution submission:** Batch buffer submission, dependency handling
- **Contexts:** GPU context creation, isolation, performance tracking
- **Synchronization:** Traditional fences, syncobj, User Fence (ufence) with sub-microsecond latency

### Performance Metrics
- **ioctl latency:** 100-500µs for simple operations
- **Batch submission:** 10-50µs per submission
- **Context switch:** 1-5µs overhead

---

## Table of Contents

1. [UAPI Architecture](#uapi-architecture)
2. [GEM Object Management](#gem-object-management)
3. [Batch Buffer Submission](#batch-buffer-submission)
4. [Context Management](#context-management)
5. [Synchronization Primitives](#synchronization-primitives)
6. [Error Handling](#error-handling)
7. [Performance Optimization](#performance-optimization)
8. [Summary & Best Practices](#summary--best-practices)

---

## UAPI Architecture

### User-Kernel Interface Stack

```plaintext
User Space Application
    │
    ├─ libdrm (abstraction layer)
    │  ├─ DRM Device Handle
    │  ├─ GEM Object IDs
    │  └─ Context Handles
    │
    ├─ Kernel UAPI
    │  ├─ DRM_IOCTL_I915_GEM_CREATE
    │  ├─ DRM_IOCTL_I915_GEM_EXECBUFFER2
    │  ├─ DRM_IOCTL_I915_GEM_CONTEXT_CREATE
    │  └─ DRM_IOCTL_I915_GEM_WAIT
    │
    └─ i915 Driver
       └─ GPU Hardware
```

### UAPI Version Management

```c
// UAPI version tracking
#define I915_UAPI_VERSION_MAJOR 2
#define I915_UAPI_VERSION_MINOR 0

// Query driver capabilities
struct drm_i915_query_topology_info {
    __u16 flags;
    __u16 max_slices;
    __u16 max_subslices;
    __u16 max_eus_per_subslice;
    // ...
};

// Check feature support
static int query_driver_capabilities(int fd)
{
    struct drm_i915_query query = {0};
    struct drm_i915_query_topology_info *topo;
    
    query.query_id = I915_QUERY_TOPOLOGY_INFO;
    
    if (ioctl(fd, DRM_IOCTL_I915_QUERY, &query) < 0) {
        perror("Query failed");
        return -1;
    }
    
    return 0;
}
```

---

## GEM Object Management

### GEM Create and Memory Binding

```plaintext
GEM Object Lifecycle
┌──────────────────────────────────┐
│ 1. Create (DRM_IOCTL_I915_GEM_CREATE)
│    └─ Allocate system RAM pages
└──────┬───────────────────────────┘
       │
       ├─→ 2. Map (mmap or mmap from ioctl)
       │   └─ Virtual address in user space
       │
       ├─→ 3. Bind to GGTT (GPU Global TLB)
       │   ├─ Allocate GGTT entries
       │   └─ Enable GPU access
       │
       ├─→ 4. Execute GPU Commands
       │   └─ GPU reads/writes via GGTT
       │
       ├─→ 5. Unmap (if using user mmap)
       │   └─ Release virtual address
       │
       ├─→ 6. Unbind (if needed)
       │   └─ Remove GGTT mapping
       │
       └─→ 7. Destroy (close DRM device)
           └─ Release all resources
```

### GEM Create Implementation

```c
// User-space GEM creation
struct drm_i915_gem_create {
    __u64 size;           // Requested size in bytes
    __u32 handle;         // Output: GEM handle
    __u32 pad;
};

// Kernel-side GEM create ioctl
static int i915_gem_create_ioctl(struct drm_device *dev,
                                 void *data,
                                 struct drm_file *file)
{
    struct drm_i915_gem_create *args = data;
    struct drm_i915_private *i915 = to_i915(dev);
    struct i915_gem_object *obj;
    int ret;
    
    // 1. Validate size
    if (args->size == 0 || args->size > 1ull << 40) {
        return -EINVAL;
    }
    
    // 2. Create GEM object
    obj = i915_gem_object_create_shmem(i915, args->size);
    if (IS_ERR(obj)) {
        return PTR_ERR(obj);
    }
    
    // 3. Create handle for user space
    ret = drm_gem_handle_create(file, &obj->base, &args->handle);
    
    i915_gem_object_put(obj);
    
    return ret;
}

// User-space example
int create_gem_buffer(int fd, unsigned long size, uint32_t *handle)
{
    struct drm_i915_gem_create create = {
        .size = size,
    };
    
    if (drmIoctl(fd, DRM_IOCTL_I915_GEM_CREATE, &create)) {
        return -1;
    }
    
    *handle = create.handle;
    return 0;
}
```

### GEM Memory Type Control

```c
// Control memory domain/cache behavior
struct drm_i915_gem_caching {
    __u32 handle;        // GEM handle
    __u32 caching;       // Caching mode
};

// Caching modes
#define I915_CACHING_NONE        0  // Uncached
#define I915_CACHING_CACHED      1  // CPU cache enabled
#define I915_CACHING_DISPLAY     2  // Optimized for display

// Set caching policy
static int gem_set_caching(int fd, uint32_t handle, uint32_t caching)
{
    struct drm_i915_gem_caching cache = {
        .handle = handle,
        .caching = caching,
    };
    
    return drmIoctl(fd, DRM_IOCTL_I915_GEM_SET_CACHING, &cache);
}
```

---

## Batch Buffer Submission

### Execbuf Structure

```c
// Batch buffer execution structure
struct drm_i915_gem_execbuffer2 {
    __u32 buffers_ptr;         // Pointer to exec object array
    __u32 buffer_count;        // Number of objects
    __u32 batch_start_offset;  // Start offset in batch buffer
    __u32 batch_len;           // Batch buffer length
    __u32 DR1;                 // HW context (reserved)
    __u32 DR4;                 // HW context (reserved)
    __u32 num_cliprects;       // Clip rectangle count
    __u32 cliprects_ptr;       // Pointer to clip rects
    __u32 flags;               // Execution flags
    __u32 rsvd1;               // Reserved
    __u64 rsvd2;               // Reserved
};

// Exec object - describes a buffer involved in execution
struct drm_i915_gem_exec_object2 {
    __u32 handle;              // GEM handle
    __u32 relocation_count;    // Number of relocations
    __u64 relocs_ptr;          // Pointer to relocations
    __u64 alignment;           // Required alignment
    __u64 offset;              // GPU virtual address
    __u64 flags;               // Object flags
    __u64 rsvd1;               // Reserved
    __u64 rsvd2;               // Reserved
};

// Relocation - fix up addresses in batch buffer
struct drm_i915_gem_relocation_entry {
    __u32 target_handle;       // Target GEM object
    __u32 delta;               // Offset within target
    __u64 offset;              // Offset in batch buffer
    __u64 presumed_offset;     // Pre-computed offset
    __u32 read_domains;        // Read domain requirements
    __u32 write_domain;        // Write domain requirement
};
```

### Batch Submission Flow

```plaintext
User-Space Batch Submission
┌──────────────────────────────────┐
│ Create Batch Buffer              │
│ (GEM object with GPU commands)    │
└──────┬───────────────────────────┘
       │
       ├─→ Create Exec Objects
       │   └─ List all buffers needed
       │
       ├─→ Setup Relocations
       │   └─ Fix GPU address references
       │
       ├─→ Call execbuf ioctl
       │   ├─ Kernel validates
       │   ├─ Allocates GPU memory
       │   ├─ Submits to GPU
       │   └─ Returns fence for sync
       │
       └─→ Wait for Completion
           ├─ Poll fence or ioctl
           └─ Access results
```

### Kernel Execbuf Handler

```c
// Handle batch submission ioctl
static int i915_gem_execbuffer2_ioctl(struct drm_device *dev,
                                      void *data,
                                      struct drm_file *file)
{
    struct drm_i915_gem_execbuffer2 *args = data;
    struct drm_i915_private *i915 = to_i915(dev);
    struct i915_execbuffer eb = {0};
    int ret;
    
    // 1. Validate input
    if (args->buffer_count > MAX_EXEC_OBJECTS)
        return -EINVAL;
    
    // 2. Parse exec objects
    ret = i915_gem_do_execbuffer(dev, data, file, &eb);
    if (ret)
        return ret;
    
    // 3. Validate batch buffer
    ret = i915_execbuffer_parse_batch(eb.batch,
                                      eb.batch_len);
    if (ret)
        return ret;
    
    // 4. Allocate GPU memory for buffers
    ret = i915_gem_exec_reserve(&eb);
    if (ret)
        return ret;
    
    // 5. Apply relocations
    ret = i915_gem_apply_relocations(&eb);
    if (ret)
        goto out_unreserve;
    
    // 6. Build submission request
    ret = i915_gem_do_exec_request(&eb);
    if (ret)
        goto out_unreserve;
    
out_unreserve:
    i915_gem_exec_unreserve(&eb);
    
    return ret;
}
```

---

## Context Management

### Context Creation

```plaintext
GPU Context Hierarchy
┌──────────────────────────────┐
│ Device Context               │
│ (driver-level state)         │
└────────┬─────────────────────┘
         │
         ├─→ Process Context
         │   └─ Per-application state
         │
         ├─→ GPU Context #0
         │   ├─ Engine state
         │   ├─ Page tables
         │   ├─ Register state
         │   └─ Completion status
         │
         ├─→ GPU Context #1
         │   └─ [Similar to #0]
         │
         └─→ GPU Context #N
             └─ [Similar to #0]
```

### Context Isolation

```c
// GPU context - isolated GPU state per context
struct intel_context {
    struct kref ref;
    
    // Engine and address space
    struct intel_engine_cs *engine;
    struct i915_address_space *vm;
    
    // Ring for this context
    struct intel_ring *ring;
    
    // Execution state
    u32 lrc_reg_state[LRC_STATE_PN];  // Local register context
    
    // Scheduling state
    struct list_head link;  // Scheduler queue
    bool active;
};

// Create GPU context
struct drm_i915_gem_context_create {
    __u32 ctx_id;     // Output: context ID
    __u32 flags;      // Creation flags
};

// Kernel context create ioctl
static int i915_gem_context_create_ioctl(struct drm_device *dev,
                                         void *data,
                                         struct drm_file *file)
{
    struct drm_i915_gem_context_create *args = data;
    struct drm_i915_private *i915 = to_i915(dev);
    struct i915_gem_context *ctx;
    int ret;
    
    // 1. Create context
    ctx = i915_gem_context_create(i915);
    if (IS_ERR(ctx))
        return PTR_ERR(ctx);
    
    // 2. Assign ID
    ret = idr_alloc(&i915->contexts.idr, ctx,
                    0, 0, GFP_KERNEL);
    if (ret < 0)
        goto out_ctx_put;
    
    args->ctx_id = ret;
    
    // 3. Return handle to user space
    return 0;
    
out_ctx_put:
    i915_gem_context_put(ctx);
    return ret;
}
```

### Context Parameter Configuration

```c
// Query/set context parameters
struct drm_i915_gem_context_param {
    __u32 ctx_id;      // Context ID
    __u32 param;       // Parameter type
    __u64 value;       // Parameter value
    __u64 reserved;    // Reserved
};

// Parameter types
#define I915_CONTEXT_PARAM_VM              0  // Address space
#define I915_CONTEXT_PARAM_PRIORITY        1  // Priority (-1023 to +1023)
#define I915_CONTEXT_PARAM_BANNABLE        2  // Can be banned
#define I915_CONTEXT_PARAM_NO_ERROR_CAPTURE 3 // Skip error recording
#define I915_CONTEXT_PARAM_PERSISTENCE    4  // Keep context on hang

// Set high priority for real-time context
static int set_context_priority(int fd, uint32_t ctx_id, int priority)
{
    struct drm_i915_gem_context_param param = {
        .ctx_id = ctx_id,
        .param = I915_CONTEXT_PARAM_PRIORITY,
        .value = priority,
    };
    
    return drmIoctl(fd, DRM_IOCTL_I915_GEM_CONTEXT_SETPARAM, &param);
}
```

---

## Synchronization Primitives

### Fence Wait

```c
// Wait for GPU fence completion
struct drm_i915_gem_wait {
    __u32 bo_handle;        // GEM object handle
    __u32 flags;            // Wait flags
    __s64 timeout_ns;       // Timeout in nanoseconds (-1 = infinite)
};

// Wait flags
#define I915_WAIT_IOCTL_EXEC_QUEUE    (1 << 0)  // Wait for exec queue
#define I915_WAIT_IOCTL_GEM_SEQNO     (1 << 1)  // Wait for seqno

// User-space wait example
int wait_for_completion(int fd, uint32_t handle, int timeout_ms)
{
    struct drm_i915_gem_wait wait = {
        .bo_handle = handle,
        .timeout_ns = timeout_ms * 1000000LL,
    };
    
    int ret = drmIoctl(fd, DRM_IOCTL_I915_GEM_WAIT, &wait);
    
    if (ret == 0) {
        // Object has been read from GPU
        return 0;
    } else if (errno == ETIME) {
        // Timeout occurred
        return -ETIMEDOUT;
    }
    
    return -1;
}
```

### Syncobj - Modern Synchronization

```c
// Synchronization object - replaces legacy fences
struct drm_syncobj {
    struct kref refcount;
    struct dma_fence *fence;
    struct list_head cb_list;  // Completion callbacks
};

// Create and use syncobj
struct drm_i915_gem_execbuffer_ext_batch_dependencies {
    __u64 dependency_count;
    __u64 dependencies_ptr;  // Array of syncobj handles
};

// User-space example with syncobj
int submit_with_syncobj_deps(int fd, uint32_t ctx_id,
                              uint32_t *dep_syncobjs,
                              int dep_count)
{
    struct drm_i915_gem_exec_object2 objs[1];
    struct drm_i915_gem_execbuffer_ext_batch_dependencies ext = {
        .dependency_count = dep_count,
        .dependencies_ptr = (uintptr_t)dep_syncobjs,
    };
    
    struct drm_i915_gem_execbuffer2_ext exec = {
        .base = {
            .buffers_ptr = (uintptr_t)objs,
            .buffer_count = 1,
            .batch_start_offset = 0,
            .batch_len = 16,
        },
        .extensions = (uintptr_t)&ext,
    };
    
    return drmIoctl(fd, DRM_IOCTL_I915_GEM_EXECBUFFER2_EXT, &exec);
}
```

### User Fence (ufence) - Low-Latency Synchronization

User Fence is a high-performance synchronization primitive that allows GPU completion notifications directly to user-space memory without system calls, achieving sub-microsecond latency.

**Architecture:**
```
User Fence Notification Stack
┌──────────────────────────┐
│ User Fence (ufence)      │  ← Application-level API
│ ├─ VM Bind notification  │
│ ├─ Execbuffer completion │
│ └─ Wait conditional      │
└────────────┬─────────────┘
             │
┌────────────▼──────────────┐
│ dma_fence_work           │  ← Work queue wrapper
│ └─ Async task execution  │
└────────────┬──────────────┘
             │
┌────────────▼──────────────┐
│ Linux dma_fence          │  ← Kernel sync primitive
│ ├─ DMA ops               │
│ ├─ Completion callbacks  │
│ └─ Reference counting    │
└──────────────────────────┘
```

**Use Cases:**

1. **VM Bind Notifications** - Async VM bind operations notify completion via ufence
2. **Execbuffer Completion** - Batch buffer submission completion signals user fence
3. **Conditional Wait** - Wait with operators (EQ, NEQ, GT, GTE, LT, LTE) on memory value
4. **TLB Synchronization** - Direct TLB shootdown notifications

**UAPI Structures:**

```c
// User fence descriptor - passed in execbuffer flags/extensions
struct drm_i915_user_fence {
    __u32 handle;           // GEM handle to user-allocated buffer
    __u32 offset;           // Offset within buffer (must be 8-byte aligned)
};

// User fence value structure in memory
struct i915_user_fence_value {
    __u64 value;            // Atomic value for comparison
    __u64 timestamp;        // Optional: completion timestamp
};

// VM Bind with user fence
struct drm_i915_gem_vm_bind {
    __u64 vm_id;
    __u64 start;            // VM address
    __u64 length;           // Mapping length
    __u64 offset;           // Object offset
    __u32 flags;
    __u32 extensions;       // Extended via batch_buffer_reloc
};

// User fence wait IOCTL
struct drm_i915_gem_wait_user_fence {
    __u64 vm_id;            // Address space context
    __u64 fence_addr;       // GPA of fence value (PPGTT)
    __u64 fence_value;      // Expected value to wait for
    __u32 op;               // Operation: EQ, NEQ, GT, GTE, LT, LTE
    __u32 flags;            // I915_WAIT_NOHANG, I915_WAIT_ABSOLUTE
    __s64 timeout_ns;       // Timeout in nanoseconds
};

// Wait operation codes
#define DRM_I915_USER_FENCE_EQ   0  // Wait until fence == value
#define DRM_I915_USER_FENCE_NEQ  1  // Wait until fence != value
#define DRM_I915_USER_FENCE_GT   2  // Wait until fence > value
#define DRM_I915_USER_FENCE_GTE  3  // Wait until fence >= value
#define DRM_I915_USER_FENCE_LT   4  // Wait until fence < value
#define DRM_I915_USER_FENCE_LTE  5  // Wait until fence <= value
```

**Execution Modes:**

| Mode | Description | Latency | Use Case |
|------|-------------|---------|----------|
| `ufence_page_ops` | Direct page write (fast) | ~100ns | Standard operations |
| `ufence_mm_ops` | Memory management ops | ~500ns | Complex memory states |
| `ufence_nops` | No-op (error/disabled) | N/A | Testing/fallback |

**Performance Characteristics:**

- **Notification Latency:** ~100ns (atomic memory write)
- **No System Call:** Direct user-space memory write on GPU completion
- **Conditional Waiting:** Hardware-assisted value comparison
- **TLB Aware:** Integrated TLB synchronization

**User-Space Example:**

```c
// 1. Allocate user fence buffer
uint32_t fence_handle;
struct drm_i915_gem_create create = {
    .size = 4096,  // One page for fence values
};
drmIoctl(fd, DRM_IOCTL_I915_GEM_CREATE, &create);
fence_handle = create.handle;

// 2. Map fence buffer to user space
void *fence_addr = mmap(NULL, 4096, PROT_READ | PROT_WRITE,
                        MAP_SHARED, fd, fence_handle << 12);

// 3. Prepare VM bind with user fence
struct drm_i915_gem_vm_bind vm_bind = {
    .vm_id = my_vm_id,
    .start = 0x1000000,
    .length = 0x10000,
    .offset = 0,
    .flags = I915_GEM_VM_BIND_ASYNC,
    // User fence notification setup would be in extensions
};

// 4. Execute batch with user fence notification
struct drm_i915_gem_execbuffer2 exec = {
    .buffers_ptr = (uintptr_t)objects,
    .buffer_count = 1,
    .batch_start_offset = 0,
    .batch_len = batch_size,
    // User fence for completion notification
};
drmIoctl(fd, DRM_IOCTL_I915_GEM_EXECBUFFER2, &exec);

// 5. Wait for completion with conditional check
struct drm_i915_gem_wait_user_fence wait = {
    .vm_id = my_vm_id,
    .fence_addr = (uintptr_t)fence_addr + fence_offset,
    .fence_value = expected_completion_value,
    .op = DRM_I915_USER_FENCE_EQ,
    .flags = 0,
    .timeout_ns = 1000000000LL,  // 1 second
};
int ret = drmIoctl(fd, DRM_IOCTL_I915_GEM_WAIT_USER_FENCE, &wait);

if (ret == 0) {
    printf("Execution completed!\n");
} else if (errno == ETIME) {
    printf("Timeout waiting for completion\n");
}
```

**Implementation Details:**

```c
// Kernel-side: i915_gem_vm_bind_object.c (simplified)
int i915_gem_vm_bind_with_ufence(struct drm_i915_gem_vm_bind *args)
{
    struct i915_user_fence *ufence = extract_ufence(args);
    struct dma_fence_work *work;
    
    // 1. Create async bind operation
    work = i915_gem_vm_bind_create_work(args);
    
    // 2. Attach user fence for completion
    if (ufence) {
        attach_user_fence_to_work(work, ufence);
    }
    
    // 3. Queue for async execution
    queue_work(work);
    
    // 4. On completion: dma_fence_work triggers user fence write
    //    - GPU writes to user-allocated buffer at specified offset
    //    - User space reads value without system call (polling/wait)
    
    return 0;
}

// User fence notification callback
static void ufence_notify(struct dma_fence_work *work)
{
    struct user_fence *ufence = work->ufence;
    
    // 1. Get user page (already pinned during setup)
    struct page *page = ufence->page;
    
    // 2. Write completion value atomically
    atomic64_set((atomic64_t *)(page_address(page) + ufence->offset),
                 ufence->value);
    
    // 3. TLB shootdown if needed
    if (ufence->needs_tlb_sync)
        i915_tlb_invalidate(ufence->vm);
}
```

**Key Advantages Over Traditional Fences:**

1. **Sub-microsecond Latency** - No system call overhead
2. **Conditional Operations** - Hardware-assisted comparisons
3. **Batch Optimizations** - Multiple fences in single operation
4. **Memory Efficient** - Reuse allocated buffers
5. **GPU-Aware** - Integrated with TLB synchronization

**Limitations & Considerations:**

- User buffer must be pinned during operation
- Address must be within user's allocated PPGTT
- 8-byte alignment requirement
- Timeout handling varies by operation
- Not suitable for inter-process synchronization

---

## Error Handling

### Common UAPI Errors

```plaintext
UAPI Error Codes
┌─────────────────────────────────────┐
│ -EINVAL (22)                        │
│ ├─ Invalid arguments                │
│ ├─ Invalid handle                   │
│ └─ Invalid operation                │
│                                     │
│ -ENOENT (2)                         │
│ ├─ Handle not found                 │
│ └─ Operation not supported          │
│                                     │
│ -ENOMEM (12)                        │
│ ├─ Out of memory                    │
│ ├─ GPU memory exhausted             │
│ └─ Buffer too large                 │
│                                     │
│ -EFAULT (14)                        │
│ ├─ Bad user-space pointer           │
│ └─ Copy from/to user failed         │
│                                     │
│ -EIO (5)                            │
│ ├─ GPU error                        │
│ ├─ GPU hang                         │
│ └─ Device in error state            │
└─────────────────────────────────────┘
```

### Error Handling in User Space

```c
// Robust execbuf with error handling
int safe_execbuf(int fd, struct drm_i915_gem_execbuffer2 *exec)
{
    int ret = drmIoctl(fd, DRM_IOCTL_I915_GEM_EXECBUFFER2, exec);
    
    if (ret == -1) {
        switch (errno) {
        case EINVAL:
            fprintf(stderr, "Invalid execbuf arguments\n");
            return -EINVAL;
        
        case EIO:
            fprintf(stderr, "GPU error detected\n");
            // Recover or retry
            return -EIO;
        
        case ENOMEM:
            fprintf(stderr, "GPU memory exhausted\n");
            // Reduce workload or free memory
            return -ENOMEM;
        
        case EFAULT:
            fprintf(stderr, "Bad user-space pointer\n");
            return -EFAULT;
        
        default:
            fprintf(stderr, "Unexpected error: %s\n",
                    strerror(errno));
            return -1;
        }
    }
    
    return 0;
}
```

---

## Performance Optimization

### Batch Buffer Optimization

```c
// Reduce ioctl overhead with large batch submission
void optimize_batch_submission(int fd)
{
    struct drm_i915_gem_execbuffer2 exec;
    struct drm_i915_gem_exec_object2 objs[MAX_OBJS];
    
    // 1. Use large batch sizes
    exec.batch_len = 16 * 1024;  // 16KB batch (more efficient)
    
    // 2. Minimize relocations
    // Pre-allocate and reuse memory regions
    
    // 3. Batch multiple submits
    // Don't call execbuf for every job - batch them
    
    // 4. Use persistent mappings
    // Keep GPU memory persistently mapped when possible
}
```

### Context Caching

```c
// Reuse contexts to avoid creation overhead
struct cached_contexts {
    uint32_t high_priority_ctx;
    uint32_t normal_ctx;
    uint32_t low_priority_ctx;
};

void init_cached_contexts(int fd, struct cached_contexts *cache)
{
    struct drm_i915_gem_context_create create = {0};
    
    // Create reusable contexts
    drmIoctl(fd, DRM_IOCTL_I915_GEM_CONTEXT_CREATE, &create);
    cache->high_priority_ctx = create.ctx_id;
    
    drmIoctl(fd, DRM_IOCTL_I915_GEM_CONTEXT_CREATE, &create);
    cache->normal_ctx = create.ctx_id;
    
    drmIoctl(fd, DRM_IOCTL_I915_GEM_CONTEXT_CREATE, &create);
    cache->low_priority_ctx = create.ctx_id;
    
    // Set priorities
    struct drm_i915_gem_context_param param = {
        .param = I915_CONTEXT_PARAM_PRIORITY,
    };
    
    param.ctx_id = cache->high_priority_ctx;
    param.value = 512;
    drmIoctl(fd, DRM_IOCTL_I915_GEM_CONTEXT_SETPARAM, &param);
    
    param.ctx_id = cache->low_priority_ctx;
    param.value = -512;
    drmIoctl(fd, DRM_IOCTL_I915_GEM_CONTEXT_SETPARAM, &param);
}
```

---

## Summary & Best Practices

### Key Takeaways

1. **Handle lifecycle:** Create once, reuse multiple times
2. **Batch operations:** Minimize ioctl calls
3. **Async operations:** Use fences for synchronization
4. **Memory efficiency:** Reuse buffers when possible
5. **Error handling:** Always check ioctl return codes

### Best Practices

**For Application Design:**
- Cache contexts and GEM objects
- Use async wait with timeouts
- Batch multiple submits
- Monitor memory usage
- Handle -EINVAL gracefully (driver update)

**For Performance:**
- Minimize ioctl frequency
- Use large batch buffers (reduce overhead)
- Pre-allocate and reuse memory
- Use proper priorities for latency-critical work
- Profile ioctl latency

**For Reliability:**
- Validate all ioctl arguments
- Handle transient errors with retry
- Monitor GPU memory usage
- Detect GPU hangs early
- Test error paths

---

## References

- [UAPI Documentation](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/include/uapi/drm/i915_drm.h)
- [GEM Execution](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/i915/gem)
- Related: [03-Context-Management.md](03-Context-Management.md), [06-Ring-Buffer-Management.md](06-Ring-Buffer-Management.md)

---

**Next Steps:**
- Study libdrm wrapper implementations
- Profile ioctl overhead on real workloads
- Implement context pooling for optimization
- Test error handling with stress tests

---

**Document Status:** ✅ Complete  
**Version:** 1.0  
**Last Updated:** February 8, 2026
