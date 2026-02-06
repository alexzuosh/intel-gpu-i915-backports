# Virtual Memory Management System (MMU, VMA, Binding)

## Overview

Virtual Memory Management in the i915 driver handles the mapping between GPU virtual addresses (GA) and physical memory. This system provides isolation, protection, and efficient memory utilization through translation tables (GGTT/PPGTT) and Virtual Memory Address (VMA) objects.

**Key Concepts:**
- **GGTT (Global Graphics Translation Table):** Shared address space visible to all GPU contexts
- **PPGTT (Per-Process GTT):** Per-context private address spaces for isolation
- **VMA (Virtual Memory Address):** Represents a binding of a GEM object into an address space
- **Binding/Unbinding:** Operations to map/unmap objects into GPU address spaces
- **Migration:** Moving data between different memory regions

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────┐
│         User Application / GPU Context                    │
│              (Needs GPU memory access)                    │
└────────────────────┬─────────────────────────────────────┘
                     ↓
        ┌────────────────────────────────┐
        │  GEM Object (CPU memory view)  │
        │  - Represents GPU memory       │
        │  - Allocated by driver         │
        └────────────────┬───────────────┘
                         ↓
        ┌────────────────────────────────┐
        │  i915_vma.c (VMA Management)   │
        │  - Virtual Memory Address      │
        │  - Binding/Unbinding           │
        │  - Address space mapping       │
        └────────────────┬───────────────┘
                         ↓
        ┌────────────────────────────────────────┐
        │   Address Space Selection              │
        │  ┌──────────────────────────────────┐  │
        │  │ GGTT (Global/Shared)             │  │
        │  │ - Visible to all contexts        │  │
        │  │ - Used for shared resources      │  │
        │  └──────────────────────────────────┘  │
        │              OR                        │
        │  ┌──────────────────────────────────┐  │
        │  │ PPGTT (Per-Process Private)      │  │
        │  │ - Context-specific isolation     │  │
        │  │ - Security boundary              │  │
        │  └──────────────────────────────────┘  │
        └────────────────┬───────────────────────┘
                         ↓
        ┌────────────────────────────────────┐
        │   Page Table Programming           │
        │  - Update PTE (Page Table Entry)   │
        │  - Write to PAGING STRUCTURES      │
        │  - Set permissions (R/W/X)         │
        └────────────────┬───────────────────┘
                         ↓
        ┌────────────────────────────────────┐
        │   GPU Hardware MMU                  │
        │  - Caches page tables (TLB)        │
        │  - Performs VA→PA translation      │
        │  - Enforces permissions            │
        └────────────────┬───────────────────┘
                         ↓
              Physical Memory Access
```

---

## Core Components

### 1. **Virtual Memory Address (VMA) - i915_vma.c**

**Purpose:** Represents a GEM object bound into a specific address space

**Key Structure:**
```c
struct i915_vma {
    // GEM object this VMA refers to
    struct drm_i915_gem_object *obj;
    
    // Address space (GGTT or PPGTT) where bound
    struct i915_address_space *vm;
    
    // Virtual address in the address space
    u64 node;                          // Offset/address in VM
    
    /* Binding state */
    atomic_t open_count;               // References to this VMA
    atomic_t pages_count;              // Pin count
    
    unsigned long flags;
    #define I915_VMA_GLOBAL_BIND       0
    #define I915_VMA_LOCAL_BIND        1
    #define I915_VMA_BIND_MASK         (I915_VMA_GLOBAL_BIND | I915_VMA_LOCAL_BIND)
    
    /* Execution state */
    struct i915_active active;         // Active tracking
    struct list_head active_link;      // List of active VMAs
    
    /* Statistics */
    struct {
        ktime_t bind_time;
        ktime_t unbind_time;
        u32 bind_count;
        u32 unbind_count;
    } stats;
    
    /* Memory pressure */
    struct list_head evict_link;       // For eviction lists
};
```

**VMA Lifecycle:**

```
┌─────────────────────────────────────┐
│  User wants to use GEM object       │
│  in a GPU context                   │
└────────────┬──────────────────────┘
             ↓
    ┌────────────────────────────┐
    │ Create VMA                 │
    │ - Allocate VMA structure   │
    │ - Link GEM object          │
    │ - Select address space     │
    └────────────┬───────────────┘
                 ↓
    ┌────────────────────────────┐
    │ Reserve Address Space      │
    │ - Allocate GPU VA range    │
    │ - Check for collisions     │
    │ - Register in drm_mm tree  │
    └────────────┬───────────────┘
                 ↓
    ┌────────────────────────────┐
    │ Bind VMA                   │
    │ - Get object pages         │
    │ - Program page tables      │
    │ - Flush TLB                │
    │ - Mark as active           │
    └────────────┬───────────────┘
                 ↓
    ┌────────────────────────────┐
    │ Active / In Use            │
    │ - GPU can access via VA    │
    │ - Reference counted        │
    │ - Can be evicted under     │
    │   memory pressure          │
    └────────────┬───────────────┘
                 ↓
    ┌────────────────────────────┐
    │ Unbind (on eviction)       │
    │ - Clear page tables        │
    │ - Flush TLB                │
    │ - Release address space    │
    │ - Mark inactive            │
    └────────────┬───────────────┘
                 ↓
    ┌────────────────────────────┐
    │ Destroy VMA                │
    │ - Free VMA structure       │
    │ - Release all resources    │
    └────────────────────────────┘
```

---

### 2. **Global Graphics Translation Table (GGTT) - intel_ggtt.c**

**Purpose:** Shared GPU address space visible to all contexts and engines

**Key Structure:**
```c
struct i915_ggtt {
    struct i915_address_space vm;      // Base address space
    
    /* Hardware configuration */
    size_t total_size;                 // Total GGTT size
    u32 stolen_size;                   // Reserved for system
    
    /* Memory management */
    struct drm_mm vm_free_list;        // Free address space
    struct list_head bound_list;       // Bound VMAs
    
    /* Page table management */
    struct pt_entry *table;            // Page table entries
    u32 last_entry;                    // Last programmed entry
    
    /* Hardware registers */
    void __iomem *reg_iomap;           // Register access
    
    /* Flushing */
    struct work_struct flush_work;     // Deferred flush
    
    /* Statistics */
    struct {
        u64 total_allocated;
        u64 total_freed;
        u32 current_bindings;
    } stats;
};
```

**GGTT Characteristics:**
```
┌─────────────────────────────────────────────────┐
│          GGTT (Global Graphics Translation Table) │
├─────────────────────────────────────────────────┤
│  Properties:                                     │
│  • Size: Typically 256MB - 2GB (hardware limit)  │
│  • Visibility: Shared by ALL GPU contexts        │
│  • Isolation: NO - all apps see same VA space    │
│  • Usage: System buffers, ring buffers, etc.     │
│                                                  │
│  Layout:                                         │
│  ┌───────────────────────────────────────────┐   │
│  │  0x0000_0000 - Firmware/System            │   │
│  ├───────────────────────────────────────────┤   │
│  │  Ring buffers (exec, render, etc.)        │   │
│  ├───────────────────────────────────────────┤   │
│  │  Stolen memory region (framebuffer)       │   │
│  ├───────────────────────────────────────────┤   │
│  │  GuC/HUC firmware allocations             │   │
│  ├───────────────────────────────────────────┤   │
│  │  Shared buffers (DMA-BUF, prime)          │   │
│  ├───────────────────────────────────────────┤   │
│  │  Free space (available for allocation)    │   │
│  └───────────────────────────────────────────┘   │
│  Limit: Hardware register MAX_GGTT_ADDRESS       │
└─────────────────────────────────────────────────┘
```

**GGTT Binding Example:**

```c
int intel_ggtt_bind_vma(struct i915_vma *vma)
{
    struct i915_ggtt *ggtt = vma->vm;
    struct drm_i915_gem_object *obj = vma->obj;
    
    // Step 1: Get pages from object
    struct page **pages = obj->mm.pages;
    
    // Step 2: Get virtual address for VMA
    u64 start = vma->node.start;
    u64 size = vma->node.size;
    
    // Step 3: Program PTEs (Page Table Entries)
    for (u64 offset = 0; offset < size; offset += PAGE_SIZE) {
        u32 pte_offset = (start + offset) / PAGE_SIZE;
        struct page *page = pages[offset / PAGE_SIZE];
        
        // Build PTE: physical address + flags
        u64 pte = page_to_phys(page);
        pte |= I915_PTE_VALID;  // Mark as valid
        pte |= I915_PTE_READ;   // Enable read
        pte |= I915_PTE_WRITE;  // Enable write
        
        // Write to GGTT table
        iowrite64(ggtt->table + pte_offset, pte);
    }
    
    // Step 4: Flush TLB (Translation Lookaside Buffer)
    intel_ggtt_flush(ggtt);
    
    // Step 5: Mark as bound
    set_bit(I915_VMA_GLOBAL_BIND, &vma->flags);
    
    return 0;
}
```

---

### 3. **Per-Process Graphics Translation Table (PPGTT) - intel_ppgtt.c**

**Purpose:** Per-context private address spaces for memory isolation

**Key Structure:**
```c
struct i915_ppgtt {
    struct i915_address_space vm;      // Base address space
    
    /* Context ownership */
    struct i915_gem_context *ctx;      // Owning context
    
    /* Page tables */
    struct i915_page_directory *pd;    // Top-level page directory
    
    /* Address space configuration */
    u64 total_size;                    // Virtual address space size
    u32 mode;                          // PPGTT mode (2/3/4 level)
    
    /* Scratch pages (for unused PTEs) */
    struct page *scratch_pages[4];     // For each level
    
    /* Statistics */
    struct {
        u64 tlb_flushes;
        u64 page_table_updates;
    } stats;
};
```

**PPGTT Characteristics:**
```
┌─────────────────────────────────────────────────┐
│   PPGTT (Per-Process Graphics Translation Table)  │
├─────────────────────────────────────────────────┤
│  Properties:                                     │
│  • Size: 48-bit VA space (~256TB addressable)    │
│  • Visibility: PRIVATE per context               │
│  • Isolation: YES - apps cannot see each other   │
│  • Usage: Application GPU memory, shaders, etc.  │
│                                                  │
│  Per-Context Isolation:                         │
│  ┌─────────────────────────────────────────┐    │
│  │  Context A                              │    │
│  │  ┌─────────────────────────────────┐    │    │
│  │  │ PPGTT_A Virtual Address Space   │    │    │
│  │  │ 0x0000000000 - 0xFFFFFFFFFF     │    │    │
│  │  │ Maps to different physical mem  │    │    │
│  │  └─────────────────────────────────┘    │    │
│  └─────────────────────────────────────────┘    │
│                                                  │
│  ┌─────────────────────────────────────────┐    │
│  │  Context B                              │    │
│  │  ┌─────────────────────────────────┐    │    │
│  │  │ PPGTT_B Virtual Address Space   │    │    │
│  │  │ 0x0000000000 - 0xFFFFFFFFFF     │    │    │
│  │  │ Maps to different physical mem  │    │    │
│  │  └─────────────────────────────────┘    │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘

Security Benefit:
  • Context A cannot read/write Context B memory
  • Prevents information leakage
  • Essential for multi-user systems
```

**PPGTT Page Table Hierarchy:**

```
PPGTT (48-bit address space) - 4-level page tables:

Virtual Address: [47:39][38:30][29:21][20:12][11:0]
                 ------  ------  ------  ------  ----
                   PDL    PD     PT      PTE    Offset
                   
Level 1: Page Directory Lookup (9 bits)
┌──────────────────────────────────────┐
│ Points to Page Directories (512)    │
│ One per level 2                      │
└──────────────────────────────────────┘
         │
         └─→ Level 2: Page Directory (9 bits)
             ┌──────────────────────────────────────┐
             │ Points to Page Tables (512)         │
             │ One per level 3                      │
             └──────────────────────────────────────┘
                     │
                     └─→ Level 3: Page Table (9 bits)
                         ┌──────────────────────────────────────┐
                         │ Contains Page Table Entries (512)    │
                         │ Maps to physical pages               │
                         └──────────────────────────────────────┘
                                 │
                                 └─→ Level 4: Page Offset (12 bits)
                                     ┌──────────────────────────────────────┐
                                     │ Offset within 4KB page               │
                                     │ Physical address = PTE_BASE + offset │
                                     └──────────────────────────────────────┘
```

---

## Binding and Unbinding Operations

### Binding Process

```c
// Complete binding flow
int i915_vma_bind(struct i915_vma *vma, 
                  enum i915_cache_level cache_level,
                  u32 flags)
{
    struct drm_i915_gem_object *obj = vma->obj;
    struct i915_address_space *vm = vma->vm;
    
    // Step 1: Validate VMA state
    if (vma->flags & I915_VMA_BOUND)
        return 0;  // Already bound
    
    // Step 2: Pin pages (prevent eviction)
    int ret = i915_gem_object_pin_pages(obj);
    if (ret < 0)
        return ret;
    
    // Step 3: Get cache level
    obj->cache_level = cache_level;
    
    // Step 4: Prepare VM for binding
    ret = vm->bind_vma(vm, vma, cache_level, flags);
    if (ret < 0) {
        i915_gem_object_unpin_pages(obj);
        return ret;
    }
    
    // Step 5: Mark as bound
    set_bit(I915_VMA_BOUND, &vma->flags);
    
    // Step 6: Record binding
    list_add(&vma->bound_link, &vm->bound_list);
    
    // Step 7: Update statistics
    vm->stats.current_bindings++;
    
    return 0;
}
```

**Binding Diagram:**

```
Bind Request Arrives
        │
        ↓
┌───────────────────────────────────┐
│ Check if already bound            │
│ - If yes, return success          │
│ - If no, proceed                  │
└───────────┬───────────────────────┘
            ↓
┌───────────────────────────────────┐
│ Pin object pages                  │
│ - Increment page pin count        │
│ - Prevent page eviction           │
│ - Get page list ready             │
└───────────┬───────────────────────┘
            ↓
┌───────────────────────────────────┐
│ Program translation tables        │
│ - Walk GGTT or PPGTT hierarchy    │
│ - Build page table entries        │
│ - Write PTEs to hardware          │
│ - Set cache levels                │
│ - Mark pages as valid             │
└───────────┬───────────────────────┘
            ↓
┌───────────────────────────────────┐
│ Flush TLB (Translation Lookaside  │
│ Buffer) in GPU                    │
│ - Clear cached translations       │
│ - Force re-lookup from tables     │
└───────────┬───────────────────────┘
            ↓
┌───────────────────────────────────┐
│ Update VMA state                  │
│ - Mark as bound                   │
│ - Add to bound lists              │
│ - Record in statistics            │
└───────────┬───────────────────────┘
            ↓
            ✓ Binding Complete
            GPU can now access object
            via virtual address
```

### Unbinding Process

```c
// Complete unbinding flow
int i915_vma_unbind(struct i915_vma *vma)
{
    struct drm_i915_gem_object *obj = vma->obj;
    struct i915_address_space *vm = vma->vm;
    
    // Step 1: Check if bound
    if (!(vma->flags & I915_VMA_BOUND))
        return 0;
    
    // Step 2: Ensure GPU not using this VMA
    int ret = i915_vma_wait_for_activity(vma);
    if (ret < 0)
        return ret;
    
    // Step 3: Unbind from VM
    ret = vm->unbind_vma(vm, vma);
    if (ret < 0)
        return ret;
    
    // Step 4: Clear page tables
    // (hardware-specific implementation)
    intel_ppgtt_clear_range(vm->ppgtt, 
                           vma->node.start,
                           vma->node.size);
    
    // Step 5: Flush TLB
    vm->flush_ggtt_write();
    
    // Step 6: Unpin pages
    i915_gem_object_unpin_pages(obj);
    
    // Step 7: Update state
    clear_bit(I915_VMA_BOUND, &vma->flags);
    list_del(&vma->bound_link);
    
    // Step 8: Release address space
    drm_mm_remove_node(&vma->node);
    
    return 0;
}
```

**Unbinding Diagram:**

```
Unbind Request (on eviction)
        │
        ↓
┌───────────────────────────────────┐
│ Wait for GPU to finish            │
│ - Check active references         │
│ - Wait for requests to complete   │
│ - Ensure no pending operations    │
└───────────┬───────────────────────┘
            ↓
┌───────────────────────────────────┐
│ Clear page table entries          │
│ - Mark PTEs as invalid            │
│ - Clear permissions               │
│ - Deallocate page table levels    │
│   if empty                        │
└───────────┬───────────────────────┘
            ↓
┌───────────────────────────────────┐
│ Flush GPU TLB                     │
│ - Ensure GPU sees changes         │
│ - Invalidate cached translations  │
│ - Full invalidation context-wide  │
└───────────┬───────────────────────┘
            ↓
┌───────────────────────────────────┐
│ Release virtual address space     │
│ - Return VA range to free pool    │
│ - Update drm_mm allocator         │
│ - Defragment if needed            │
└───────────┬───────────────────────┘
            ↓
┌───────────────────────────────────┐
│ Unpin pages from memory           │
│ - Decrement pin count             │
│ - Allow eviction/swapping         │
│ - Release page references         │
└───────────┬───────────────────────┘
            ↓
┌───────────────────────────────────┐
│ Update bookkeeping                │
│ - Mark VMA unbound                │
│ - Remove from tracking lists      │
│ - Update statistics               │
└───────────┬───────────────────────┘
            ↓
            ✓ Unbinding Complete
            GPU cannot access anymore
            Pages can be evicted/freed
```

---

## Memory Migration

Memory migration moves GPU allocations between different memory regions (system RAM ↔ local VRAM).

**Migration Scenarios:**

```
Trigger Events:

1. User Explicit Migration
   • Application requests move to specific region
   • Use EXEC_OBJECT_PLACEMENT flags
   • Via gem_vm_bind with region parameter

2. Eviction (Memory Pressure)
   • System memory low
   • Migrate LMEM objects to system
   • Reduce physical GPU memory usage

3. Optimization
   • Move frequently accessed to fast memory
   • Use predictive migration

4. Coherency
   • Ensure CPU-GPU cache coherency
   • Migrate to coherent region if needed
```

**Migration Process:**

```
┌─────────────────────────────────────┐
│ Migration Triggered                 │
│ (pressure, request, optimization)   │
└────────────┬──────────────────────┘
             ↓
    ┌────────────────────────────┐
    │ Create source mapping      │
    │ - Bind to source region    │
    │ - Ensure accessible        │
    └────────────┬───────────────┘
                 ↓
    ┌────────────────────────────┐
    │ Create destination mapping │
    │ - Allocate in target region│
    │ - Prepare for data copy    │
    └────────────┬───────────────┘
                 ↓
    ┌────────────────────────────┐
    │ Copy Data                  │
    │ Options:                   │
    │ - GPU copy (DMA engine)    │
    │ - CPU memcpy               │
    │ - Async copy via ring      │
    └────────────┬───────────────┘
                 ↓
    ┌────────────────────────────┐
    │ Update VMA Mappings        │
    │ - Unbind from source       │
    │ - Bind to destination      │
    │ - Update references        │
    └────────────┬───────────────┘
                 ↓
    ┌────────────────────────────┐
    │ Release Source Memory      │
    │ - Deallocate old region    │
    │ - Return to free pool      │
    │ - Update statistics        │
    └────────────┬───────────────┘
                 ↓
    ┌────────────────────────────┐
    │ Verify Migration           │
    │ - Check data integrity     │
    │ - Verify accessibility     │
    │ - Test GPU access          │
    └────────────┬───────────────┘
                 ↓
            ✓ Migration Complete
```

**Migration Code Example:**

```c
int i915_gem_object_migrate(struct drm_i915_gem_object *obj,
                            intel_memory_region *target_region)
{
    struct drm_i915_private *i915 = obj->base.dev->dev_private;
    
    // Step 1: Validate
    if (obj->mm.region == target_region)
        return 0;  // Already in target region
    
    // Step 2: Unbind from all address spaces
    i915_gem_object_unbind_all(obj);
    
    // Step 3: Get source pages
    struct page **src_pages = obj->mm.pages;
    size_t size = obj->base.size;
    
    // Step 4: Allocate in target region
    struct page **dst_pages = target_region->alloc(size);
    if (!dst_pages)
        return -ENOMEM;
    
    // Step 5: Copy data (GPU or CPU method)
    if (can_use_gpu_copy) {
        // Use GPU DMA copy
        intel_migrate_copy(i915->gpu,
                          src_pages, 
                          dst_pages, 
                          size);
    } else {
        // CPU memcpy
        for (size_t i = 0; i < size / PAGE_SIZE; i++) {
            void *src = kmap(src_pages[i]);
            void *dst = kmap(dst_pages[i]);
            memcpy(dst, src, PAGE_SIZE);
            kunmap(src_pages[i]);
            kunmap(dst_pages[i]);
        }
    }
    
    // Step 6: Update object
    obj->mm.region = target_region;
    obj->mm.pages = dst_pages;
    
    // Step 7: Free old pages
    target_region->free(src_pages, size);
    
    // Step 8: Re-bind if needed
    if (obj->active_count > 0) {
        i915_gem_object_bind_to_vm(obj, NULL);
    }
    
    return 0;
}
```

---

## Page Table Management

### Page Table Entry (PTE) Format

```
Generic PTE Format (64-bit):

63  [Physical Address (39 bits)]  24
    ┌──────────────────────────┐
    │ Physical Page Address    │
    │ Bits 63:24 = PA[51:12]   │
    │ (Points to 4KB page)     │
    └──────────────────────────┘

23-12: [Unused/Reserved] (12 bits)
    ┌──────────────────────────┐
    │ Reserved for SW/HW       │
    └──────────────────────────┘

11-8: [Cache Control] (4 bits)
    ┌──────────────────────────┐
    │ 0: UC (Uncached)         │
    │ 1: WC (Write-Combine)    │
    │ 2: WT (Write-Through)    │
    │ 3: WB (Write-Back)       │
    └──────────────────────────┘

7-4: [Flags]
    ┌──────────────────────────┐
    │ Bit 7: WRITABLE          │
    │ Bit 6: READABLE          │
    │ Bit 5: EXECUTABLE        │
    │ Bit 4: RESERVED          │
    └──────────────────────────┘

3: Unused

2: ACCESSED (Set by HW on access)
   ┌──────────────────────────┐
   │ Used for page statistics │
   └──────────────────────────┘

1: DIRTY (Set by HW on write)
   ┌──────────────────────────┐
   │ Used for dirty tracking  │
   └──────────────────────────┘

0: VALID (Entry is valid)
   ┌──────────────────────────┐
   │ 1 = Valid entry          │
   │ 0 = Invalid/Unmapped     │
   └──────────────────────────┘
```

### TLB (Translation Lookaside Buffer) Management

```
TLB Hierarchy:

L1 TLB (Per-Engine)
┌─────────────────────────────────┐
│ Caches most recent translations │
│ Very fast (~1 cycle)            │
│ Small: 32-64 entries            │
└─────────────────────────────────┘
    ↓ Miss
L2 TLB (Shared)
┌─────────────────────────────────┐
│ Larger translation cache        │
│ Fast (~10 cycles)               │
│ Larger: 256-1024 entries        │
└─────────────────────────────────┘
    ↓ Miss
Page Table Walk
┌─────────────────────────────────┐
│ Walk page table hierarchy       │
│ Slow (~100+ cycles)             │
│ Accesses main memory            │
└─────────────────────────────────┘

TLB Flush Operations:
```

**TLB Flushing:**

```c
// Full TLB flush
void intel_ppgtt_flush_tlb(struct i915_ppgtt *ppgtt)
{
    struct intel_uncore *uncore = ppgtt->uncore;
    
    // Method 1: Global invalidation (all entries)
    intel_uncore_write_fw(uncore, GEN12_TTCNTRL,
                         INVLPG_INVALIDATE_GLOBAL);
    
    // Wait for completion
    intel_uncore_read_fw(uncore, GEN12_TTCNTRL);
}

// Selective TLB flush (range)
void intel_ppgtt_flush_tlb_range(struct i915_ppgtt *ppgtt,
                                 u64 start, u64 end)
{
    struct intel_uncore *uncore = ppgtt->uncore;
    
    // Method 2: Invalidate range (newer HW)
    for (u64 addr = start; addr < end; addr += PAGE_SIZE) {
        intel_uncore_write_fw(uncore, GEN12_INVLPG,
                             addr | INVLPG_ADDR_VALID);
    }
}

// Page table invalidation without TLB flush
void i915_gem_invalidate_pages(struct drm_i915_gem_object *obj)
{
    // Mark as needing re-binding
    obj->mm.dirty = true;
    
    // Will be re-validated on next access
}
```

---

## Address Space Allocator (drm_mm)

The `drm_mm` allocator manages free/allocated ranges in address spaces.

```
Address Space Allocation Tree:

VM Address Space (256GB for PPGTT):
┌─────────────────────────────────────────────────┐
│ [Allocated] [Free] [Allocated] [Free] [Allocated] │
│  0-64MB    64-128MB 128-256MB 256MB+ ...        │
└─────────────────────────────────────────────────┘

drm_mm tracks as:
- Free nodes (for allocation)
- Allocated nodes (currently in use)
- Linked list with color/size

Allocation Policy:
- First-fit: Use first available hole
- Best-fit: Use smallest fitting hole
- Buddy: Power-of-2 alignment
```

**drm_mm Operations:**

```c
// Allocate VA range
int drm_mm_insert_node(struct drm_mm *mm,
                       struct drm_mm_node *node,
                       u64 size,
                       u64 alignment,
                       unsigned long flags)
{
    // Find best-fit hole
    struct drm_mm_node *hole = find_hole(mm, size, alignment);
    
    if (!hole)
        return -ENOSPC;  // No space available
    
    // Split hole if necessary
    if (hole->size > size)
        split_node(hole, size);
    
    // Record allocation
    node->start = hole->start;
    node->size = size;
    list_add(&node->link, &hole->link);
    
    return 0;
}

// Free VA range
void drm_mm_remove_node(struct drm_mm_node *node)
{
    // Mark node as free
    list_del(&node->link);
    
    // Try to merge with neighbors
    struct drm_mm_node *prev = prev_node(node);
    struct drm_mm_node *next = next_node(node);
    
    if (is_free(prev) && adjacent(prev, node)) {
        merge_nodes(prev, node);
    }
    
    if (is_free(next) && adjacent(node, next)) {
        merge_nodes(node, next);
    }
}
```

---

## Code Flow Examples

### Complete Bind Flow

```c
// User-initiated binding
int i915_gem_vm_bind_ioctl(struct drm_device *dev,
                           void *data,
                           struct drm_file *file)
{
    struct drm_i915_gem_vm_bind *args = data;
    
    // Step 1: Get GEM object
    struct drm_i915_gem_object *obj =
        i915_gem_object_lookup(file, args->handle);
    
    // Step 2: Get address space (VM)
    struct i915_address_space *vm =
        i915_gem_vm_lookup(file, args->vm_id);
    
    // Step 3: Create VMA
    struct i915_vma *vma = i915_vma_create(obj, vm);
    if (!vma)
        return -ENOMEM;
    
    // Step 4: Allocate address range
    u64 start = args->address;
    u64 size = obj->base.size;
    
    int ret = drm_mm_insert_node(&vm->mm,
                                 &vma->node,
                                 size,
                                 PAGE_SIZE,
                                 DRM_MM_SEARCH_DEFAULT);
    if (ret < 0)
        goto err_vma;
    
    // Step 5: Bind
    ret = i915_vma_bind(vma, I915_CACHE_LEVEL_UC, 0);
    if (ret < 0)
        goto err_node;
    
    // Step 6: Return VA to user
    args->offset = vma->node.start;
    
    return 0;

err_node:
    drm_mm_remove_node(&vma->node);
err_vma:
    i915_vma_put(vma);
    return ret;
}
```

### Eviction Unbinding

```c
// Called when memory pressure evicts object
int i915_gem_object_evict(struct drm_i915_gem_object *obj)
{
    // Step 1: Get all VMAs for this object
    struct i915_vma *vma, *next;
    list_for_each_entry_safe(vma, next, &obj->vma_list, obj_link) {
        // Step 2: Unbind from address space
        int ret = i915_vma_unbind(vma);
        if (ret < 0)
            return ret;  // Failed to unbind, cannot evict
    }
    
    // Step 3: Object is now unbound and evictable
    return 0;
}
```

---

## Cache Coherency Considerations

```
Cache Levels:

L1: Per-GPU Engine Cache
    ├─ Fastest
    ├─ Smallest
    └─ Per-engine

L2: Shared GPU Cache
    ├─ Medium speed
    ├─ Larger
    └─ Shared across engines

L3: Last-level GPU Cache
    ├─ Largest GPU cache
    ├─ Slowest cache
    └─ Shared

Main Memory

Coherency Issues:
```

**Cache Coherency Handling:**

```c
// Set cache level for VMA
int i915_vma_set_cache_level(struct i915_vma *vma,
                             enum i915_cache_level level)
{
    struct drm_i915_gem_object *obj = vma->obj;
    
    // Store cache level
    obj->cache_level = level;
    
    // If already bound, need to rebind with new level
    if (vma->flags & I915_VMA_BOUND) {
        // Re-program page tables with new cache bits
        // (Hardware-specific)
        update_pte_cache_bits(vma, level);
        
        // Flush caches to be safe
        intel_gt_flush_ggtt_writes(vma->vm->gt);
    }
    
    return 0;
}

// Cache levels available:
enum i915_cache_level {
    I915_CACHE_NONE,           // Uncached
    I915_CACHE_LLC,            // CPU L3 (Haswell+)
    I915_CACHE_ELLC,           // eDRAM (Broadwell)
    I915_CACHE_L3_LLC,         // GPU L3 + CPU L3
};
```

---

## Performance Optimization Tips

### 1. **Minimize TLB Flushes**
- Batch address space changes
- Use range invalidation vs. full flush
- Defer flushes when possible

### 2. **Optimize Address Space Usage**
- Pre-allocate likely ranges
- Use power-of-2 alignments
- Reduce fragmentation

### 3. **Cache-Aware Binding**
- Use appropriate cache levels for working set
- Keep hot data in L3
- Use write-combine for ring buffers

### 4. **Migration Optimization**
- Use GPU copy for large migrations
- Batch migrations under pressure
- Predict and prefetch

---

## Debugging & Inspection

### Debugfs Commands:

```bash
# List all VMAs
cat /sys/kernel/debug/dri/0/i915_vma_list

# Check address space usage
cat /sys/kernel/debug/dri/0/mm_allocator

# View page table statistics
cat /sys/kernel/debug/dri/0/i915_ppgtt_info

# TLB flush statistics
cat /sys/kernel/debug/dri/0/i915_tlb_stats
```

### Kernel Logging:

```bash
# Enable detailed VM logging
echo "module i915 +p" > /proc/dynamic_debug/control
echo "file drivers/gpu/drm/i915/i915_vma.c +p" > /proc/dynamic_debug/control

# Monitor address space allocation
dmesg | grep "VMA.*alloc"
```

---

## Related Components

- **GEM Objects:** `gem/i915_gem_object.c` - Memory representation
- **Page Management:** `gem/i915_gem_pages.c` - Physical page tracking
- **Address Spaces:** `gt/intel_ggtt.c`, `gt/intel_ppgtt.c`
- **Reset:** `gt/intel_reset.c` - TLB flush on GPU reset
- **Power:** `gt/intel_rc6.c` - Address space during RC6

---

## References

- **Source:** `i915_vma.c`, `gt/intel_ggtt.c`, `gt/intel_ppgtt.c`, `gt/gen8_ppgtt.c`
- **Headers:** `i915_vma.h`, `intel_gtt.h`
- **Selftests:** `gt/selftest_gtt.c`
- **Documentation:** `Documentation/gpu/i915.rst#vm-and-address-spaces`
