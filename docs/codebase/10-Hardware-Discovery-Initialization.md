# Hardware Discovery and Initialization

**Document ID:** 10 | **Status:** Complete | **Last Updated:** February 2026

---

## Executive Summary

Hardware discovery and initialization is the **first and most critical phase** of the i915 driver lifecycle. This document covers how the driver detects GPU hardware, initializes hardware subsystems, and configures the device for operation.

### Key Topics
- **Boot sequence:** Firmware initialization → hardware detection → subsystem setup
- **Feature detection:** Determining GPU capabilities and version
- **Device initialization:** Configuring hardware, allocating resources
- **Performance impact:** Initialization overhead and boot time

### Performance Baseline (iGPU)
- **Boot-to-ready:** ~1-2 seconds
- **Subsystem init:** ~10-100ms per component
- **Memory init:** ~50-200ms

---

## Table of Contents

1. [Boot and Driver Loading](#boot-and-driver-loading)
2. [Hardware Detection](#hardware-detection)
3. [Feature Flag Management](#feature-flag-management)
4. [Subsystem Initialization](#subsystem-initialization)
5. [Resource Allocation](#resource-allocation)
6. [Device Readiness](#device-readiness)
7. [Initialization Debugging](#initialization-debugging)
8. [Summary & Best Practices](#summary--best-practices)

---

## Boot and Driver Loading

### Module Loading and Bind

```plaintext
┌─────────────────────────────────────────────────────────┐
│ System Boot                                             │
│   └─ PCI Bus Enumeration                               │
│       └─ detect i915 PCI devices                       │
└────────────────┬────────────────────────────────────────┘
                 │
         ┌───────▼────────┐
         │ PCI Probe Callback
         │ i915_pci_probe()
         └───────┬────────┘
                 │
    ┌────────────▼──────────────┐
    │ Allocate i915_drm_private │
    │ Initialize PCI device     │
    │ Request memory regions    │
    └────────────┬──────────────┘
                 │
    ┌────────────▼──────────────┐
    │ drm_dev_register()        │
    │   └─ i915_load_firmware()│
    │   └─ intel_device_info_init()
    │   └─ init_subsystems()    │
    └────────────┬──────────────┘
                 │
         ┌───────▼────────┐
         │ Device Ready   │
         └────────────────┘
```

### PCI Device Initialization

```c
// Key initialization function
static int i915_pci_probe(struct pci_dev *pdev,
                          const struct pci_device_id *ent)
{
    struct i915_drm_private *i915;
    
    // 1. Allocate private data structure
    i915 = devm_drm_dev_alloc(&pdev->dev, &i915_drm_driver,
                               typeof(*i915), drm);
    
    // 2. Enable PCI device
    pci_enable_device(pdev);
    
    // 3. Request memory regions and interrupts
    pci_request_regions(pdev, driver.name);
    
    // 4. Map memory-mapped I/O regions
    i915->uncore.regs = pci_iomap(pdev, 0, 0);
    
    // 5. Detect hardware and initialize
    intel_detect_device(i915);
    
    // 6. Register with DRM subsystem
    drm_dev_register(&i915->drm, 0);
    
    return 0;
}
```

---

## Hardware Detection

### Device ID Recognition

The driver identifies GPU capabilities through device ID and revision:

```plaintext
┌──────────────────────────────────────────────────────┐
│ PCI Device ID Lookup                                 │
│   └─ device_id (e.g., 0x1916 = Skylake GT2)        │
│       └─ match against pci_device_id_list[]         │
│           └─ retrieve platform_info                 │
└─────────────┬────────────────────────────────────────┘
              │
    ┌─────────▼────────────────────────────┐
    │ struct intel_platform_info            │
    │   ├─ gen (generation)                 │
    │   ├─ gt_count (# of graphics tiles)  │
    │   ├─ num_slices                       │
    │   ├─ eu_per_subslice                 │
    │   ├─ has_* flags (capabilities)      │
    │   └─ frequencies (min, max, RPN)     │
    └─────────────────────────────────────┘
```

### MMIO Enumeration

```c
// Detect hardware version and capabilities
static void intel_device_info_init(struct i915_drm_private *i915)
{
    u32 ver = intel_detect_device_gen(i915);
    u32 eu_count = intel_detect_eu_count(i915);
    u32 slice_count = intel_detect_slice_count(i915);
    
    // Store capabilities
    i915->gt.info.engine_mask = detect_engines(i915);
    i915->gt.info.num_slices = slice_count;
    i915->gt.info.eu_per_subslice = eu_count;
    
    // Enable generation-specific features
    if (GRAPHICS_VER(i915) >= 12) {
        i915->gt.info.has_render_engine = true;
        i915->gt.info.has_compute = true;
    }
}

// Detect GPU generation via register read
static u32 intel_detect_device_gen(struct i915_drm_private *i915)
{
    u32 ver = intel_uncore_read(&i915->uncore, RING_TIMESTAMP);
    return GRAPHICS_VER_FROM_REGISTER(ver);
}
```

### Engine Detection

```plaintext
┌─────────────────────────────────────────────────┐
│ Detect Available GPU Engines                    │
│   (RCS, BCS, VCS, VECS, compute, etc.)         │
└────────────┬────────────────────────────────────┘
             │
    ┌────────▼─────────────────────────┐
    │ Read VDBOX_CTL (Media engines)   │
    │ Read VEBOX_CTL (Enhancement)     │
    │ Read COMPUTE_CTL (Compute units) │
    │ Read CCS_CTL (Compute shaders)   │
    └────────┬──────────────────────────┘
             │
    ┌────────▼──────────────────────────┐
    │ Build engine_mask                  │
    │ Map engine_id → physical instance |
    │ Initialize per-engine structures  │
    └────────────────────────────────────┘
```

---

## Feature Flag Management

### Platform Features

The driver uses feature flags to enable/disable functionality based on hardware:

```c
// Feature flag structure
struct intel_platform_info {
    // GPU generation
    u8 gen;
    
    // Engine counts
    u8 num_rcs;  // Render engines
    u8 num_bcs;  // Blitter engines
    u8 num_vcs;  // Video engines
    u8 num_vecs; // Enhancement engines
    
    // Memory and cache
    u32 l3_cache_size_kb;
    u32 llc_size_kb;
    bool has_shared_memory;
    
    // Execution features
    bool has_preemption;
    bool has_eu_slice_queue;
    bool supports_slm;  // Shared local memory
    
    // Power features
    bool has_rps;  // Dynamic frequency
    bool has_rc6;  // Power gating
    bool has_slpc; // Autonomous power control
    
    // Display features
    bool has_hdmi;
    bool has_dp;
    u8 num_pipes;
    
    // Debug and security
    bool has_pml4;      // 4-level page tables
    bool supports_full_ppgtt;
    bool has_eu_slice_queues;
};

// Runtime feature detection
if (HAS_RC6(i915)) {
    init_rc6_power_gating();
}

if (HAS_PREEMPTION(i915)) {
    setup_preemption_handlers();
}

if (GRAPHICS_VER(i915) >= 12) {
    // Alchemist+ features
    enable_compute_units();
}
```

### Generation-Specific Initialization

```plaintext
┌─────────────────────────────────────┐
│ Generation Detection                │
└────┬────────────────────────────────┘
     │
     ├─→ Gen 11 (Icelake)
     │   ├─ Init VDBOX 2
     │   ├─ Setup CCS (compute)
     │   └─ Configure GuC submission
     │
     ├─→ Gen 12 (Alchemist)
     │   ├─ Init multiple render units
     │   ├─ Setup compute DSS (dual-subslice)
     │   ├─ Configure Xe-HPM features
     │   └─ Enable media codec engines
     │
     └─→ Gen 13+ (Ponte Vecchio+)
         ├─ Init multi-tile configuration
         ├─ Setup tile interconnect
         ├─ Configure fault tolerance
         └─ Enable high bandwidth memory
```

---

## Subsystem Initialization

### Initialization Order

Subsystems must initialize in correct order (dependencies shown):

```plaintext
┌─────────────────────────────────────────────────────────┐
│ Phase 1: Hardware Setup (prerequisite for all)          │
│   ├─ MMIO regions mapping                               │
│   ├─ Interrupt configuration                            │
│   └─ Clock gating and reset handling                    │
└────┬────────────────────────────────────────────────────┘
     │
     ├─────────────────────────────────────────────────────┐
     │                                                      │
     ├─→ Phase 2: Memory System (needed by execution)      │
     │   ├─ GGTT initialization                            │
     │   ├─ Memory buddy allocator setup                   │
     │   └─ Gen-specific page table structures             │
     │                                                      │
     ├─→ Phase 3: Execution Hardware (depends on memory)   │
     │   ├─ Ring buffer allocation                         │
     │   ├─ Context allocation                             │
     │   └─ GuC firmware loading (if supported)            │
     │                                                      │
     ├─→ Phase 4: Scheduling & Power (depends on exec)     │
     │   ├─ Request scheduler                              │
     │   ├─ RPS/RC6 power management                       │
     │   └─ Thermal management                             │
     │                                                      │
     └─→ Phase 5: Display (independent)                    │
         ├─ Framebuffer allocation                         │
         ├─ Display pipes configuration                    │
         └─ Monitor/connector detection                    │
```

### Core Subsystems

```c
// Initialization order in i915_driver_load()
static int i915_driver_load(struct i915_drm_private *i915)
{
    int ret;
    
    // 1. Initialize global device state
    i915_params_copy(&i915->params, &i915_modparams);
    
    // 2. Setup memory management
    ret = i915_gem_init_early(i915);
    if (ret)
        return ret;
    
    // 3. Initialize execution engine
    ret = i915_gt_init_early(i915, &i915->gt);
    if (ret)
        goto cleanup_gem;
    
    // 4. Load GuC firmware (if supported)
    if (intel_uc_wants_guc(&i915->gt.uc)) {
        ret = intel_uc_fw_upload(&i915->gt.uc);
        if (ret)
            goto cleanup_gt;
    }
    
    // 5. Setup power management
    intel_pmc_setup(i915);
    intel_rps_init(&i915->gt.rps);
    intel_rc6_init(&i915->gt.rc6);
    
    // 6. Initialize display
    ret = intel_display_driver_probe(i915);
    if (ret)
        goto cleanup_pm;
    
    return 0;
    
cleanup_pm:
    // cleanup power management
cleanup_gt:
    // cleanup execution
cleanup_gem:
    // cleanup memory
    return ret;
}
```

---

## Resource Allocation

### Memory Resources

```plaintext
┌──────────────────────────────────────┐
│ GPU Address Space (Virtual)          │
│   Total: 256 GB - 512 GB             │
└────┬─────────────────────────────────┘
     │
     ├─→ GGTT (Global GTT)
     │   ├─ 1-16 MB
     │   ├─ Shared by all processes
     │   └─ Reserved for firmware
     │
     ├─→ PPGTT Per-Process GTT)
     │   ├─ 256 GB per process
     │   ├─ 4-level page tables (gen 12+)
     │   └─ Private to each context
     │
     ├─→ WC (Write-Combining) Regions
     │   ├─ Ring buffers
     │   ├─ Command streams
     │   └─ Batch buffers
     │
     ├─→ Framebuffers
     │   ├─ Display memory
     │   ├─ Color buffers
     │   └─ Z/stencil buffers
     │
     └─→ General GPU Memory
         ├─ Textures
         ├─ Shaders
         └─ User data
```

### Ring Buffer Allocation

```c
// Initialize ring buffers for command submission
static int init_ring_buffers(struct i915_gt *gt)
{
    // Each engine gets a ring buffer
    for_each_engine(engine, gt, id) {
        struct intel_ring *ring;
        
        // Allocate ring buffer (typically 16 KB)
        ring = intel_ring_create(engine, 16 * PAGE_SIZE);
        
        // Initialize ring pointers
        ring->head = 0;
        ring->tail = 0;
        ring->space = ring->size;
        
        // Map to GGTT (global page tables)
        ring->vaddr = ioremap_wc(ring->phys_addr, ring->size);
        
        engine->ring = ring;
    }
    
    return 0;
}
```

---

## Device Readiness

### Initialization Verification

```plaintext
┌────────────────────────────────────────────────┐
│ Device Readiness Checks                        │
└───┬────────────────────────────────────────────┘
    │
    ├─ Hardware Registers Accessible?
    │  └─ Read RING_TIMESTAMP register ✓
    │
    ├─ Memory System Operational?
    │  ├─ GGTT configured ✓
    │  ├─ TLBs initialized ✓
    │  └─ Buddy allocator ready ✓
    │
    ├─ Execution Ready?
    │  ├─ Ring buffers allocated ✓
    │  ├─ GuC firmware loaded (if needed) ✓
    │  └─ Command submission working ✓
    │
    ├─ Power Management Active?
    │  ├─ RPS frequency control ✓
    │  ├─ RC6 gates configured ✓
    │  └─ Thermal monitoring ready ✓
    │
    └─ Display Functional?
       ├─ Pipes configured ✓
       ├─ Connectors detected ✓
       └─ Framebuffer allocated ✓
```

### Startup Sequence Validation

```c
// Verify device is ready for use
static int verify_device_readiness(struct i915_drm_private *i915)
{
    u32 timestamp;
    int ret;
    
    // Test 1: Hardware registers accessible
    timestamp = intel_uncore_read(&i915->uncore, RING_TIMESTAMP);
    if (timestamp == 0)
        return -EIO;
    
    // Test 2: Memory system functional
    ret = i915_ggtt_test_write(&i915->ggtt);
    if (ret)
        return ret;
    
    // Test 3: Execution ready
    ret = i915_gem_create_test_object(i915);
    if (ret)
        return ret;
    
    // Test 4: Power management initialized
    if (!i915_pmu_is_active(i915))
        return -EIO;
    
    return 0;  // Device ready
}
```

---

## Initialization Debugging

### Common Initialization Failures

| Issue | Symptom | Diagnosis |
|-------|---------|-----------|
| **PCI device not found** | No device in lspci | BIOS disabled GPU or bus not enumerated |
| **MMIO mapping failed** | "Cannot map MMIO" error | Memory region not accessible or already mapped |
| **Firmware load failed** | "GuC fw load failed" | Firmware file missing or corrupted |
| **Ring initialization failed** | "Ring init timeout" | Memory allocation or hardware issue |
| **Interrupt not firing** | Commands hang indefinitely | IRQ configuration incorrect |
| **TLB invalidation timeout** | "TLB invalidation timeout" | Hardware state corrupted |

### Debug Output

```bash
# Enable debug logging
dmesg | grep i915       # Kernel messages
cat /sys/kernel/debug/dri/0/i915_runtime_pm_status  # PM state
cat /sys/kernel/debug/dri/0/i915_gem_objects  # Memory usage
cat /sys/kernel/debug/dri/0/i915_freq_info   # Frequency info
```

### Key Debug Points

```c
// Add debug logging during initialization
#define i915_init_debug(dev, fmt, ...) \
    dev_info(&(dev)->pdev->dev, "[i915-init] " fmt, ##__VA_ARGS__)

static int i915_driver_load(struct i915_drm_private *i915)
{
    i915_init_debug(i915, "Starting device initialization");
    
    // ... initialization code ...
    
    i915_init_debug(i915, "Hardware generation: %d", GRAPHICS_VER(i915));
    i915_init_debug(i915, "Engines available: 0x%x", 
                    i915->gt.info.engine_mask);
    i915_init_debug(i915, "Memory: %llu MB total",
                    i915->ggtt.total >> 20);
    
    i915_init_debug(i915, "Initialization complete");
    return 0;
}
```

---

## Summary & Best Practices

### Key Takeaways

1. **Hardware detection is critical:** Must correctly identify GPU generation and features
2. **Initialization order matters:** Dependencies between subsystems must be respected
3. **Fail fast:** Early validation prevents cryptic errors later
4. **Feature flags enable flexibility:** Same code supports multiple hardware generations
5. **Resource allocation must be precise:** MMIO regions, memory sizes, frequency ranges

### Best Practices

**For Driver Development:**
- Always check feature flags before using generation-specific code
- Validate hardware state early in initialization
- Use DRM_ERROR for critical failures, DRM_INFO for milestones
- Test on multiple hardware generations if possible

**For Debugging:**
- Check kernel messages (dmesg) for initialization errors
- Verify PCI device is recognized: `lspci | grep VGA`
- Use debug registers: RING_TIMESTAMP, FORCEWAKE, etc.
- Test with simpler commands first (e.g., read register)

**For Performance:**
- Initialization happens once per boot, not in hot path
- Use async initialization for non-critical subsystems
- Lazy-load optional features to speed up probe
- Cache hardware capabilities in global structure

### Initialization Timeline

| Stage | Time (ms) | Operation |
|-------|-----------|-----------|
| **PCI probe** | 1-5 | Device detection and MMIO mapping |
| **Hardware detection** | 5-10 | Device ID lookup, feature detection |
| **GGTT init** | 10-20 | Global page tables setup |
| **Memory allocator** | 5-15 | Buddy allocator initialization |
| **GuC firmware load** | 50-200 | Firmware download and verification |
| **RPS/RC6 setup** | 20-50 | Power management initialization |
| **Display probe** | 100-300 | Monitor detection and setup |
| **Total boot-to-ready** | 200-600 | Depends on hardware and config |

---

## References

- [Intel i915 Driver Source](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/i915)
- [Device Initialization Sequence](https://01.org/linuxgraphics/documentation/hardware-specification)
- [Feature Detection Code](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/i915/i915_pci.c)
- Related: [03-Context-Management.md](03-Context-Management.md), [01-Memory-Management.md](01-Memory-Management.md), [02-GuC-Firmware.md](02-GuC-Firmware.md)

---

**Next Steps:**
- Study the i915_pci_probe() function for real implementation details
- Review feature flags in intel_platform_info structure
- Check hardware-specific initialization code for your GPU generation
- Test initialization with debug logging enabled

---

**Document Status:** ✅ Complete  
**Version:** 1.0  
**Last Updated:** February 8, 2026
