# Runtime Power Management

**Document ID:** 16 | **Status:** Complete | **Last Updated:** February 2026

---

## Executive Summary

Runtime power management optimizes GPU energy consumption during idle and active periods. This document covers RC6 sleep states, dynamic frequency scaling (RPS), and power-aware scheduling.

### Key Topics
- **RC6/RC7 states:** GPU sleep modes with different power savings
- **RPS (Render P-State):** Dynamic frequency and voltage scaling
- **Autosuspend:** Automated power gating during inactivity
- **Thermal management:** Temperature-aware throttling

### Performance Impact
- **Idle power:** 5-15W → 0.5-2W with RC6 enabled
- **Frequency switch:** 1-10ms transition time
- **Latency cost:** 1-5ms wake-up time from RC6

---

## Table of Contents

1. [Power Management Architecture](#power-management-architecture)
2. [RC6 Sleep States](#rc6-sleep-states)
3. [RPS and Frequency Scaling](#rps-and-frequency-scaling)
4. [Autosuspend and D3Cold](#autosuspend-and-d3cold)
5. [Thermal Management](#thermal-management)
6. [ASPM and PCIe Power](#aspm-and-pcie-power)
7. [Power Management Debugging](#power-management-debugging)
8. [Summary & Best Practices](#summary--best-practices)

---

## Power Management Architecture

### Power State Hierarchy

```plaintext
GPU Power States (Lowest to Highest Power)
┌─────────────────────────────────────┐
│ D3Cold (PCI Device Off)             │
│ ├─ Power consumption: < 0.1W        │
│ ├─ Wake latency: ~100ms             │
│ └─ Requires PCIe hotplug support    │
├─────────────────────────────────────┤
│ RC7 (Render Sleep)                  │
│ ├─ Power consumption: < 1W          │
│ ├─ Wake latency: 20-50ms            │
│ └─ Some context memory powered      │
├─────────────────────────────────────┤
│ RC6 (Render Sleep)                  │
│ ├─ Power consumption: 1-5W          │
│ ├─ Wake latency: 5-20ms             │
│ └─ Shader engines off, memory on    │
├─────────────────────────────────────┤
│ Idle (Low Frequency)                │
│ ├─ Power consumption: 2-8W          │
│ ├─ Wake latency: < 1ms              │
│ └─ Minimum frequency (~300MHz)      │
├─────────────────────────────────────┤
│ Active (Dynamic Frequency)          │
│ ├─ Power consumption: 10-40W        │
│ ├─ Frequency: 300-1500MHz+          │
│ └─ RPS adjusts based on load        │
└─────────────────────────────────────┘
```

### Power Domain Management

```c
// Power domains - independent power gating units
struct i915_power_domains {
    struct mutex lock;
    struct i915_power_well *power_wells;
    int num_power_wells;
    
    // Refcounts per domain
    struct refcount_t refs[I915_NUM_POWER_DOMAINS];
};

// Power domain types
enum i915_power_domain_id {
    POWER_DOMAIN_RENDER,        // GPU execution engines
    POWER_DOMAIN_PIPE_A,        // Display pipe A
    POWER_DOMAIN_PIPE_B,        // Display pipe B
    POWER_DOMAIN_PIPE_C,        // Display pipe C
    POWER_DOMAIN_MEMORY,        // GPU memory controllers
    POWER_DOMAIN_VDSC,          // Video codec domain
    POWER_DOMAIN_RM,            // Render/Memory combined
};

// Enable specific power domain
void intel_display_power_get(struct i915_runtime_pm *rpm,
                             enum i915_power_domain domain)
{
    mutex_lock(&rpm->power_domains.lock);
    
    refcount_inc(&rpm->power_domains.refs[domain]);
    
    if (refcount_read(&rpm->power_domains.refs[domain]) == 1) {
        // First user of this domain, enable it
        intel_power_well_enable(rpm, domain);
    }
    
    mutex_unlock(&rpm->power_domains.lock);
}

// Disable power domain
void intel_display_power_put(struct i915_runtime_pm *rpm,
                            enum i915_power_domain domain)
{
    mutex_lock(&rpm->power_domains.lock);
    
    refcount_dec(&rpm->power_domains.refs[domain]);
    
    if (refcount_read(&rpm->power_domains.refs[domain]) == 0) {
        // Last user, power down this domain
        intel_power_well_disable(rpm, domain);
    }
    
    mutex_unlock(&rpm->power_domains.lock);
}
```

---

## RC6 Sleep States

### RC6 Entry and Exit

```plaintext
RC6 Sleep Cycle
┌──────────────────────────────┐
│ GPU Active                   │
│ ├─ Executing commands        │
│ └─ All engines running       │
└──────┬───────────────────────┘
       │
       ├─ No work for N ms
       │  └─ GPU idle detected
       │
       ├─→ RC6 Entry
       │   ├─ Flush pipelines
       │   ├─ Save state registers
       │   ├─ Gate clock domains
       │   └─ Enter low-power state
       │
       ├─ Powered down (Sleep)
       │  └─ Power consumption: 1-5W
       │
       ├─ GPU interrupt or work queued
       │  └─ Wake signal detected
       │
       ├─→ RC6 Exit
       │   ├─ Restore clock gating
       │   ├─ Restore state registers
       │   └─ Resume execution
       │
       └─→ GPU Active
           └─ Ready to execute
```

### RC6 Configuration

```c
// RC6 sleep state settings
struct intel_rps {
    // RC6 state configuration
    bool rc6_enabled;
    bool rc6p_enabled;     // Deeper RC6 variant
    bool rc6pp_enabled;    // Even deeper RC6 variant
    
    // Thresholds
    unsigned long rc6_residency_us;  // Expected residency
    unsigned long rc6_entry_delay_us; // Time to enter RC6
};

// Enable RC6 sleep states
static void intel_rc6_init(struct i915_drm_private *i915)
{
    struct intel_uncore *uncore = &i915->uncore;
    u32 rc6_ctrl;
    
    // 1. Check if RC6 is supported
    if (!HAS_RC6(i915)) {
        dev_info(i915->drm.dev, "RC6 not supported\n");
        return;
    }
    
    // 2. Configure RC6 entry criteria
    // Set how long GPU must be idle before entering RC6
    intel_uncore_write(uncore, GEN6_RC_CONTROL,
                       GEN6_RC_CTL_EW_ENABLE |
                       GEN6_RC_CTL_RC6_ENABLE);
    
    // 3. Set RC6 entry timer (microseconds to milliseconds)
    intel_uncore_write(uncore, GEN6_RC_STATE,
                       RC6_ENTRY_TIMER_US);
    
    // 4. Program RC6 departure latency
    intel_uncore_write(uncore, GEN6_RC6p_THRESHOLD,
                       RC6P_LATENCY_US);
    
    // 5. Enable RC6 in hardware
    rc6_ctrl = intel_uncore_read(uncore, GEN6_RC_CONTROL);
    rc6_ctrl |= GEN6_RC_CTL_RC6_ENABLE;
    intel_uncore_write(uncore, GEN6_RC_CONTROL, rc6_ctrl);
    
    dev_info(i915->drm.dev, "RC6 enabled (residency: %ldus)\n",
             i915->rps.rc6_residency_us);
}

// Disable RC6 (for debugging)
static void intel_rc6_disable(struct i915_drm_private *i915)
{
    struct intel_uncore *uncore = &i915->uncore;
    u32 rc6_ctrl;
    
    rc6_ctrl = intel_uncore_read(uncore, GEN6_RC_CONTROL);
    rc6_ctrl &= ~GEN6_RC_CTL_RC6_ENABLE;
    intel_uncore_write(uncore, GEN6_RC_CONTROL, rc6_ctrl);
    
    dev_info(i915->drm.dev, "RC6 disabled\n");
}
```

### RC6 Residency Tracking

```c
// Track how long GPU spends in RC6
struct intel_rc6_residency {
    u64 entry_time;
    u64 total_residency;
    unsigned long entry_count;
};

// Read RC6 residency counter
static u64 read_rc6_residency(struct i915_drm_private *i915)
{
    struct intel_uncore *uncore = &i915->uncore;
    
    // RC6 counter is typically in 10ms units
    u32 rc6_count = intel_uncore_read(uncore, GEN6_RC6_RESIDENCY);
    
    // Convert to microseconds
    return rc6_count * 10000;
}

// Monitor RC6 effectiveness
static void check_rc6_residency(struct i915_drm_private *i915)
{
    static u64 last_residency = 0;
    u64 current_residency = read_rc6_residency(i915);
    
    u64 delta = current_residency - last_residency;
    
    // If very low RC6 residency, warn user
    if (delta < 1000000) {  // Less than 1 second in 1 minute
        dev_warn(i915->drm.dev,
                "Low RC6 residency: %llums/min\n",
                delta / 1000);
    }
    
    last_residency = current_residency;
}
```

---

## RPS and Frequency Scaling

### Dynamic Frequency Adjustment

```plaintext
RPS Frequency Scaling (Render P-State)
┌────────────────────────────────────┐
│ GPU Frequency Selection            │
└────┬───────────────────────────────┘
     │
     ├─ Load < 25%?
     │  └─ Decrease frequency (save power)
     │     ├─ From 1200MHz to 1000MHz
     │     └─ Power saved: ~50W
     │
     ├─ Load 25-75%?
     │  └─ Maintain frequency (balanced)
     │
     ├─ Load > 75%?
     │  └─ Increase frequency (maximize performance)
     │     ├─ From 1000MHz to 1500MHz
     │     └─ Power increase: +100W
     │
     └─ Thermal limit exceeded?
         └─ Throttle frequency (safety)
             └─ Reduce to safe level
```

### RPS Implementation

```c
// RPS (Render P-State) configuration
struct intel_rps {
    // Frequency range
    u32 min_freq_softlimit;    // Minimum allowed frequency
    u32 max_freq_softlimit;    // Maximum allowed frequency
    u32 min_freq;              // Hardware minimum
    u32 max_freq;              // Hardware maximum
    u32 current_freq;          // Current frequency
    
    // RPS thresholds
    u32 up_threshold;          // Load to increase freq
    u32 down_threshold;        // Load to decrease freq
    
    // Delayed work for frequency adjustment
    struct delayed_work work;
};

// RPS loop - periodically adjust frequency
static void intel_rps_work(struct work_struct *work)
{
    struct intel_rps *rps = container_of(work,
                                         struct intel_rps,
                                         work.work);
    struct i915_drm_private *i915 = rps_to_i915(rps);
    u32 gpu_load;
    u32 target_freq;
    int ret;
    
    // 1. Measure current GPU load
    gpu_load = intel_gpu_measure_load(i915);
    
    // 2. Determine target frequency
    if (gpu_load > rps->up_threshold) {
        // GPU heavily loaded - increase frequency
        target_freq = min(rps->current_freq + 50,
                         rps->max_freq_softlimit);
    } else if (gpu_load < rps->down_threshold) {
        // GPU lightly loaded - decrease frequency
        target_freq = max(rps->current_freq - 50,
                         rps->min_freq_softlimit);
    } else {
        // Load stable - no change
        target_freq = rps->current_freq;
    }
    
    // 3. Apply frequency change if needed
    if (target_freq != rps->current_freq) {
        ret = intel_set_rps_freq(i915, target_freq);
        if (ret == 0) {
            rps->current_freq = target_freq;
            dev_dbg(i915->drm.dev,
                   "RPS frequency: %uMHz (load: %u%%)\n",
                   target_freq, gpu_load);
        }
    }
    
    // 4. Reschedule for next iteration
    queue_delayed_work(system_unbound_wq, &rps->work,
                      msecs_to_jiffies(RPS_INTERVAL_MS));
}

// Set GPU frequency
static int intel_set_rps_freq(struct i915_drm_private *i915,
                              u32 freq_mhz)
{
    struct intel_uncore *uncore = &i915->uncore;
    u32 freq_code;
    
    // 1. Validate frequency is in range
    if (freq_mhz < i915->rps.min_freq ||
        freq_mhz > i915->rps.max_freq) {
        return -EINVAL;
    }
    
    // 2. Convert frequency to hardware code
    freq_code = intel_rps_freq_to_code(freq_mhz);
    
    // 3. Request frequency change
    intel_uncore_write(uncore, GEN6_RPNSWREQ,
                      GEN6_TURBO_DISABLE |
                      (freq_code << GEN6_TURBO_SHIFT));
    
    // 4. Wait for hardware to acknowledge
    return intel_wait_for_register(uncore,
                                  GEN6_RPSTAT1,
                                  GEN6_RPSTAT_BUSY,
                                  0,
                                  10);
}
```

### User Control of RPS

```bash
# View RPS parameters
cat /sys/class/drm/card0/gt/gt_cur_freq_mhz     # Current frequency
cat /sys/class/drm/card0/gt/gt_max_freq_mhz     # Maximum frequency
cat /sys/class/drm/card0/gt/gt_min_freq_mhz     # Minimum frequency
cat /sys/class/drm/card0/gt/gt_rps_boost_freq_mhz # Boost frequency

# Set frequency limits (requires root)
echo 1000 > /sys/class/drm/card0/gt/gt_max_freq_mhz    # Cap at 1000MHz
echo 300 > /sys/class/drm/card0/gt/gt_min_freq_mhz     # Min 300MHz

# Disable RPS (fixed frequency)
echo performance > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

---

## Autosuspend and D3Cold

### Autosuspend Logic

```plaintext
Autosuspend State Machine
┌──────────────────────────┐
│ GPU Active               │
│ (executing commands)     │
└──────┬───────────────────┘
       │
       ├─ No work submitted
       │  └─ Idle counter starts
       │
       ├─ Idle for N seconds?
       │  └─ (typically 10-60 sec)
       │
       ├─→ Runtime Suspend
       │   ├─ Stop GPU
       │   ├─ Gate power domains
       │   ├─ Save minimal state
       │   └─ Enter autosuspend
       │
       ├─ GPU suspended
       │  └─ Power consumption: ~0.5-2W
       │
       ├─ New work submitted
       │  └─ Wake request queued
       │
       ├─→ Runtime Resume
       │   ├─ Restore power domains
       │   ├─ Reinitialize GPU
       │   └─ Restart execution
       │
       └─→ GPU Active
           └─ Ready for work
```

### Autosuspend Implementation

```c
// Runtime PM (power management) structure
struct i915_runtime_pm {
    struct device *dev;
    
    // Reference counting
    atomic_t wakeref_count;
    
    // Autosuspend timer
    struct delayed_work autosuspend_work;
    unsigned int autosuspend_delay;  // milliseconds
};

// Get wakeref - prevent autosuspend
static intel_wakeref_t intel_runtime_pm_get(
    struct i915_runtime_pm *rpm)
{
    int ret;
    
    // Increment wakeref count
    if (atomic_add_return(1, &rpm->wakeref_count) == 1) {
        // First wakeref, ensure GPU is on
        ret = pm_runtime_get_sync(rpm->dev);
        if (ret < 0) {
            atomic_dec(&rpm->wakeref_count);
            return -ret;
        }
    }
    
    return 1;  // Wakeref obtained
}

// Put wakeref - allow autosuspend
static void intel_runtime_pm_put(
    struct i915_runtime_pm *rpm,
    intel_wakeref_t wakeref)
{
    // Decrement wakeref
    if (atomic_sub_return(1, &rpm->wakeref_count) == 0) {
        // Last wakeref released, schedule autosuspend
        queue_delayed_work(system_unbound_wq,
                          &rpm->autosuspend_work,
                          msecs_to_jiffies(rpm->autosuspend_delay));
    }
}

// Autosuspend timer fired
static void intel_autosuspend_work(struct work_struct *work)
{
    struct i915_runtime_pm *rpm = container_of(work,
                                              struct i915_runtime_pm,
                                              autosuspend_work.work);
    
    // Check if any wakeref taken
    if (atomic_read(&rpm->wakeref_count) > 0) {
        return;  // Someone grabbed wakeref, don't suspend
    }
    
    // Safe to suspend
    pm_runtime_put_autosuspend(rpm->dev);
}

// Usage pattern
void gpu_operation_with_pm(struct i915_drm_private *i915)
{
    intel_wakeref_t wakeref;
    
    // Ensure GPU is on for this operation
    wakeref = intel_runtime_pm_get(&i915->runtime_pm);
    if (wakeref < 0) {
        dev_err(i915->drm.dev, "Failed to get wakeref\n");
        return;
    }
    
    // Do GPU operations here
    // GPU is guaranteed to be powered on
    
    // Release GPU, allowing autosuspend
    intel_runtime_pm_put(&i915->runtime_pm, wakeref);
}
```

### D3Cold Support

```c
// D3Cold - deepest PCIe power state
static int intel_d3cold_enter(struct i915_drm_private *i915)
{
    struct pci_dev *pdev = i915->drm.pdev;
    int ret;
    
    // 1. Check D3Cold capability
    if (!pci_can_enter_d3cold(pdev)) {
        dev_dbg(i915->drm.dev, "D3Cold not supported\n");
        return -EOPNOTSUPP;
    }
    
    // 2. Perform GPU shutdown
    ret = i915_gem_suspend(i915);
    if (ret) {
        return ret;
    }
    
    // 3. Set PCIe device to D3Cold
    pci_set_power_state(pdev, PCI_D3cold);
    
    // 4. Notify platform ACPI of power transition
    acpi_bus_set_power(pci_device_acpi_handle(pdev),
                      ACPI_STATE_D3_COLD);
    
    dev_info(i915->drm.dev, "Entered D3Cold state\n");
    
    return 0;
}

// Resume from D3Cold
static int intel_d3cold_exit(struct i915_drm_private *i915)
{
    struct pci_dev *pdev = i915->drm.pdev;
    int ret;
    
    // 1. Restore PCIe device power
    pci_set_power_state(pdev, PCI_D0);
    
    // 2. Restore ACPI power state
    acpi_bus_set_power(pci_device_acpi_handle(pdev),
                      ACPI_STATE_D0);
    
    // 3. Reinitialize GPU
    ret = i915_gem_resume(i915);
    if (ret) {
        dev_err(i915->drm.dev, "Failed to resume from D3Cold\n");
        return ret;
    }
    
    dev_info(i915->drm.dev, "Exited D3Cold state\n");
    
    return 0;
}
```

---

## Thermal Management

### Temperature Monitoring

```c
// Read GPU temperature
static int i915_read_temperature(struct i915_drm_private *i915,
                                 u32 *temp_c)
{
    struct intel_uncore *uncore = &i915->uncore;
    u32 temp_status;
    u32 raw_temp;
    
    // Read temperature register
    temp_status = intel_uncore_read(uncore, GEN12_TEMP_STATUS);
    
    // Extract temperature
    raw_temp = (temp_status >> 16) & 0x3FF;
    
    // Convert to Celsius (format varies by platform)
    *temp_c = raw_temp - 50;  // Typical offset
    
    return 0;
}

// Thermal throttling response
static void intel_thermal_handler(struct i915_drm_private *i915)
{
    u32 temp;
    int ret;
    
    ret = i915_read_temperature(i915, &temp);
    if (ret)
        return;
    
    // Check thermal thresholds
    if (temp >= THERMAL_CRITICAL) {
        // Critical: immediate shutdown
        dev_crit(i915->drm.dev,
                "Critical thermal: %u°C, throttling to 300MHz\n",
                temp);
        intel_set_rps_freq(i915, 300);
        
    } else if (temp >= THERMAL_WARNING) {
        // Warning: reduce to half frequency
        dev_warn(i915->drm.dev,
                "Thermal warning: %u°C, reducing frequency\n",
                temp);
        u32 reduced = i915->rps.max_freq / 2;
        intel_set_rps_freq(i915, reduced);
        
    } else if (temp < THERMAL_RECOVERY) {
        // Cool enough to resume normal operation
        if (i915->rps.current_freq < i915->rps.max_freq) {
            dev_info(i915->drm.dev,
                    "Temperature recovered: %u°C\n",
                    temp);
            // Resume normal frequency scaling
            // (RPS loop will increase frequency)
        }
    }
}
```

---

## ASPM and PCIe Power

### ASPM Configuration

```c
// Active State Power Management (ASPM) - PCIe link states
static int intel_aspm_enable(struct i915_drm_private *i915)
{
    struct pci_dev *pdev = i915->drm.pdev;
    u32 aspm_control;
    
    // Check ASPM support
    if (!pci_aspm_enabled(pdev)) {
        dev_info(i915->drm.dev, "ASPM not enabled\n");
        return 0;
    }
    
    // Enable L0s and L1 states
    aspm_control = ASPM_L0S | ASPM_L1;
    pci_disable_link_state(pdev, ~aspm_control);
    
    dev_info(i915->drm.dev, "ASPM enabled\n");
    
    return 0;
}

// Check ASPM latency impact
static void check_aspm_latency(struct i915_drm_private *i915)
{
    // L0s: 1-2µs exit latency
    // L1: 10-100µs exit latency
    
    // Applications sensitive to latency (<1ms) may want to disable
    
    dev_dbg(i915->drm.dev, "ASPM may add 1-100µs latency\n");
}
```

---

## Power Management Debugging

### Power Monitoring

```bash
# View current frequency
cat /sys/class/drm/card0/gt/cur_freq_mhz

# View RC6 status
cat /sys/kernel/debug/dri/0/i915_rc6_info

# Monitor power draw (requires external measurement)
echo 1 > /sys/kernel/debug/dri/0/i915_energy_monitoring

# View temperature
cat /sys/class/drm/card0/gt/temp

# Check autosuspend status
cat /sys/devices/pci0000:00/0000:00:02.0/power/runtime_status
```

### Debug Output

```c
// Log power state transitions
#define PM_DEBUG(i915, fmt, ...)                        \
    dev_dbg(i915->drm.dev,                             \
            "[PM] " fmt, ##__VA_ARGS__)

// Usage
PM_DEBUG(i915, "RPS frequency changed: %u -> %uMHz\n",
         old_freq, new_freq);
PM_DEBUG(i915, "RC6 entered: residency=%llums\n",
         residency_ms);
```

---

## Summary & Best Practices

### Key Takeaways

1. **Multi-level power:** RC6 + RPS + autosuspend
2. **Latency trade-off:** Deeper sleep = longer wake time
3. **Thermal limits:** Temperature controls frequency
4. **Wakeref pattern:** Always pair get/put calls
5. **Monitoring:** Track residency and temperatures

### Best Practices

**For Deployments:**
- Enable RC6 for power savings
- Use autosuspend for idle systems
- Monitor thermal throttling
- Profile power per workload
- Test D3Cold on supported platforms

**For Performance:**
- Minimize wakeref scope
- Avoid rapid suspend/resume cycles
- Use RPS for balanced performance
- Disable ASPM if latency-critical
- Avoid extreme frequency scaling

**For Reliability:**
- Handle RPS frequency failures
- Monitor temperature continuously
- Test power state transitions
- Validate RC6 residency
- Check ACPI integration

---

## References

- [Runtime PM](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/i915/intel_runtime_pm.c)
- [RPS Implementation](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/i915/gt/intel_rps.c)
- Related: [04-Power-Management.md](04-Power-Management.md), [11-Error-Handling-Recovery.md](11-Error-Handling-Recovery.md)

---

**Next Steps:**
- Study runtime PM patterns in i915
- Profile power consumption with measurements
- Implement custom power monitoring
- Test power state transitions

---

**Document Status:** ✅ Complete  
**Version:** 1.0  
**Last Updated:** February 8, 2026
