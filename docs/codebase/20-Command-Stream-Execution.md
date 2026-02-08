# 20. Command Stream Execution

## Executive Summary

Command stream execution is the core mechanism by which the i915 GPU driver submits work to the hardware. This document covers the complete flow from user-space batch buffer submission through command parsing, ring management, and hardware execution. Understanding command streams is essential for optimizing GPU workload submission, debugging submission issues, and understanding the driver's architecture.

**Key Concepts:**
- Batch buffer submission and ring management
- Command stream parsing and validation
- Ring buffer architecture and pointer management
- Semaphore and pipeline synchronization
- Error handling and command validation

**Table of Contents**
1. [Command Stream Architecture](#architecture)
2. [Batch Buffer Submission](#batch-submission)
3. [Ring Buffer Management](#ring-management)
4. [Command Parsing and Validation](#parsing-validation)
5. [Pipeline Control and Synchronization](#synchronization)
6. [Error Handling and Recovery](#error-handling)
7. [Performance Optimization](#optimization)
8. [Debugging Command Streams](#debugging)
9. [Best Practices](#best-practices)
10. [References](#references)

---

## Architecture

### Command Submission Flow

```
User Space (libdrm)
    |
    v
Execbuf IOCTL
    |
    v
Batch Buffer Validation
    |
    v
Ring Buffer Update (pointer advance)
    |
    v
Hardware Execution
    |
    v
Context Switch / Interrupt Handling
```

The command stream execution pipeline involves multiple stages:

1. **User-Space Submission**: Applications submit work via execbuf IOCTL with GEM object references
2. **Kernel Validation**: Commands are validated and dependencies resolved
3. **Ring Buffer Update**: Validated commands are placed in the ring buffer
4. **Hardware Fetch**: GPU fetches commands from the ring at its own pace
5. **Execution and Completion**: GPU executes commands and generates completion signals

### Ring Buffer Architecture

The ring buffer is a circular queue that stores GPU commands:

```
Ring Buffer:
┌─────────────────────────────────┐
│  Head Pointer (GPU position)    │
│  Tail Pointer (Driver position) │
│  Wrap-around at size boundary   │
└─────────────────────────────────┘

Space calculation:
  free_space = (head - tail) % ring_size
  Can submit if command_size <= free_space
```

**Key Ring Parameters:**
- **Ring Size**: Typically 1-2 MB, must be power-of-2
- **Head Pointer**: GPU-maintained position (read-only from driver)
- **Tail Pointer**: Driver-maintained position (write-once to GPU)
- **Seqno/Timeline**: Per-context execution order tracking

---

## Batch Buffer Submission

### Execbuf IOCTL Flow

```c
// User space submission
struct drm_i915_gem_execbuffer2 exec_params = {
    .buffers_ptr = (uint64_t)&exec_objects,
    .buffer_count = obj_count,
    .batch_start_offset = 0,
    .batch_len = batch_size,
    .DR1 = 0,  /* Ring DW0 */
    .DR4 = 0,  /* Ring DW4 */
    .num_cliprects = 0,
    .cliprects_ptr = 0,
    .flags = I915_EXEC_RENDER,  /* Ring selection */
    .rsvd1 = 0,
    .rsvd2 = 0,
};

ioctl(fd, DRM_IOCTL_I915_GEM_EXECBUFFER2, &exec_params);
```

**Execution Parameters:**
- `buffers_ptr`: Array of exec_object2 structures (GEM objects to validate)
- `batch_start_offset`: Offset into batch buffer for command start
- `batch_len`: Size of batch buffer in bytes
- `flags`: Ring selection (RENDER, BLT, VIDEO, COMPUTE)
- `DR1/DR4`: Render context registers (legacy, mostly unused)

### Batch Buffer Validation

The kernel validates batch buffers before execution:

```c
// Simplified validation flow
static int validate_batch_buffer(struct drm_i915_gem_request *req,
                                 struct drm_i915_gem_object *batch)
{
    u32 *cmd;
    unsigned int offset;
    
    for (offset = 0; offset < batch->batch_size; offset += 4) {
        cmd = (u32 *)((char *)batch->vaddr + offset);
        
        switch (*cmd >> 24) {  /* Opcode in upper byte */
        case 0x02:  /* Non-pipelined state */
            if (needs_clflush(cmd))
                clflush_range(cmd, cmd_length);
            break;
            
        case 0x03:  /* Pipelined state */
            validate_pipelined_command(cmd, offset);
            break;
            
        case 0x04:  /* 3D command */
            validate_3d_command(cmd);
            break;
        }
    }
    
    return 0;  /* Valid */
}
```

**Validation Checks:**
- Command alignment and size constraints
- Restricted register access (privileged commands blocked)
- Buffer offset validity
- Fence synchronization
- Memory dependencies

---

## Ring Buffer Management

### Ring Buffer Pointers

The driver maintains strict pointer management to prevent overwriting in-flight commands:

```c
struct intel_engine_cs_ringbuffer {
    struct drm_i915_gem_object *obj;
    void *vaddr;           /* CPU-mapped virtual address */
    u32 size;              /* Ring size in bytes */
    u32 effective_size;    /* Usable size (size - reserved) */
    u32 head;              /* GPU position (from HW) */
    u32 tail;              /* Driver position (our next write) */
};

static inline u32 intel_ring_space(struct intel_ring *ring)
{
    return (ring->head - ring->tail) & (ring->size - 1);
}

static inline bool intel_ring_stopped(struct intel_ring *ring)
{
    return !!(I915_READ(RING_CTL(ring->mmio_base)) & RING_WAIT);
}
```

### Command Space Reservation

Before submitting commands, the driver reserves space:

```c
static int
intel_ring_begin(struct drm_i915_gem_request *req, int num_dwords)
{
    struct intel_ring *ring = req->ring;
    int max_size = num_dwords * sizeof(u32);
    
    /* Ensure sufficient space exists */
    while (intel_ring_space(ring) < max_size) {
        if (signal_pending(current))
            return -ERESTARTSYS;
            
        /* Wait for GPU to advance head pointer */
        wait_event(ring->irq_queue,
                   intel_ring_space(ring) >= max_size);
    }
    
    ring->reserved_tail = ring->tail;
    ring->reserved_space = max_size;
    return 0;
}

static void intel_ring_advance(struct drm_i915_gem_request *req)
{
    struct intel_ring *ring = req->ring;
    
    /* Update tail pointer to GPU */
    I915_WRITE(RING_TAIL(ring->mmio_base), ring->tail);
    
    /* Ensure write completes */
    POSTING_READ(RING_TAIL(ring->mmio_base));
}
```

### Ring Buffer Wrapping

When commands wrap around the ring buffer:

```c
static u32 *intel_ring_wrap(struct intel_ring *ring, u32 *cmds)
{
    if ((void *)cmds >= (void *)ring->vaddr + ring->size)
        return (u32 *)ring->vaddr + ((void *)cmds - 
                (void *)(ring->vaddr + ring->size));
    return cmds;
}

/* Usage */
u32 *cmd = intel_ring_begin(req, 10);
*cmd++ = MI_NOOP;
*cmd++ = MI_STORE_DATA_IMM;
*cmd = offset;
intel_ring_advance(req);
```

---

## Command Parsing and Validation

### Command Format and Decoding

GPU commands have a standardized format:

```
Command Header (DWORD 0):
┌─────────────────────────────────────┐
│ Length | Type | SubOpcode | Opcode │
│ 31-16  │ 15-14│ 13-8      │ 7-0    │
└─────────────────────────────────────┘

Length: Number of DWORDs - 2
Type: 00=MI, 01=Single, 10=2D, 11=3D
Opcode: Command type
```

### Command Stream Execution Examples

**Example 1: NoOp Instruction**

```c
/* Simplest possible command */
struct mi_noop {
    u32 header;
};

#define MI_NOOP                         0x00
static inline u32 mi_noop(void)
{
    return MI_NOOP;
}

/* Usage */
OUT_RING(mi_noop());  /* 0x00000000 */
```

**Example 2: Memory Instruction**

```c
/* Store immediate data to memory */
struct mi_store_data_imm {
    u32 header;
    u32 flags;
    u32 address_low;
    u32 address_high;
    u32 data_low;
    u32 data_high;
};

#define MI_STORE_DATA_IMM               0x20
#define MI_STORE_DATA_IMM_USE_GGTT      (1 << 22)
#define MI_STORE_DATA_IMM_QW_EN         (1 << 21)

static inline void mi_store_data_imm(u32 *cmds, u64 address, u64 data)
{
    *cmds++ = MI_STORE_DATA_IMM | MI_STORE_DATA_IMM_QW_EN | 4;
    *cmds++ = 0;
    *cmds++ = lower_32_bits(address);
    *cmds++ = upper_32_bits(address);
    *cmds++ = lower_32_bits(data);
    *cmds++ = upper_32_bits(data);
}
```

**Example 3: 3D State Command**

```c
/* Pipeline state register write */
#define GFX_OP_3D_STATE_VS              (0x3 << 29 | 0x0 << 24)
#define GFX_OP_3D_STATE_GS              (0x3 << 29 | 0x1 << 24)
#define GFX_OP_3D_STATE_PS              (0x3 << 29 | 0x2 << 24)

static inline void emit_3d_state_vs(u32 *cmds, u32 offset)
{
    *cmds++ = GFX_OP_3D_STATE_VS | 6;  /* Length-1 */
    *cmds++ = 0;
    *cmds++ = offset;
    *cmds++ = 0;
    *cmds++ = 0;
    *cmds++ = 0;
    *cmds++ = 0;
}
```

---

## Pipeline Control and Synchronization

### Pipeline Control Commands

```c
/* Pipeline control - synchronization point */
struct gen8_pipeline_control {
    u32 header;
    u32 dw1;
    u64 address;
    u32 immediate_data_low;
    u32 immediate_data_high;
};

#define PIPE_CONTROL_COMMAND_OPCODE         0x7a000004
#define PIPE_CONTROL_AMFS_FLUSH             (1 << 13)
#define PIPE_CONTROL_ISP_DIS                (1 << 9)
#define PIPE_CONTROL_TLB_INVALIDATE         (1 << 8)
#define PIPE_CONTROL_FLUSH_L3               (1 << 7)
#define PIPE_CONTROL_DC_FLUSH_ENABLE        (1 << 5)
#define PIPE_CONTROL_QW_WRITE               (1 << 14)

static void emit_pipeline_control(u32 *cmds, u32 flags)
{
    *cmds++ = PIPE_CONTROL_COMMAND_OPCODE;
    *cmds++ = flags;
    *cmds++ = 0;  /* Address low */
    *cmds++ = 0;  /* Address high */
}
```

### Semaphore Synchronization

```c
/* GPU-to-GPU synchronization via semaphores */
#define GFX_OP_SEMAPHORE                (0x3 << 29 | 0x1d << 24)
#define SEMAPHORE_TARGET_RENDER_RING    (1 << 15)
#define SEMAPHORE_COMPARE_GREATER       (1 << 14)

static void emit_semaphore_wait(u32 *cmds, int mbox, u32 seqno)
{
    *cmds++ = GFX_OP_SEMAPHORE | SEMAPHORE_TARGET_RENDER_RING |
              SEMAPHORE_COMPARE_GREATER | 4;
    *cmds++ = seqno;
    *cmds++ = mbox;  /* Mailbox index */
}

static void emit_semaphore_signal(u32 *cmds, int mbox, u32 seqno)
{
    *cmds++ = GFX_OP_SEMAPHORE | 4;
    *cmds++ = seqno;
    *cmds++ = mbox;
}
```

---

## Error Handling and Recovery

### Batch Buffer Errors

```c
/* Detect and handle batch execution errors */
static void handle_batch_error(struct drm_i915_private *dev_priv,
                               struct intel_engine_cs *engine)
{
    u32 ipehr = I915_READ(RING_IPEHR(engine->mmio_base));
    u32 instdone = I915_READ(RING_INSTDONE(engine->mmio_base));
    
    DRM_ERROR("Batch error detected on %s\n", engine->name);
    DRM_ERROR("  IPEHR: 0x%08x (Current instruction)\n", ipehr);
    DRM_ERROR("  INSTDONE: 0x%08x (Instruction completions)\n", instdone);
    
    /* Analyze instruction pointer */
    if (ipehr == 0 || ipehr == GFX_OP_MI_ARB_ON_OFF)
        return;  /* Known safe errors */
    
    /* Trigger recovery: GPU hang detected */
    i915_handle_error(dev_priv, engine_mask(engine));
}
```

### Command Validation Failures

```c
/* Strict whitelist validation for batch commands */
#define CMD_ALLOWED     0x00
#define CMD_REJECTED    0x01
#define CMD_NEEDS_CLFLUSH 0x02

static const u8 cmd_validation_table[] = {
    [0x00] = CMD_ALLOWED,           /* MI_NOOP */
    [0x04] = CMD_ALLOWED,           /* MI_WAIT_FOR_EVENT */
    [0x0c] = CMD_ALLOWED,           /* MI_ARB_CHECK */
    [0x1c] = CMD_NEEDS_CLFLUSH,    /* MI_BATCH_BUFFER_START */
    [0x20] = CMD_ALLOWED,           /* MI_STORE_DATA_IMM */
    [0x31] = CMD_REJECTED,          /* MI_CONDITIONAL_BATCH_BUFFER_END */
};

static int validate_command(u32 cmd_opcode)
{
    u8 validation = cmd_validation_table[cmd_opcode & 0xff];
    
    if (validation == CMD_REJECTED) {
        DRM_ERROR("Rejected command: 0x%02x\n", cmd_opcode);
        return -EINVAL;
    }
    
    return 0;
}
```

---

## Performance Optimization

### Batch Buffer Size Optimization

```c
/* Optimal batch buffer sizes for caching */
#define BATCH_MIN_SIZE      512     /* Minimum batch allocation */
#define BATCH_NORMAL_SIZE   4096    /* Most common size */
#define BATCH_LARGE_SIZE    65536   /* Large workloads */

/* Reuse pool for frequently allocated batches */
struct drm_i915_gem_batch_pool {
    struct list_head cache_list[BATCH_POOL_BUCKETS];
    unsigned long active_count;
};

static struct drm_i915_gem_object *
i915_gem_batch_pool_get(struct i915_gem_batch_pool *pool, size_t size)
{
    struct drm_i915_gem_object *obj = NULL;
    int bucket = get_bucket_index(size);
    
    list_for_each_entry_safe(obj, tmp, &pool->cache_list[bucket], batch_link) {
        if (obj->base.size >= size) {
            list_move(&obj->batch_link, &pool->active_list);
            return obj;
        }
    }
    
    /* No suitable buffer in pool, allocate new */
    return i915_gem_object_create(dev, round_up(size, PAGE_SIZE));
}
```

### Ring Buffer Caching Strategy

```c
/* Minimize ring stalls by proactive buffer management */
static void maintain_ring_health(struct intel_ring *ring)
{
    u32 space = intel_ring_space(ring);
    
    if (space < ring->effective_size / 4) {
        /* Warning level: 75% full */
        schedule_work(&ring->cleanup_work);
    }
    
    if (space < 256) {
        /* Critical: cannot fit typical batch */
        flush_pending_requests(ring);
        wait_for_ring_space(ring, 256);
    }
}
```

### Command Pipelining

```c
/* Overlap command submission and GPU execution */
static void pipeline_commands(struct drm_i915_gem_request *req)
{
    /* Advance tail while GPU is still executing previous batch */
    struct intel_ring *ring = req->ring;
    
    /* Query current GPU position (from interrupt) */
    u32 gpu_head = I915_READ(RING_HEAD(ring->mmio_base));
    
    /* Calculate safe submission point */
    u32 safe_tail = (gpu_head + MIN_AHEAD_DISTANCE) & (ring->size - 1);
    
    if (ring->tail != safe_tail) {
        I915_WRITE(RING_TAIL(ring->mmio_base), ring->tail);
        POSTING_READ(RING_TAIL(ring->mmio_base));
    }
}
```

---

## Debugging Command Streams

### Ring Buffer Dump

```c
/* Dump ring buffer contents for debugging */
static void dump_ring_buffer(struct intel_ring *ring)
{
    u32 *ptr = (u32 *)ring->vaddr;
    u32 head = I915_READ(RING_HEAD(ring->mmio_base));
    u32 tail = ring->tail;
    int i;
    
    DRM_DEBUG_DRIVER("Ring buffer dump (head=0x%x, tail=0x%x):\n", head, tail);
    
    for (i = 0; i < ring->size / sizeof(u32); i += 4) {
        DRM_DEBUG_DRIVER("[0x%04x] %08x %08x %08x %08x\n",
                        i * 4, ptr[i], ptr[i+1], ptr[i+2], ptr[i+3]);
    }
}
```

### Command Logging

```c
/* Log submitted commands for debugging */
static void log_batch_submission(struct drm_i915_gem_request *req,
                                 struct drm_i915_gem_object *batch)
{
    u32 *cmds = (u32 *)batch->vaddr;
    int i;
    
    DRM_DEBUG_DRIVER("Batch submission: %d bytes\n", batch->base.size);
    
    for (i = 0; i < batch->base.size / 4; i += 4) {
        DRM_DEBUG_DRIVER("  [0x%04x] %08x %08x %08x %08x\n",
                        i * 4, cmds[i], cmds[i+1], cmds[i+2], cmds[i+3]);
    }
}
```

### State Trace

```c
/* Trace GPU state during command execution */
static void trace_ring_state(struct intel_ring *ring, const char *event)
{
    u32 head = I915_READ(RING_HEAD(ring->mmio_base));
    u32 tail = I915_READ(RING_TAIL(ring->mmio_base));
    u32 status = I915_READ(RING_INSTPM(ring->mmio_base));
    
    trace_printk("Ring %s: head=0x%x tail=0x%x instpm=0x%x\n",
                 event, head, tail, status);
}
```

---

## Best Practices

### 1. **Batch Buffer Management**
- Pre-allocate batch buffers from a pool for frequently-used sizes
- Validate all commands before submission
- Maintain proper alignment (typically 8-byte or 16-byte boundaries)

### 2. **Ring Buffer Efficiency**
- Reserve ring space before starting command generation
- Minimize ring stalls by proactive space monitoring
- Use pipeline control wisely to avoid unnecessary flushes

### 3. **Synchronization**
- Use semaphores for inter-engine coordination
- Emit pipeline control commands at logical boundaries
- Track completion via seqno or fences

### 4. **Error Handling**
- Validate all batch buffers before execution
- Implement comprehensive error injection for testing
- Log ring state on GPU hangs for post-mortem analysis

### 5. **Performance**
- Batch multiple commands together to reduce ring updates
- Use instruction caches where available
- Monitor ring occupancy and adjust submission rate dynamically

---

## References

- **i915 Driver Documentation**: Command stream specification and validation rules
- **GPU ISA Manuals**: Specific instruction formats and encodings
- **Ring Buffer Implementation**: `drivers/gpu/drm/i915/intel_ringbuffer.c`
- **Execbuf Path**: `drivers/gpu/drm/i915/i915_gem_execbuffer.c`
- **Batch Validation**: Instruction whitelist and security policies

**Related Topics:**
- [Hardware Discovery and Initialization](10-Hardware-Discovery-Initialization.md)
- [Scheduling and Arbitration](21-Scheduling-Arbitration.md)
- [Performance Monitoring (OA)](14-Performance-Monitoring-OA.md)
- [User-Space Interface (UAPI)](15-User-Space-Interface-UAPI.md)
