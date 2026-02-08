# Performance Monitoring and Observation Architecture (OA)

**Document ID:** 14 | **Status:** Complete | **Last Updated:** February 2026

---

## Executive Summary

Performance monitoring is critical for understanding GPU behavior and optimizing applications. i915 provides the Observation Architecture (OA) for detailed performance counter collection and analysis.

### Key Topics
- **OA framework:** Configurable performance counter collection
- **Counter types:** Engine metrics, memory bandwidth, shader utilization
- **Sampling modes:** Periodic interrupt, on-demand, triggered collection
- **Power and thermal:** Energy tracking, temperature monitoring
- **Analysis tools:** Metrics aggregation, performance visualization

### Performance Impact
- **Monitoring overhead:** 1-5% CPU/GPU overhead when active
- **Counter resolution:** Sub-millisecond timestamps
- **Data rate:** 10-100KB per second depending on configuration

---

## Table of Contents

1. [Observation Architecture Overview](#observation-architecture-overview)
2. [Counter Configuration](#counter-configuration)
3. [Sampling Modes and Triggering](#sampling-modes-and-triggering)
4. [Counter Collection and Reporting](#counter-collection-and-reporting)
5. [Power and Thermal Monitoring](#power-and-thermal-monitoring)
6. [Performance Analysis](#performance-analysis)
7. [Debugging Monitoring Issues](#debugging-monitoring-issues)
8. [Summary & Best Practices](#summary--best-practices)

---

## Observation Architecture Overview

### OA Unit Architecture

```plaintext
GPU Hardware Performance Counters
┌──────────────────────────────────┐
│ OA Unit (Observation Architecture)│
│ Integrated in GPU                │
└────────┬─────────────────────────┘
         │
         ├─→ Counter Collection
         │   ├─ GPU utilization
         │   ├─ Memory bandwidth
         │   ├─ Engine load
         │   ├─ Cache hits/misses
         │   └─ Shader metrics
         │
         ├─→ Event Generation
         │   ├─ Periodic timer
         │   ├─ Interrupt trigger
         │   └─ Software trigger
         │
         ├─→ Data Buffering
         │   ├─ Circular buffer (OA buffer)
         │   ├─ Auto-wraparound
         │   └─ Overflow handling
         │
         └─→ Delivery to Driver
             ├─ DMA to system RAM
             ├─ Interrupt notification
             └─ Read from userspace
```

### Counter Types

```c
// Performance counter definitions
struct i915_oa_metric_set {
    const char *name;
    const char *desc;
    
    // Counter definitions
    const struct i915_oa_counter *counters;
    int num_counters;
    
    // Sampling configuration
    u32 sample_rate;        // Samples per second
    u32 report_format;      // Data format version
};

// Individual counter definition
struct i915_oa_counter {
    const char *name;
    const char *symbol;
    const char *description;
    
    enum oa_counter_type type;  // Percentage, count, time, etc
    enum oa_counter_source src; // GPU/Memory/Thermal source
    
    u32 offset;             // Byte offset in sample
    u32 width;              // Bits per counter value
};

// Counter types
enum oa_counter_type {
    OA_COUNTER_TYPE_PERCENT,        // 0-100%
    OA_COUNTER_TYPE_COUNT,          // Event count
    OA_COUNTER_TYPE_TIME,           // Duration in nanoseconds
    OA_COUNTER_TYPE_BANDWIDTH,      // Bytes per second
    OA_COUNTER_TYPE_FREQUENCY,      // MHz
    OA_COUNTER_TYPE_TEMPERATURE,    // Celsius
    OA_COUNTER_TYPE_POWER,          // Watts
};
```

---

## Counter Configuration

### Metric Set Selection

```plaintext
Available Metric Sets by Generation
┌─────────────────────────────────────┐
│ Gen12+ (Newer)                      │
│ - Render Metrics (shader, RW)       │
│ - Memory Metrics (L3, bandwidth)    │
│ - Video Metrics (codec utilization) │
├─────────────────────────────────────┤
│ Gen9-Gen11                          │
│ - Render Metrics                    │
│ - Memory Metrics                    │
│ - Compute Metrics (optional)        │
├─────────────────────────────────────┤
│ Gen8 (Limited)                      │
│ - Basic render metrics only         │
└─────────────────────────────────────┘
```

### Setting up OA Counter Collection

```c
// OA context initialization
struct i915_oa_client {
    struct i915_drm_private *i915;
    struct pid *pid;
    struct rcu_head rcu;
    
    // Configured metric set
    const struct i915_oa_metric_set *metrics;
    
    // Sampling parameters
    u32 sample_rate;           // Samples/second
    bool pinned_ctx_oa;        // Pin context during OA
    
    // Buffers
    struct i915_gem_object *obj_buffer;
    void *virtual_addr;
    u32 buffer_size;
    
    // State tracking
    bool sampling;
    bool use_marks;
    struct i915_perf_stream *stream;
};

// Initialize OA for performance measurement
static int i915_oa_init(struct i915_drm_private *i915,
                       struct i915_oa_client *client,
                       struct drm_i915_perf_open_param *param)
{
    struct i915_oa_metric_set *metric_set;
    int ret;
    
    // 1. Validate and select metric set
    metric_set = i915_oa_select_metric_set(i915, param->metrics_set_id);
    if (!metric_set) {
        dev_dbg(i915->drm.dev,
                "Invalid metric set: %u\n",
                param->metrics_set_id);
        return -EINVAL;
    }
    
    client->metrics = metric_set;
    
    // 2. Set sampling parameters
    if (param->sample_rate > 0) {
        if (param->sample_rate > MAX_OA_SAMPLE_RATE) {
            return -EINVAL;
        }
        client->sample_rate = param->sample_rate;
    } else {
        client->sample_rate = DEFAULT_OA_SAMPLE_RATE;
    }
    
    // 3. Allocate OA buffer
    client->obj_buffer = i915_gem_object_create_shmem(i915,
                                                       OA_BUFFER_SIZE);
    if (IS_ERR(client->obj_buffer)) {
        return PTR_ERR(client->obj_buffer);
    }
    
    // 4. Map buffer to userspace
    client->virtual_addr = i915_gem_dmabuf_vmap(client->obj_buffer);
    if (!client->virtual_addr) {
        ret = -ENOMEM;
        goto out_put;
    }
    
    // 5. Configure GPU OA unit
    ret = i915_oa_configure(i915, client);
    if (ret)
        goto out_vunmap;
    
    client->sampling = true;
    
    return 0;
    
out_vunmap:
    i915_gem_dmabuf_vunmap(client->obj_buffer,
                           client->virtual_addr);
out_put:
    i915_gem_object_put(client->obj_buffer);
    return ret;
}

// Configure GPU OA unit
static int i915_oa_configure(struct i915_drm_private *i915,
                            struct i915_oa_client *client)
{
    const struct i915_oa_metric_set *metrics = client->metrics;
    struct intel_uncore *uncore = &i915->uncore;
    
    // 1. Stop any existing OA
    intel_uncore_rmw(uncore, GEN12_OA_CTL,
                     GEN12_OA_CTL_ENABLE, 0);
    
    // 2. Configure metric set
    for (int i = 0; i < metrics->num_counters; i++) {
        intel_uncore_write(uncore,
                          GEN12_OA_COUNTER_BASE + i * 4,
                          metrics->counters[i].offset);
    }
    
    // 3. Configure sampling rate
    intel_uncore_write(uncore, GEN12_OA_SAMPLE_RATE,
                       client->sample_rate);
    
    // 4. Configure OA buffer
    intel_uncore_write(uncore, GEN12_OA_BUFFER_BASE,
                       i915_gem_object_ggtt_offset(client->obj_buffer));
    
    intel_uncore_write(uncore, GEN12_OA_BUFFER_SIZE,
                       client->buffer_size >> PAGE_SHIFT);
    
    // 5. Start OA collection
    intel_uncore_rmw(uncore, GEN12_OA_CTL,
                     0, GEN12_OA_CTL_ENABLE);
    
    return 0;
}
```

---

## Sampling Modes and Triggering

### Periodic Sampling

```plaintext
Periodic OA Sampling
┌─────────────────────────────────────┐
│ OA Unit (configured for 10kHz)      │
└────────┬────────────────────────────┘
         │
         ├─ 0.0ms  → Sample 1 (GPU state snapshot)
         │
         ├─ 0.1ms  → Sample 2 (GPU state snapshot)
         │
         ├─ 0.2ms  → Sample 3 (GPU state snapshot)
         │
         ├─ 0.3ms  → Sample 4 (GPU state snapshot)
         │
         └─ ...continues until stopped

Samples captured every 100µs at 10kHz rate
```

### Event-Triggered Sampling

```c
// Trigger OA sample on specific event
static int i915_oa_trigger_sample(struct i915_oa_client *client,
                                  enum oa_trigger_event event)
{
    struct intel_uncore *uncore =
        &client->i915->uncore;
    u32 ctrl;
    
    switch (event) {
    case OA_TRIGGER_SUBMIT:
        // Sample when work submitted
        ctrl = GEN12_OA_TRIGGER_SUBMIT;
        break;
        
    case OA_TRIGGER_COMPLETE:
        // Sample when work completes
        ctrl = GEN12_OA_TRIGGER_COMPLETE;
        break;
        
    case OA_TRIGGER_PREEMPT:
        // Sample on context preemption
        ctrl = GEN12_OA_TRIGGER_PREEMPT;
        break;
        
    default:
        return -EINVAL;
    }
    
    // Trigger immediate sample
    intel_uncore_write(uncore,
                      GEN12_OA_TRIGGER,
                      ctrl);
    
    return 0;
}
```

### Marks and Synchronization

Correlate GPU time with CPU time:

```c
// Place timing mark in OA stream
static int i915_oa_place_mark(struct i915_oa_client *client,
                              u64 cpu_time)
{
    struct i915_oa_mark {
        u64 cpu_timestamp;
        u32 seqno;
        u32 reserved;
    };
    
    struct i915_oa_mark mark;
    struct i915_request *rq;
    int ret;
    
    // 1. Create timing marker request
    rq = i915_request_create(client->context);
    if (IS_ERR(rq))
        return PTR_ERR(rq);
    
    // 2. Record CPU time
    mark.cpu_timestamp = cpu_time;
    mark.seqno = rq->fence.seqno;
    
    // 3. Write marker to OA buffer
    ret = i915_oa_write_mark(client, &mark);
    
    i915_request_put(rq);
    return ret;
}
```

---

## Counter Collection and Reporting

### Reading OA Data

```c
// Read performance counter data from OA buffer
static ssize_t i915_oa_read(struct i915_oa_client *client,
                            char __user *buf,
                            size_t count)
{
    struct i915_oa_buffer *oa_buffer;
    const struct i915_oa_metric_set *metrics;
    u8 *sample_data;
    ssize_t bytes_read = 0;
    
    oa_buffer = &client->i915->perf.oa_buffer;
    metrics = client->metrics;
    
    // 1. Check for available data
    if (oa_buffer->head == oa_buffer->tail)
        return 0;  // No new samples
    
    // 2. Walk through samples
    while (bytes_read < count &&
           oa_buffer->head != oa_buffer->tail) {
        
        // Get next sample from circular buffer
        sample_data = oa_buffer->vaddr + oa_buffer->head;
        
        // 3. Copy sample to userspace
        if (copy_to_user(buf + bytes_read,
                        sample_data,
                        metrics->sample_size)) {
            return -EFAULT;
        }
        
        bytes_read += metrics->sample_size;
        
        // 4. Advance buffer pointer
        oa_buffer->head += metrics->sample_size;
        if (oa_buffer->head >= oa_buffer->size)
            oa_buffer->head = 0;  // Wrap around
    }
    
    return bytes_read;
}

// Parse sample data
static void i915_oa_parse_sample(const u8 *sample,
                                 const struct i915_oa_metric_set *metrics,
                                 struct i915_oa_sample_data *parsed)
{
    // 1. Extract timestamp
    parsed->timestamp = *(u64 *)sample;
    
    // 2. Parse counter values
    for (int i = 0; i < metrics->num_counters; i++) {
        const struct i915_oa_counter *counter = 
            &metrics->counters[i];
        
        u32 value = *(u32 *)(sample + counter->offset);
        
        // Store counter value
        parsed->counters[i] = value;
    }
    
    // 3. Extract context information
    parsed->ctx_id = *(u32 *)(sample + CONTEXT_ID_OFFSET);
    parsed->request_id = *(u32 *)(sample + REQUEST_ID_OFFSET);
}
```

### Data Format

```plaintext
OA Sample Format (Typical 256 bytes)
┌──────────────────────────────────┐
│ Timestamp (8 bytes)               │  ← GPU timestamp
├──────────────────────────────────┤
│ Report ID (4 bytes)               │  ← Sample identifier
├──────────────────────────────────┤
│ Context ID (4 bytes)              │  ← Context tag
├──────────────────────────────────┤
│ GPU Utilization (4 bytes)         │  ← 0-100%
├──────────────────────────────────┤
│ EU Array Utilization (4 bytes)    │  ← Shader engines active
├──────────────────────────────────┤
│ Memory Read Bandwidth (4 bytes)   │  ← GB/s
├──────────────────────────────────┤
│ Memory Write Bandwidth (4 bytes)  │  ← GB/s
├──────────────────────────────────┤
│ L3 Cache Hit Rate (4 bytes)       │  ← 0-100%
├──────────────────────────────────┤
│ GPU Frequency (4 bytes)           │  ← MHz
├──────────────────────────────────┤
│ GPU Temperature (2 bytes)         │  ← Celsius
├──────────────────────────────────┤
│ Reserved (variable)               │  ← Padding
└──────────────────────────────────┘
Total: 256 bytes per sample
```

---

## Power and Thermal Monitoring

### Energy Consumption Tracking

```c
// Monitor GPU energy consumption
struct i915_energy_info {
    u64 timestamp;
    u32 energy_units;      // Joules
    u32 power_units;       // Watts
    u32 instantaneous_power;
    u64 cumulative_energy;
};

// Read energy counters
static int i915_read_energy(struct i915_drm_private *i915,
                           struct i915_energy_info *info)
{
    struct intel_uncore *uncore = &i915->uncore;
    u32 energy_status;
    
    info->timestamp = ktime_get_ns();
    
    // 1. Read energy status register
    energy_status = intel_uncore_read(uncore, MSR_RAPL_STATUS);
    
    // 2. Extract energy in fixed units
    // Typical: 65536 energy counts = 1 Joule
    info->energy_units = (energy_status & 0xFFFFFF) / 65536;
    
    // 3. Calculate instantaneous power
    // Subtract previous sample and divide by time delta
    static u64 last_energy = 0;
    static u64 last_time = 0;
    
    u64 energy_delta = info->energy_units - last_energy;
    u64 time_delta_ns = info->timestamp - last_time;
    u64 time_delta_s = time_delta_ns / 1000000000;
    
    if (time_delta_s > 0) {
        info->instantaneous_power = energy_delta / time_delta_s;
        info->cumulative_energy += energy_delta;
    }
    
    last_energy = info->energy_units;
    last_time = info->timestamp;
    
    return 0;
}
```

### Temperature Monitoring

```c
// Monitor GPU temperature
static int i915_read_temperature(struct i915_drm_private *i915,
                                 u32 *temperature_c)
{
    struct intel_uncore *uncore = &i915->uncore;
    u32 temp_status;
    u32 raw_temp;
    
    // Read temperature sensor
    temp_status = intel_uncore_read(uncore, TS_CTRL);
    
    // Extract temperature value
    raw_temp = (temp_status >> TS_TEMP_SHIFT) & TS_TEMP_MASK;
    
    // Convert to Celsius (format depends on platform)
    // Typical: raw_temp * 1°C, offset -50°C
    *temperature_c = raw_temp - 50;
    
    return 0;
}

// Thermal throttle monitoring
static void i915_thermal_notify(struct i915_drm_private *i915)
{
    u32 temp;
    
    i915_read_temperature(i915, &temp);
    
    if (temp > THERMAL_CRITICAL) {
        dev_crit(i915->drm.dev,
                "GPU critical thermal: %u°C\n",
                temp);
        // Trigger emergency shutdown
        intel_rps_set(i915, RPS_MIN_FREQ);
    } else if (temp > THERMAL_WARNING) {
        dev_warn(i915->drm.dev,
                "GPU thermal warning: %u°C\n",
                temp);
        // Reduce frequency
        intel_rps_set(i915, RPS_FREQ_50);
    }
}
```

---

## Performance Analysis

### Aggregating Metrics

```c
// Aggregate samples into metrics
struct i915_oa_metrics_summary {
    u64 duration_ns;
    u32 samples_collected;
    
    // Averages
    u32 avg_gpu_utilization;
    u32 avg_eu_utilization;
    u32 avg_frequency_mhz;
    
    // Peaks
    u32 peak_power_watts;
    u32 peak_temperature_c;
    
    // Memory
    u64 total_mem_read_bytes;
    u64 total_mem_write_bytes;
    
    // Cache
    u32 avg_l3_hit_rate;
};

// Compute metrics from samples
static void i915_oa_compute_metrics(
    const struct i915_oa_sample_data *samples,
    int num_samples,
    struct i915_oa_metrics_summary *summary)
{
    u64 gpu_util_sum = 0;
    u64 freq_sum = 0;
    u32 max_power = 0;
    u32 max_temp = 0;
    
    for (int i = 0; i < num_samples; i++) {
        const struct i915_oa_sample_data *s = &samples[i];
        
        // Sum for averaging
        gpu_util_sum += s->gpu_utilization;
        freq_sum += s->frequency_mhz;
        
        // Track peaks
        if (s->power_watts > max_power)
            max_power = s->power_watts;
        if (s->temperature_c > max_temp)
            max_temp = s->temperature_c;
    }
    
    // Calculate averages
    summary->samples_collected = num_samples;
    summary->avg_gpu_utilization = gpu_util_sum / num_samples;
    summary->avg_frequency_mhz = freq_sum / num_samples;
    summary->peak_power_watts = max_power;
    summary->peak_temperature_c = max_temp;
}
```

### Visualization and Export

```c
// Export metrics in JSON format
static int i915_oa_export_json(
    const struct i915_oa_metrics_summary *summary,
    char *buf, size_t buf_size)
{
    return snprintf(buf, buf_size,
        "{\n"
        "  \"duration_ms\": %llu,\n"
        "  \"samples\": %u,\n"
        "  \"gpu_utilization_pct\": %u,\n"
        "  \"frequency_mhz\": %u,\n"
        "  \"peak_power_w\": %u,\n"
        "  \"peak_temp_c\": %u,\n"
        "  \"memory_read_gb\": %llu,\n"
        "  \"memory_write_gb\": %llu,\n"
        "  \"l3_hit_rate_pct\": %u\n"
        "}\n",
        summary->duration_ns / 1000000,
        summary->samples_collected,
        summary->avg_gpu_utilization,
        summary->avg_frequency_mhz,
        summary->peak_power_watts,
        summary->peak_temperature_c,
        summary->total_mem_read_bytes / (1024*1024*1024),
        summary->total_mem_write_bytes / (1024*1024*1024),
        summary->avg_l3_hit_rate);
}
```

---

## Debugging Monitoring Issues

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **No OA samples** | Collection not started | Check OA enable register |
| **Buffer overflow** | Sample rate too high | Reduce rate or increase buffer |
| **Inaccurate values** | Wrong metric set | Verify metric set ID matches GPU |
| **Timestamp misalignment** | CPU/GPU clock drift | Use synchronization marks |
| **Thermal spikes** | Measurement noise | Apply smoothing filter |

### Enable OA Debugging

```bash
# Enable OA-related debugging
echo "module i915 +p" > /sys/kernel/debug/dynamic_debug/control

# Enable performance monitoring tracing
echo 1 > /sys/kernel/debug/tracing/events/i915_perf/enable

# View OA buffer status
cat /sys/kernel/debug/dri/0/i915_oa_buffer_status
```

---

## Summary & Best Practices

### Key Takeaways

1. **Metric selection:** Choose right set for workload
2. **Sample rate trade-off:** Higher rate = more overhead
3. **Buffer management:** Prevent overflow with large buffers
4. **Synchronization:** Use marks for CPU-GPU correlation
5. **Analysis:** Aggregate samples for trend detection

### Best Practices

**For Profiling:**
- Start with 1kHz sampling, increase if needed
- Use appropriate metric set for workload type
- Capture marks for CPU time correlation
- Analyze off-GPU to reduce overhead
- Use smoothing filters for noisy data

**For Optimization:**
- Monitor GPU utilization and frequency
- Track memory bandwidth saturation
- Detect thermal throttling patterns
- Identify synchronization stalls
- Profile power consumption per workload

**For Debugging:**
- Enable OA tracing for detailed analysis
- Check metric set compatibility
- Verify buffer size is sufficient
- Validate timestamp accuracy
- Test with synthetic workloads

---

## References

- [Performance Monitoring](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/i915/i915_perf.c)
- [OA Unit Configuration](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/i915/oa)
- Related: [04-Power-Management.md](04-Power-Management.md), [09-i915-Fence-Timeline-Study.md](09-i915-Fence-Timeline-Study.md)

---

**Next Steps:**
- Study OA configuration in i915_perf.c
- Analyze metric set definitions
- Profile a real workload with OA
- Implement custom metric analysis tools

---

**Document Status:** ✅ Complete  
**Version:** 1.0  
**Last Updated:** February 8, 2026
