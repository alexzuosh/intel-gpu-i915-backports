# 22. GPU Security and Sandboxing

## Executive Summary

GPU security encompasses access control, memory isolation, and protection mechanisms that prevent unauthorized GPU access and ensure workload isolation. The i915 driver implements multiple layers of security including context isolation, command validation, virtual address space separation, and GuC firmware security. Understanding GPU security is essential for system administrators, embedded systems engineers, and anyone concerned with multi-tenant GPU deployments.

**Key Concepts:**
- Context and address space isolation
- Privileged command filtering
- GuC firmware security and verification
- Memory protection and access control
- Hardware-based security features

**Table of Contents**
1. [Security Architecture](#architecture)
2. [Context Isolation](#context-isolation)
3. [Command Validation and Filtering](#command-validation)
4. [Virtual Address Space Protection](#virtual-address-space)
5. [GuC Security](#guc-security)
6. [Hardware Security Features](#hardware-features)
7. [Sandboxing Strategies](#sandboxing)
8. [Access Control](#access-control)
9. [Security Auditing](#auditing)
10. [Best Practices](#best-practices)

---

## Architecture

### Security Model Overview

```
Multi-User System Security Model:
┌────────────────────────────────────────────────────────┐
│ Application 1 (User A) | Application 2 (User B)       │
├────────────────────────────────────────────────────────┤
│ i915 Kernel Driver (Validation Layer)                  │
│ ├─ Command whitelist validation                        │
│ ├─ Memory access control                               │
│ ├─ IOMMU/GGTT translation                             │
│ └─ Context isolation enforcement                       │
├────────────────────────────────────────────────────────┤
│ GPU Hardware                                            │
│ ├─ GPU contexts (isolated address spaces)              │
│ ├─ Translation lookaside buffer (TLB) per-context      │
│ └─ Hardware access control (if available)              │
└────────────────────────────────────────────────────────┘
```

### Security Layers

```
Layer 1: User Space Validation
   ├─ Application should submit valid batches
   └─ Not trusted (user can be malicious)

Layer 2: Kernel Validation (PRIMARY SECURITY)
   ├─ Strict command whitelist enforcement
   ├─ Memory access checking
   ├─ IOMMU page table validation
   └─ Trusted execution

Layer 3: Hardware Isolation
   ├─ GPU virtual address space per-context
   ├─ Hardware context isolation
   ├─ Memory translation isolation
   └─ Engine-specific access control

Layer 4: Firmware (GuC) Enforcement
   ├─ Final command validation
   ├─ Preemption security
   └─ Hardware protection
```

---

## Context Isolation

### Context and Address Space

```c
/* Each GPU context has isolated address space */
struct i915_gem_context {
    struct drm_i915_private *i915;
    
    /* Context identity */
    unsigned long flags;
    u32 user_handle;                /* User-visible context handle */
    int id;                         /* Internal context ID */
    char *name;
    
    /* Address space (virtual address mapping) */
    struct i915_address_space *vm;  /* Per-context VM */
    
    /* Security attributes */
    struct {
        bool protected;             /* Protected content context */
        bool restricted;            /* Restricted permissions */
    } security;
    
    /* Execution tracking */
    struct list_head active_requests;
};

/* Virtual address space per-context */
struct i915_address_space {
    struct drm_mm vm;               /* Virtual address allocator */
    struct drm_i915_gem_object *scratch_page;
    
    /* Page tables (GPU-controlled) */
    struct drm_i915_gem_object *pd;    /* Page directory */
    struct drm_i915_gem_object *pt[VM_MAX_LEVELS]; /* Page tables */
    
    /* IOMMU protection */
    struct iommu_domain *iommu;
};
```

### Memory Isolation

```c
/* Ensure context cannot access memory outside its VM */
static int validate_memory_access(struct i915_gem_context *ctx,
                                  u64 gpu_address,
                                  u64 size)
{
    struct i915_address_space *vm = ctx->vm;
    struct drm_mm_node *node;
    int ret;
    
    /* Check if address is within context's VM range */
    node = drm_mm_search_range(&vm->vm, gpu_address,
                               gpu_address + size);
    
    if (!node) {
        DRM_DEBUG_DRIVER("Access outside VM range: 0x%llx (size: 0x%llx)\n",
                        gpu_address, size);
        return -EFAULT;
    }
    
    /* Verify page table permissions */
    ret = check_page_table_permissions(vm, node);
    if (ret < 0) {
        DRM_DEBUG_DRIVER("Page table permission denied\n");
        return ret;
    }
    
    return 0;
}
```

### Preventing Cross-Context Access

```c
/* Prevent unauthorized memory access between contexts */
static void setup_context_isolation(struct i915_gem_context *ctx)
{
    /* Set up per-context page tables */
    setup_page_tables(ctx->vm);
    
    /* Configure TLB to be per-context */
    if (HAS_CONTEXT_ISOLATION(ctx->i915)) {
        struct drm_i915_private *dev_priv = ctx->i915;
        
        /* Enable VM (virtual memory) context */
        I915_WRITE(RING_VM_CTX_LO(RENDER_RING_BASE),
                  lower_32_bits(ctx->vm->pd_gpu_addr));
        I915_WRITE(RING_VM_CTX_HI(RENDER_RING_BASE),
                  upper_32_bits(ctx->vm->pd_gpu_addr));
        
        /* Flush TLB for new context */
        gen6_flush_tlb(dev_priv);
    }
}
```

---

## Command Validation and Filtering

### Privileged Command Whitelist

```c
/* Commands allowed in user-submitted batches */
enum command_validation_result {
    ALLOWED = 0,           /* Safe for untrusted batch */
    REQUIRES_CLFLUSH = 1,  /* Needs cache flush */
    REJECTED = -1,         /* Privileged/unsafe */
};

/* Command validation table */
static const char cmd_validation[] = {
    /* MI (Memory Interface) commands */
    [0x00] = ALLOWED,                   /* MI_NOOP */
    [0x04] = ALLOWED,                   /* MI_WAIT_FOR_EVENT */
    [0x08] = ALLOWED,                   /* MI_ARB_ON_OFF */
    [0x0c] = ALLOWED,                   /* MI_ARB_CHECK */
    [0x1c] = REQUIRES_CLFLUSH,         /* MI_BATCH_BUFFER_START (dangerous) */
    [0x20] = ALLOWED,                   /* MI_STORE_DATA_IMM */
    [0x21] = ALLOWED,                   /* MI_STORE_DATA_INDEX */
    [0x24] = ALLOWED,                   /* MI_FLUSH */
    [0x31] = REJECTED,                  /* MI_CONDITIONAL_BATCH_BUFFER_END */
    [0x33] = REJECTED,                  /* MI_STORE_DWORD_INDEX */
    
    /* All other MI commands: REJECTED */
};

static enum command_validation_result
validate_command(u32 cmd_opcode)
{
    u32 idx = (cmd_opcode >> 23) & 0xff;  /* Extract command bits */
    
    return cmd_validation[idx];
}
```

### Per-Command Validation

```c
/* Detailed validation for specific dangerous commands */
static int validate_mi_store_data_imm(u32 *cmd, int cmd_length)
{
    u64 address;
    
    if (cmd_length < 4) {
        DRM_ERROR("MI_STORE_DATA_IMM too short\n");
        return -EINVAL;
    }
    
    /* Extract address from command */
    address = ((u64)cmd[3] << 32) | cmd[2];
    
    /* Whitelist of allowed addresses */
    if (address == DEFERRED_HWS_ADDRESS ||
        address == SCRATCH_PAGE_ADDRESS ||
        address == CONTEXT_SAVE_ADDRESS) {
        return 0;  /* Allowed */
    }
    
    /* All other addresses rejected */
    DRM_ERROR("MI_STORE_DATA_IMM to unauthorized address: 0x%llx\n",
             address);
    return -EPERM;
}
```

### Register Access Control

```c
/* Prevent privileged register access via batch commands */
static const u32 restricted_registers[] = {
    RING_BUFFER_TAIL(RENDER_RING_BASE),
    RING_BUFFER_HEAD(RENDER_RING_BASE),
    RING_BUFFER_CTL(RENDER_RING_BASE),
    RENDER_HWS_PGA(RENDER_RING_BASE),
    RING_TIMESTAMP(RENDER_RING_BASE),
    /* ... more privileged registers ... */
};

static int is_register_restricted(u32 reg_addr)
{
    int i;
    
    for (i = 0; i < ARRAY_SIZE(restricted_registers); i++) {
        if (reg_addr == restricted_registers[i])
            return 1;  /* Restricted */
    }
    
    return 0;  /* Not restricted */
}
```

---

## Virtual Address Space Protection

### IOMMU Integration

```c
/* IOMMU (Input/Output Memory Management Unit) protects GPU access */
static int setup_iommu_for_context(struct i915_gem_context *ctx)
{
    struct drm_i915_private *dev_priv = ctx->i915;
    struct iommu_domain *domain;
    
    if (!dev_priv->iommu_enabled)
        return 0;
    
    /* Create IOMMU domain for context */
    domain = iommu_domain_alloc(&pci_bus_type);
    if (!domain)
        return -ENOMEM;
    
    /* Attach GPU device to IOMMU domain */
    if (iommu_attach_device(domain, &dev_priv->drm.pdev->dev)) {
        iommu_domain_free(domain);
        return -ENODEV;
    }
    
    ctx->vm->iommu = domain;
    
    DRM_DEBUG_DRIVER("IOMMU enabled for context %d\n", ctx->id);
    return 0;
}

/* Translate GPU virtual address to physical via IOMMU */
static phys_addr_t get_iommu_mapped_address(struct i915_address_space *vm,
                                           u64 gpu_addr)
{
    phys_addr_t phys;
    
    if (!vm->iommu)
        return 0;  /* No IOMMU, invalid */
    
    phys = iommu_iova_to_phys(vm->iommu, gpu_addr);
    
    return phys;
}
```

### Page Table Protection

```c
/* Validate all GPU page table accesses */
static int validate_page_table_entry(struct i915_address_space *vm,
                                    u64 pte_value)
{
    /* Extract address from PTE */
    u64 phys_addr = pte_value & I915_PTE_ADDR_MASK;
    
    /* Ensure address is not in privileged memory */
    if (phys_addr >= BIOS_AREA_START && phys_addr <= BIOS_AREA_END) {
        DRM_ERROR("PTE points to BIOS area: 0x%llx\n", phys_addr);
        return -EPERM;
    }
    
    /* Ensure address is in valid system memory */
    if (!is_valid_system_memory(phys_addr)) {
        DRM_ERROR("PTE points to invalid memory: 0x%llx\n", phys_addr);
        return -EFAULT;
    }
    
    /* Check memory permissions (if supported) */
    if (HAS_MEMORY_PERMISSIONS(vm->i915)) {
        int perms = get_memory_permissions(phys_addr);
        
        if (!(perms & MEM_PERM_GPU_READABLE)) {
            DRM_ERROR("Memory not readable by GPU: 0x%llx\n", phys_addr);
            return -EACCES;
        }
    }
    
    return 0;  /* Valid */
}
```

---

## GuC Security

### Firmware Verification

```c
/* Verify GuC firmware before loading */
static int verify_guc_firmware(struct intel_guc *guc,
                              const struct firmware *fw)
{
    struct guc_fw_header *header;
    u32 expected_checksum, calculated_checksum;
    int ret;
    
    if (fw->size < sizeof(*header)) {
        DRM_ERROR("GuC firmware too small\n");
        return -EINVAL;
    }
    
    header = (struct guc_fw_header *)fw->data;
    
    /* Validate signature */
    ret = verify_guc_signature(fw->data, fw->size, header->signature);
    if (ret < 0) {
        DRM_ERROR("GuC firmware signature invalid\n");
        return ret;
    }
    
    /* Verify checksum */
    expected_checksum = header->checksum;
    calculated_checksum = compute_firmware_checksum(
        fw->data + sizeof(*header),
        fw->size - sizeof(*header));
    
    if (expected_checksum != calculated_checksum) {
        DRM_ERROR("GuC firmware checksum mismatch\n");
        return -EINVAL;
    }
    
    DRM_DEBUG_DRIVER("GuC firmware verified successfully\n");
    return 0;
}

/* GuC firmware header */
struct guc_fw_header {
    u32 magic;              /* Magic number validation */
    u32 version;
    u32 size;
    u32 checksum;
    u8 signature[256];      /* RSA signature */
};
```

### GuC Command Sandboxing

```c
/* Validate GuC command before transmission */
static int validate_guc_command(const u32 *action, u32 len)
{
    u32 cmd_id = action[0];
    
    switch (cmd_id) {
    /* Allowed commands */
    case INTEL_GUC_ACTION_SAMPLE_FORCEWAKE:
    case INTEL_GUC_ACTION_ALLOCATE_DOORBELL:
    case INTEL_GUC_ACTION_DEALLOCATE_DOORBELL:
    case INTEL_GUC_ACTION_SUBMIT_EXEC_QUEUE:
        return 0;  /* Safe */
        
    /* Privileged commands (kernel-only) */
    case INTEL_GUC_ACTION_ENABLE_SCHEDULING:
    case INTEL_GUC_ACTION_DISABLE_SCHEDULING:
    case INTEL_GUC_ACTION_RESET_CONTEXT:
        if (!in_kernel_context())
            return -EPERM;  /* User cannot execute */
        return 0;
        
    default:
        DRM_ERROR("Unknown GuC command: 0x%x\n", cmd_id);
        return -EINVAL;
    }
}
```

---

## Hardware Security Features

### TLB and Translation Protection

```c
/* Hardware translation lookaside buffer (TLB) isolation */
static void setup_tlb_isolation(struct intel_engine_cs *engine)
{
    u32 tlb_mask = 0;
    
    /* Configure per-context TLB (if supported) */
    if (HAS_PER_CONTEXT_TLB(engine->i915)) {
        /* Enable per-context TLB for isolation */
        tlb_mask = GFX_TLB_INVALIDATE_PER_CONTEXT;
    }
    
    /* Flush TLB on context switch */
    I915_WRITE(RING_INSTPM(engine->mmio_base),
              I915_READ(RING_INSTPM(engine->mmio_base)) | tlb_mask);
    
    DRM_DEBUG_DRIVER("TLB isolation configured for %s\n", engine->name);
}

/* Invalidate TLB for security */
static void invalidate_tlb(struct intel_engine_cs *engine)
{
    /* Full TLB invalidation (slow but secure) */
    u32 val = I915_READ(RING_INSTPM(engine->mmio_base));
    I915_WRITE(RING_INSTPM(engine->mmio_base), val | INSTPM_TLB_INVALIDATE);
    
    /* Wait for invalidation to complete */
    if (wait_for((I915_READ(RING_INSTPM(engine->mmio_base)) & 
                 INSTPM_TLB_INVALIDATE) == 0, 1000)) {
        DRM_ERROR("TLB invalidation timeout\n");
    }
}
```

### Hardware Access Control

```c
/* Hardware-enforced access control (on supported platforms) */
#define HAS_HW_ACCESS_CONTROL(dev) (INTEL_GEN(dev) >= 12)

static int setup_hw_access_control(struct i915_gem_context *ctx)
{
    struct drm_i915_private *dev_priv = ctx->i915;
    
    if (!HAS_HW_ACCESS_CONTROL(dev_priv))
        return 0;  /* Not supported */
    
    /* Configure hardware access control registers */
    I915_WRITE(RING_EXTENDED_CONTEXT(RENDER_RING_BASE),
              I915_READ(RING_EXTENDED_CONTEXT(RENDER_RING_BASE)) |
              EXTENDED_CONTEXT_ENABLE_ACCESS_CONTROL);
    
    /* Set context-specific access mask */
    I915_WRITE(CONTEXT_ACCESS_MASK(ctx->hw_id), 
              CONTEXT_ACCESS_MASK_DEFAULT);
    
    DRM_DEBUG_DRIVER("Hardware access control configured\n");
    return 0;
}
```

---

## Sandboxing Strategies

### Protected Memory Regions

```c
/* Designate protected memory regions for sensitive content */
struct protected_region {
    u64 start;          /* Physical address start */
    u64 end;            /* Physical address end */
    int context_id;     /* Owning context (-1 = kernel only) */
    int access_mask;    /* Who can access (read/write flags) */
};

static int allocate_protected_memory(struct i915_gem_context *ctx,
                                     u64 size,
                                     struct protected_region *region)
{
    struct drm_i915_private *dev_priv = ctx->i915;
    
    /* Allocate from protected memory pool */
    region->start = allocate_protected_pool(dev_priv, size);
    region->end = region->start + size;
    region->context_id = ctx->id;
    region->access_mask = ACCESS_OWNER_ONLY;
    
    /* Register with IOMMU for protection */
    setup_iommu_protection(region);
    
    DRM_DEBUG_DRIVER("Protected region allocated: 0x%llx-0x%llx\n",
                    region->start, region->end);
    return 0;
}
```

### Execution Timeout Enforcement

```c
/* Enforce execution timeout to prevent DoS */
struct execution_timeout {
    u32 context_id;
    unsigned long deadline;
    struct timer_list timer;
};

static void timeout_expired(unsigned long data)
{
    struct execution_timeout *timeout = (struct execution_timeout *)data;
    struct intel_context *ctx = find_context(timeout->context_id);
    
    DRM_WARN("Execution timeout for context %d\n", timeout->context_id);
    
    /* Preempt the runaway context */
    i915_context_preempt(ctx);
    
    /* Optionally disable context on repeated timeouts */
    if (++ctx->timeout_count > MAX_TIMEOUT_COUNT)
        i915_context_disable(ctx);
}

#define DEFAULT_EXECUTION_TIMEOUT_MS 5000
#define MAX_TIMEOUT_COUNT 10
```

---

## Access Control

### User Privilege Checking

```c
/* Enforce user privilege levels */
enum drm_i915_gem_privilege {
    I915_PRIVILEGE_USER = 0,        /* Normal user */
    I915_PRIVILEGE_SYSTEM = 1,      /* System (kernel) */
    I915_PRIVILEGE_ADMIN = 2,       /* Administrator */
};

static int check_user_privilege(struct drm_file *file,
                               enum drm_i915_gem_privilege required_priv)
{
    int user_priv = get_user_privilege(file);
    
    if (user_priv < required_priv) {
        DRM_DEBUG_DRIVER("Insufficient privilege: required %d, have %d\n",
                        required_priv, user_priv);
        return -EPERM;
    }
    
    return 0;
}

static enum drm_i915_gem_privilege get_user_privilege(struct drm_file *file)
{
    if (capable(CAP_SYS_ADMIN))
        return I915_PRIVILEGE_ADMIN;
    
    /* Check if user has GPU access permission */
    if (file->is_master)
        return I915_PRIVILEGE_SYSTEM;
    
    return I915_PRIVILEGE_USER;
}
```

### Resource Limits per Context

```c
/* Enforce resource limits to prevent DoS */
struct context_resource_limits {
    u64 max_memory;              /* Max memory allocation */
    u32 max_contexts;            /* Max sub-contexts */
    u32 max_pending_requests;    /* Max queued work */
    unsigned int max_execution_time_ms;
};

static int check_resource_limits(struct i915_gem_context *ctx,
                                u64 allocation_size)
{
    struct context_resource_limits *limits = &ctx->resource_limits;
    
    if (ctx->total_memory + allocation_size > limits->max_memory) {
        DRM_DEBUG_DRIVER("Memory limit exceeded: %llu + %llu > %llu\n",
                        ctx->total_memory, allocation_size,
                        limits->max_memory);
        return -ENOSPC;
    }
    
    if (list_count_nodes(&ctx->active_requests) >= 
        limits->max_pending_requests) {
        DRM_DEBUG_DRIVER("Too many pending requests\n");
        return -EBUSY;
    }
    
    return 0;
}
```

---

## Security Auditing

### Security Event Logging

```c
/* Log security-relevant events for auditing */
enum security_event_type {
    SEC_EVENT_COMMAND_REJECTED = 0,
    SEC_EVENT_CONTEXT_VIOLATION = 1,
    SEC_EVENT_PRIVILEGE_DENIED = 2,
    SEC_EVENT_MEMORY_ACCESS_VIOLATION = 3,
    SEC_EVENT_TIMEOUT = 4,
};

static void log_security_event(enum security_event_type event,
                              const char *description,
                              u32 context_id)
{
    struct timespec64 ts;
    
    ktime_get_real_ts64(&ts);
    
    DRM_AUDIT("SECURITY: type=%d ctx=%d event=%s ts=%lld.%09ld\n",
             event, context_id, description,
             ts.tv_sec, ts.tv_nsec);
}
```

### Violation Detection

```c
/* Detect and report security violations */
static void detect_security_violations(struct drm_i915_private *dev_priv)
{
    u32 violation_status = I915_READ(GPU_SECURITY_VIOLATION_STATUS);
    
    if (violation_status & MEMORY_PROTECTION_VIOLATION) {
        DRM_ERROR("Memory protection violation detected\n");
        log_security_event(SEC_EVENT_MEMORY_ACCESS_VIOLATION,
                          "Memory protection violation", 0);
        trigger_gpu_hang_recovery(dev_priv);
    }
    
    if (violation_status & PRIVILEGE_ESCALATION_ATTEMPT) {
        DRM_ERROR("Privilege escalation attempt detected\n");
        log_security_event(SEC_EVENT_PRIVILEGE_DENIED,
                          "Privilege escalation attempt", 0);
    }
    
    /* Clear violation flags */
    I915_WRITE(GPU_SECURITY_VIOLATION_STATUS, violation_status);
}
```

---

## Best Practices

### 1. **Defense in Depth**
- Implement multiple security layers (kernel, firmware, hardware)
- Don't rely on a single security mechanism
- Validate at each layer independently

### 2. **Principle of Least Privilege**
- Grant minimum necessary permissions
- Use whitelisting instead of blacklisting commands
- Restrict register access strictly

### 3. **Isolation Strategy**
- Maintain separate address spaces per context
- Use IOMMU for additional isolation when available
- Enforce TLB invalidation on context switches

### 4. **Audit and Monitoring**
- Log all security-relevant events
- Monitor execution timeouts and violations
- Implement intrusion detection mechanisms

### 5. **Firmware Security**
- Verify firmware signatures before loading
- Keep firmware up-to-date with security patches
- Limit firmware command capabilities

---

## References

- **Intel GPU Security**: Hardware and firmware security mechanisms
- **Context Isolation**: Per-context address space and memory protection
- **Command Validation**: Whitelist-based command security
- **IOMMU Integration**: I/O memory management for GPU protection
- **GuC Firmware Security**: Firmware verification and sandboxing

**Related Topics:**
- [Hardware Discovery and Initialization](10-Hardware-Discovery-Initialization.md)
- [Firmware Loading and Management](13-Firmware-Loading-Management.md)
- [Context Management](03-Context-Management.md)
- [User-Space Interface (UAPI)](15-User-Space-Interface-UAPI.md)
