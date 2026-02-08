# Hardware Workarounds and Errata

**Document ID:** 19 | **Status:** Complete | **Last Updated:** February 2026

---

## Executive Summary

GPU hardware often has errata requiring software workarounds. This document covers common hardware bugs, their detection, and mitigation strategies across GPU generations.

### Key Topics
- **Errata identification:** Platform detection, stepping-based quirks
- **Workaround patterns:** Register tweaks, command stream fixes
- **Performance impact:** Cache vs correctness trade-offs
- **Regression prevention:** Testing and validation

### Common Errata
- **Gen9:** Display FIFO underruns, memory arbitration issues
- **Gen10+:** Compute mode restrictions, L3 cache issues
- **All:** Thermal throttling timing, power gating edge cases

---

## Table of Contents

1. [Hardware Errata Database](#hardware-errata-database)
2. [Errata Detection](#errata-detection)
3. [Common Workarounds](#common-workarounds)
4. [Performance Impact Analysis](#performance-impact-analysis)
5. [Testing Workarounds](#testing-workarounds)
6. [Documentation and Tracking](#documentation-and-tracking)
7. [Summary & Best Practices](#summary--best-practices)

---

## Hardware Errata Database

### Errata Registry Structure

```c
// Hardware workaround registry
struct intel_wl_info {
    u8 gen;              // GPU generation
    u8 stepping_min;     // Minimum affected stepping
    u8 stepping_max;     // Maximum affected stepping
    
    const char *name;
    const char *description;
    
    // Fix applied by function
    void (*apply)(struct i915_drm_private *i915);
    void (*unapply)(struct i915_drm_private *i915);
    
    bool enabled;        // Is workaround active
    unsigned long flags;
};

// Example errata entries
static const struct intel_wl_info skl_errata[] = {
    {
        .gen = 9,
        .stepping_min = 0,
        .stepping_max = 4,
        .name = "WaDisplayUnderflowFIFO",
        .description = "Display FIFO can underflow under "
                      "specific memory arbitration conditions",
        .apply = wa_display_fifo_underflow_apply,
    },
    {
        .gen = 9,
        .stepping_min = 0,
        .stepping_max = 0xFF,
        .name = "WaDisableChickenBitTSGBarrierAckForFFTC",
        .description = "TSG barrier ack can cause GPU hang "
                      "in specific scenarios",
        .apply = wa_tsg_barrier_ack_apply,
    },
};

// Gen12 errata
static const struct intel_wl_info gen12_errata[] = {
    {
        .gen = 12,
        .stepping_min = 0,
        .stepping_max = 2,
        .name = "WaL3BankAddressHashing",
        .description = "L3 cache bank addressing issue "
                      "affects coherency",
        .apply = wa_l3_bank_hashing_apply,
    },
};
```

### Errata Lookup

```c
// Find applicable errata for current platform
static void apply_platform_errata(struct i915_drm_private *i915)
{
    const struct intel_wl_info *errata;
    int num_errata;
    int i;
    
    // Get errata list for this generation
    if (IS_SKYLAKE(i915)) {
        errata = skl_errata;
        num_errata = ARRAY_SIZE(skl_errata);
    } else if (IS_KABYLAKE(i915)) {
        errata = kbl_errata;
        num_errata = ARRAY_SIZE(kbl_errata);
    } else if (IS_GEN12(i915)) {
        errata = gen12_errata;
        num_errata = ARRAY_SIZE(gen12_errata);
    } else {
        return;  // No errata for this generation
    }
    
    // Apply applicable workarounds
    for (i = 0; i < num_errata; i++) {
        const struct intel_wl_info *wa = &errata[i];
        
        // Check if this stepping is affected
        if (i915->stepping.stepping < wa->stepping_min ||
            i915->stepping.stepping > wa->stepping_max) {
            continue;  // Not affected
        }
        
        // Apply workaround
        dev_info(i915->drm.dev, "Applying %s\n", wa->name);
        if (wa->apply) {
            wa->apply(i915);
        }
        
        wa->enabled = true;
    }
}
```

---

## Errata Detection

### Platform and Stepping Detection

```c
// Detect GPU platform and stepping
static void detect_platform_info(struct i915_drm_private *i915)
{
    struct pci_dev *pdev = i915->drm.pdev;
    u16 device_id = pdev->device;
    u8 revision;
    
    // Read device revision ID
    pci_read_config_byte(pdev, PCI_REVISION_ID, &revision);
    
    // Decode platform from device ID
    i915->platform.id = decode_device_id(device_id);
    
    // Decode stepping from revision
    i915->stepping.stepping = revision & 0x0F;
    
    dev_info(i915->drm.dev,
            "GPU: %s (0x%04x) Stepping: %c\n",
            get_platform_name(i915->platform.id),
            device_id,
            stepping_to_letter(i915->stepping.stepping));
}

// Check for specific errata
bool has_errata(struct i915_drm_private *i915,
                const char *name)
{
    const struct intel_wl_info *errata;
    int num_errata;
    int i;
    
    // Get errata list
    errata = get_errata_for_gen(i915);
    num_errata = get_errata_count(i915);
    
    // Search for errata by name
    for (i = 0; i < num_errata; i++) {
        if (strcmp(errata[i].name, name) == 0) {
            // Check if this stepping is affected
            if (i915->stepping.stepping <= errata[i].stepping_max)
                return true;
        }
    }
    
    return false;
}
```

---

## Common Workarounds

### Display FIFO Underflow

```c
// Workaround: Display FIFO can underflow in specific conditions
// Solution: Increase FIFO threshold and reduce burst size

static void wa_display_fifo_underflow_apply(
    struct i915_drm_private *i915)
{
    struct intel_uncore *uncore = &i915->uncore;
    u32 fifo_config;
    
    // Read current FIFO configuration
    fifo_config = intel_uncore_read(uncore,
                                   DSPARB_PER_PLANE_CTL);
    
    // 1. Increase minimum FIFO threshold
    fifo_config &= ~DSPARB_MIN_THRESHOLD_MASK;
    fifo_config |= DSPARB_MIN_THRESHOLD_HIGH;
    
    // 2. Reduce maximum burst size
    fifo_config &= ~DSPARB_MAX_BURST_MASK;
    fifo_config |= DSPARB_MAX_BURST_64;
    
    // Write back configuration
    intel_uncore_write(uncore, DSPARB_PER_PLANE_CTL,
                      fifo_config);
    
    dev_info(i915->drm.dev, "Applied WaDisplayUnderflowFIFO\n");
}

static void wa_display_fifo_underflow_unapply(
    struct i915_drm_private *i915)
{
    struct intel_uncore *uncore = &i915->uncore;
    
    // Restore default FIFO configuration
    intel_uncore_write(uncore, DSPARB_PER_PLANE_CTL,
                      DSPARB_DEFAULT_CONFIG);
}
```

### Memory Arbitration

```c
// Workaround: Memory arbitration can cause GPU hang
// Solution: Modify arbitration weights

static void wa_memory_arbitration_apply(
    struct i915_drm_private *i915)
{
    struct intel_uncore *uncore = &i915->uncore;
    u32 arb_config;
    
    // Adjust memory arbitration weights
    arb_config = intel_uncore_read(uncore, ARB_CONFIG);
    
    // Increase priority for GPU render
    arb_config |= ARB_RENDER_PRIORITY_HIGH;
    
    // Decrease priority for display
    arb_config &= ~ARB_DISPLAY_PRIORITY_MASK;
    arb_config |= ARB_DISPLAY_PRIORITY_LOW;
    
    intel_uncore_write(uncore, ARB_CONFIG, arb_config);
    
    dev_info(i915->drm.dev, "Applied WaMemoryArbitration\n");
}
```

### Cache Coherency

```c
// Workaround: L3 cache incoherency under load
// Solution: Disable L3 cache optimization

static void wa_l3_cache_incoherence_apply(
    struct i915_drm_private *i915)
{
    struct intel_engine_cs *engine;
    int i;
    
    // For each engine, disable L3 optimization
    for_each_engine(engine, &i915->gt, i) {
        struct i915_request *rq;
        u32 *cmd;
        
        rq = i915_request_create(engine);
        if (IS_ERR(rq))
            continue;
        
        // Build command to disable L3 cache hints
        cmd = intel_ring_begin(rq, 4);
        if (IS_ERR(cmd)) {
            i915_request_put(rq);
            continue;
        }
        
        *cmd++ = MI_LOAD_REGISTER_IMM(1);
        *cmd++ = L3_CACHE_HINTS;
        *cmd++ = L3_CACHE_HINTS_DISABLE;
        *cmd++ = MI_NOOP;
        
        intel_ring_advance(rq, cmd);
        i915_request_submit(rq);
        i915_request_put(rq);
    }
    
    dev_info(i915->drm.dev, "Applied WaL3CacheIncoherence\n");
}
```

### Thermal Throttling

```c
// Workaround: Thermal throttling timing inaccurate
// Solution: Add delay before checking

static void wa_thermal_throttle_timing_apply(
    struct i915_drm_private *i915)
{
    i915->thermal.throttle_check_delay_ms = 100;  // 100ms delay
    
    dev_info(i915->drm.dev,
            "Applied WaThermalThrottleTiming "
            "(delay: %dms)\n",
            i915->thermal.throttle_check_delay_ms);
}
```

---

## Performance Impact Analysis

### Workaround Performance Trade-offs

```plaintext
Workaround Performance Impact Analysis
┌──────────────────────────────────────────┐
│ Workaround         │ Performance Impact   │
├────────────────────┼─────────────────────┤
│ FIFO Underflow     │ -2-5% GPU throughput │
│ Memory Arbitration │ -1-3% latency        │
│ L3 Cache Disable   │ -5-10% memory BW     │
│ Thermal Throttle   │ -0-2% (safety)       │
│ TSG Barrier        │ < 1% overhead        │
└──────────────────────────────────────────┘

Trade-off: Stability vs Performance
- Most workarounds are necessary for correctness
- Performance cost acceptable vs data corruption
- Newer steppings fix underlying issues
```

### Measuring Workaround Impact

```c
// Measure performance with/without workaround
struct perf_benchmark {
    u64 throughput_gbps;
    u64 latency_ns;
    unsigned long errors;
};

static struct perf_benchmark benchmark_with_wa;
static struct perf_benchmark benchmark_without_wa;

// Run benchmark and record metrics
static void measure_workaround_impact(
    struct i915_drm_private *i915,
    const char *wa_name)
{
    struct perf_benchmark result;
    int i;
    
    // Run 10 iterations
    for (i = 0; i < 10; i++) {
        run_synthetic_benchmark(&result);
        
        dev_dbg(i915->drm.dev,
               "%s performance: %lld GB/s, "
               "latency: %lld ns\n",
               wa_name,
               result.throughput_gbps,
               result.latency_ns);
    }
}
```

---

## Testing Workarounds

### Workaround Validation

```c
// Validate workaround is actually applied
static int validate_workaround(struct i915_drm_private *i915,
                               const char *wa_name)
{
    if (!strcmp(wa_name, "WaDisplayUnderflowFIFO")) {
        u32 fifo_config = intel_uncore_read(&i915->uncore,
                                            DSPARB_PER_PLANE_CTL);
        
        // Check if workaround bits are set
        if ((fifo_config & DSPARB_MIN_THRESHOLD_MASK) !=
            DSPARB_MIN_THRESHOLD_HIGH) {
            dev_warn(i915->drm.dev,
                    "%s not properly applied\n",
                    wa_name);
            return -EINVAL;
        }
    }
    
    return 0;
}
```

### Regression Testing

```bash
# Test with workarounds disabled (for affected stepping)
echo "disable_workarounds=1" | tee /sys/kernel/debug/i915_modparam

# Run workload and monitor for errors
stress-ng --gpu 4 --gpu-ops 1000000 &
sleep 10
# Monitor for hangs, glitches, memory corruption

# Re-enable and test again
echo "disable_workarounds=0" | tee /sys/kernel/debug/i915_modparam
```

---

## Documentation and Tracking

### Errata Documentation Format

```markdown
# WaDisplayUnderflowFIFO

**Affected Platforms:** SKL A0-D0
**Issue:** Display FIFO can underflow causing visual artifacts
**Root Cause:** Memory arbitration starves display pixel fetch
**Workaround:** Increase FIFO threshold and limit burst size
**Mitigation:** Pending in stepping E0+
**Performance Impact:** -2% GPU throughput
**Tested By:** [names]
**Validation:** Stress tested with high memory load
```

### Tracking System

```c
// Track which errata are active
struct intel_workaround_status {
    struct {
        const char *name;
        bool enabled;
        ktime_t applied_time;
        unsigned long affected_count;
    } active_workarounds[MAX_WORKAROUNDS];
    
    int num_active;
};

// Report active workarounds
void report_active_workarounds(struct i915_drm_private *i915)
{
    struct intel_workaround_status *status = &i915->wa_status;
    int i;
    
    dev_info(i915->drm.dev,
            "Active workarounds: %d\n",
            status->num_active);
    
    for (i = 0; i < status->num_active; i++) {
        dev_info(i915->drm.dev,
                "  [%d] %s (applied: %lldms ago)\n",
                i,
                status->active_workarounds[i].name,
                ktime_ms_delta(ktime_get(),
                              status->active_workarounds[i].applied_time));
    }
}
```

---

## Summary & Best Practices

### Key Takeaways

1. **Generation-specific:** Different errata per GPU generation
2. **Stepping-dependent:** Newer steppings may fix issues
3. **Performance cost:** Acceptable trade-off for correctness
4. **Documentation:** Track all applied workarounds
5. **Validation:** Test with and without workarounds

### Best Practices

**For Workaround Implementation:**
- Always document errata and solution
- Make workarounds conditional on stepping
- Provide disable mechanism for testing
- Measure performance impact
- Test both with and without

**For Deployment:**
- Keep workaround list up-to-date
- Test on actual target hardware
- Monitor for errata-related issues
- Plan for stepping updates
- Archive historical errata info

**For Debugging:**
- Enable workaround logging
- Use module parameters to disable
- Monitor for missed errata
- Track performance regressions
- Document stepping-specific issues

---

## References

- [Workaround Tracking](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/i915/gt)
- [Intel GPU Errata Database](https://www.intel.com/content/dam/www/public/us/en/documents/errata/)
- Related: [10-Hardware-Discovery-Initialization.md](10-Hardware-Discovery-Initialization.md), [04-Power-Management.md](04-Power-Management.md)

---

**Next Steps:**
- Review active workarounds for your platform
- Test performance impact of critical workarounds
- Develop custom workarounds for new errata
- Document any platform-specific issues

---

**Document Status:** ✅ Complete  
**Version:** 1.0  
**Last Updated:** February 8, 2026
