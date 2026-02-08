# Error Handling and Recovery

**Document ID:** 11 | **Status:** Complete | **Last Updated:** February 2026

---

## Executive Summary

Error handling and recovery is critical for GPU driver reliability. This document covers how i915 detects errors, recovers from failures, and maintains system stability when GPU operations fail.

### Key Topics
- **Error detection:** Timeout detection, GPU hangs, hardware errors
- **Recovery mechanisms:** GPU reset, request retry, state recovery
- **Deadlock prevention:** Watchdog timers, forced resets
- **Error reporting:** Logging, debugging, user notification

### Performance Impact
- **Undetected hang:** System freezes indefinitely
- **Detection latency:** 2-10 seconds for hang detection
- **Recovery time:** 1-5 seconds for GPU reset

---

## Table of Contents

1. [Error Detection](#error-detection)
2. [Timeout Handling](#timeout-handling)
3. [GPU Hang Recovery](#gpu-hang-recovery)
4. [Hardware Error Handling](#hardware-error-handling)
5. [State Recovery and Cleanup](#state-recovery-and-cleanup)
6. [Error Injection and Testing](#error-injection-and-testing)
7. [Debugging Failed Operations](#debugging-failed-operations)
8. [Summary & Best Practices](#summary--best-practices)

---

## Error Detection

### Error Sources and Types

```plaintext
┌─────────────────────────────────────┐
│ GPU Error Sources                   │
└────┬────────────────────────────────┘
     │
     ├─→ Execution Errors
     │   ├─ Invalid instruction
     │   ├─ Segmentation fault
     │   ├─ Memory access violation
     │   └─ Arithmetic exception
     │
     ├─→ Timeout Errors
     │   ├─ Request not completing
     │   ├─ Batch buffer hang
     │   ├─ Submission deadlock
     │   └─ Fence timeout
     │
     ├─→ Hardware Errors
     │   ├─ Temperature overheat
     │   ├─ Power domain failure
     │   ├─ Engine lockup
     │   └─ Memory corruption
     │
     ├─→ Driver Errors
     │   ├─ Invalid command stream
     │   ├─ Memory leak
     │   ├─ Resource exhaustion
     │   └─ Logic bug
     │
     └─→ System Errors
         ├─ Out of memory
         ├─ Interrupt storm
         ├─ PCIe link failure
         └─ Thermal shutdown
```

### Heartbeat Monitoring

The driver monitors GPU activity using a heartbeat mechanism:

```c
// Periodic heartbeat check
struct intel_engine_heartbeat {
    struct delayed_work work;
    unsigned long last_exec;  // Last execution time
    unsigned long thresholds[I915_HEARTBEAT_FREQ];
};

// Heartbeat function called every ~1 second
static void intel_engine_heartbeat(struct work_struct *work)
{
    struct intel_engine_cs *engine =
        container_of(work, struct intel_engine_cs, 
                     heartbeat.work.work);
    
    // Check if requests are completing
    intel_engine_flush_submission(engine);
    
    // Check for stuck requests
    if (request_is_stuck(engine->last_request)) {
        dev_err(engine->i915->drm.dev,
                "GPU hang detected on %s\n", engine->name);
        queue_work(system_unbound_wq, &engine->reset.work);
        return;
    }
    
    // Schedule next heartbeat
    mod_delayed_work(system_unbound_wq, &engine->heartbeat.work,
                     I915_HEARTBEAT_INTERVAL);
}
```

---

## Timeout Handling

### Timeout Thresholds

Different operations have different timeout requirements:

```plaintext
┌────────────────────────────────────────┐
│ Timeout Thresholds                     │
└───┬────────────────────────────────────┘
    │
    ├─ Fence Wait
    │  ├─ User space: up to 60 seconds
    │  ├─ Kernel internal: 2-10 seconds
    │  └─ Heartbeat: 1-2 seconds
    │
    ├─ Request Execution
    │  ├─ Short tasks: 0.1-1 second
    │  ├─ Long tasks: 5-10 seconds
    │  ├─ Batch jobs: up to 30+ seconds
    │  └─ Compute: workload dependent
    │
    ├─ Memory Operations
    │  ├─ TLB invalidation: 10-100 ms
    │  ├─ Eviction: 100 ms - 1 second
    │  └─ Migration: 1-5 seconds
    │
    ├─ Power Transitions
    │  ├─ RC6 entry: 10 ms
    │  ├─ Frequency change: 1-10 ms
    │  └─ Power well on: 10-100 ms
    │
    └─ Synchronization
       ├─ Request dependency: 10-100 ms
       ├─ Semaphore: 1 second
       └─ Barrier: 5 seconds
```

### Watchdog Timer Implementation

```c
// Request watchdog - detects hung requests
struct i915_request {
    struct dma_fence fence;
    struct delayed_work watchdog;
    unsigned long timeout;
};

// Start watchdog when request is submitted
void i915_request_start_watchdog(struct i915_request *rq)
{
    rq->timeout = jiffies + I915_REQUEST_TIMEOUT;
    
    mod_delayed_work(system_highpri_wq, &rq->watchdog,
                     I915_REQUEST_TIMEOUT);
}

// Called when watchdog fires
static void i915_watchdog_fired(struct work_struct *work)
{
    struct i915_request *rq = container_of(work,
        struct i915_request, watchdog.work);
    
    // Check if request has completed
    if (dma_fence_is_signaled(&rq->fence)) {
        return;  // Completed, all good
    }
    
    // Request still not done - GPU might be hung
    dev_err(rq->engine->i915->drm.dev,
            "Request timeout on %s after %ums\n",
            rq->engine->name,
            I915_REQUEST_TIMEOUT);
    
    // Trigger GPU reset
    i915_gpu_reset(rq->engine->i915);
}
```

---

## GPU Hang Recovery

### Hang Detection Flow

```plaintext
┌──────────────────────────────┐
│ Request Submitted            │
│ Start Heartbeat              │
└──────┬───────────────────────┘
       │
       ├─ Every 1 second
       │  └─ Check request progress
       │
       ├─ Progress detected? ✓
       │  └─ Continue execution
       │
       ├─ No progress after 5 sec? ✗
       │  └─ GPU Hang Suspected
       │
       ├─ Confirm with 2nd check
       │  └─ Re-check after 1 more sec
       │
       ├─ Still no progress? ✗
       │  └─ GPU Hang Confirmed
       │
       └─→ Trigger GPU Reset
           ├─ Flag reset in progress
           ├─ Wake up all waiters
           ├─ Disable interrupts
           ├─ Wait for engines idle
           ├─ Reset hardware
           ├─ Reinitialize GPU state
           ├─ Resume execution
           └─ Report to userspace
```

### GPU Reset Implementation

```c
// Full GPU reset sequence
static int i915_gpu_reset(struct i915_drm_private *i915)
{
    int ret, i;
    
    // 1. Mark reset as in progress
    if (test_and_set_bit(I915_RESET_IN_PROGRESS,
                          &i915->gpu_error.flags)) {
        // Another thread is already resetting
        return -EBUSY;
    }
    
    // 2. Disable new work submission
    i915_gem_set_wedged(i915);
    
    // 3. Wake all waiting processes
    wake_up_all(&i915->gpu_error.wait_queue);
    
    // 4. Wait for in-flight requests to timeout/complete
    for (i = 0; i < I915_NUM_ENGINES; i++) {
        struct intel_engine_cs *engine = i915->engines[i];
        if (!engine)
            continue;
            
        // Stop accepting new work
        intel_engine_stop_cs(engine);
        
        // Wait for pending work
        ret = intel_engine_reset(engine, "GPU reset");
        if (ret)
            goto out;
    }
    
    // 5. Reinitialize hardware
    ret = i915_ggtt_init(i915);
    if (ret)
        goto out;
    
    // 6. Restore GPU state
    ret = intel_engines_reinit(i915);
    if (ret)
        goto out;
    
    // 7. Replay lost requests (optional)
    ret = i915_gem_replay_failed_requests(i915);
    
out:
    // 8. Re-enable work submission
    clear_bit(I915_RESET_IN_PROGRESS, &i915->gpu_error.flags);
    
    return ret;
}
```

### Per-Engine Reset

For less critical hangs, reset individual engines:

```c
// Reset single engine (lighter weight than full reset)
static int intel_engine_reset(struct intel_engine_cs *engine,
                              const char *reason)
{
    int ret;
    
    GEM_TRACE("engine_reset(%s)", engine->name);
    
    dev_notice(engine->i915->drm.dev,
               "%s engine reset [%s]\n",
               engine->name, reason);
    
    // 1. Stop command submission
    intel_engine_stop_cs(engine);
    
    // 2. Wait for in-flight requests
    while (!list_empty(&engine->active.requests)) {
        struct i915_request *rq = list_first_entry(
            &engine->active.requests,
            struct i915_request, sched.link);
        
        // Force completion
        i915_request_mark_complete(rq);
        list_del_init(&rq->sched.link);
    }
    
    // 3. Reset the engine
    ret = engine->reset.reset(engine, NULL);
    
    // 4. Reinitialize engine state
    ret = engine->init_hw(engine);
    
    return ret;
}
```

---

## Hardware Error Handling

### Interrupt-Driven Error Detection

```plaintext
GPU ISR (Interrupt Service Routine)
    │
    ├─→ Read ISR/IIR registers
    │
    ├─→ Parse interrupt type
    │   ├─ Render error?
    │   ├─ Timeout interrupt?
    │   ├─ Memory error?
    │   └─ Thermal warning?
    │
    ├─→ Hardware Error?
    │   ├─ Save error state
    │   ├─ Log error details
    │   ├─ Queue recovery work
    │   └─ Inform userspace
    │
    └─→ Clear interrupt
        └─ Continue execution
```

### Error State Capture

When an error occurs, save GPU state for debugging:

```c
// Capture GPU error state for debugging
struct i915_error_state {
    unsigned long timestamp;
    int uptime;
    
    // Engine states
    struct {
        char name[16];
        u32 RING_HEAD;
        u32 RING_TAIL;
        u32 RING_CTL;
        u32 RING_STATUS;
        u32 RING_MODE;
        struct {
            u32 cpu_ring_head;
            u32 cpu_ring_tail;
            u32 last_seqno;
        } ringbuffer;
    } engines[I915_NUM_ENGINES];
    
    // Memory state
    struct {
        u32 ERROR;    // Error status register
        u32 FAULT;    // Page fault info
    } memory;
    
    // Thermal state
    struct {
        u32 TEMP;     // Temperature
        u32 STATUS;   // Throttle status
    } thermal;
};

// Called in error handler
void i915_capture_error_state(struct i915_drm_private *i915)
{
    struct i915_error_state *es;
    struct intel_engine_cs *engine;
    int i;
    
    es = kmalloc(sizeof(*es), GFP_ATOMIC);
    if (!es)
        return;
    
    es->timestamp = ktime_get_real_ns();
    
    for_each_engine(engine, &i915->gt, i) {
        es->engines[i].RING_HEAD = 
            intel_uncore_read(&i915->uncore,
                              RING_HEAD(engine->mmio_base));
        es->engines[i].RING_TAIL =
            intel_uncore_read(&i915->uncore,
                              RING_TAIL(engine->mmio_base));
        // ... capture more registers ...
    }
    
    i915->error_state = es;
    
    // Make available to userspace via debugfs
    // /sys/kernel/debug/dri/0/i915_error_state
}
```

### Thermal Management

```c
// Handle thermal errors
static void handle_thermal_error(struct i915_drm_private *i915)
{
    u32 temp = read_temperature_sensor(i915);
    
    if (temp > THERMAL_CRITICAL_TEMP) {
        // Critical overheat - immediate shutdown
        dev_crit(i915->drm.dev,
                "Critical thermal error: %d C\n", temp);
        i915_gpu_reset(i915);
        queue_work(system_unbound_wq,
                   &i915->l3_parity.error_work);
    } else if (temp > THERMAL_WARNING_TEMP) {
        // Warning - reduce performance
        dev_warn(i915->drm.dev,
                "Thermal warning: %d C\n", temp);
        intel_rps_set(i915, RPS_MIN_FREQ);
    }
}
```

---

## State Recovery and Cleanup

### Wedged Device Recovery

When GPU is in bad state, mark as "wedged" and prevent new work:

```plaintext
┌──────────────────────────────┐
│ GPU Wedged (Hung)            │
│ - No recovery attempted      │
│ - No new work accepted       │
│ - Existing work drains       │
└──────┬───────────────────────┘
       │
       ├─ Userspace detection
       │  └─ ioctl returns -EIO
       │
       ├─ All waiters woken
       │  └─ dma_fence_signal with error
       │
       ├─ In-flight requests completed
       │  └─ Fences marked as failed
       │
       ├─ Resources cleaned up
       │  ├─ Ring buffers flushed
       │  ├─ Pending work aborted
       │  └─ Memory freed
       │
       └─ Device can be unloaded
           └─ rmmod i915 succeeds
```

### Request Cleanup on Error

```c
// Abort pending requests on error
void i915_gem_submit_request_error(struct i915_request *rq,
                                   int error)
{
    struct intel_ring *ring = rq->ring;
    
    // 1. Stop processing this request
    intel_ring_advance(ring, rq);
    
    // 2. Mark fence as signaled with error
    dma_fence_set_error(&rq->fence, error);
    dma_fence_signal(&rq->fence);
    
    // 3. Wake all waiters
    intel_context_exit(rq->context);
    
    // 4. Release resources
    i915_request_put(rq);
}
```

---

## Error Injection and Testing

### Debug Error Injection

```c
// Module parameter for error injection
static int inject_errors = 0;
module_param(inject_errors, int, 0644);
MODULE_PARM_DESC(inject_errors,
    "Inject GPU errors for testing (0=none, 1=hang, 2=reset)");

// Inject error on next submission
void i915_inject_gpu_error(struct i915_drm_private *i915)
{
    switch (inject_errors) {
    case 1:
        // Inject hang
        dev_info(i915->drm.dev, "Injecting GPU hang\n");
        queue_delayed_work(system_unbound_wq,
                          &i915->gpu_error.hangcheck_work, 0);
        break;
        
    case 2:
        // Inject reset
        dev_info(i915->drm.dev, "Injecting GPU reset\n");
        i915_gpu_reset(i915);
        break;
        
    default:
        break;
    }
}
```

### Stress Testing

```bash
# Inject hang error
echo 1 > /sys/module/i915/parameters/inject_errors

# Monitor error state
cat /sys/kernel/debug/dri/0/i915_error_state

# Check recovery
lspci | grep VGA  # Should still show GPU

# Verify reset count
cat /sys/kernel/debug/dri/0/i915_reset_count
```

---

## Debugging Failed Operations

### Common Failure Patterns

| Pattern | Cause | Solution |
|---------|-------|----------|
| **Timeout after N seconds** | Hang in batch buffer | Check batch contents, look for infinite loops |
| **Immediate return -EIO** | GPU wedged | Check error state, may need driver reload |
| **Request never completes** | Fence not signaled | Check interrupt delivery, heartbeat logs |
| **Memory corruption** | TLB stale entries | Force TLB invalidation, check GGTT |
| **Thermal throttle** | Temperature too high | Check cooling, reduce frequency |

### Debug Information

```bash
# Check GPU error state
cat /sys/kernel/debug/dri/0/i915_error_state

# Kernel hang detection
dmesg | grep -i "hang\|timeout\|reset"

# GPU heartbeat
cat /sys/kernel/debug/dri/0/i915_heartbeat

# Request status
cat /sys/kernel/debug/dri/0/i915_requests

# Engine status
cat /sys/kernel/debug/dri/0/i915_engine_info
```

### Logging Configuration

```c
// Enable detailed logging
#define DRM_DEBUG_DRIVER(fmt, ...)                  \
    drm_dev_dbg(dev, DRM_UT_DRIVER, fmt, ##__VA_ARGS__)

// In error handler
DRM_DEBUG_DRIVER("GPU hang detected, initiating recovery\n");
DRM_DEBUG_DRIVER("Error state: head=0x%x tail=0x%x\n",
                 head, tail);
```

---

## Summary & Best Practices

### Key Takeaways

1. **Proactive monitoring:** Heartbeat detection prevents long hangs
2. **Fast recovery:** Per-engine reset is faster than full GPU reset
3. **State capture:** Error state debugging helps identify root cause
4. **Graceful degradation:** Wedged device still allows cleanup
5. **Timeout diversity:** Different operations need different timeouts

### Best Practices

**For Driver Development:**
- Always set appropriate timeouts for long operations
- Use heartbeat monitoring for GPU responsiveness
- Implement proper error state capture for debugging
- Test recovery paths with error injection
- Log enough context for root cause analysis

**For Debugging:**
- Check kernel messages first (dmesg)
- Capture error state before system crash
- Use periodic heartbeat to confirm GPU responsiveness
- Monitor temperature and power for hardware issues
- Test with `inject_errors` module parameter

**For Reliability:**
- Use watchdog timers on long-running operations
- Implement exponential backoff for retries
- Keep error state for post-mortem analysis
- Test error paths regularly
- Monitor for patterns of repeated failures

### Recovery Decision Tree

```
GPU Hang Detected
    │
    ├─ Still responsive to interrupts? NO
    │  └─ Try per-engine reset
    │      ├─ Success? → Resume execution
    │      └─ Fail? → Full GPU reset
    │
    ├─ Still responsive after reset? NO
    │  └─ GPU Wedged
    │      ├─ Prevent new work
    │      ├─ Drain in-flight requests
    │      ├─ Cleanup resources
    │      └─ Report error to userspace
    │
    └─ Recovery successful? YES
       └─ Resume normal operation
```

---

## References

- [GPU Reset Implementation](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/i915/gt/intel_reset.c)
- [Heartbeat Monitoring](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/i915/gt/intel_engine_heartbeat.c)
- [Error State Capture](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/i915/i915_gpu_error.c)
- Related: [09-i915-Fence-Timeline-Study.md](09-i915-Fence-Timeline-Study.md), [04-Power-Management.md](04-Power-Management.md)

---

**Next Steps:**
- Study the GPU reset flow in intel_reset.c
- Review heartbeat implementation in intel_engine_heartbeat.c
- Test error recovery with inject_errors parameter
- Analyze error state dumps from failed operations

---

**Document Status:** ✅ Complete  
**Version:** 1.0  
**Last Updated:** February 8, 2026
