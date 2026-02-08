# DMA Buffer Operations

**Document ID:** 18 | **Status:** Complete | **Last Updated:** February 2026

---

## Executive Summary

DMA (Direct Memory Access) engines perform fast data transfers without CPU intervention. This document covers DMA operations, buffer synchronization, and inter-device sharing.

### Key Topics
- **DMA engines:** Blitter, command streamers
- **Buffer operations:** Copies, fills, transformations
- **Synchronization:** Fences, ordering guarantees
- **Buffer sharing:** DMABUF, external device access

### Performance Metrics
- **DMA bandwidth:** 20-50GB/s (Gen12+)
- **Setup latency:** 100-500µs per operation
- **Copy throughput:** 10+ GB/s sustained

---

## Table of Contents

1. [DMA Engine Architecture](#dma-engine-architecture)
2. [Blitter (BLT) Operations](#blitter-blt-operations)
3. [Buffer Synchronization](#buffer-synchronization)
4. [DMABUF and Buffer Sharing](#dmabuf-and-buffer-sharing)
5. [Performance Optimization](#performance-optimization)
6. [Debugging DMA Issues](#debugging-dma-issues)
7. [Summary & Best Practices](#summary--best-practices)

---

## DMA Engine Architecture

### DMA Engine Types

```plaintext
Intel GPU DMA Engines
┌──────────────────────────────────┐
│ Copy Engine (Data movement)      │
│ ├─ Copies between buffers        │
│ ├─ Bandwidth: 30GB/s             │
│ └─ Power efficient               │
├──────────────────────────────────┤
│ Blitter (2D Operations)          │
│ ├─ Rectangle copies              │
│ ├─ Color fills                   │
│ ├─ Format conversions            │
│ └─ Bandwidth: 40GB/s             │
├──────────────────────────────────┤
│ Render Engine                    │
│ ├─ 3D transformations            │
│ ├─ Shader execution              │
│ └─ Bandwidth: 50GB/s             │
└──────────────────────────────────┘
```

### DMA Buffer Structure

```c
// DMA buffer allocation
struct i915_gem_object {
    struct drm_gem_object base;
    
    // Physical pages
    struct sg_table *pages;
    unsigned long num_pages;
    
    // Caching info
    enum i915_cache_level cache_level;
    
    // Memory domain tracking
    unsigned int read_domains;
    unsigned int write_domain;
    
    // Pinning (for DMA)
    struct list_head pin_list;
    int pinned_count;
};

// Allocate buffer for DMA
struct i915_gem_object *i915_gem_object_create_shmem_dma(
    struct i915_drm_private *i915,
    size_t size)
{
    struct i915_gem_object *obj;
    int ret;
    
    // Allocate GEM object
    obj = i915_gem_object_create_shmem(i915, size);
    if (IS_ERR(obj))
        return obj;
    
    // Pin in memory for DMA
    ret = i915_gem_object_pin_pages(obj);
    if (ret) {
        i915_gem_object_put(obj);
        return ERR_PTR(ret);
    }
    
    return obj;
}
```

---

## Blitter (BLT) Operations

### Blitter Command Submission

```plaintext
BLT Operation Flow
┌──────────────────────────────────┐
│ Application Request (Copy Data)  │
└──────┬───────────────────────────┘
       │
       ├─→ Create BLT Command
       │   ├─ Source buffer address
       │   ├─ Destination buffer address
       │   ├─ Width and height
       │   └─ Pitch and format
       │
       ├─→ Submit to BCS Engine
       │   ├─ Queue command buffer
       │   ├─ Set start offset
       │   └─ Trigger execution
       │
       ├─→ BLT Engine Executes
       │   ├─ Fetch source data
       │   ├─ Process pixels
       │   └─ Write to destination
       │
       └─→ Complete and Signal
           ├─ Set completion flag
           ├─ Generate interrupt
           └─ Notify driver/app
```

### BLT Implementation

```c
// Blitter copy operation
static int i915_blt_copy(struct i915_drm_private *i915,
                        struct drm_i915_gem_object *src,
                        struct drm_i915_gem_object *dst,
                        u32 width, u32 height)
{
    struct i915_request *rq;
    struct i915_vma *src_vma, *dst_vma;
    u32 *cmd;
    int ret;
    
    // 1. Allocate request
    rq = i915_request_create(i915->engines[BCS0]);
    if (IS_ERR(rq))
        return PTR_ERR(rq);
    
    // 2. Bind buffers to GPU memory
    src_vma = i915_vma_instance(src, &i915->ggtt, NULL);
    dst_vma = i915_vma_instance(dst, &i915->ggtt, NULL);
    
    ret = i915_vma_pin(src_vma, 0, 0, PIN_GLOBAL);
    if (ret)
        goto out;
    
    ret = i915_vma_pin(dst_vma, 0, 0, PIN_GLOBAL);
    if (ret)
        goto unpin_src;
    
    // 3. Build BLT command
    cmd = intel_ring_begin(rq, 8);
    if (IS_ERR(cmd)) {
        ret = PTR_ERR(cmd);
        goto unpin_dst;
    }
    
    // BLT_SRC_COPY command
    *cmd++ = (BLT_OPCODE_SRC_COPY_BLT |
             (width - 1) << BLT_WIDTH_SHIFT |
             (height - 1) << BLT_HEIGHT_SHIFT);
    
    *cmd++ = dst_vma->node.start + height * dst->pitch;
    *cmd++ = 0x0;  // Destination offset
    *cmd++ = height << 16 | width;
    *cmd++ = src_vma->node.start;
    *cmd++ = src->pitch;
    *cmd++ = MI_BATCH_BUFFER_END;
    
    intel_ring_advance(rq, cmd);
    
    // 4. Submit for execution
    ret = i915_request_submit(rq);
    
unpin_dst:
    i915_vma_unpin(dst_vma);
unpin_src:
    i915_vma_unpin(src_vma);
out:
    i915_request_put(rq);
    return ret;
}

// Rectangle fill operation
static int i915_blt_fill(struct i915_drm_private *i915,
                        struct drm_i915_gem_object *dst,
                        u32 color, u32 width, u32 height)
{
    struct i915_request *rq;
    struct i915_vma *dst_vma;
    u32 *cmd;
    int ret;
    
    rq = i915_request_create(i915->engines[BCS0]);
    if (IS_ERR(rq))
        return PTR_ERR(rq);
    
    dst_vma = i915_vma_instance(dst, &i915->ggtt, NULL);
    ret = i915_vma_pin(dst_vma, 0, 0, PIN_GLOBAL);
    if (ret)
        goto out;
    
    // BLT fill command
    cmd = intel_ring_begin(rq, 6);
    if (IS_ERR(cmd)) {
        ret = PTR_ERR(cmd);
        goto unpin;
    }
    
    *cmd++ = (BLT_OPCODE_COLOR_BLT |
             (width - 1) << BLT_WIDTH_SHIFT |
             (height - 1) << BLT_HEIGHT_SHIFT);
    *cmd++ = dst_vma->node.start;
    *cmd++ = dst->pitch;
    *cmd++ = color;  // Fill color
    *cmd++ = MI_BATCH_BUFFER_END;
    
    intel_ring_advance(rq, cmd);
    ret = i915_request_submit(rq);
    
unpin:
    i915_vma_unpin(dst_vma);
out:
    i915_request_put(rq);
    return ret;
}
```

---

## Buffer Synchronization

### Fence-Based Synchronization

```plaintext
DMA Buffer Synchronization
┌──────────────────────────────────┐
│ Application Issues DMA Copy      │
│ "Copy 1MB from A to B"           │
└──────┬───────────────────────────┘
       │
       ├─→ Driver Creates Fence
       │   └─ Tracks operation completion
       │
       ├─→ DMA Operation Queued
       │   └─ Added to ring buffer
       │
       ├─→ GPU Executes
       │   └─ Data transferred
       │
       ├─→ Fence Signals
       │   ├─ Operation complete
       │   └─ Data valid
       │
       └─→ Application Waits
           ├─ Polls or sleeps on fence
           └─ Uses result when ready
```

### Memory Domain Tracking

```c
// Memory domain tracking
struct i915_gem_object {
    // Cache domain tracking
    unsigned int read_domains;    // Who has read permission
    unsigned int write_domain;    // Who has write permission
};

// Memory domain definitions
#define I915_GEM_DOMAIN_CPU        (1 << 0)  // CPU cache
#define I915_GEM_DOMAIN_RENDER     (1 << 1)  // GPU render engine
#define I915_GEM_DOMAIN_SAMPLER    (1 << 2)  // GPU texture sampler
#define I915_GEM_DOMAIN_COMMAND    (1 << 3)  // GPU command parser
#define I915_GEM_DOMAIN_INSTRUCTION (1 << 4) // GPU instruction cache
#define I915_GEM_DOMAIN_VERTEX     (1 << 5)  // GPU vertex cache
#define I915_GEM_DOMAIN_GTT        (1 << 6)  // GTT mappings

// Ensure buffer is in correct domain before access
static int i915_gem_object_set_domain(
    struct i915_gem_object *obj,
    unsigned int read_domains,
    unsigned int write_domain)
{
    if (obj->write_domain == write_domain &&
        (obj->read_domains & read_domains) == read_domains)
        return 0;  // Already in correct domain
    
    // Need to flush caches
    if (obj->write_domain) {
        // Flush GPU write cache
        intel_engine_flush_submission(
            write_domain_to_engine(obj->write_domain));
    }
    
    // Update domain tracking
    obj->read_domains |= read_domains;
    obj->write_domain = write_domain;
    
    return 0;
}

// Wait for domain transition
static int i915_gem_object_wait_rendering(
    struct i915_gem_object *obj,
    bool write_domain)
{
    struct i915_request *rq;
    long ret;
    
    // Get last request for this object
    rq = i915_gem_object_get_active_request(obj);
    if (!rq)
        return 0;
    
    // Wait for GPU to finish with object
    if (write_domain)
        ret = dma_fence_wait(&rq->fence, false);
    else
        ret = dma_fence_wait(&rq->fence, false);
    
    i915_request_put(rq);
    return ret < 0 ? ret : 0;
}
```

---

## DMABUF and Buffer Sharing

### DMABUF Export

```c
// Export GEM object as DMABUF for sharing
static int i915_gem_prime_get_fd(struct drm_i915_gem_object *obj)
{
    struct dma_buf *dma_buf;
    int fd;
    
    // Check if already exported
    dma_buf = drm_gem_dmabuf_export(&obj->base);
    if (IS_ERR(dma_buf))
        return PTR_ERR(dma_buf);
    
    // Create file descriptor for DMABUF
    fd = dma_buf_fd(dma_buf, O_CLOEXEC);
    
    if (fd < 0)
        dma_buf_put(dma_buf);
    
    return fd;
}

// Import DMABUF from other driver
static struct i915_gem_object *i915_gem_prime_import(
    struct drm_device *dev,
    struct dma_buf *dma_buf)
{
    struct i915_drm_private *i915 = to_i915(dev);
    struct i915_gem_object *obj;
    int ret;
    
    // Create GEM object for imported DMABUF
    obj = i915_gem_object_alloc();
    if (!obj)
        return ERR_PTR(-ENOMEM);
    
    // Attach to DMABUF
    obj->dma_buf = dma_buf;
    ret = dma_buf_attach(dma_buf, &i915->drm.pdev->dev,
                        &i915_dmabuf_ops);
    if (ret) {
        kfree(obj);
        return ERR_PTR(ret);
    }
    
    return obj;
}

// DMABUF ops
static const struct dma_buf_ops i915_dmabuf_ops = {
    .attach = i915_dmabuf_attach,
    .detach = i915_dmabuf_detach,
    .map_dma_buf = i915_dmabuf_map,
    .unmap_dma_buf = i915_dmabuf_unmap,
    .release = i915_dmabuf_release,
    .begin_cpu_access = i915_dmabuf_begin_cpu_access,
    .end_cpu_access = i915_dmabuf_end_cpu_access,
};
```

### DMABUF Synchronization

```c
// Synchronize imported DMABUF before use
static int i915_dmabuf_sync(struct dma_buf *dma_buf,
                            bool write_access)
{
    struct dma_buf_attachment *attach;
    struct sg_table *sgt;
    int ret;
    
    // Get attachment for this driver
    attach = dma_buf_attachment_get(dma_buf);
    if (!attach)
        return -ENODEV;
    
    // Request DMA mapping
    sgt = dma_buf_map_attachment(attach, write_access ?
                                 DMA_BIDIRECTIONAL :
                                 DMA_FROM_DEVICE);
    if (IS_ERR(sgt)) {
        ret = PTR_ERR(sgt);
        goto out;
    }
    
    // Sync for GPU access
    dma_sync_sg_for_device(&i915->drm.pdev->dev,
                          sgt->sgl, sgt->nents,
                          write_access ?
                          DMA_BIDIRECTIONAL :
                          DMA_FROM_DEVICE);
    
    ret = 0;
    
out:
    dma_buf_attachment_put(attach);
    return ret;
}
```

---

## Performance Optimization

### Copy Performance Tuning

```c
// Optimize copy operations
static int i915_blt_copy_optimized(
    struct i915_drm_private *i915,
    struct i915_gem_object *src,
    struct i915_gem_object *dst)
{
    size_t size = src->base.size;
    
    // 1. Choose optimal copy method
    if (size < 4096) {
        // Small copy: use CPU or small GPU copy
        return i915_gem_object_pwrite_fast(src, dst);
    } else if (size < 1MB) {
        // Medium: BLT engine
        return i915_blt_copy(i915, src, dst, 
                            src->stride, src->height);
    } else {
        // Large: render engine or DMA engine
        return i915_render_copy(i915, src, dst);
    }
}

// Batch multiple operations
struct blt_batch {
    struct i915_request *rq;
    u32 *cmd;
    int cmd_count;
};

// Add operation to batch
static void add_blt_copy_to_batch(struct blt_batch *batch,
                                  struct i915_gem_object *src,
                                  struct i915_gem_object *dst)
{
    // Add BLT command without submitting
    batch->cmd_count++;
    
    // Only submit when batch is full
    if (batch->cmd_count >= MAX_BLT_OPS) {
        i915_request_submit(batch->rq);
        batch->cmd_count = 0;
    }
}
```

---

## Debugging DMA Issues

### Common DMA Problems

| Issue | Cause | Solution |
|-------|-------|----------|
| **Copy fails** | Source not pinned | Pin pages before DMA |
| **Slow copy** | Wrong engine used | Use render for large buffers |
| **Data corruption** | Sync race condition | Add proper fences |
| **Timeout** | Stuck operation | Check BCS engine state |
| **Memory errors** | Invalid addresses | Validate VMA |

### DMA Debugging Tools

```bash
# Check BCS engine status
cat /sys/kernel/debug/dri/0/i915_bcs_info

# Monitor DMA operations
cat /sys/kernel/debug/dri/0/i915_requests

# Enable DMA tracing
echo 1 > /sys/kernel/debug/tracing/events/i915/enable
cat /sys/kernel/debug/tracing/trace_pipe | grep blt
```

---

## Summary & Best Practices

### Key Takeaways

1. **Engine selection:** Choose right engine for workload
2. **Pinning:** Always pin buffers before DMA
3. **Synchronization:** Use fences for safety
4. **DMABUF:** Standard for inter-device sharing
5. **Optimization:** Batch operations for efficiency

### Best Practices

**For DMA Operations:**
- Pin pages before DMA access
- Use proper memory domains
- Add explicit synchronization fences
- Batch small operations
- Validate addresses

**For Performance:**
- Use BCS for simple copies
- Use render for complex transforms
- Batch multiple operations
- Minimize CPU involvement
- Profile actual performance

**For Reliability:**
- Handle DMABUF import/export
- Validate synchronization
- Test with various sizes
- Monitor engine status
- Log DMA errors

---

## References

- [DMA Operations](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/i915/gt/intel_bcs.c)
- [DMABUF Support](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/i915/gem/i915_gem_dmabuf.c)
- Related: [01-Memory-Management.md](01-Memory-Management.md), [06-Ring-Buffer-Management.md](06-Ring-Buffer-Management.md)

---

**Next Steps:**
- Study BLT engine operation in intel_bcs.c
- Implement optimized copy for your workload
- Test DMABUF sharing with other drivers
- Profile DMA performance

---

**Document Status:** ✅ Complete  
**Version:** 1.0  
**Last Updated:** February 8, 2026
