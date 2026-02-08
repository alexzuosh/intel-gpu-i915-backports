# CPU-GPU Coherency Quick Reference

**Document:** Quick guide to understanding and implementing CPU-GPU coherency  
**Related:** [06a-CPU-GPU-Coherency.md](06a-CPU-GPU-Coherency.md)  
**Audience:** GPU driver engineers, memory subsystem engineers

---

## TL;DR: Choose Your Path

### Integrated GPU (iGPU) - Intel Arc, Iris, UHD

```
Use: I915_CACHE_LLC (default)

Sync with:
  - store-release (CPU)
  - load-acquire (GPU)

Benefit:
  ✓ Automatic coherency via shared L3
  ✓ Fast: ~100ns latency
  ✓ Simple: hardware handles it

Example:
  ggtt_map(obj, I915_CACHE_LLC);
  store-release(&obj->flag, 1);
  // GPU sees latest value immediately
```

### Discrete GPU (dGPU) - Arc A770+

```
Use: I915_BO_GPU_CACHED (accept non-coherent)

Sync with:
  - clflush(buffer)       // Flush CPU cache
  - mb()                  // Full barrier
  - DMA transfer          // Copy to VRAM
  - wait_for_dma()        // Verify complete
  - PIPE_CONTROL flush    // GPU barrier

Benefit:
  ✓ Explicit control
  ✓ Predictable performance
  ✓ Clear semantics

Example:
  alloc_dma_buffer();
  write_data(buffer);
  clflush(buffer);        // Flush caches
  mb();
  initiate_dma(buf→vram);
  wait_dma_complete();
  gpu_execute(kernel);    // Operates on VRAM
```

---

## Decision Matrix

### When to Use Which Approach

| Requirement | iGPU Approach | dGPU Approach |
|------------|--------------|--------------|
| **Shared data** | Use LLC | Copy before/after |
| **Frequent sync** | store-release | DMA overhead |
| **Low latency** | 100ns | 10-100μs |
| **Simple code** | Yes | Complex |
| **Predictable perf** | Varies | Stable |
| **CPU-GPU share** | Yes | No (separate mem) |

---

## Common Patterns

### Pattern 1: iGPU Coherent Read (Fast)

```c
// CPU writes, GPU reads
struct drm_i915_gem_object *obj;

// Set up as coherent
i915_gem_object_set_cache_level(obj, I915_CACHE_LLC);

// CPU side: Write data
cpu_write_data(obj, value);
smp_store_release(&obj->valid, 1);  // Release to GPU

// GPU side: Read (automatic coherency)
// GPU reads via shared L3 - NO clflush needed!
```

**Performance:** ~100ns latency (cache-to-cache)

### Pattern 2: dGPU Non-Coherent Copy

```c
// CPU prepares data, GPU processes, CPU reads result
struct drm_i915_gem_object *sys_buf;  // System RAM
struct drm_i915_gem_object *vram_buf; // GPU VRAM

// CPU writes to system RAM
cpu_write_data(sys_buf, value);

// Prepare for DMA copy
clflush_cache_range(cpu_virt_addr(sys_buf), size);  // Flush caches
smp_mb();                              // Full barrier

// DMA copy CPU RAM → GPU VRAM
dma_transfer(sys_buf → vram_buf);
wait_for_dma_complete();               // Sync point

// GPU reads from VRAM (its local memory)
submit_gpu_work(vram_buf);             // GPU has data
gpu_wait_completion();

// DMA copy GPU VRAM → CPU RAM
dma_transfer(vram_buf → sys_buf);
wait_for_dma_complete();

// CPU reads result
result = cpu_read_data(sys_buf);
```

**Performance:** 10-100μs total (DMA setup + transfer)

### Pattern 3: Mixed Mode (iGPU shared + dGPU local)

```c
// Discrete GPU with some shared data

// Shared command structures (frequent updates)
struct cmd_struct *cmd = alloc_coherent(I915_CACHE_LLC);

// GPU-local compute data (bulk transfer once)
struct data_buf *data = alloc_lmem();

// Update commands frequently (coherent)
update_command_ring(cmd);  // store-release implicit
smp_store_release(&cmd->ready, 1);

// Transfer bulk data once (DMA)
dma_transfer(user_buf → data);
wait_dma_complete();

// GPU executes using both:
// - Commands via coherent access (fast)
// - Data from local VRAM (fast GPU access)
gpu_submit(cmd, data);
```

---

## Debugging Checklist

### iGPU Coherency Problems

- [ ] Object marked `I915_CACHE_LLC`?
- [ ] Using `store-release` / `load-acquire`?
- [ ] Memory barrier between CPU write and GPU read?
- [ ] Check `/sys/kernel/debug/dri/0/i915_gem_objects` flags
- [ ] Enable debug logging: `echo 'drm:*' > /proc/dynamic_debug/control`

### dGPU Coherency Problems

- [ ] `clflush()` called before DMA?
- [ ] `mb()` called after DMA?
- [ ] Waited for DMA complete with `wait_for_completion()`?
- [ ] GPU PIPE_CONTROL flush before dependent loads?
- [ ] Check GPU stalled on fence?

### General Coherency Debugging

```bash
# Check cache levels
cat /sys/kernel/debug/dri/0/i915_gem_objects | grep cache_level

# Enable coherency tracing
echo "file drivers/gpu/drm/i915/gem/i915_gem_object.c +p" \
  > /proc/dynamic_debug/control

# Dump page tables to see cache bits
cat /sys/kernel/debug/dri/0/i915_page_tables | grep -i cache

# Watch for coherency-related hangs
dmesg -f | grep -i "coherency\|flush\|clflush"
```

---

## Performance Tips

### iGPU Optimization

```c
// ✓ Good: Use shared L3 cache
i915_gem_object_set_cache_level(obj, I915_CACHE_LLC);

// ✓ Good: Batch updates with single release
do {
    write_data_1();
    write_data_2();
    write_data_3();
} while(...);
smp_store_release(&all_ready, 1);  // One release for all

// ✗ Bad: Release per write (overhead)
for each write {
    smp_store_release(...);  // Too many barriers!
}
```

### dGPU Optimization

```c
// ✓ Good: Batch large DMA transfers (amortize setup)
dma_transfer(1GB buffer);  // ~25ms, good throughput

// ✓ Good: Async DMA while CPU does other work
dma_start();
cpu_do_other_work();
dma_wait();  // At end

// ✗ Bad: Many small transfers (setup overhead dominates)
for i in 1000 {
    dma_transfer(1MB);  // 1000 × setup overhead!
}

// ✗ Bad: Sync DMA (blocks CPU)
dma_transfer();
dma_wait();  // CPU blocks here
cpu_continue();
```

---

## Key Insight Matrix

| Concept | iGPU | dGPU |
|---------|------|------|
| **Shared Memory** | Yes (DDR4) | No (separate VRAM) |
| **Cache Coherency** | Automatic (L3) | Manual (DMA) |
| **Sync Primitive** | store-release | clflush + mb |
| **Latency** | 100ns | 10-100μs |
| **Bandwidth** | 100+ GB/s (L3) | 25-40 GB/s (PCIe) |
| **Default Safe** | Yes (LLC) | No (requires DMA) |
| **Code Complexity** | Low | High |
| **Performance** | Excellent | Limited by PCIe |

---

## Real Code Examples

### From i915 Source: Setting Coherency

```c
// From: drivers/gpu/drm/i915/gem/i915_gem_object.c

void i915_gem_object_set_cache_level(
    struct drm_i915_gem_object *obj,
    enum i915_cache_level level)
{
    obj->cache_level = level;
    
    // For coherent access
    if (level == I915_CACHE_LLC)
        set_bits(obj, I915_BO_COHERENT);
    
    // Update all VMAs bound to this object
    list_for_each_entry(vma, &obj->vma_list, obj_link) {
        i915_vma_update_cache_level(vma, level);
    }
}
```

### From i915 Source: Emitting GPU Barrier

```c
// From: drivers/gpu/drm/i915/gt/gen6_ppgtt.c

static void emit_tlb_flush(struct i915_request *rq)
{
    u32 *cs;
    
    cs = intel_ring_begin(rq, 4);
    
    // PIPE_CONTROL: Flush GPU caches
    *cs++ = GFX_OP_PIPE_CONTROL(6);
    *cs++ = (PIPE_CONTROL_RENDER_TARGET_CACHE_FLUSH |
             PIPE_CONTROL_INSTRUCTION_CACHE_INVALIDATE |
             PIPE_CONTROL_TEXTURE_CACHE_INVALIDATE);
    *cs++ = 0;
    
    intel_ring_advance(rq, cs);
}
```

---

## Resources

**Read First:**
- [06a-CPU-GPU-Coherency.md](06a-CPU-GPU-Coherency.md) - Complete coherency guide
- [01-Memory-Management.md](01-Memory-Management.md) - GEM objects
- [06-Virtual-Memory.md](06-Virtual-Memory.md) - Page table coherency bits

**Implementation Reference:**
- `drivers/gpu/drm/i915/gem/i915_gem_object.c` - Object cache management
- `drivers/gpu/drm/i915/intel_memory_region.c` - Region coherency
- `drivers/gpu/drm/i915/gt/gen*.c` - PIPE_CONTROL emission

**External References:**
- Intel Architecture Manual - Memory Coherency
- PCIe Specification - Coherency mechanisms
- ARM Memory Model - Memory barriers

---

## FAQ

**Q: My iGPU GPU reads stale data. What's wrong?**  
A: Check if object is marked `I915_CACHE_LLC`. Use `store-release` on CPU side. Enable debug logging to verify.

**Q: dGPU is slow. How do I optimize?**  
A: Batch large DMA transfers (1MB+). Use async DMA while CPU does work. Avoid many small copies.

**Q: When should I use clflush()?**  
A: Only for dGPU before DMA or P2P transfers. iGPU doesn't need it (shared L3). Always pair with `mb()`.

**Q: What's PIPE_CONTROL?**  
A: GPU-side memory barrier. Flushes GPU caches. Required after store before load from different location.

**Q: Can I read GPU memory from CPU on dGPU?**  
A: Yes, via PCIe P2P, but 1000x slower than iGPU. Best to DMA copy back to CPU RAM first.

**Q: How much does coherency overhead cost?**  
A: iGPU: ~50ns per sync (negligible). dGPU: ~50μs per MB (DMA dominated).

---

**Last Updated:** February 8, 2026  
**Version:** 1.0  
**Status:** Ready for team use
