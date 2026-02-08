# Display and Output Management

**Document ID:** 17 | **Status:** Complete | **Last Updated:** February 2026

---

## Executive Summary

The display engine manages GPU outputs to monitors and external devices. This document covers the display pipeline, mode setting, EDID handling, and hotplug detection.

### Key Topics
- **Display pipeline:** Encoders, connectors, transcoders
- **Mode setting:** Resolution, refresh rate, color depth configuration  
- **EDID discovery:** Monitor capability detection via DDC
- **Hotplug handling:** Dynamic connection/disconnection events

### Performance Metrics
- **Mode set time:** 50-200ms for CRTC configuration
- **Hotplug detection:** 100-500ms event delivery
- **EDID read:** 10-50ms over DDC bus

---

## Table of Contents

1. [Display Architecture](#display-architecture)
2. [Display Pipeline Components](#display-pipeline-components)
3. [Mode Setting and Configuration](#mode-setting-and-configuration)
4. [EDID and DDC](#edid-and-ddc)
5. [Hotplug Detection](#hotplug-detection)
6. [Multi-Display Support](#multi-display-support)
7. [Debugging Display Issues](#debugging-display-issues)
8. [Summary & Best Practices](#summary--best-practices)

---

## Display Architecture

### Display Output Flow

```plaintext
GPU Frame Buffer to Monitor
┌─────────────────────────────────────┐
│ Frame Buffer in GPU Memory          │
│ (Contains pixel data)               │
└────────┬────────────────────────────┘
         │
         ├─→ Pipe/Crtc
         │   ├─ Scans out pixels
         │   ├─ Adds timing signals
         │   └─ Outputs pixel stream
         │
         ├─→ Plane
         │   ├─ Fetches pixels from buffer
         │   ├─ Applies color transformations
         │   └─ Handles scaling/rotation
         │
         ├─→ Encoder
         │   ├─ Converts pixel stream
         │   ├─ Encodes HDMI/DP signal
         │   └─ Adds metadata
         │
         ├─→ Connector/Port
         │   ├─ Physical output (HDMI/DP/VGA)
         │   ├─ Manages signaling levels
         │   └─ Detects connection
         │
         └─→ Monitor Display
             └─ Shows the image
```

### Display Components

```c
// Display pipeline structure
struct intel_display {
    struct intel_crtc *crtcs[I915_MAX_CRTCS];
    struct intel_encoder *encoders[I915_MAX_ENCODERS];
    struct intel_connector *connectors[I915_MAX_CONNECTORS];
    struct intel_plane *planes[I915_MAX_PLANES];
};

// CRTC - Cathode Ray Tube Controller (timing generator)
struct intel_crtc {
    struct drm_crtc base;
    enum intel_crtc_id crtc_id;  // Pipe A, B, C
    
    // Timing configuration
    struct drm_display_mode mode;
    int clock;                     // Pixel clock in kHz
    int h_total, h_active, h_sync_start, h_sync_end;
    int v_total, v_active, v_sync_start, v_sync_end;
    
    // Connected to connector
    struct intel_connector *connector;
};

// Encoder - converts pixel stream to display signal
struct intel_encoder {
    enum intel_output_type type;   // HDMI, DP, VGA, etc
    struct intel_crtc *crtc;       // Connected CRTC
    
    // Type-specific configuration
    union {
        struct intel_hdmi_encoder hdmi;
        struct intel_dp_encoder dp;
        struct intel_dvo_encoder dvo;
    };
};

// Connector - physical output connector
struct intel_connector {
    enum intel_output_type type;   // Output type
    enum connector_status status;  // Connected/Disconnected
    
    // EDID information
    struct edid *edid;
    struct drm_display_info display_info;
    
    // Hotplug detection
    struct delayed_work hotplug_work;
};
```

---

## Display Pipeline Components

### CRTC Configuration

```c
// Configure CRTC for specific mode
static int intel_crtc_set_mode(struct intel_crtc *crtc,
                               struct drm_display_mode *mode)
{
    struct i915_drm_private *i915 = crtc_to_i915(crtc);
    struct intel_uncore *uncore = &i915->uncore;
    
    // 1. Disable CRTC during reconfiguration
    intel_uncore_rmw(uncore, PIPECONF(crtc->crtc_id),
                     PIPECONF_ENABLE, 0);
    
    intel_wait_for_vblank(crtc);
    
    // 2. Configure horizontal timing
    intel_uncore_write(uncore, HTOTAL(crtc->crtc_id),
                      ((mode->htotal - 1) << 16) |
                      (mode->hdisplay - 1));
    
    intel_uncore_write(uncore, HBLANK(crtc->crtc_id),
                      ((mode->htotal - 1) << 16) |
                      (mode->hdisplay - 1));
    
    intel_uncore_write(uncore, HSYNC(crtc->crtc_id),
                      ((mode->hsync_end - 1) << 16) |
                      (mode->hsync_start - 1));
    
    // 3. Configure vertical timing
    intel_uncore_write(uncore, VTOTAL(crtc->crtc_id),
                      ((mode->vtotal - 1) << 16) |
                      (mode->vdisplay - 1));
    
    intel_uncore_write(uncore, VBLANK(crtc->crtc_id),
                      ((mode->vtotal - 1) << 16) |
                      (mode->vdisplay - 1));
    
    intel_uncore_write(uncore, VSYNC(crtc->crtc_id),
                      ((mode->vsync_end - 1) << 16) |
                      (mode->vsync_start - 1));
    
    // 4. Set pixel clock
    intel_crtc_set_clock(crtc, mode->clock);
    
    // 5. Enable CRTC
    intel_uncore_rmw(uncore, PIPECONF(crtc->crtc_id),
                     0, PIPECONF_ENABLE);
    
    dev_info(i915->drm.dev,
            "CRTC %c mode set: %ux%u@%uHz\n",
            crtc_name(crtc), mode->hdisplay,
            mode->vdisplay, drm_mode_vrefresh(mode));
    
    return 0;
}
```

### Encoder Configuration

```c
// HDMI encoder setup
static int intel_hdmi_set_output(struct intel_encoder *encoder)
{
    struct intel_uncore *uncore = encoder_to_uncore(encoder);
    u32 hdmi_config = 0;
    
    // 1. Get HDMI register offset
    unsigned int hdmi_reg = HDMI_CTRL(encoder->port);
    
    // 2. Configure HDMI parameters
    if (encoder->crtc->mode.flags & DRM_MODE_FLAG_PHSYNC)
        hdmi_config |= HDMI_HSYNC_ACTIVE_HIGH;
    else
        hdmi_config |= HDMI_HSYNC_ACTIVE_LOW;
    
    if (encoder->crtc->mode.flags & DRM_MODE_FLAG_PVSYNC)
        hdmi_config |= HDMI_VSYNC_ACTIVE_HIGH;
    else
        hdmi_config |= HDMI_VSYNC_ACTIVE_LOW;
    
    // 3. Enable HDMI output
    hdmi_config |= HDMI_ENABLE;
    
    intel_uncore_write(uncore, hdmi_reg, hdmi_config);
    
    return 0;
}

// DisplayPort (DP) encoder setup
static int intel_dp_set_output(struct intel_encoder *encoder)
{
    struct intel_dp *dp = enc_to_intel_dp(encoder);
    u32 dp_ctrl = 0;
    
    // 1. Configure DP link rate
    dp_ctrl |= DP_LINK_TRAIN_PAT_1;
    dp_ctrl |= (dp->link_bw << DP_LINK_BW_SHIFT);
    
    // 2. Configure DP lane count
    dp_ctrl |= ((dp->lane_count - 1) << DP_LANE_COUNT_SHIFT);
    
    // 3. Configure color depth
    dp_ctrl |= DP_COLOR_DEPTH_8BIT;
    
    // 4. Enable DP link training
    intel_dp_start_link_train(dp);
    
    // 5. Wait for link to be trained
    if (!intel_dp_wait_link_ready(dp))
        return -EIO;
    
    return 0;
}
```

---

## Mode Setting and Configuration

### Atomic Mode Setting

```plaintext
Atomic Mode Setting Flow
┌──────────────────────────────────┐
│ New Mode Requested               │
│ (resolution, refresh rate, etc)  │
└──────┬───────────────────────────┘
       │
       ├─→ Build Atomic State
       │   ├─ Validate new config
       │   ├─ Check resource availability
       │   ├─ Compute required changes
       │   └─ Build atomic transaction
       │
       ├─→ Validate State
       │   ├─ Check mode validity
       │   ├─ Verify encoder assignment
       │   ├─ Check power requirements
       │   └─ Validate physical constraints
       │
       ├─→ Disable Old Mode (if active)
       │   ├─ Stop scanout
       │   ├─ Disable planes
       │   └─ Power down encoder
       │
       ├─→ Apply New Configuration
       │   ├─ Configure CRTC timing
       │   ├─ Setup encoder
       │   ├─ Enable planes
       │   └─ Start scanout
       │
       └─→ Mode Set Complete
           └─ Display shows new resolution
```

### Mode Set Implementation

```c
// Set display mode atomically
static int intel_display_set_mode(struct drm_crtc *crtc,
                                  struct drm_display_mode *mode)
{
    struct drm_device *dev = crtc->dev;
    struct drm_atomic_state *state;
    struct drm_crtc_state *crtc_state;
    int ret;
    
    // 1. Allocate atomic state
    state = drm_atomic_state_alloc(dev);
    if (!state)
        return -ENOMEM;
    
    state->acquire_ctx = drm_modeset_legacy_acquire_ctx(crtc);
    
    // 2. Get CRTC state and modify
    crtc_state = drm_atomic_get_crtc_state(state, crtc);
    if (IS_ERR(crtc_state)) {
        ret = PTR_ERR(crtc_state);
        goto out;
    }
    
    // 3. Set new mode
    ret = drm_atomic_set_mode_for_crtc(crtc_state, mode);
    if (ret)
        goto out;
    
    // 4. Validate the atomic state
    ret = drm_atomic_check_only(state);
    if (ret)
        goto out;
    
    // 5. Commit the atomic state
    ret = drm_atomic_commit(state);
    if (ret)
        goto out;
    
    dev_info(dev->dev, "Mode set to %ux%u@%u\n",
            mode->hdisplay, mode->vdisplay,
            drm_mode_vrefresh(mode));
    
out:
    drm_atomic_state_put(state);
    return ret;
}
```

---

## EDID and DDC

### EDID Reading

```plaintext
EDID Discovery Process
┌──────────────────────────────────┐
│ Monitor Connected (hotplug)      │
└──────┬───────────────────────────┘
       │
       ├─→ Read EDID via DDC
       │   ├─ Send I2C request to monitor
       │   ├─ Monitor returns EDID block
       │   ├─ Retry on CRC error
       │   └─ Parse EDID data
       │
       ├─→ Extract Monitor Capabilities
       │   ├─ Native resolution
       │   ├─ Supported refresh rates
       │   ├─ Aspect ratio
       │   ├─ Color gamut
       │   └─ Supported features
       │
       ├─→ Validate EDID
       │   ├─ Check header signature
       │   ├─ Verify checksum
       │   └─ Check version
       │
       └─→ Build Display Modes
           ├─ Create mode for each capability
           ├─ Add preferred mode first
           └─ Return to userspace
```

### EDID Parsing

```c
// Read EDID from monitor via DDC
static struct edid *intel_connector_get_edid(
    struct intel_connector *connector)
{
    struct edid *edid;
    int retry;
    
    // Try reading EDID multiple times (DDC can be flaky)
    for (retry = 0; retry < 3; retry++) {
        edid = drm_get_edid(&connector->base,
                           connector->ddc);
        
        if (edid) {
            // Validate EDID checksum
            if (edid_valid(edid)) {
                return edid;
            }
            kfree(edid);
        }
        
        // Wait before retry
        msleep(100);
    }
    
    dev_warn(connector_to_i915(connector)->drm.dev,
            "Failed to read EDID from monitor\n");
    
    return NULL;
}

// Parse EDID and extract modes
static void intel_connector_parse_edid(
    struct intel_connector *connector)
{
    struct edid *edid = connector->edid;
    struct drm_display_info *info = &connector->display_info;
    struct drm_display_mode *mode;
    int i;
    
    if (!edid) {
        return;
    }
    
    // Extract basic info
    info->width_mm = edid->width_cm * 10;
    info->height_mm = edid->height_cm * 10;
    
    // Extract supported resolutions
    for (i = 0; i < EDID_MODE_COUNT; i++) {
        if (edid->modes[i].h_active == 0)
            break;
        
        // Create DRM mode for this capability
        mode = drm_mode_create(connector->base.dev);
        mode->hdisplay = edid->modes[i].h_active;
        mode->vdisplay = edid->modes[i].v_active;
        mode->vrefresh = edid->modes[i].refresh_rate;
        
        // Add to connector's mode list
        drm_mode_probed_add(&connector->base, mode);
    }
}
```

### DDC Communication

```c
// I2C access over DDC bus
static int intel_ddc_get_modes(struct intel_connector *connector)
{
    struct i2c_adapter *ddc = connector->ddc;
    struct edid *edid;
    int num_modes = 0;
    
    // 1. Read EDID via DDC
    edid = intel_connector_get_edid(connector);
    if (!edid) {
        dev_warn(ddc->dev, "DDC read failed\n");
        return 0;
    }
    
    // 2. Parse EDID for modes
    num_modes = drm_add_edid_modes(&connector->base, edid);
    
    // 3. Store EDID for later use
    connector->edid = edid;
    
    return num_modes;
}
```

---

## Hotplug Detection

### Hotplug Interrupt Handling

```plaintext
Hotplug Event Detection
┌──────────────────────────────────┐
│ Monitor Connection Changed       │
│ (Plugged in or unplugged)        │
└──────┬───────────────────────────┘
       │
       ├─→ GPIO Interrupt Fired
       │   └─ Connector status changed
       │
       ├─→ I915 Hotplug ISR
       │   ├─ Detect which port
       │   ├─ Queue hotplug worker
       │   └─ Return from interrupt
       │
       ├─→ Hotplug Worker Thread
       │   ├─ Read connector status
       │   ├─ Read EDID if connected
       │   ├─ Update display modes
       │   └─ Notify userspace
       │
       └─→ Userspace Notification
           ├─ udev event
           ├─ DRM event
           └─ Application response
```

### Hotplug Implementation

```c
// Hotplug detection routine
static void intel_hotplug_work_func(struct work_struct *work)
{
    struct delayed_work *delayed_work =
        container_of(work, struct delayed_work, work);
    
    struct intel_connector *connector =
        container_of(delayed_work, struct intel_connector,
                     hotplug_work);
    
    struct drm_connector *base = &connector->base;
    enum connector_status old_status = base->status;
    enum connector_status new_status;
    
    // 1. Detect current connector status
    new_status = intel_connector_detect(connector);
    
    // 2. If status changed, handle event
    if (new_status != old_status) {
        dev_info(connector_to_i915(connector)->drm.dev,
                "%s hotplug event: %s\n",
                connector->name,
                new_status == connector_connected ?
                "Connected" : "Disconnected");
        
        base->status = new_status;
        
        if (new_status == connector_connected) {
            // Monitor plugged in
            intel_connector_update_edid(connector);
        } else {
            // Monitor unplugged
            intel_connector_clear_edid(connector);
        }
        
        // Notify userspace
        drm_sysfs_hotplug_event(base->dev);
    }
}

// Hotplug ISR handler
static void intel_hotplug_irq_handler(struct i915_drm_private *i915)
{
    u32 hotplug_status;
    int i;
    
    // Read which connectors have hotplug events
    hotplug_status = intel_uncore_read(&i915->uncore,
                                       PORT_HOTPLUG_STAT);
    
    if (!hotplug_status)
        return;
    
    // Clear the status
    intel_uncore_write(&i915->uncore,
                      PORT_HOTPLUG_STAT,
                      hotplug_status);
    
    // Queue work for each connector with event
    for (i = 0; i < I915_MAX_CONNECTORS; i++) {
        struct intel_connector *connector =
            i915->connectors[i];
        
        if (!connector)
            continue;
        
        // Check if this connector has hotplug event
        if (hotplug_status & connector->hotplug_pin) {
            queue_delayed_work(system_unbound_wq,
                              &connector->hotplug_work,
                              msecs_to_jiffies(100));
        }
    }
}
```

---

## Multi-Display Support

### Dual Display Configuration

```c
// Configure multiple displays
static int intel_setup_dual_displays(struct i915_drm_private *i915)
{
    struct intel_crtc *crtc_a, *crtc_b;
    struct intel_connector *conn_hdmi, *conn_dp;
    struct drm_display_mode mode_hdmi, mode_dp;
    
    // 1. Get CRTC pipes
    crtc_a = &i915->display.crtcs[PIPE_A];
    crtc_b = &i915->display.crtcs[PIPE_B];
    
    // 2. Get connectors
    conn_hdmi = find_connector_by_type(i915, DRM_MODE_CONNECTOR_HDMI);
    conn_dp = find_connector_by_type(i915, DRM_MODE_CONNECTOR_DisplayPort);
    
    if (!conn_hdmi || !conn_dp)
        return -ENODEV;
    
    // 3. Set mode for HDMI (1920x1080@60Hz)
    drm_mode_create_from_cmdline(&mode_hdmi, "1920x1080");
    crtc_a->connector = conn_hdmi;
    intel_crtc_set_mode(crtc_a, &mode_hdmi);
    
    // 4. Set mode for DP (2560x1440@60Hz)
    drm_mode_create_from_cmdline(&mode_dp, "2560x1440");
    crtc_b->connector = conn_dp;
    intel_crtc_set_mode(crtc_b, &mode_dp);
    
    dev_info(i915->drm.dev, "Dual display configured\n");
    
    return 0;
}
```

---

## Debugging Display Issues

### Common Issues and Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| **No display output** | CRTC disabled or encoder off | Check PIPECONF register |
| **Wrong resolution** | Mode not set correctly | Verify mode parameters |
| **EDID not detected** | DDC communication failure | Check DDC I2C bus |
| **Hotplug not working** | Interrupt disabled | Enable hotplug ISR |
| **Flickering** | Timing mismatch | Adjust sync parameters |

### Display Debug Tools

```bash
# View connected monitors
cat /sys/kernel/debug/dri/0/i915_display_info

# Check CRTC configuration
cat /sys/kernel/debug/dri/0/i915_crtc_info

# Monitor hotplug events
cat /sys/kernel/debug/dri/0/i915_hotplug_info

# Enable display debugging
echo "module i915 +p" > /sys/kernel/debug/dynamic_debug/control
grep "i915_pch\|hotplug\|display" /sys/kernel/debug/dynamic_debug/control
```

---

## Summary & Best Practices

### Key Takeaways

1. **Pipeline order:** Pipe → Plane → Encoder → Connector
2. **Mode setting:** Always use atomic operations
3. **EDID reliability:** Retry DDC reads on failure
4. **Hotplug handling:** Defer work to avoid ISR stalls
5. **Multi-display:** Independent CRTCs per output

### Best Practices

**For Display Configuration:**
- Read EDID to discover modes
- Use atomic mode setting
- Validate all timing parameters
- Handle missing EDID gracefully
- Support multiple displays

**For Hotplug Handling:**
- Debounce hotplug events
- Queue work outside ISR
- Update EDID on connection
- Clear EDID on disconnection
- Notify userspace of changes

**For Debugging:**
- Monitor ISR hotplug delivery
- Check DDC/I2C communication
- Verify CRTC register values
- Enable display tracing
- Test with multiple monitors

---

## References

- [Display Code](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/i915/display)
- [EDID Handling](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/i915/display/intel_connector.c)
- Related: [12-Interrupt-Handling.md](12-Interrupt-Handling.md), [04-Power-Management.md](04-Power-Management.md)

---

**Next Steps:**
- Study display pipeline in drivers/gpu/drm/i915/display
- Implement custom hotplug handler
- Test EDID parsing with various monitors
- Profile mode setting latency

---

**Document Status:** ✅ Complete  
**Version:** 1.0  
**Last Updated:** February 8, 2026
