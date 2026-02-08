# Interrupt Handling and Management

**Document ID:** 12 | **Status:** Complete | **Last Updated:** February 2026

---

## Executive Summary

Interrupt handling is fundamental to i915's real-time responsiveness. This document explains how i915 detects GPU events, manages interrupt vectors, and ensures timely processing of GPU completion signals.

### Key Topics
- **Interrupt types:** Render, display, memory, error, thermal
- **Interrupt delivery:** MSI, MSI-X, legacy INTx
- **Handler architecture:** ISR, deferred work, tasklets
- **Performance:** Interrupt coalescing, optimization strategies

### Performance Metrics
- **Interrupt latency:** Sub-microsecond detection of GPU events
- **Handler overhead:** ~100 cycles per interrupt
- **Completion latency:** <1ms from interrupt to fence signaling

---

## Table of Contents

1. [Interrupt Architecture](#interrupt-architecture)
2. [Interrupt Types and Sources](#interrupt-types-and-sources)
3. [Interrupt Delivery Mechanisms](#interrupt-delivery-mechanisms)
4. [ISR and Handler Implementation](#isr-and-handler-implementation)
5. [Deferred Work and Tasklets](#deferred-work-and-tasklets)
6. [Interrupt Coalescing](#interrupt-coalescing)
7. [Interrupt Optimization Strategies](#interrupt-optimization-strategies)
8. [Debugging and Monitoring](#debugging-and-monitoring)
9. [Summary & Best Practices](#summary--best-practices)

---

## Interrupt Architecture

### High-Level Flow

```plaintext
┌─────────────────────────────────────┐
│ GPU Event Occurs                    │
│ (e.g., Render Complete)             │
└──────┬──────────────────────────────┘
       │
       ├─→ GPU Sets Status Register
       │   └─ Marks interrupt pending
       │
       ├─→ GPU Asserts IRQ Line
       │   ├─ Legacy INTx: Asserts line
       │   ├─ MSI: Sends message
       │   └─ MSI-X: Sends vector
       │
       ├─→ CPU Interrupt Controller
       │   ├─ Routes to CPU
       │   └─ Preempts current code
       │
       ├─→ ISR Handler (Interrupt Service Routine)
       │   ├─ Runs in interrupt context
       │   ├─ Minimal processing
       │   ├─ Schedules deferred work
       │   └─ Returns IRQ_HANDLED
       │
       ├─→ IRQ Handler Thread
       │   ├─ Runs at high priority
       │   ├─ Processes interrupts
       │   ├─ Signals fences
       │   ├─ Updates completion trackers
       │   └─ Wakes waiters
       │
       └─→ Wait Queue Woken
           └─ Userspace notified
```

### Interrupt Controller Integration

```c
// PCIe interrupt configuration
struct intel_runtime_pm {
    struct pci_dev *pdev;
    struct pci_irq_affinity_mask irq_affinity;
};

// Initialize interrupt handling
static int i915_driver_load(struct i915_drm_private *i915)
{
    // 1. Request interrupt vector
    int irq = pci_alloc_irq_vectors(i915->drm.pdev,
                                     1, 1,
                                     PCI_IRQ_MSIX |
                                     PCI_IRQ_MSI |
                                     PCI_IRQ_LEGACY);
    
    i915->drm.pdev->irq = irq;
    
    // 2. Register ISR handler
    devm_request_irq(i915->drm.dev, irq,
                     i915_irq_handler,
                     IRQF_SHARED,
                     "i915", i915);
    
    // 3. Enable interrupts
    i915_irq_enable(i915);
    
    return 0;
}
```

---

## Interrupt Types and Sources

### GPU Interrupt Types

```plaintext
GPU Interrupts (IIR - Interrupt Identity Register)
    │
    ├─ Render Completion Interrupts
    │  ├─ RCS completion (0x1)
    │  ├─ BCS completion (0x2)
    │  ├─ VCS completion (0x4)
    │  └─ VECS completion (0x8)
    │
    ├─ Display Interrupts
    │  ├─ Pipe A VBLANK (0x100)
    │  ├─ Pipe B VBLANK (0x200)
    │  ├─ Pipe C VBLANK (0x400)
    │  ├─ Hotplug detect (0x800)
    │  └─ Sprite flip (0x1000)
    │
    ├─ Memory Interrupts
    │  ├─ GGTT PTE write (0x2000)
    │  ├─ PPGTT page fault (0x4000)
    │  └─ Out of memory (0x8000)
    │
    ├─ Error Interrupts
    │  ├─ GPU hang timeout (0x10000)
    │  ├─ Memory error (0x20000)
    │  ├─ L3 parity error (0x40000)
    │  └─ FIFO underrun (0x80000)
    │
    ├─ Thermal Interrupts
    │  ├─ Temperature warning (0x100000)
    │  ├─ Frequency threshold (0x200000)
    │  └─ Power limit (0x400000)
    │
    └─ System Interrupts
       ├─ Debug message (0x800000)
       ├─ Semaphore signal (0x1000000)
       └─ PCIe link change (0x2000000)
```

### Per-Engine Interrupt Status

```c
// Engine-specific interrupt registers
struct intel_engine_cs {
    struct {
        u32 IER;     // Interrupt Enable Register
        u32 IIR;     // Interrupt Identity Register
        u32 ISR;     // Interrupt Status Register
        u32 IMR;     // Interrupt Mask Register
    } irq;
    
    unsigned long irq_flags;
};

// Interrupt status bits per engine
#define RENDER_RING_TAIL_ADVANCED  (1 << 0)  // RCS batch complete
#define RENDER_RING_BUFFER_WRAP    (1 << 1)  // Ring wrapped around
#define RENDER_RING_CTX_SWITCH     (1 << 2)  // Context switch done
#define RENDER_SYNC_STATUS         (1 << 3)  // Synchronization event
#define RENDER_SEMAPHORE_SLOW      (1 << 4)  // Semaphore timeout
```

---

## Interrupt Delivery Mechanisms

### Message Signaled Interrupts (MSI)

Modern GPUs use MSI or MSI-X for reliable, high-performance interrupt delivery:

```plaintext
MSI Delivery
┌─────────────────────────────────────┐
│ GPU Event                           │
└─────────────┬───────────────────────┘
              │
              ├─→ GPU composes MSI message
              │   ├─ Address field
              │   ├─ Data field
              │   └─ Timestamp
              │
              ├─→ Writes MSI packet
              │   └─ To memory address in PCIe
              │
              ├─→ Memory controller receives
              │   └─ Triggers CPU interrupt
              │
              └─→ ISR Handled
                  └─ MSI already acknowledged
```

```c
// Enable MSI or MSI-X
static int i915_pci_probe(struct pci_dev *pdev,
                          const struct pci_device_id *ent)
{
    struct intel_runtime_pm *rpm;
    int ret;
    
    // Try MSI-X first (multiple vectors)
    ret = pci_alloc_irq_vectors(pdev,
                                I915_NUM_ENGINES + 1,
                                I915_NUM_ENGINES + 1,
                                PCI_IRQ_MSIX);
    if (ret > 0) {
        dev_info(&pdev->dev, "Using MSI-X with %d vectors\n", ret);
        goto success;
    }
    
    // Fall back to MSI (single vector)
    ret = pci_alloc_irq_vectors(pdev, 1, 1, PCI_IRQ_MSI);
    if (ret > 0) {
        dev_info(&pdev->dev, "Using MSI\n");
        goto success;
    }
    
    // Fall back to legacy INTx
    ret = pci_alloc_irq_vectors(pdev, 1, 1, PCI_IRQ_LEGACY);
    if (ret > 0) {
        dev_warn(&pdev->dev, "Using legacy INTx (slow)\n");
        goto success;
    }
    
    return -ENODEV;
    
success:
    pdev->irq = pci_irq_vector(pdev, 0);
    return 0;
}
```

### Legacy INTx Interrupts

Older systems use line-based interrupts:

```plaintext
INTx Delivery (Slower)
┌─────────────────────────────────────┐
│ GPU Event                           │
└─────────────┬───────────────────────┘
              │
              ├─→ GPU asserts INT# line
              │   └─ Holds line low
              │
              ├─→ PCIe root complex sees
              │   └─ Routes through IOAPIC
              │
              ├─→ IOAPIC routes to CPU
              │   └─ May be shared with other devices
              │
              ├─→ CPU handles interrupt
              │   ├─ May need to poll devices
              │   └─ Typically slower
              │
              └─→ GPU must deassert line
                  └─ After ack from driver
```

### MSI-X Vector Assignment

For systems supporting MSI-X, assign separate vectors to different events:

```c
// MSI-X vector allocation
#define I915_MSI_RENDER_COMPLETE  0
#define I915_MSI_DISPLAY_HOTPLUG  1
#define I915_MSI_THERMAL_WARNING  2
#define I915_MSI_ERROR_RECOVERY   3
#define I915_NUM_MSIX_VECTORS     4

// Request vectors
static int setup_msix_vectors(struct i915_drm_private *i915)
{
    struct pci_dev *pdev = i915->drm.pdev;
    int ret;
    
    ret = pci_alloc_irq_vectors(pdev,
                                I915_NUM_MSIX_VECTORS,
                                I915_NUM_MSIX_VECTORS,
                                PCI_IRQ_MSIX);
    
    if (ret == I915_NUM_MSIX_VECTORS) {
        for (int i = 0; i < I915_NUM_MSIX_VECTORS; i++) {
            int irq = pci_irq_vector(pdev, i);
            devm_request_irq(i915->drm.dev, irq,
                           i915_msix_handler[i],
                           IRQF_NO_AUTOEN,
                           irq_names[i], i915);
        }
        return 0;
    }
    
    return ret;
}
```

---

## ISR and Handler Implementation

### Fast Path ISR Handler

The ISR must be fast to avoid stalling CPU execution:

```c
// Main interrupt handler - runs in interrupt context
static irqreturn_t i915_irq_handler(int irq, void *arg)
{
    struct i915_drm_private *i915 = arg;
    int ret = IRQ_NONE;
    
    // 1. Disable interrupts while processing
    // (kernel handles re-enabling after handler returns)
    
    // 2. Read and clear interrupt status
    u32 iir = intel_uncore_read(&i915->uncore, GEN8_DE_IIR);
    if (!iir) {
        // No interrupt - must not be ours
        return IRQ_NONE;
    }
    
    // 3. Clear interrupt bits
    intel_uncore_write(&i915->uncore, GEN8_DE_IIR, iir);
    intel_uncore_posting_read(&i915->uncore, GEN8_DE_IIR);
    
    // 4. Quick sanity check
    if (unlikely(iir & GEN8_DE_IIR_ERROR_BITS)) {
        dev_err(i915->drm.dev, "GPU error detected: 0x%08x\n", iir);
        // Queue error handler, don't process here
        queue_work(system_unbound_wq, &i915->gpu_error.work);
    }
    
    // 5. Dispatch work to handler thread for real processing
    queue_irq_work(&i915->irq_work);
    
    return IRQ_HANDLED;
}
```

### Deferred ISR Processing

The actual work is done outside interrupt context:

```c
// Run in handler thread (higher context level)
static void i915_irq_work_handler(struct irq_work *work)
{
    struct i915_drm_private *i915 =
        container_of(work, struct i915_drm_private,
                     irq_work.work);
    
    // Read engine-specific interrupt status
    for_each_engine(engine, &i915->gt, i) {
        u32 iir = intel_uncore_read(&i915->uncore,
                                    RING_ISR(engine->mmio_base));
        
        if (!(iir & engine->irq.enabled))
            continue;
        
        // Clear status bits
        intel_uncore_write(&i915->uncore,
                          RING_ISR(engine->mmio_base),
                          iir);
        
        // Process engine-specific interrupts
        if (iir & RENDER_RING_TAIL_ADVANCED) {
            // Render job completed
            intel_engine_signal_breadcrumbs(engine);
        }
        
        if (iir & RENDER_RING_CTX_SWITCH) {
            // Context switch completed
            intel_execlists_process_context_switch(engine);
        }
    }
    
    // Handle display interrupts
    if (iir & GEN8_DE_HOTPLUG_MASK) {
        intel_display_hotplug_handler(i915);
    }
}
```

---

## Deferred Work and Tasklets

### Tasklet-Based Processing

Traditional approach using tasklets for interrupt processing:

```c
// Tasklet for deferred processing
struct intel_engine_cs {
    struct tasklet_struct irq_tasklet;
    unsigned long irq_posted;
};

// Schedule tasklet from ISR
static void engine_irq_tasklet_handler(struct tasklet_struct *t)
{
    struct intel_engine_cs *engine =
        from_tasklet(engine, t, irq_tasklet);
    
    // Process all completed requests
    intel_engine_process_completions(engine);
    
    // Service seqno-based waiters
    intel_engine_signal_breadcrumbs(engine);
}

// Called from ISR
static void intel_engine_irq_signal(struct intel_engine_cs *engine)
{
    struct tasklet_struct *tl = &engine->irq_tasklet;
    
    // Mark tasklet as needing run
    __tasklet_hi_schedule_first(tl);
}
```

### Request Completion Processing

```c
// Process completions for an engine
static void intel_engine_process_completions(
    struct intel_engine_cs *engine)
{
    struct i915_request *rq, *rn;
    
    // Walk all active requests
    list_for_each_entry_safe(rq, rn,
                             &engine->active.requests,
                             sched.link) {
        // Check if request has completed
        if (!dma_fence_is_signaled(&rq->fence))
            break;  // Not complete, so earlier ones aren't either
        
        // Mark complete
        __list_del_entry(&rq->sched.link);
        
        // Release resources
        i915_request_retire(rq);
        i915_request_put(rq);
    }
}

// Signal breadcrumb (completion) waiters
static void intel_engine_signal_breadcrumbs(
    struct intel_engine_cs *engine)
{
    struct list_head *pos, *next;
    unsigned long flags;
    
    spin_lock_irqsave(&engine->breadcrumb.irq_lock, flags);
    
    // Wake all processes waiting on this engine
    list_for_each_safe(pos, next,
                       &engine->breadcrumb.sig_list) {
        struct i915_request *rq =
            list_entry(pos, struct i915_request,
                      signal_link);
        
        // Signal completion
        dma_fence_signal(&rq->fence);
        
        // Wake waiters
        wake_up(&rq->submit.wait);
    }
    
    spin_unlock_irqrestore(&engine->breadcrumb.irq_lock, flags);
}
```

---

## Interrupt Coalescing

### Interrupt Rate Limiting

Too many interrupts can waste CPU cycles. Coalescing reduces interrupt rate:

```plaintext
Without Coalescing
────────────────────────────────────────────
Job 1 done  Job 2 done  Job 3 done  Job 4 done
   IRQ         IRQ         IRQ         IRQ
────────────────────────────────────────────
100% CPU usage, high latency, power consumption

With Coalescing (e.g., 2ms or 4 jobs)
────────────────────────────────────────────
Job 1,2,3,4 done
       IRQ (batched)
────────────────────────────────────────────
Lower CPU, higher latency trade-off
```

### Implementing Coalescing

```c
// Interrupt coalescing parameters
struct intel_irq_coalesce {
    bool enabled;
    unsigned int timeout_ms;  // Wait up to N ms
    unsigned int batch_size;  // Wait for N jobs
    unsigned long last_irq;
};

// Coalesced interrupt handler
static void i915_irq_handler_coalesced(
    struct intel_engine_cs *engine)
{
    struct intel_irq_coalesce *coalesce = &engine->irq_coalesce;
    unsigned long now = jiffies;
    unsigned int completed;
    
    // Count completed requests since last interrupt
    completed = intel_engine_count_completions(engine);
    
    // Check if we should defer this interrupt
    if (coalesce->enabled) {
        unsigned long elapsed = jiffies_to_msecs(now - coalesce->last_irq);
        
        // Defer if too soon and below batch size
        if (elapsed < coalesce->timeout_ms &&
            completed < coalesce->batch_size) {
            // Don't process yet, let more jobs complete
            return;
        }
    }
    
    // Process now
    intel_engine_process_completions(engine);
    coalesce->last_irq = now;
}
```

### Adaptive Coalescing

Adjust coalescing based on workload:

```c
// Dynamically adjust coalescing
static void update_irq_coalescing(struct intel_engine_cs *engine)
{
    struct intel_irq_coalesce *coalesce = &engine->irq_coalesce;
    unsigned long load = engine->avg_load;  // Load percentage
    
    if (load > 80) {
        // High load - coalesce more aggressively
        coalesce->timeout_ms = 5;
        coalesce->batch_size = 8;
    } else if (load > 50) {
        // Medium load - balanced coalescing
        coalesce->timeout_ms = 2;
        coalesce->batch_size = 4;
    } else {
        // Low load - minimal coalescing
        coalesce->timeout_ms = 0;
        coalesce->batch_size = 1;
    }
}
```

---

## Interrupt Optimization Strategies

### Affinity and CPU Pinning

Pin interrupts to specific CPU cores for cache locality:

```bash
# View interrupt affinity
cat /proc/irq/47/smp_affinity

# Pin interrupt to CPU 2
echo 4 > /proc/irq/47/smp_affinity

# Pin interrupt to CPUs 2-3
echo c > /proc/irq/47/smp_affinity

# Enable IRQ balance awareness
echo on > /proc/irq/47/rps_hint
```

### Interrupt Balancing

Modern kernels can auto-balance interrupts:

```c
// Request affinity mask
static int setup_irq_affinity(struct i915_drm_private *i915)
{
    struct pci_dev *pdev = i915->drm.pdev;
    struct pci_irq_affinity desc = {
        .pre_vectors = 0,        // Don't pre-reserve
        .post_vectors = 0,       // Don't post-reserve
        .nr_sets = 1,            // Single set of vectors
    };
    
    // Linux will distribute vectors across CPUs
    int ret = pci_alloc_irq_vectors_affinity(pdev,
                                              1, 1,
                                              PCI_IRQ_MSIX | PCI_IRQ_MSI,
                                              &desc);
    
    return ret;
}
```

### Interrupt Nesting and Priority

Handle high-priority interrupts first:

```c
// Set handler priority
static int register_irq_handlers(struct i915_drm_private *i915)
{
    // Critical error handler - high priority
    devm_request_irq(i915->drm.dev,
                     i915->irq_error,
                     i915_error_irq_handler,
                     IRQF_NODELAY | IRQF_TRIGGER_HIGH,
                     "i915-error", i915);
    
    // Render completion - normal priority
    devm_request_irq(i915->drm.dev,
                     i915->irq_render,
                     i915_render_irq_handler,
                     IRQF_SHARED,
                     "i915-render", i915);
    
    // Display - low priority
    devm_request_irq(i915->drm.dev,
                     i915->irq_display,
                     i915_display_irq_handler,
                     IRQF_SHARED | IRQF_LOWPRI,
                     "i915-display", i915);
}
```

---

## Debugging and Monitoring

### Interrupt Statistics

```bash
# View interrupt count per CPU
cat /proc/interrupts | grep i915

# Monitor interrupt rate in real-time
watch -n 1 'grep i915 /proc/interrupts'

# Count interrupts for IRQ 47
watch "grep 47 /proc/interrupts"
```

### Enable Interrupt Tracing

```bash
# Enable i915 interrupt tracing
echo 1 > /sys/kernel/debug/tracing/events/i915/enable

# Read tracing output
cat /sys/kernel/debug/tracing/trace_pipe | grep i915_irq

# View trace statistics
cat /sys/kernel/debug/tracing/events/i915/i915_irq_handler/stat
```

### Debug Output

```c
// Log interrupt details
#define INTEL_IRQ_DEBUG(engine, fmt, ...)                   \
    do {                                                    \
        if (INTEL_DEBUG & DEBUG_INTERRUPTS) {              \
            dev_dbg(engine->i915->drm.dev,                 \
                    "IRQ %s: " fmt,                        \
                    engine->name, ##__VA_ARGS__);          \
        }                                                   \
    } while (0)

// Usage
INTEL_IRQ_DEBUG(engine, "Processing %d completions\n", count);
```

### Interrupt Rate Monitoring

```c
// Track interrupt rate
struct intel_irq_stats {
    unsigned long count;
    unsigned long last_sec;
    unsigned int rate_per_sec;
};

// Update stats
static void update_irq_stats(struct intel_engine_cs *engine)
{
    struct intel_irq_stats *stats = &engine->irq_stats;
    unsigned long now = jiffies;
    
    stats->count++;
    
    if (time_after(now, stats->last_sec + HZ)) {
        stats->rate_per_sec = stats->count;
        stats->count = 0;
        stats->last_sec = now;
        
        if (stats->rate_per_sec > INTEL_IRQ_MAX_RATE) {
            dev_warn(engine->i915->drm.dev,
                    "%s: high IRQ rate: %d/sec\n",
                    engine->name, stats->rate_per_sec);
        }
    }
}
```

---

## Summary & Best Practices

### Key Takeaways

1. **Interrupt type matters:** Use MSI-X when available for better latency
2. **Keep handlers fast:** Defer actual work to handler threads
3. **Batch completions:** Coalescing reduces overhead
4. **Monitor rates:** Excessive interrupts indicate inefficiency
5. **Pin affinity:** CPU-local processing improves cache efficiency

### Best Practices

**For Performance:**
- Prefer MSI-X > MSI > INTx for interrupt delivery
- Keep ISR handler minimal (read status, schedule work)
- Use tasklets or workqueues for heavy lifting
- Implement adaptive interrupt coalescing
- Monitor interrupt rates for anomalies

**For Debugging:**
- Enable interrupt tracing for detailed analysis
- Monitor /proc/interrupts for rate changes
- Check CPU affinity for optimal placement
- Log interrupt details with timestamps
- Test with high-load workloads

**For Reliability:**
- Verify interrupt delivery in probe path
- Test with different interrupt delivery modes
- Handle interrupt storms gracefully
- Monitor for spurious interrupts
- Validate handler registration

### Performance Tuning Checklist

| Optimization | Impact | Complexity |
|-------------|--------|-----------|
| **MSI vs INTx** | 10-20% | Low |
| **Handler affinity** | 5-15% | Low |
| **Interrupt coalescing** | 10-30% | Medium |
| **Tasklet batching** | 5-10% | Medium |
| **MSI-X per-engine** | 20-40% | High |

---

## References

- [Interrupt Controller](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/i915/i915_irq.c)
- [Tasklet Processing](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/kernel/softirq.c)
- [MSI Configuration](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/pci/msi.c)
- Related: [12-Error-Handling-Recovery.md](11-Error-Handling-Recovery.md), [03-Context-Management.md](03-Context-Management.md)

---

**Next Steps:**
- Study interrupt handler flow in i915_irq.c
- Profile interrupt rate with high-load workloads
- Test with different MSI configurations
- Implement custom interrupt tracing

---

**Document Status:** ✅ Complete  
**Version:** 1.0  
**Last Updated:** February 8, 2026
