# GGTT/PPGTT Implementation Guide - Practical Usage

**Companion Document:** 06c-GGTT-PPGTT-Deep-Dive.md  
**Focus:** How to work with GGTT and PPGTT in i915 code  
**Real Examples:** From actual i915 implementation

---

## Table of Contents

1. [Quick Start](#quick-start)
2. [GGTT Operations](#ggtt-operations)
3. [PPGTT Operations](#ppgtt-operations)
4. [VMA Binding](#vma-binding)
5. [Common Patterns](#common-patterns)
6. [Real-World Examples](#real-world-examples)
7. [Error Handling](#error-handling)
8. [Performance Tips](#performance-tips)
9. [Debugging Guide](#debugging-guide)

---

## Quick Start

### Choosing Which Table to Use

```c
// Decision flowchart in code:

struct drm_i915_gem_object *obj = get_gpu_memory();

// Should this go in GGTT or PPGTT?

if (is_kernel_internal(obj)) {
    // Kernel work (firmware, status, display)
    bind_to_ggtt(obj);          // Use GGTT
} else if (is_user_application(obj)) {
    // User app memory
    bind_to_ppgtt(obj);         // Use PPGTT
} else if (needs_visible_everywhere(obj)) {
    // Shared across contexts
    bind_to_ggtt(obj);          // Use GGTT
} else if (large_allocation(obj)) {
    // Too big for GGTT anyway
    bind_to_ppgtt(obj);         // Use PPGTT (256TB)
} else {
    // Default: if in doubt, use PPGTT (safer)
    bind_to_ppgtt(obj);         // Use PPGTT
}
```

### Basic Binding Pattern

```c
// GGTT binding (kernel work)

struct i915_vma *vma;
struct drm_i915_gem_object *obj;

// 1. Get or create GGTT
struct i915_ggtt *ggtt = &dev_priv->ggtt;

// 2. Create VMA
vma = i915_vma_create(obj, &ggtt->vm, NULL);
if (IS_ERR(vma))
    return PTR_ERR(vma);

// 3. Bind to GGTT
ret = i915_vma_pin(vma, 0, 0, PIN_GLOBAL);
if (ret < 0) {
    i915_vma_destroy(vma);
    return ret;
}

// Now obj is mapped in GGTT at vma->node.start
uint32_t ggtt_offset = i915_ggtt_offset(vma);
dev_priv->hw_status_page_offset = ggtt_offset;

// 4. Later: unbind
i915_vma_unpin(vma);
i915_vma_destroy(vma);
```

### PPGTT Binding Pattern

```c
// PPGTT binding (user application)

struct i915_vma *vma;
struct drm_i915_gem_object *obj;
struct i915_gem_context *ctx;

// 1. Get context's PPGTT
struct i915_ppgtt *ppgtt = ctx->ppgtt;

// 2. Create VMA for context
vma = i915_vma_create(obj, &ppgtt->vm, ctx);
if (IS_ERR(vma))
    return PTR_ERR(vma);

// 3. Bind to context's PPGTT
ret = i915_vma_pin(vma, 0, size, PIN_USER);
if (ret < 0) {
    i915_vma_destroy(vma);
    return ret;
}

// Now obj is mapped in context's PPGTT
uint64_t gpu_va = i915_vma_offset(vma);
return_to_userspace(gpu_va);

// 4. Later: unbind
i915_vma_unpin(vma);
i915_vma_destroy(vma);
```

---

## GGTT Operations

### 1. GGTT Initialization

```c
// During driver probe (happens once)

static int i915_ggtt_probe_hw(struct drm_i915_private *dev_priv)
{
    struct i915_ggtt *ggtt = &dev_priv->ggtt;
    
    // 1. Determine GGTT size from hardware
    ggtt->total_size = get_ggtt_size_from_hw();
    // Result: 256MB, 512MB, 1GB, or 2GB
    
    // 2. Determine mappable range (CPU-accessible)
    ggtt->mappable_end = determine_cpu_mappable_range();
    // Typically: 256MB max CPU access
    
    // 3. Allocate GGTT page table
    ret = allocate_ggtt_page_table(ggtt);
    if (ret)
        return ret;
    
    // 4. Initialize address space
    ret = i915_address_space_init(&ggtt->vm, dev_priv);
    if (ret)
        return ret;
    
    // 5. Setup scratch page (for invalid PTEs)
    ret = setup_scratch_page(ggtt);
    if (ret)
        return ret;
    
    return 0;
}
```

### 2. Mapping Object to GGTT

```c
// Bind GEM object to GGTT

struct i915_vma *i915_gem_object_ggtt_pin(
    struct drm_i915_gem_object *obj,
    const struct i915_ggtt_view *view,
    u64 size,
    u32 alignment,
    u64 flags)
{
    struct i915_ggtt *ggtt = &obj->base.dev->dev_private->ggtt;
    struct i915_vma *vma;
    
    // 1. Find or create VMA for this object
    vma = i915_vma_instance(obj, &ggtt->vm, view);
    if (IS_ERR(vma)) {
        DRM_ERROR("Failed to create GGTT VMA\n");
        return vma;
    }
    
    // 2. Pin (prevent eviction)
    int ret = i915_vma_pin(vma, size, alignment, 
                           PIN_GLOBAL | flags);
    if (ret) {
        DRM_ERROR("Failed to pin to GGTT: %d\n", ret);
        return ERR_PTR(ret);
    }
    
    // 3. Object now in GGTT at vma->node.start
    GEM_BUG_ON(!drm_mm_node_allocated(&vma->node));
    
    return vma;
}

// Usage:
vma = i915_gem_object_ggtt_pin(obj, NULL, size, 4096, 0);
ggtt_offset = vma->node.start;
```

### 3. Accessing GGTT-Mapped Memory

```c
// Reading/writing GGTT-mapped memory

// Case 1: From kernel (CPU access)
void read_ggtt_memory(struct i915_vma *vma, void *buf, size_t len)
{
    // Method 1: Via kernel virtual address (if in mappable range)
    if (vma->node.start < ggtt->mappable_end) {
        // Kernel has virtual mapping to GGTT
        void *kernel_addr = ggtt->mappable_addr + vma->node.start;
        memcpy(buf, kernel_addr, len);
    } else {
        // Beyond mappable range, use WC (write-combine) copy
        memcpy_fromio(buf, ggtt->base_addr + vma->node.start, len);
    }
}

// Case 2: From GPU (in shader/kernel)
void submit_gpu_command_using_ggtt(struct i915_request *rq,
                                   struct i915_vma *vma)
{
    // GPU shader code receives GGTT offset
    // Can directly access from any GPU context
    
    uint32_t ggtt_offset = vma->node.start;
    
    // In GPU command stream:
    // mov %eax, [ggtt_offset]  // Read from GGTT
    // mov [ggtt_offset], %ebx  // Write to GGTT
}

// Case 3: GPU firmware access
void firmware_read_status_page(void)
{
    // GPU firmware automatically sees GGTT
    // Status page at known GGTT offset
    // Firmware reads GPU completion status from GGTT
}
```

### 4. Unmapping from GGTT

```c
// Unbind and cleanup

void i915_gem_object_ggtt_unpin(struct i915_vma *vma)
{
    // 1. Unpin (allow eviction)
    i915_vma_unpin(vma);
    
    // 2. Destroy VMA
    i915_vma_destroy(vma);
    
    // Object now unmapped from GGTT
    // Physical memory can be freed or migrated
}
```

---

## PPGTT Operations

### 1. Context and PPGTT Creation

```c
// Create GPU context with PPGTT

struct i915_gem_context *i915_gem_context_create(
    struct drm_i915_private *dev_priv)
{
    struct i915_gem_context *ctx;
    int ret;
    
    // 1. Allocate context structure
    ctx = kzalloc(sizeof(*ctx), GFP_KERNEL);
    if (!ctx)
        return ERR_PTR(-ENOMEM);
    
    // 2. Create PPGTT for this context
    ctx->ppgtt = i915_ppgtt_create(dev_priv);
    if (IS_ERR(ctx->ppgtt)) {
        ret = PTR_ERR(ctx->ppgtt);
        kfree(ctx);
        return ERR_PTR(ret);
    }
    
    // 3. Initialize hardware state (LRC)
    ret = i915_lrc_init(ctx);
    if (ret) {
        i915_ppgtt_put(ctx->ppgtt);
        kfree(ctx);
        return ERR_PTR(ret);
    }
    
    // 4. Context now ready to use
    // PPGTT is private to this context
    
    return ctx;
}
```

### 2. Creating PPGTT

```c
// Initialize PPGTT (4-level page table)

struct i915_ppgtt *i915_ppgtt_create(
    struct drm_i915_private *dev_priv)
{
    struct i915_ppgtt *ppgtt;
    int ret;
    
    // 1. Allocate PPGTT structure
    ppgtt = kzalloc(sizeof(*ppgtt), GFP_KERNEL);
    if (!ppgtt)
        return ERR_PTR(-ENOMEM);
    
    // 2. Determine page table levels
    // Gen6-7: 2-level, Gen8+: 4-level
    ppgtt->mode = determine_ppgtt_mode(dev_priv);
    
    // 3. Allocate top-level page directory
    if (ppgtt->mode == 4_LEVEL) {
        ppgtt->pml4 = alloc_page_table_level();
        if (!ppgtt->pml4)
            return ERR_PTR(-ENOMEM);
    } else {
        ppgtt->pdp = alloc_page_table_level();
        if (!ppgtt->pdp)
            return ERR_PTR(-ENOMEM);
    }
    
    // 4. Map page tables to GGTT (kernel needs access)
    ret = bind_ppgtt_to_ggtt(ppgtt);
    if (ret)
        return ERR_PTR(ret);
    
    // 5. Initialize address space
    ret = i915_address_space_init(&ppgtt->vm, dev_priv);
    if (ret)
        return ERR_PTR(ret);
    
    return ppgtt;
}
```

### 3. Binding to PPGTT

```c
// Map object into context's PPGTT

struct i915_vma *i915_ppgtt_bind(
    struct drm_i915_gem_object *obj,
    struct i915_gem_context *ctx,
    u64 size,
    u32 alignment)
{
    struct i915_ppgtt *ppgtt = ctx->ppgtt;
    struct i915_vma *vma;
    int ret;
    
    // 1. Create VMA in context's PPGTT
    vma = i915_vma_create(obj, &ppgtt->vm, ctx);
    if (IS_ERR(vma))
        return vma;
    
    // 2. Pin into PPGTT (allocate VA range)
    ret = i915_vma_pin(vma, size, alignment, PIN_USER);
    if (ret) {
        i915_vma_destroy(vma);
        return ERR_PTR(ret);
    }
    
    // 3. Program PPGTT page tables
    // For each page:
    //   1. Find PML4 entry (bits 47-39)
    //   2. If PML4 invalid, allocate PDP
    //   3. Find PDP entry (bits 38-30)
    //   4. If PDP invalid, allocate PD
    //   5. Find PD entry (bits 29-21)
    //   6. If PD invalid, allocate PT
    //   7. Find PT entry (bits 20-12)
    //   8. Write PTE with physical address
    
    // Kernel does this during i915_vma_pin():
    ret = ppgtt->vm_ops->bind_vma(vma);
    if (ret) {
        i915_vma_unpin(vma);
        i915_vma_destroy(vma);
        return ERR_PTR(ret);
    }
    
    // Object now in PPGTT at vma->node.start
    uint64_t gpu_va = vma->node.start;
    
    return vma;
}
```

### 4. Accessing PPGTT-Mapped Memory from GPU

```c
// GPU shader accessing PPGTT memory

// Userspace application creates buffer in PPGTT:
uint64_t gpu_va = bind_buffer_to_ppgtt(buffer);

// Application submits shader:
vkCmdDispatch(command_buffer, 1024, 1024, 1);

// GPU executes shader:
__global__ void compute_kernel(
    uint64_t input_gpu_va,   // PPGTT address
    uint64_t output_gpu_va)  // PPGTT address
{
    // 1. GPU translates VA via PPGTT
    //    VA 0x1000 → page walk → physical frame
    
    // 2. GPU reads/writes using physical address
    float input_value = *(float*)(input_gpu_va);
    float output_value = compute(input_value);
    *(float*)(output_gpu_va) = output_value;
    
    // All translation happens automatically in GPU MMU
}

// From GPU perspective:
//   mov %eax, [gpua_va]  → GPU MMU translates via PPGTT
//                         → Physical memory accessed
```

### 5. Context Switching and PPGTT

```c
// Context switch choreography

void i915_gpu_context_switch(struct i915_gem_context *old_ctx,
                             struct i915_gem_context *new_ctx)
{
    // 1. Ensure GPU idle (wait for completion)
    i915_gpu_wait_for_idle();
    
    // 2. Save old context's hardware state
    i915_lrc_save(&old_ctx->lrc);
    
    // 3. Update PPGTT register to new context's PPGTT base
    uint64_t ppgtt_base = i915_ppgtt_base_address(new_ctx->ppgtt);
    i915_write(RING_PPGTT_BASE(engine), ppgtt_base);
    
    // 4. Optional: Flush TLB
    // (Some GPUs use ASID tagging instead)
    if (!supports_asid()) {
        i915_write(RING_TLB_INVALIDATE, 1);
    } else {
        // Update ASID register instead
        i915_write(RING_ASID(engine), new_ctx->asid);
    }
    
    // 5. Load new context's hardware state
    i915_lrc_load(&new_ctx->lrc);
    
    // 6. Resume GPU execution
    // GPU now uses new PPGTT for address translation
    
    // GGTT mappings unchanged (still visible)
}
```

---

## VMA Binding

### VMA Structure

```c
struct i915_vma {
    struct drm_i915_gem_object *obj;      // The object
    
    // Address space
    struct i915_address_space *vm;        // GGTT or PPGTT?
    
    // Position in address space
    struct drm_mm_node node;              // VA range allocated
    
    // Binding state
    u16 flags;
    #define I915_VMA_GLOBAL_BIND    BIT(0)  // Bound in GGTT
    #define I915_VMA_LOCAL_BIND     BIT(1)  // Bound in PPGTT
    #define I915_VMA_PIN_HIGH       BIT(2)
    #define I915_VMA_PIN_GLOBAL     BIT(3)
    
    // Page tables
    struct i915_page_table *page_table;   // For PPGTT
    
    // Optimization info
    unsigned long active;                 // Refcount for activity
};
```

### VMA Lifecycle

```c
// Complete VMA lifecycle

// 1. Create VMA
struct i915_vma *vma = i915_vma_instance(obj, vm, view);

// 2. Pin (bind to address space)
int ret = i915_vma_pin(vma, size, alignment, flags);
// At this point:
//   - Virtual address allocated
//   - Page tables updated
//   - Physical memory pinned (not evictable)
//   - Object accessible via GPU

// 3. Use in GPU work
// ... submit GPU commands using vma->node.start ...

// 4. Unpin (cleanup)
i915_vma_unpin(vma);
// At this point:
//   - Virtual address released
//   - Page tables cleared
//   - Physical memory can be evicted
//   - Object no longer accessible from GPU

// 5. Destroy VMA
i915_vma_destroy(vma);

// 6. Object might still exist (for other uses)
```

---

## Common Patterns

### Pattern 1: Kernel Operation Using GGTT

```c
// Kernel-driven memory copy in GPU

struct i915_request *copy_via_gpu_ggtt(
    struct drm_i915_gem_object *src_obj,
    struct drm_i915_gem_object *dst_obj,
    size_t size)
{
    struct i915_request *rq;
    struct i915_vma *src_vma, *dst_vma;
    
    // 1. Bind source to GGTT
    src_vma = i915_gem_object_ggtt_pin(src_obj, NULL, size, 0, 0);
    if (IS_ERR(src_vma))
        return (void*)src_vma;
    
    // 2. Bind destination to GGTT
    dst_vma = i915_gem_object_ggtt_pin(dst_obj, NULL, size, 0, 0);
    if (IS_ERR(dst_vma)) {
        i915_vma_unpin(src_vma);
        return (void*)dst_vma;
    }
    
    // 3. Create GPU request (work to do)
    rq = i915_request_create(engine);
    if (IS_ERR(rq)) {
        i915_vma_unpin(src_vma);
        i915_vma_unpin(dst_vma);
        return rq;
    }
    
    // 4. Add copy command to request
    // Using GGTT addresses (visible to all contexts)
    uint32_t src_offset = src_vma->node.start;
    uint32_t dst_offset = dst_vma->node.start;
    
    // GPU copy command (simplified):
    // for i in 0..size:
    //   dst[dst_offset + i] = src[src_offset + i]
    
    emit_copy_command(rq, src_offset, dst_offset, size);
    
    // 5. Submit request
    i915_request_add(rq);
    
    // 6. Cleanup (after GPU completes)
    // In callback:
    //   i915_vma_unpin(src_vma);
    //   i915_vma_unpin(dst_vma);
    
    return rq;
}
```

### Pattern 2: User Buffer in PPGTT

```c
// Application buffer mapping

long i915_gem_execbuffer_ioctl(struct drm_device *dev,
                               void *data,
                               struct drm_file *file)
{
    struct drm_i915_gem_execbuffer2 *args = data;
    struct i915_gem_context *ctx;
    struct drm_i915_gem_object *obj;
    struct i915_vma *vma;
    int i;
    
    // 1. Get user's context (has PPGTT)
    ctx = get_context_from_handle(file, args->rsvd1);
    
    // 2. For each buffer in submission
    for (i = 0; i < args->buffer_count; i++) {
        struct drm_i915_relocation_entry *reloc = 
            &args->buffers[i];
        
        // 3. Get user's GEM object
        obj = drm_gem_object_lookup(file, reloc->handle);
        
        // 4. Bind to context's PPGTT
        vma = i915_vma_instance(obj, &ctx->ppgtt->vm, NULL);
        if (IS_ERR(vma))
            return PTR_ERR(vma);
        
        // 5. Pin (allocate VA in PPGTT)
        int ret = i915_vma_pin(vma, 0, 0, PIN_USER);
        if (ret)
            return ret;
        
        // 6. Get VA for user's buffer
        uint64_t gpu_va = vma->node.start;
        reloc->offset = gpu_va;  // Return to userspace
    }
    
    // 7. Submit GPU work
    // Buffers now in user context's PPGTT
    // GPU can access via returned VA
    
    return 0;
}
```

### Pattern 3: Mixed GGTT + PPGTT

```c
// Memory migration example (uses both)

int migrate_object_to_vram(struct drm_i915_gem_object *obj,
                           struct i915_gem_context *ctx)
{
    struct i915_vma *src_vma, *dst_vma;
    struct i915_vma *copy_vma;  // Kernel work
    struct i915_request *rq;
    
    // 1. Create source binding (original location)
    // Typically System RAM in PPGTT
    src_vma = i915_vma_instance(obj, &ctx->ppgtt->vm, NULL);
    i915_vma_pin(src_vma, 0, 0, PIN_USER);
    
    // 2. Allocate VRAM for destination
    struct drm_i915_gem_object *vram_obj = 
        allocate_from_vram(obj->size);
    
    // 3. Create copy batch in GGTT (kernel work)
    struct drm_i915_gem_object *copy_batch =
        i915_gem_object_create_kernel(i915, COPY_BATCH_SIZE);
    
    copy_vma = i915_gem_object_ggtt_pin(copy_batch, NULL, 
                                        COPY_BATCH_SIZE, 0, 0);
    
    // 4. Bind VRAM destination to GGTT (temp)
    dst_vma = i915_gem_object_ggtt_pin(vram_obj, NULL, 
                                       obj->size, 0, 0);
    
    // 5. Create copy request
    rq = i915_request_create(engine);
    
    // 6. Fill copy batch
    // src = PPGTT address (app memory)
    // dst = GGTT address (VRAM, kernel temp)
    uint32_t src_addr = src_vma->node.start;  // PPGTT
    uint32_t dst_addr = dst_vma->node.start;  // GGTT
    
    fill_copy_batch(copy_batch, src_addr, dst_addr, obj->size);
    
    // 7. Submit
    i915_request_add(rq);
    
    // 8. After completion: rebind obj in PPGTT to VRAM
    i915_vma_unpin(src_vma);
    i915_ppgtt_rebind(src_vma, vram_obj);  // Points to VRAM now
    
    // 9. Cleanup temp bindings
    i915_vma_unpin(dst_vma);
    i915_vma_unpin(copy_vma);
    
    return 0;
}
```

---

## Real-World Examples

### Example 1: Display Framebuffer Setup

```c
// From intel_display.c

int intel_display_set_plane(struct drm_plane *plane,
                            struct drm_crtc *crtc,
                            struct drm_framebuffer *fb,
                            int crtc_x, int crtc_y)
{
    struct intel_framebuffer *intel_fb = 
        to_intel_framebuffer(fb);
    struct drm_i915_gem_object *obj = intel_fb->obj;
    struct i915_vma *vma;
    int ret;
    
    // 1. Bind framebuffer to GGTT
    // (display engine not a GPU context)
    vma = i915_gem_object_ggtt_pin(
        obj, 
        NULL,
        fb->pitches[0],  // Pitch for scanning
        0,
        PIN_MAPPABLE);
    
    if (IS_ERR(vma)) {
        DRM_ERROR("Failed to pin framebuffer\n");
        return PTR_ERR(vma);
    }
    
    // 2. Get GGTT address for display controller
    uint32_t display_addr = i915_ggtt_offset(vma);
    
    // 3. Program display registers
    // Display controller will scan GGTT address
    I915_WRITE(DSPADDR(plane), display_addr);
    I915_WRITE(DSPTILEOFF(plane), crtc_x | (crtc_y << 16));
    I915_WRITE(DSPSTRIDE(plane), fb->pitches[0]);
    
    // 4. Enable plane
    I915_WRITE(DSPCONTR(plane), 
               DISPLAY_PLANE_ENABLE | /* other bits */);
    
    // Framebuffer now being scanned from GGTT
    // Will remain pinned while displayed
    
    return 0;
}
```

### Example 2: GPU Status Page

```c
// From intel_gt.c

int intel_gt_setup_status_page(struct intel_gt *gt)
{
    struct i915_vma *vma;
    struct page *page;
    int ret;
    
    // 1. Allocate physical page
    page = alloc_page(GFP_KERNEL);
    if (!page)
        return -ENOMEM;
    
    // 2. Create GEM object for status page
    struct drm_i915_gem_object *obj =
        i915_gem_object_create_from_page(i915, page);
    if (IS_ERR(obj))
        return PTR_ERR(obj);
    
    // 3. Bind to GGTT (firmware needs access)
    vma = i915_gem_object_ggtt_pin(obj, NULL, PAGE_SIZE, 0, 0);
    if (IS_ERR(vma)) {
        i915_gem_object_put(obj);
        return PTR_ERR(vma);
    }
    
    // 4. Get GGTT offset
    gt->status_page_offset = i915_ggtt_offset(vma);
    gt->status_page = page_address(page);
    
    // 5. Tell GPU where to write status
    I915_WRITE(GFXPAUSE_STATUS_PAGE_ADDR(gt),
               gt->status_page_offset);
    
    // 6. GPU firmware now uses GGTT address
    // Can write completion status to this page
    // Visible to all GPU contexts (in GGTT)
    
    return 0;
}
```

### Example 3: User Buffer Binding

```c
// From gem_exec_ioctl.c

long i915_gem_execbuffer_ioctl_impl(
    struct drm_device *dev,
    struct drm_i915_gem_execbuffer2 *args,
    struct drm_file *file)
{
    struct drm_i915_private *dev_priv = 
        to_drm_i915_private(dev);
    struct i915_gem_context *ctx;
    struct i915_execbuffer eb;
    int ret;
    
    // 1. Get user's context (with PPGTT)
    ctx = i915_gem_context_lookup(file->driver_priv, 
                                  args->rsvd1);
    if (!ctx)
        return -ENOENT;
    
    // 2. Initialize execbuffer
    memset(&eb, 0, sizeof(eb));
    eb.context = ctx;
    eb.file = file;
    
    // 3. Process validation list (buffers)
    ret = i915_gem_execbuffer_parse_vmas(dev_priv, &eb,
                                         args);
    if (ret)
        goto unlock;
    
    // i915_gem_execbuffer_parse_vmas does:
    // for each buffer in args->buffers:
    //     obj = lookup_gem_object(handle)
    //     vma = i915_vma_instance(obj, &ctx->ppgtt->vm, NULL)
    //     i915_vma_pin(vma, size, alignment, PIN_USER)
    //     // Now object bound in context's PPGTT
    //     // GPU can access via vma->node.start
    
    // 4. Submit GPU work
    ret = i915_gem_execbuffer_move_to_gpu(dev_priv, &eb);
    if (ret)
        goto unpin;
    
    // 5. Unpin (allows eviction later)
    i915_gem_execbuffer_unpin_vmas(dev_priv, &eb);
    
    return ret;
}
```

---

## Error Handling

### GGTT Exhaustion

```c
// Handle "No space in GGTT" error

int handle_ggtt_exhaustion(struct i915_ggtt *ggtt,
                          size_t needed)
{
    // 1. Check remaining space
    size_t available = ggtt_available_space(ggtt);
    if (available >= needed)
        return 0;  // Unexpected
    
    DRM_WARN("GGTT exhausted: need %zu, have %zu\n",
             needed, available);
    
    // 2. Try to evict objects
    int ret = i915_gem_evict_ggtt(ggtt);
    if (ret)
        return -ENOSPC;
    
    // 3. Retry
    available = ggtt_available_space(ggtt);
    if (available < needed)
        return -ENOSPC;  // Still not enough
    
    return 0;
}

// Usage:
struct i915_vma *vma = i915_gem_object_ggtt_pin(obj, ...);
if (IS_ERR(vma)) {
    if (PTR_ERR(vma) == -ENOSPC) {
        ret = handle_ggtt_exhaustion(ggtt, obj->size);
        if (!ret)
            vma = i915_gem_object_ggtt_pin(obj, ...);
    }
}
```

### PPGTT Page Table Allocation Failure

```c
// Handle page table allocation failure

int handle_ppgtt_alloc_failure(struct i915_ppgtt *ppgtt,
                              u64 va, size_t size)
{
    // 1. Check what failed
    // Page table allocation failed during binding
    
    DRM_ERROR("PPGTT page table allocation failed\n");
    DRM_DEBUG("  VA: 0x%llx\n", va);
    DRM_DEBUG("  Size: %zu bytes\n", size);
    
    // 2. Options:
    
    // Option A: Evict other objects to free memory
    int ret = i915_gem_evict_ppgtt_range(ppgtt, va, size);
    if (ret)
        return -ENOMEM;
    
    // Option B: Reduce size request
    // Application may need to split allocation
    
    // Option C: Use GGTT for small allocations
    // (if applicable for this use case)
    
    return -ENOMEM;
}
```

---

## Performance Tips

### Tip 1: Minimize GGTT Usage

```c
// ANTI-PATTERN: Everything in GGTT
for (int i = 0; i < 1000; i++) {
    vma = i915_gem_object_ggtt_pin(large_obj[i], ...);
    // Fragments GGTT quickly!
}

// BETTER: Use PPGTT for user objects
for (int i = 0; i < 1000; i++) {
    vma = i915_vma_instance(large_obj[i], &ppgtt->vm, NULL);
    i915_vma_pin(vma, ...);
    // Spreads load across 48TB PPGTT address space
}
```

### Tip 2: Batch PPGTT Bindings

```c
// SLOW: Bind one by one (TLB thrashing)
for (each buffer) {
    ppgtt_bind_single_buffer(buffer);
    // Causes TLB misses for each binding
}

// FASTER: Batch bindings
struct list_head buffers;
// Collect all buffers...
i915_ppgtt_bind_batch(&ppgtt->vm, &buffers);
// Single TLB flush, better cache behavior
```

### Tip 3: Reuse VMAs

```c
// SLOW: Create/destroy VMA repeatedly
for (each frame) {
    vma = i915_vma_instance(obj, vm, NULL);
    i915_vma_pin(vma, ...);
    // ... use vma ...
    i915_vma_unpin(vma);
    i915_vma_destroy(vma);
}

// BETTER: Reuse VMA
vma = i915_vma_instance(obj, vm, NULL);
i915_vma_pin(vma, ...);
for (each frame) {
    // ... use same vma ...
}
i915_vma_unpin(vma);
i915_vma_destroy(vma);
```

---

## Debugging Guide

### Check GGTT Usage

```bash
# View GGTT bindings
cat /sys/kernel/debug/dri/0/i915_ggtt_bindings

# Example output:
# Address     Size   Object
# 0x0         0x1000 status_page
# 0x1000      0x1000 display_fb
# 0x100000    0x10000 gpu_command_buffer
```

### Check PPGTT

```bash
# View context info
cat /sys/kernel/debug/dri/0/i915_contexts

# View specific PPGTT
cat /sys/kernel/debug/dri/0/i915_ppgtt_0

# Check page tables
cat /sys/kernel/debug/dri/0/i915_page_tables
```

### Monitor Performance

```bash
# Track page table updates
perf trace -e "gpu_mem:ppgtt_*" ./workload

# Monitor TLB behavior
perf stat -e tlb_miss,tlb_hit,dtlb_miss,itlb_miss ./workload

# Profile GGTT/PPGTT operations
perf record -e gpu_mem -g ./workload
perf report
```

---

**End of GGTT/PPGTT Implementation Guide**
