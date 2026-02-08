# 21. Scheduling and Arbitration

## Executive Summary

GPU scheduling and arbitration determine how the i915 driver multiplexes multiple GPU contexts and workloads across hardware resources. The GuC firmware scheduler handles fair scheduling, priority enforcement, and context switching. Understanding scheduling mechanisms is critical for achieving predictable latency, fair resource allocation, and optimizing multi-application GPU workload performance.

**Key Concepts:**
- Context switching and preemption mechanisms
- Priority levels and fairness algorithms
- Task queue management and scheduling policies
- Workload balancing across GPU engines
- GuC-based scheduling architecture

**Table of Contents**
1. [Scheduling Architecture](#architecture)
2. [GuC Scheduler](#guc-scheduler)
3. [Context Management and Switching](#context-management)
4. [Priority and Fairness](#priority-fairness)
5. [Task Queue Management](#task-queues)
6. [Preemption and Timeslicing](#preemption)
7. [Workload Balancing](#balancing)
8. [Performance Tuning](#performance-tuning)
9. [Debugging and Monitoring](#debugging)
10. [Best Practices](#best-practices)

---

## Architecture

### Scheduling Overview

```
User Space (Multiple Contexts)
    |
    +-- Context A (App 1)
    +-- Context B (App 2)
    +-- Context C (Render Engine)
    |
    v
Kernel i915 Driver
    |
    v
GuC Firmware Scheduler
    |
    +-- Priority Queue Management
    +-- Context Switching
    +-- Preemption Logic
    |
    v
Hardware Execution Units
    +-- Render Engine
    +-- Blitter Engine
    +-- Video Decode Engine
    +-- Video Encode Engine
```

### Scheduling Models

**1. Fixed Priority Scheduling (Legacy)**
```
Priority Levels:  HIGH → NORMAL → LOW
Strategy:         Always run highest priority task
Preemption:       Preempt lower priority
Latency:          Low for high-priority tasks
Fairness:         Lower priority starvation possible
```

**2. Time-Sliced Scheduling (Modern)**
```
Round-Robin Scheduling:
Context A (10ms) → Context B (10ms) → Context C (10ms) → ...
                ↑
         GuC handles switching
```

**3. Hybrid Model (Production)**
```
Priority Levels:
  0-255: Priority range
  0-3:   System (kernel) priority
  4-255: User priority (distributed fairly)

Mixing fairness with priorities
```

---

## GuC Scheduler

### GuC Architecture

```c
/* GuC scheduler handles all GPU task scheduling */
struct guc_preempt_work {
    struct work_struct work;
    struct intel_engine_cs *engine;
    struct preempt_work_item items[MAX_PREEMPT_ITEMS];
};

struct guc_shared_ctx_data {
    u32 addr;           /* GPU virtual address */
    struct guc_ct_pool *pool;
    u32 pool_count;
};

struct intel_guc {
    /* Firmware attributes */
    u32 fw_version;
    bool loaded;
    bool running;
    
    /* Communication */
    struct guc_ct ct;   /* Command transport */
    
    /* Scheduling state */
    struct guc_preempt_work preempt_work;
    struct guc_shared_ctx_data shared_data;
    
    /* Workqueues */
    struct work_struct work;
    struct delayed_work stat_work;
};
```

### GuC Communication

```c
/* GuC command transmission */
static int guc_send_command(struct intel_guc *guc,
                           const u32 *action,
                           u32 len,
                           u32 *response_buf)
{
    int ret;
    
    /* Prepare command packet */
    ret = intel_guc_ct_send(&guc->ct, action, len,
                           response_buf, RESPONSE_LEN);
    
    if (ret < 0) {
        DRM_ERROR("GuC command failed: %d\n", ret);
        return ret;
    }
    
    /* Command successfully transmitted */
    return ret;
}

/* Action commands */
#define INTEL_GUC_ACTION_SAMPLE_FORCEWAKE    0x0
#define INTEL_GUC_ACTION_ALLOCATE_DOORBELL   0x1
#define INTEL_GUC_ACTION_DEALLOCATE_DOORBELL 0x2
#define INTEL_GUC_ACTION_PARK_CONTEXT        0x3
#define INTEL_GUC_ACTION_UNPARK_CONTEXT      0x4
#define INTEL_GUC_ACTION_SUBMIT_EXEC_QUEUE   0x5
```

### Context Descriptor

```c
/* Context descriptor sent to GuC */
struct guc_context_desc {
    u32 context_id;
    u32 state;              /* Context state */
    u32 priority;           /* Scheduling priority */
    u32 hw_id;              /* Hardware context ID */
    
    /* Workqueue management */
    u32 wq_base;            /* Work queue base address */
    u32 wq_head;            /* Work queue head pointer */
    u32 wq_tail;            /* Work queue tail pointer */
    
    /* Scheduling attributes */
    u32 rate_estimate;      /* Instructions per second */
    u32 submit_element_count;
    u32 schedule_state;
};

/* Context states */
#define GUC_CTX_STATE_DEFAULT       0x0
#define GUC_CTX_STATE_WAITING       0x1
#define GUC_CTX_STATE_ACTIVE        0x2
#define GUC_CTX_STATE_PENDING_PAUSE 0x3
#define GUC_CTX_STATE_PAUSED        0x4
```

---

## Context Management and Switching

### Context Registration

```c
/* Register context with GuC for scheduling */
static int guc_context_register(struct intel_guc_client *client,
                                struct intel_context *ctx)
{
    struct guc_context_desc desc;
    int ret;
    
    /* Populate context descriptor */
    desc.context_id = ctx->guc_id;
    desc.hw_id = ctx->hw_id;
    desc.priority = ctx->priority;
    desc.wq_base = ctx->workqueue.gpu_addr;
    
    /* Send registration to GuC */
    ret = guc_send_command(client->guc,
                          (u32 *)&desc,
                          sizeof(desc) / 4,
                          NULL);
    
    if (ret == 0) {
        ctx->state = CONTEXT_REGISTERED;
    }
    
    return ret;
}
```

### Context Switching

```c
/* Hardware context switch */
struct context_switch_trace {
    u32 from_context_id;
    u32 to_context_id;
    u32 switch_reason;
    u32 switch_count;
};

#define SWITCH_REASON_PREEMPTION    0x1
#define SWITCH_REASON_TIMEOUT       0x2
#define SWITCH_REASON_FAIRNESS      0x3
#define SWITCH_REASON_ERROR         0x4

static void trace_context_switch(struct intel_engine_cs *engine,
                                 struct context_switch_trace *trace)
{
    DRM_DEBUG_DRIVER("Context switch on %s: %d->%d (%s)\n",
                    engine->name,
                    trace->from_context_id,
                    trace->to_context_id,
                    switch_reason_name(trace->switch_reason));
}
```

---

## Priority and Fairness

### Priority Assignment

```c
/* Priority levels (0-255, lower = higher priority) */
#define GUC_PRIORITY_KERNEL         0   /* System priority */
#define GUC_PRIORITY_HIGH           5   /* High priority user */
#define GUC_PRIORITY_NORMAL         127 /* Normal user */
#define GUC_PRIORITY_LOW            255 /* Low priority user */

static int set_context_priority(struct intel_context *ctx, int priority)
{
    if (priority < GUC_PRIORITY_KERNEL)
        return -EINVAL;  /* Cannot elevate above kernel */
    
    if (priority > GUC_PRIORITY_LOW)
        priority = GUC_PRIORITY_LOW;
    
    ctx->priority = priority;
    
    /* Notify GuC of priority change */
    return guc_context_update_priority(ctx);
}

/* Client-accessible priority levels */
enum drm_i915_gem_context_priority {
    I915_CONTEXT_LOW_PRIORITY = 0,
    I915_CONTEXT_NORMAL_PRIORITY = 1,
    I915_CONTEXT_HIGH_PRIORITY = 2,
};
```

### Fair Scheduling

```c
/* Fair scheduling algorithm */
struct scheduling_slice {
    u32 context_id;
    u32 time_slice_duration;  /* Time slot in microseconds */
    u32 time_slice_remaining;
};

#define DEFAULT_TIMESLICE_DURATION  1000  /* 1ms */
#define MIN_TIMESLICE_DURATION      10    /* 10us */
#define MAX_TIMESLICE_DURATION      10000 /* 10ms */

static void adjust_timeslice(struct intel_engine_cs *engine,
                            struct scheduling_slice *slice)
{
    /* Increase timeslice for cooperative workloads */
    if (slice->context_id->instructions_per_frame < THRESHOLD)
        slice->time_slice_duration = MAX_TIMESLICE_DURATION;
    else
        slice->time_slice_duration = MIN_TIMESLICE_DURATION;
}
```

---

## Task Queue Management

### Work Queue Structure

```c
/* Per-context work queue for batch submission */
struct guc_work_queue {
    void *vaddr;                /* CPU-mapped address */
    u64 gpu_addr;              /* GPU virtual address */
    u32 size;                  /* Queue size in bytes */
    
    u32 head;                  /* GuC updates */
    u32 tail;                  /* Driver updates */
    
    struct drm_i915_gem_object *obj;
    struct list_head items;    /* Pending work items */
};

/* Work queue entry */
struct guc_wq_item {
    u32 context_desc;
    u32 batch_address_lo;
    u32 batch_address_hi;
    u32 fence_id;
};
```

### Submit Queue Management

```c
/* Queue work to GuC scheduler */
static void guc_submit_request(struct drm_i915_gem_request *req)
{
    struct intel_engine_cs *engine = req->engine;
    struct intel_guc *guc = &engine->i915->guc;
    
    /* Prepare work queue item */
    struct guc_wq_item item = {
        .context_desc = req->ctx->guc_id,
        .batch_address_lo = req->batch_gpu_addr & 0xffffffff,
        .batch_address_hi = req->batch_gpu_addr >> 32,
        .fence_id = req->seqno,
    };
    
    /* Push to work queue */
    guc_wq_item_append(engine->guc_wq, &item);
    
    /* Signal GuC via doorbell */
    guc_ring_doorbell(guc, engine);
    
    trace_i915_gem_request_submit(req);
}
```

### Doorbell Mechanism

```c
/* Notify GuC of new work via doorbell */
static inline void guc_ring_doorbell(struct intel_guc *guc,
                                     struct intel_engine_cs *engine)
{
    u32 doorbell_offset = guc_doorbell_offset(guc, engine->id);
    u32 doorbell_addr = guc->ggtt_pin_bias + doorbell_offset;
    
    /* Write to doorbell register triggers GuC scheduler */
    iowrite32(1, (void __iomem *)doorbell_addr);
    
    /* Ensure doorbell write reaches hardware */
    ioread32((void __iomem *)doorbell_addr);
}
```

---

## Preemption and Timeslicing

### Preemption Mechanism

```c
/* Preempt a running context */
static void preempt_context(struct intel_engine_cs *engine,
                           struct intel_context *to_preempt)
{
    struct intel_guc *guc = &engine->i915->guc;
    u32 action[] = {
        INTEL_GUC_ACTION_PREEMPT,
        engine->guc_id,
        to_preempt->guc_id,
    };
    
    /* Send preemption action to GuC */
    intel_guc_send(guc, action, ARRAY_SIZE(action));
    
    /* Wait for preemption to complete */
    wait_for_preemption(to_preempt, PREEMPT_TIMEOUT_MS);
    
    trace_i915_context_preemption(engine, to_preempt);
}

#define PREEMPT_TIMEOUT_MS 100
```

### Timeslice Expiration

```c
/* Handle context timeslice timeout */
static void timeslice_expired(struct intel_engine_cs *engine)
{
    struct intel_context *running_ctx = engine->running_context;
    
    DRM_DEBUG_DRIVER("Timeslice expired for context %d on %s\n",
                    running_ctx->guc_id, engine->name);
    
    /* Trigger fairness preemption */
    preempt_context(engine, running_ctx);
    
    /* Schedule next context */
    schedule_next_context(engine);
}

/* Timeslice interrupt handler */
static void handle_timeslice_interrupt(struct drm_i915_private *dev_priv,
                                       unsigned int engine_mask)
{
    struct intel_engine_cs *engine;
    unsigned int tmp;
    
    for_each_engine_masked(engine, dev_priv, engine_mask, tmp) {
        queue_work(system_highpri_wq, &engine->timeslice_work);
    }
}
```

---

## Workload Balancing

### Engine Load Tracking

```c
/* Track per-engine utilization for balancing decisions */
struct engine_load {
    struct intel_engine_cs *engine;
    u32 queue_depth;            /* Number of pending batches */
    u32 utilization_percent;    /* 0-100% */
    u64 last_busy_jiffies;
};

static u32 get_engine_utilization(struct intel_engine_cs *engine)
{
    u64 total_time = jiffies - engine->busy_since;
    u64 busy_time = engine->busy_jiffies;
    
    if (total_time == 0)
        return 0;
    
    return (busy_time * 100) / total_time;
}
```

### Load Balancing Strategy

```c
/* Choose optimal engine for workload */
static struct intel_engine_cs *
choose_best_engine(struct drm_i915_private *dev_priv,
                  struct i915_gem_engines *engines,
                  unsigned int class)
{
    struct intel_engine_cs *best_engine = NULL;
    u32 min_utilization = 100;
    unsigned int i;
    
    for (i = 0; i < engines->count; i++) {
        struct intel_engine_cs *engine = engines->engines[i];
        u32 utilization;
        
        if (engine->class != class)
            continue;
        
        utilization = get_engine_utilization(engine);
        
        if (utilization < min_utilization) {
            min_utilization = utilization;
            best_engine = engine;
        }
    }
    
    return best_engine ?: engines->engines[0];
}
```

---

## Performance Tuning

### Scheduling Policies

```c
/* Tunable scheduling parameters */
struct scheduling_config {
    u32 timeslice_duration_us;
    u32 preempt_timeout_us;
    bool enable_fairness;
    bool enable_timeslicing;
    int priority_boost_percent;
};

static struct scheduling_config sched_config = {
    .timeslice_duration_us = 1000,      /* 1ms */
    .preempt_timeout_us = 100000,       /* 100ms */
    .enable_fairness = true,
    .enable_timeslicing = true,
    .priority_boost_percent = 20,
};

/* Module parameter for runtime tuning */
module_param_named(timeslice_duration,
                  sched_config.timeslice_duration_us,
                  uint, 0644);
```

### Adaptive Scheduling

```c
/* Adapt scheduling based on workload characteristics */
static void adapt_scheduling(struct intel_engine_cs *engine)
{
    struct scheduling_slice *active_slice = engine->active_slice;
    u32 utilization = get_engine_utilization(engine);
    
    /* Reduce timeslice for high-utilization scenarios */
    if (utilization > 90) {
        active_slice->time_slice_duration =
            max(MIN_TIMESLICE_DURATION,
               active_slice->time_slice_duration - 10);
    }
    
    /* Increase timeslice for underutilized engines */
    if (utilization < 30) {
        active_slice->time_slice_duration =
            min(MAX_TIMESLICE_DURATION,
               active_slice->time_slice_duration + 50);
    }
    
    DRM_DEBUG_DRIVER("Adjusted timeslice to %uus (utilization: %u%%)\n",
                    active_slice->time_slice_duration, utilization);
}
```

---

## Debugging and Monitoring

### Scheduler Statistics

```c
/* Per-context scheduling statistics */
struct guc_context_stats {
    u64 total_submit_count;
    u64 total_execution_time_ns;
    u64 max_latency_ns;
    u64 context_switches;
    u32 average_batch_size;
};

static void dump_context_stats(struct intel_context *ctx)
{
    struct guc_context_stats *stats = &ctx->guc_stats;
    
    DRM_DEBUG_DRIVER("Context %d statistics:\n", ctx->guc_id);
    DRM_DEBUG_DRIVER("  Submissions: %llu\n", stats->total_submit_count);
    DRM_DEBUG_DRIVER("  Execution time: %llu ns\n", stats->total_execution_time_ns);
    DRM_DEBUG_DRIVER("  Max latency: %llu ns\n", stats->max_latency_ns);
    DRM_DEBUG_DRIVER("  Context switches: %llu\n", stats->context_switches);
    DRM_DEBUG_DRIVER("  Avg batch size: %u bytes\n", stats->average_batch_size);
}
```

### Trace Events

```c
/* Trace scheduling events for analysis */
TRACE_EVENT(guc_context_scheduled,
    TP_PROTO(struct intel_context *ctx, bool is_new),
    TP_ARGS(ctx, is_new),
    TP_STRUCT__entry(
        __field(u32, context_id)
        __field(u32, priority)
        __field(bool, is_new)
        __field(u32, queue_depth)
    ),
    TP_fast_assign(
        __entry->context_id = ctx->guc_id;
        __entry->priority = ctx->priority;
        __entry->is_new = is_new;
        __entry->queue_depth = ctx->queue_depth;
    ),
    TP_printk("ctx=%d prio=%d queue=%d %s",
             __entry->context_id,
             __entry->priority,
             __entry->queue_depth,
             __entry->is_new ? "NEW" : "RESUME")
);
```

### Performance Analysis

```c
/* Analyze scheduling latency */
static void analyze_scheduling_latency(struct intel_engine_cs *engine)
{
    struct list_head *pending = &engine->request_list;
    struct drm_i915_gem_request *req;
    u64 total_latency = 0;
    u32 count = 0;
    
    list_for_each_entry(req, pending, link) {
        u64 latency_ns = ktime_get_ns() - req->submit_time_ns;
        
        DRM_DEBUG_DRIVER("Request %lld latency: %llu ns\n",
                        req->seqno, latency_ns);
        
        total_latency += latency_ns;
        count++;
    }
    
    if (count > 0) {
        DRM_DEBUG_DRIVER("Average latency: %llu ns\n",
                        total_latency / count);
    }
}
```

---

## Best Practices

### 1. **Priority Management**
- Use priorities sparingly to avoid starvation
- Reserve high priority for time-sensitive workloads
- Monitor priority inversion scenarios

### 2. **Timeslice Configuration**
- Tune timeslice duration based on workload characteristics
- Shorter slices for interactive applications
- Longer slices for throughput-optimized compute

### 3. **Context Allocation**
- Reuse contexts where possible to reduce overhead
- Limit number of active contexts per application
- Monitor context descriptor pool exhaustion

### 4. **Fairness Assurance**
- Enable timeslicing for multi-user systems
- Monitor fairness metrics and adjust accordingly
- Prevent high-priority contexts from indefinite execution

### 5. **Performance Optimization**
- Batch multiple work items to reduce doorbell rings
- Use engine load balancing for multiple engines
- Profile scheduling overhead with GPU traces

---

## References

- **GuC Firmware Scheduler**: i915 GuC submission and scheduler architecture
- **Context Switching**: Hardware and firmware context switch mechanisms
- **Priority Scheduling**: Fair scheduling and priority handling
- **Preemption Logic**: Context preemption and timeslice interrupts
- **Performance Analysis**: Scheduling latency and throughput metrics

**Related Topics:**
- [Hardware Discovery and Initialization](10-Hardware-Discovery-Initialization.md)
- [Firmware Loading and Management](13-Firmware-Loading-Management.md)
- [Command Stream Execution](20-Command-Stream-Execution.md)
- [User-Space Interface (UAPI)](15-User-Space-Interface-UAPI.md)
