# Memory Region Management & Migration

## Overview

This document provides deep details on memory regions (system RAM, local VRAM, stolen) and the migration mechanisms that move data between them. This is critical for understanding how i915 optimizes memory placement and handles memory pressure.

**Key Topics:**
- Memory region abstractions
- Region-specific allocation strategies
- CPU ↔ GPU coherency
- Migration policies and triggers
- Performance implications

---

## Memory Regions Architecture

```
┌──────────────────────────────────────────────────┐
│         Available Memory Resources                │
├──────────────────────────────────────────────────┤
│                                                   │
│  System RAM (Host Memory)                         │
│  ┌────────────────────────────────────────────┐  │
│  │ • Allocated by kernel shmem                │  │
│  │ • CPU-accessible via virtual addressing    │  │
│  │ • Coherent with CPU (WB cache)             │  │
│  │ • Can be swapped to disk                   │  │
│  │ • Typical: Largest pool (8GB+)             │  │
│  │ • GPU access: Via GGTT mapping             │  │
│  └────────────────────────────────────────────┘  │
│                                                   │
│  Local Memory (VRAM) - Discrete Only             │
│  ┌────────────────────────────────────────────┐  │
│  │ • GPU-dedicated memory on card              │  │
│  │ • Not directly CPU-accessible               │  │
│  │ • GPU-native access (fast)                  │  │
│  │ • Cannot be swapped                         │  │
│  │ • Typical: Limited (2-16GB)                │  │
│  │ • CPU access: Via PCIe BAR or copy         │  │
│  └────────────────────────────────────────────┘  │
│                                                   │
│  Stolen Memory (Integrated GPUs Only)            │
│  ┌────────────────────────────────────────────┐  │
│  │ • Reserved from system RAM by BIOS          │  │
│  │ • Used for framebuffer, firmware            │  │
│  │ • Limited size (32MB-512MB typically)       │  │
│  │ • Fast access from GPU                      │  │
│  │ • Mostly driver-managed                     │  │
│  └────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────┘
```

---

## Memory Region Structures

### Region Representation

```c
struct intel_memory_region {
    // Region identity
    enum intel_region_id id;
    const char *name;                  // "system", "local", "stolen"
    
    // Physical bounds
    u64 start;                         // Base physical address
    u64 size;                          // Total size
    u64 avail;                         // Currently available
    
    /* Memory management */
    union {
        struct {
            // System memory uses shmem
            struct file *shmem_file;
        } system;
        
        struct {
            // Local memory uses buddy allocator
            struct i915_buddy_mm mm;
            struct list_head objects;
        } local;
        
        struct {
            // Stolen uses simple offset tracking
            u32 start_offset;
            struct drm_mm allocator;
        } stolen;
    } private;
    
    /* Region operations */
    struct intel_memory_region_ops {
        int (*init)(struct intel_memory_region *mem);
        void (*release)(struct intel_memory_region *mem);
        
        struct drm_i915_gem_object *(*create_object)(
            struct intel_memory_region *mem,
            u64 size,
            unsigned int flags);
            
        int (*free_pages)(struct drm_i915_gem_object *obj);
    } ops;
    
    /* Statistics */
    struct {
        u64 objects_allocated;
        u64 pages_allocated;
        u64 total_evictions;
        u64 total_migrations;
    } stats;
    
    struct dentry *debugfs_root;       // For debugging
};
```

### Region Types Enumeration

```c
enum intel_region_id {
    INTEL_MEMORY_SYSTEM = 0,          // System RAM
    INTEL_MEMORY_LOCAL = 1,           // Discrete VRAM
    INTEL_MEMORY_STOLEN = 2,          // Stolen from system
    
    // Sentinel
    INTEL_MEMORY_REGIONS_MAX = 3,
};
```

---

## System Memory Management

### System Region Characteristics

```
System Memory (LMEM on integrated, main on discrete):

Allocation Method: shmem (shared memory)
┌──────────────────────────────────┐
│  Kernel shmem filesystem         │
│  /tmp/ (tmpfs)                   │
│  ↓                               │
│  Page cache managed by VM        │
│  ↓                               │
│  Physical pages allocated on     │
│  demand (lazy allocation)        │
└──────────────────────────────────┘

Advantages:
  ✓ Large capacity available
  ✓ Automatic paging to disk
  ✓ Sharing between processes
  ✓ Standard Linux memory pressure handling

Disadvantages:
  ✗ Slower than dedicated VRAM
  ✗ Paging overhead if swapped
  ✗ Not ideal for real-time workloads
  ✗ CPU coherency overhead

GPU Access:
  • Via GGTT mapping (global address space)
  • Requires PTE (page table entry) setup
  • Coherency maintained by hardware
```

**System Memory Allocation Flow:**

```c
int intel_memory_region_alloc_pages_system(
    struct intel_memory_region *mem,
    struct drm_i915_gem_object *obj,
    u64 size)
{
    // Step 1: Allocate backing store via shmem
    struct address_space *mapping =
        obj->base.filp->f_mapping;
    
    // Step 2: Get pages from shmem
    // (lazy allocation - only allocates on access)
    struct page **pages = NULL;
    for (u64 i = 0; i < size / PAGE_SIZE; i++) {
        // Request page from shmem
        struct page *page = 
            shmem_read_mapping_page(mapping, i);
        
        if (IS_ERR(page)) {
            // Out of memory
            return PTR_ERR(page);
        }
        
        // Add to page array
        pages[i] = page;
    }
    
    // Step 3: Build scatter-gather list
    obj->mm.pages = pages;
    obj->mm.page_count = size / PAGE_SIZE;
    
    // Step 4: Record allocation
    mem->avail -= size;
    mem->stats.pages_allocated += size / PAGE_SIZE;
    
    return 0;
}
```

---

## Local Memory (VRAM) Management

### Local Memory Characteristics

```
Local Memory (LMEM) - Discrete GPUs Only:

Allocation Method: Buddy allocator
┌──────────────────────────────────┐
│  Fixed address range (2-16GB)    │
│  ↓                               │
│  Buddy allocator                 │
│  (power-of-2 blocks)             │
│  ↓                               │
│  Physical pages within VRAM      │
└──────────────────────────────────┘

Characteristics:
  • Fixed size (cannot grow beyond hardware limit)
  • No paging to disk
  • Direct GPU access (very fast)
  • Must be explicitly managed
  • Memory pressure → eviction to system

GPU Access Path:
  LMEM Address
    ↓
  Direct GPU access via physical address
  (No GGTT/PPGTT translation needed)
  ↓
  Fast execution (~100 GB/s bandwidth)
```

**LMEM Allocation Flow:**

```c
int intel_memory_region_alloc_pages_local(
    struct intel_memory_region *mem,
    struct drm_i915_gem_object *obj,
    u64 size)
{
    // Step 1: Allocate block from buddy allocator
    struct i915_buddy_block *block =
        i915_buddy_alloc(&mem->private.local.mm, size);
    
    if (!block) {
        // Out of LMEM - trigger eviction
        return -ENOMEM;
    }
    
    // Step 2: Get physical address from block
    u64 phys_addr = block->offset;
    
    // Step 3: Convert to pages array
    struct page *page = 
        pfn_to_page(phys_addr >> PAGE_SHIFT);
    
    obj->mm.pages = &page;
    obj->mm.page_count = 1;  // Physically contiguous
    
    // Step 4: Record allocation
    obj->mm.region = mem;
    obj->mm.buddy_block = block;
    
    mem->avail -= size;
    mem->stats.objects_allocated++;
    
    return 0;
}
```

---

## Migration Mechanisms

### Migration Triggers

```
Trigger Events (Priority Order):

1. MEMORY PRESSURE (High Priority)
   └─ Physical memory low
   └─ kswapd wakes up
   └─ Shrinker invoked
   └─ Evict LMEM → System
   └─ Free up GPU VRAM

2. EXPLICIT MIGRATION (User Request)
   └─ Application: gem_vm_bind with region flag
   └─ Driver: DRM_IOCTL_I915_GEM_MIGRATION
   └─ Move to specified region

3. OPTIMIZED PLACEMENT (Predictive)
   └─ Frequently accessed
   └─ Move to LMEM for speed
   └─ Or move out on memory pressure

4. COHERENCY REQUIREMENT
   └─ Ensure CPU-GPU alignment
   └─ Move to coherent region
   └─ Update access permissions

5. WORKLOAD AFFINITY
   └─ Encode/decode → different region
   └─ Compute → LMEM preferred
   └─ Display → System preferred
```

### Migration Architecture

```
Migration Pipeline:

Source Object (Region A)
    ↓
[Prepare Phase]
  • Get source pages
  • Allocate destination
  • Setup mappings
    ↓
[Copy Phase]
  • GPU copy (preferred)
    OR
  • CPU copy (fallback)
  • Verify data integrity
    ↓
[Binding Phase]
  • Unbind from old region
  • Bind to new region
  • Update address mappings
    ↓
[Cleanup Phase]
  • Free source pages
  • Update statistics
  • Clear references
    ↓
Destination Object (Region B)
```

**Migration Implementation:**

```c
int i915_gem_object_migrate_region(
    struct drm_i915_gem_object *obj,
    struct intel_memory_region *target_region)
{
    struct intel_memory_region *src_region = obj->mm.region;
    
    // Step 0: Validate migration
    if (src_region == target_region)
        return 0;  // Already in target
    
    if (!can_migrate_between(src_region, target_region))
        return -EBADF;  // Invalid combination
    
    // Step 1: UNBIND PHASE
    // Unbind from all address spaces
    struct i915_vma *vma, *next;
    list_for_each_entry_safe(vma, next, &obj->vma_list, obj_link) {
        i915_vma_unbind(vma);
    }
    
    // Step 2: ALLOCATE DESTINATION
    u64 size = obj->base.size;
    struct page **new_pages = 
        target_region->ops.create_object(target_region, 
                                        size, 0)->mm.pages;
    
    if (!new_pages)
        return -ENOMEM;
    
    // Step 3: COPY DATA
    // Option A: GPU copy (preferred for LMEM transfers)
    if (can_use_gpu_copy(src_region, target_region)) {
        struct drm_i915_private *i915 = obj->base.dev->dev_private;
        struct intel_context *ce = i915->kernel_context;
        
        // Create migration request
        struct i915_request *rq =
            intel_migrate_copy(ce,
                             obj->mm.pages,
                             new_pages,
                             size);
        
        // Wait for completion
        i915_request_wait(rq, 0, MAX_SCHEDULE_TIMEOUT);
        i915_request_put(rq);
    } 
    // Option B: CPU copy (system ↔ system, or small sizes)
    else {
        for (u64 i = 0; i < size / PAGE_SIZE; i++) {
            void *src = kmap(obj->mm.pages[i]);
            void *dst = kmap(new_pages[i]);
            memcpy(dst, src, PAGE_SIZE);
            kunmap(new_pages[i]);
            kunmap(obj->mm.pages[i]);
        }
    }
    
    // Step 4: VERIFY INTEGRITY
    if (verify_migration_data(obj->mm.pages, new_pages, size) < 0) {
        // Rollback on error
        target_region->ops.free_pages(new_pages, size);
        return -EIO;
    }
    
    // Step 5: SWAP PAGES
    struct page **old_pages = obj->mm.pages;
    obj->mm.pages = new_pages;
    obj->mm.region = target_region;
    
    // Step 6: FREE SOURCE
    src_region->ops.free_pages(old_pages, size);
    
    // Step 7: RE-BIND IF NEEDED
    if (obj->active_count > 0) {
        // Re-bind to all VMAs in new region
        list_for_each_entry(vma, &obj->vma_list, obj_link) {
            i915_vma_bind(vma, I915_CACHE_UC, 0);
        }
    }
    
    // Step 8: UPDATE STATISTICS
    src_region->stats.total_migrations++;
    target_region->stats.total_migrations++;
    
    return 0;
}
```

---

## Eviction under Memory Pressure

### Eviction Flow

```
Memory Pressure Detected (kswapd/direct reclaim)
        │
        ↓
┌──────────────────────────────────────────┐
│ Shrinker Invoked:                        │
│ i915_gem_shrinker_scan()                 │
│                                          │
│ Callback from kernel memory subsystem    │
└──────────┬───────────────────────────────┘
           ↓
┌──────────────────────────────────────────┐
│ Enumerate GEM Objects:                   │
│ Walk all driver allocations              │
│ Check if evictable                       │
│ - Not pinned                             │
│ - Not in use by GPU                      │
│ - Not required in place                  │
└──────────┬───────────────────────────────┘
           ↓
┌──────────────────────────────────────────┐
│ Prioritize Objects:                      │
│ - Prefer LMEM (freeing GPU memory)       │
│ - Prefer inactive objects                │
│ - Consider size vs. eviction cost        │
└──────────┬───────────────────────────────┘
           ↓
┌──────────────────────────────────────────┐
│ Migration Decision:                      │
│ IF in LMEM:                              │
│   Migrate to System or                   │
│   Free completely                        │
│ ELSE IF in System:                       │
│   Swap to disk or                        │
│   Free if recreatable                    │
└──────────┬───────────────────────────────┘
           ↓
┌──────────────────────────────────────────┐
│ Perform Eviction:                        │
│ - Unbind from all address spaces         │
│ - Perform migration if needed            │
│ - Free memory pages                      │
│ - Update tracking                        │
└──────────┬───────────────────────────────┘
           ↓
┌──────────────────────────────────────────┐
│ Report Freed Memory:                     │
│ Return freed bytes to kernel shrinker    │
└──────────────────────────────────────────┘
```

**Shrinker Implementation:**

```c
unsigned long i915_gem_shrinker_count(
    struct shrinker *shrinker,
    struct shrink_control *sc)
{
    struct drm_i915_private *i915 = container_of(shrinker, ...);
    
    // Count evictable objects
    unsigned long count = 0;
    
    list_for_each_entry(obj, &i915->gem_objects, global_link) {
        // Skip non-evictable
        if (obj->mm.madv != I915_MADV_WILLNEED)
            continue;
        
        // Skip pinned
        if (obj->mm.pages_pin_count)
            continue;
        
        // Skip active
        if (i915_active_acquire_if_busy(&obj->active))
            continue;
        
        count += obj->base.size / PAGE_SIZE;
    }
    
    return count;
}

unsigned long i915_gem_shrinker_scan(
    struct shrinker *shrinker,
    struct shrink_control *sc)
{
    struct drm_i915_private *i915 = container_of(shrinker, ...);
    unsigned long freed = 0;
    unsigned long to_scan = sc->nr_to_scan;
    
    // Scan and evict objects
    list_for_each_entry_safe(obj, next, &i915->gem_objects, ...) {
        if (freed >= to_scan)
            break;
        
        if (!i915_gem_object_is_evictable(obj))
            continue;
        
        unsigned long obj_size = obj->base.size;
        
        // Try to evict
        if (i915_gem_object_evict(obj) == 0) {
            freed += obj_size / PAGE_SIZE;
        }
    }
    
    return freed;
}
```

---

## Cache Coherency & Consistency

### Coherency Models

```
Four memory regions, different coherency:

1. System RAM (CPU Coherent)
   ┌────────────────────────┐
   │ CPU Cache: Write-Back  │
   │ GPU Access: Via GGTT   │
   │ Coherency: Hardware    │
   │ Strategy: Auto-flush   │
   └────────────────────────┘

2. Local VRAM (Not CPU Coherent)
   ┌────────────────────────┐
   │ CPU Cache: None        │
   │ GPU Access: Direct     │
   │ Coherency: Manual      │
   │ Strategy: Explicit sync│
   └────────────────────────┘

3. Stolen Memory (Partial Coherent)
   ┌────────────────────────┐
   │ CPU Cache: Varies      │
   │ GPU Access: Direct     │
   │ Coherency: Limited     │
   │ Strategy: Context-dep  │
   └────────────────────────┘

Consistency Requirements:
  • Before GPU reads CPU-written data
  • Before CPU reads GPU-written data
  • Explicit synchronization needed
```

**Coherency Helpers:**

```c
// Ensure GPU sees CPU writes
void i915_gem_object_flush_cpu_write_domain(
    struct drm_i915_gem_object *obj)
{
    if (obj->write_domain != I915_GEM_DOMAIN_CPU)
        return;  // CPU hasn't written
    
    // Flush CPU cache if needed
    if (cpu_cache_active()) {
        clflush_cache_range(obj->mm.pages, obj->base.size);
    }
    
    obj->write_domain = 0;
    obj->read_domain |= I915_GEM_DOMAIN_GPU;
}

// Ensure CPU sees GPU writes
void i915_gem_object_flush_gpu_write_domain(
    struct drm_i915_gem_object *obj)
{
    if (obj->write_domain != I915_GEM_DOMAIN_GPU)
        return;  // GPU hasn't written
    
    // Wait for GPU to finish writes
    struct i915_active *active = &obj->active;
    i915_active_wait(active);
    
    // Invalidate CPU caches (for LMEM reads)
    if (obj->mm.region->id == INTEL_MEMORY_LOCAL) {
        clflush_cache_range(obj->mm.pages, obj->base.size);
    }
    
    obj->write_domain = 0;
    obj->read_domain |= I915_GEM_DOMAIN_CPU;
}
```

---

## Performance Implications

### Memory Placement Strategy

```
Optimal Placement Decision Tree:

Is LMEM available?
  │
  ├─ NO (Integrated GPU)
  │   └─ Use System RAM only
  │
  └─ YES (Discrete GPU)
      │
      ├─ Is data frequently accessed by GPU?
      │   │
      │   ├─ YES (hot data)
      │   │   └─ Prefer LMEM
      │   │       └─ Fast execution (~100 GB/s)
      │   │       └─ No migration
      │   │
      │   └─ NO (cold data)
      │       └─ Use System RAM
      │
      ├─ Is LMEM full?
      │   │
      │   ├─ YES
      │   │   └─ Evict to System
      │   │   └─ Migrate on demand
      │   │
      │   └─ NO
      │       └─ Can allocate in LMEM
      │
      └─ Memory pressure?
          │
          ├─ HIGH (< 10% free)
          │   └─ Evict LMEM objects
          │   └─ Trigger System swap
          │
          └─ NORMAL
              └─ Let placement be
```

### Bandwidth Comparison

```
Memory Bandwidth (PCIe 4.0 system):

LMEM Direct Access:     ~100 GB/s  ✓ BEST
GGTT System RAM:        ~15-20 GB/s (PCIe bottleneck)
CPU ↔ LMEM:            ~15-20 GB/s (PCIe bottleneck)
System RAM Swap:        ~500 MB/s  ✗ VERY SLOW

Decision:
- Always prefer LMEM for GPU workload
- Migrate under memory pressure
- Keep working set in LMEM
```

---

## Region-Specific Optimizations

### System Region Optimization

```c
// Batch allocations in system memory
struct drm_i915_gem_object *
i915_gem_object_create_shmem_batch(
    struct drm_device *dev,
    struct i915_gem_create_list *reqs)
{
    // Advantage: Reduce shrinker calls
    // Allocate many objects at once
    
    for (int i = 0; i < reqs->count; i++) {
        struct drm_i915_gem_object *obj =
            i915_gem_object_create_shmem(dev, reqs[i].size);
        
        // Batch binding
        if (should_batch_bind(obj)) {
            defer_binding(obj);  // Defer until after all allocated
        }
    }
    
    // Bind all in one TLB flush
    flush_deferred_bindings();
    
    return 0;
}
```

### LMEM Region Optimization

```c
// Compact LMEM under pressure
int i915_lmem_defragment(struct intel_memory_region *mem)
{
    struct i915_buddy_mm *mm = &mem->private.local.mm;
    
    // Find fragmented areas
    struct list_head fragments = LIST_HEAD_INIT(fragments);
    
    for (int order = 0; order < i915_buddy_n_roots(mm); order++) {
        if (mm->split_list[order].count > threshold) {
            // Too fragmented - try to consolidate
            merge_adjacent_blocks(mm, order);
        }
    }
    
    // Consider evicting small objects to consolidate
    if (fragmentation_ratio > 50%) {
        evict_to_make_space_contiguous();
    }
}
```

---

## Debugging Memory Issues

### Inspecting Memory State

```bash
# Show memory regions
cat /sys/kernel/debug/dri/0/i915_memory_regions

# List objects by region
cat /sys/kernel/debug/dri/0/i915_objects_by_region

# Show LMEM buddy allocator
cat /sys/kernel/debug/dri/0/i915_lmem_buddy

# Memory statistics
cat /sys/kernel/debug/dri/0/i915_memory_stats

# Shrinker statistics
cat /sys/kernel/debug/dri/0/i915_shrinker_stats
```

### Debugging Eviction

```bash
# Enable eviction logging
echo "file drivers/gpu/drm/i915/gem/i915_gem_shrinker.c +p" \
  > /proc/dynamic_debug/control

# Monitor evictions
dmesg -f | grep "evict\|migrate\|shrinker"

# Check for memory leaks
cat /sys/kernel/debug/dri/0/i915_gem_objects | grep -E "size|region"
```

---

## Related Components

- **GEM Objects:** `gem/i915_gem_object.c` - Memory container
- **Memory Regions:** `intel_memory_region.c` - Region management
- **Buddy Allocator:** `i915_buddy.c` - LMEM allocation
- **Page Management:** `gem/i915_gem_pages.c` - Physical page tracking
- **Shrinker:** `gem/i915_gem_shrinker.c` - Memory pressure response
- **Migration:** `gem/i915_gem_region.c` - Region-to-region moves

---

## References

- **Source:** `intel_memory_region.c`, `gem/i915_gem_shrinker.c`, `gem/i915_gem_region.c`
- **Headers:** `intel_memory_region.h`
- **Selftests:** `gem/selftests/` - Memory region tests
- **Documentation:** `Documentation/gpu/i915.rst#memory-management`
