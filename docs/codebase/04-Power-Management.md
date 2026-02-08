# Power Management System

## Overview

The Intel i915 GPU driver implements comprehensive power management to optimize energy consumption while maintaining performance. The system manages multiple power domains including render (GT), display, and media engines.

**Key Components:**
- **RPS (Render Performance State):** Dynamic frequency/voltage scaling
- **RC6:** Power gating - reduces power when idle
- **SLPC:** Self-managed power scaling via GuC
- **Runtime PM:** Device-level power state control
- **Display Power Wells:** DPLL, CDCLK control

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────┐
│         Workload / Application Activity              │
└──────────┬───────────────────────────────────────────┘
           ↓
    ┌──────────────────────────┐
    │   intel_pm.c             │
    │  (Global Power Policy)   │
    └──────────┬───────────────┘
               ↓
    ┌────────────────────────────────────────┐
    │  gt/intel_rps.c                        │
    │  (Frequency Scaling)                   │
    │  - Dynamic frequency selection         │
    │  - Power budget management             │
    └──────────┬─────────────────────────────┘
               ↓
    ┌────────────────────────────────────────┐
    │  gt/intel_rc6.c                        │
    │  (Power Gating)                        │
    │  - Deep sleep states (RC6, RC6p, RC6pp)│
    │  - Context retention                   │
    └──────────┬─────────────────────────────┘
               ↓
    ┌────────────────────────────────────────┐
    │  intel_runtime_pm.c                    │
    │  (Runtime PM Framework)                │
    │  - Device suspend/resume               │
    │  - Autosuspend with timeout            │
    └──────────┬─────────────────────────────┘
               ↓
         GPU Hardware
    (Frequency/Voltage Control)
```

---

## Core Components

### 1. **Render Performance State (RPS) - intel_rps.c**

**Purpose:** Dynamically adjust GPU frequency based on load

**Key Structure:**
```c
struct intel_rps {
    struct mutex lock;                 // Serialization
    
    /* Hardware limits */
    struct {
        u8 min;                        // Minimum frequency
        u8 max;                        // Maximum frequency
        u8 rp1;                        // RP1 (high frequency)
        u8 rp0;                        // RP0 (max frequency)
    } limits;
    
    /* Current state */
    struct {
        u8 freq_req;                   // Requested frequency
        u8 actual;                     // Actual frequency
        u8 target;                     // Target frequency
    } cur;
    
    /* Load tracking */
    struct {
        u32 ei;                        // Energy Information counter
        unsigned long last_poll;       // Last polling time
    } ei;
    
    /* Interrupt handling */
    struct work_struct work;           // Frequency update work
    struct timer_list timer;           // Polling timer
    
    /* SLPC (Self-managed) mode */
    bool slpc_enabled;
};
```

**Frequency Levels:**
```
RP0 (Max)   ────────────────────────  Turbo/Boost frequency
  │
  │
RP1 (High)  ────────────────────────  Nominal frequency  
  │
  │
... (intermediate levels)
  │
  │
Min         ────────────────────────  Minimum frequency
```

**RPS Control Loop:**

```
┌──────────────────────────────────┐
│   Measure GPU Load (EI counter)  │
│   (Energy Information)           │
└────────────┬─────────────────────┘
             ↓
    ┌──────────────────────────────┐
    │  Compare with target load    │
    │  - If over target: increase  │
    │  - If under target: decrease │
    └────────────┬─────────────────┘
                 ↓
    ┌──────────────────────────────┐
    │  Calculate new frequency     │
    │  - Respect min/max limits    │
    │  - Apply ramp constraints    │
    └────────────┬─────────────────┘
                 ↓
    ┌──────────────────────────────┐
    │  Write to PUNIT              │
    │  (Power Unit Controller)     │
    └────────────┬─────────────────┘
                 ↓
    ┌──────────────────────────────┐
    │  PUNIT changes GPU frequency │
    │  and voltage (DVFS)          │
    └────────────┬─────────────────┘
                 ↓
    ┌──────────────────────────────┐
    │  Sleep (poll interval)       │
    │  Then repeat                 │
    └──────────────────────────────┘
```

**RPS Interrupt Handling:**

```c
// GPU signals RPS interrupt when load threshold crossed
void gen11_rps_irq_handler(struct intel_rps *rps)
{
    u32 pm_isr = intel_uncore_read(uncore, GEN11_PMISR);
    
    if (pm_isr & UP_EI_EXPIRED) {
        // GPU load exceeded threshold
        // Schedule frequency increase
        queue_work(rps->work_queue, &rps->work);
    }
    
    if (pm_isr & DOWN_EI_EXPIRED) {
        // GPU load below threshold
        // Schedule frequency decrease
        queue_work(rps->work_queue, &rps->work);
    }
}
```

---

### 2. **RC6 Power Gating - intel_rc6.c**

**Purpose:** Power down GPU when idle

**RC6 States:**

```
┌─────────────────────────────────────────────────────┐
│                    RC States                        │
├─────────────────────────────────────────────────────┤
│                                                      │
│  Full Power (0V)  ─────────────────────────────────│
│      │                                              │
│      ├── RC0 (On)   │ GPU actively rendering       │
│      │              │ Full voltage, max frequency │
│      │                                              │
│      └── RC1 (Idle) │ GPU idle, ready to RC6      │
│                     │ Slight power reduction       │
│                     │                              │
│          RC6 (Deep Sleep)                          │
│      ┌───────────────────────────────────────────┐ │
│      │ All GPU cores off                          │ │
│      │ Extremely low power                        │ │
│      │ Memory contents preserved                  │ │
│      │                                             │ │
│      ├─ RC6 Classic  │ Slow wakeup (~1ms)         │ │
│      │               │ Basic power saving          │ │
│      │                                             │ │
│      ├─ RC6p         │ Deeper sleep               │ │
│      │               │ GPU-level VRAM still on    │ │
│      │                                             │ │
│      └─ RC6pp        │ Deepest sleep              │ │
│                      │ Most power saving          │ │
│                      │ Longest wakeup time        │ │
│                                                    │ │
└─────────────────────────────────────────────────────┘
```

**RC6 Entry/Exit:**

```
                GPU Active
                    │
                    ↓
        ┌───────────────────────┐
        │ GPU becomes idle      │
        │ No pending requests   │
        └───────────┬───────────┘
                    ↓
        ┌───────────────────────────────┐
        │ Wait for idle threshold       │
        │ (default ~100ms)              │
        └───────────┬───────────────────┘
                    ↓
        ┌───────────────────────────────┐
        │ RC6 eligible:                 │
        │ - No active contexts          │
        │ - Interrupts disabled         │
        │ - Power well allowed off      │
        └───────────┬───────────────────┘
                    ↓
        ┌───────────────────────────────┐
        │ GPU enters RC6                │
        │ - Save context to memory      │
        │ - Power off GPU cores         │
        │ - Reduce voltage              │
        └───────────┬───────────────────┘
                    ↓
            (Very Low Power)
                    ↓
        ┌───────────────────────────────┐
        │ New request arrives / Interrupt│
        └───────────┬───────────────────┘
                    ↓
        ┌───────────────────────────────┐
        │ RC6 exit:                     │
        │ - Restore context             │
        │ - Power on GPU                │
        │ - Restore voltage/frequency   │
        └───────────┬───────────────────┘
                    ↓
                GPU Active
```

**RC6 Configuration:**

```c
struct intel_rc6 {
    struct drm_i915_private *i915;
    
    /* RC6 mode (bitmask) */
    unsigned int level;
    #define INTEL_RC6_ENABLE    (1 << 0)
    #define INTEL_RC6p_ENABLE   (1 << 1)
    #define INTEL_RC6pp_ENABLE  (1 << 2)
    
    /* Wake up time */
    u32 ctx_wake_latency;
    u32 media_wake_latency;
    
    /* Thresholds */
    u32 idle_threshold;                // Time before RC6 entry
    
    /* State tracking */
    bool enabled;
    bool hw_enabled;
};
```

---

### 3. **SLPC (Self-managed Performance) - intel_guc_slpc.c**

**Purpose:** GuC firmware autonomously manages frequency based on workload

**Benefits over CPU-based RPS:**
- Lower CPU overhead (firmware monitors load directly)
- Faster response to load changes
- Tighter feedback loop
- Better integration with GuC submission

**SLPC Flow:**

```
┌─────────────────────────────────────┐
│  GuC Firmware (Running on GuC MC)  │
│                                     │
│  Continuously monitors:             │
│  - GPU load indicators              │
│  - EI (Energy Information)          │
│  - Performance counters             │
└────────────┬──────────────────────┘
             ↓
    ┌────────────────────────────────┐
    │ Apply SLPC scheduling policy   │
    │ - Current workload type        │
    │ - Power budget constraints     │
    │ - Performance targets          │
    └────────────┬───────────────────┘
                 ↓
    ┌────────────────────────────────┐
    │ Calculate target frequency     │
    │ - Within min/max bounds        │
    │ - Respect ramp limits          │
    └────────────┬───────────────────┘
                 ↓
    ┌────────────────────────────────┐
    │ Write to hardware PUNIT        │
    │ (DVFS - Dynamic V/F)           │
    └────────────┬───────────────────┘
                 ↓
    ┌────────────────────────────────┐
    │ Optional: Notify driver        │
    │ (via CT message)               │
    │ for monitoring/debugging       │
    └────────────────────────────────┘
```

**SLPC Configuration:**

```c
int intel_guc_slpc_set_boost(struct intel_guc_slpc *slpc,
                             bool boost_enabled)
{
    // Enable/disable turbo boost mode
    // Message to GuC via CT
    return send_guc_ct_message(slpc->guc,
        SLPC_MSG_BOOST_ENABLE, 
        boost_enabled);
}

int intel_guc_slpc_set_max_freq(struct intel_guc_slpc *slpc,
                                u32 max_freq_mhz)
{
    // Set maximum frequency limit
    return send_guc_ct_message(slpc->guc,
        SLPC_MSG_SET_MAX_FREQ,
        max_freq_mhz);
}
```

---

### 4. **Runtime PM - intel_runtime_pm.c**

**Purpose:** System-level power state management (suspend/resume)

**Power Domains:**

```
GPU Device
    ├── Display Power Domain
    │   ├── DPLL (Display PLL)
    │   ├── CDCLK (Core Display Clock)
    │   └── Power Wells
    │
    ├── GT (Graphics) Power Domain
    │   ├── Render Engine
    │   ├── Media Engines
    │   └── GPU cores
    │
    └── Uncore (Shared)
        ├── System Agent
        ├── PCH (Platform Controller Hub)
        └── Fabric
```

**Runtime PM States:**

```
Active (0 refs)
     │
     │ pm_runtime_put_autosuspend()
     │
     ↓
Autosuspend Timer (default 10s)
     │
     │ [timeout expires]
     │
     ↓
Suspended (D3Hot)
     │
     │ pm_runtime_get() or activity
     │
     ↓
Resume
     │
     ↓
Active
```

**Key Operations:**

```c
// Get runtime PM reference
int intel_runtime_pm_get(struct intel_runtime_pm *pm)
{
    struct drm_i915_private *i915 = pm->i915;
    
    // Increment reference count
    int ret = pm_runtime_get(&i915->drm.dev);
    
    if (ret < 0) {
        // Failed to resume
        return ret;
    }
    
    // Device now active
    return 0;
}

// Release runtime PM reference
void intel_runtime_pm_put(struct intel_runtime_pm *pm)
{
    struct drm_i915_private *i915 = pm->i915;
    
    // Decrement reference count
    pm_runtime_put_autosuspend(&i915->drm.dev);
    
    // May trigger autosuspend if count reaches 0
}
```

---

## Code Flow Examples

### RPS Frequency Update

```c
void intel_rps_boost(struct intel_rps *rps)
{
    struct intel_uncore *uncore = rps->uncore;
    
    // Request high frequency
    rps->cur.freq_req = rps->limits.rp0;
    
    // Write to hardware register
    intel_uncore_write_fw(uncore, GEN6_RPNSWREQ,
                         rps->cur.freq_req << GEN6_RPNSWREQ_SHIFT);
    
    // Signal change
    intel_punit_req_freq_notify(uncore);
}
```

### RC6 Entry Setup

```c
int intel_rc6_init(struct intel_rc6 *rc6)
{
    struct drm_i915_private *i915 = rc6->i915;
    struct intel_uncore *uncore = &i915->uncore;
    
    // Configure RC6 parameters
    intel_uncore_write(uncore, GEN6_RC6_THRESHOLD,
                      rc6->idle_threshold);
    
    // Set RC6 mode
    u32 rc_mode = 0;
    if (rc6->level & INTEL_RC6_ENABLE)
        rc_mode |= GEN6_RC_CTL_RC6_ENABLE;
    if (rc6->level & INTEL_RC6p_ENABLE)
        rc_mode |= GEN6_RC_CTL_RC6p_ENABLE;
        
    intel_uncore_write(uncore, GEN6_RC_CONTROL, rc_mode);
    
    // Enable interrupts
    intel_uncore_write(uncore, GEN6_PMIMR,
                      ~GEN6_PM_RPS_EVENTS);
    
    return 0;
}
```

### GPU Suspend/Resume

```c
int i915_drm_suspend(struct drm_device *dev)
{
    struct drm_i915_private *i915 = dev->dev_private;
    
    // Step 1: Disable interrupts
    i915_irq_uninstall(dev);
    
    // Step 2: Stop RPS/RC6
    intel_rc6_save(&i915->rc6);
    intel_rps_idle(i915);
    
    // Step 3: Save display state
    intel_display_suspend(i915);
    
    // Step 4: Wait for pending GPU work
    intel_gt_wait_for_idle(i915);
    
    // Step 5: Save GPU state
    intel_pm_save_context(i915);
    
    return 0;
}

int i915_drm_resume(struct drm_device *dev)
{
    struct drm_i915_private *i915 = dev->dev_private;
    
    // Step 1: Restore GPU state
    intel_pm_restore_context(i915);
    
    // Step 2: Restore display
    intel_display_resume(i915);
    
    // Step 3: Restart RPS
    intel_rps_enable(&i915->rps);
    
    // Step 4: Restart RC6
    intel_rc6_restore(&i915->rc6);
    
    // Step 5: Re-enable interrupts
    i915_irq_install(dev);
    
    return 0;
}
```

---

## Power Budget & Thermal Management

### Thermal Throttling:

```
Temperature Monitor
        ↓
Temperature Threshold Reached
        ↓
Reduce Maximum Frequency
        ↓
GPU frequency capped
        ↓
Temperature drops
        ↓
Restore normal frequency limits
```

### Power Budget (Intel Xe):

```
Total Available Power
        ↓
    Divided among:
    - Render Engine (40%)
    - Media Engines (30%)
    - Display (20%)
    - Uncore/System (10%)
        ↓
Dynamic allocation based on activity
```

---

## Deep Dive: Power Management State Transitions

### Complete RPS (Frequency Scaling) Workflow

```plantuml
@startuml
title RPS (Render Performance State): Frequency Scaling Flow

participant "GPU Load Monitor" as monitor
participant "RPS Driver" as rps
participant "Frequency Controller" as freq
participant "Power Domain" as pwr

== Monitor Workload ==

monitor -> monitor: Poll/measure GPU load\n(Energy Information counter)

monitor -> rps: Report EI counter update\nevery ~130ms

== Load Analysis ==

rps -> rps: Calculate GPU utilization:\nutil% = ei_delta / time_delta

rps -> rps: Compare with thresholds:
rps -> rps: - Too busy? (util > 90%)
rps -> rps: - Good load? (60-90%)
rps -> rps: - Light load? (< 10%)

== Frequency Decision ==

alt High Load Detected
  rps -> freq: Increase frequency\n(up to RP0/boost)
  freq -> pwr: Adjust voltage\nfor new frequency
  pwr -> pwr: Enable higher freq domain
else Light Load
  rps -> freq: Decrease frequency\n(down to RPn/min)
  freq -> pwr: Lower voltage
  pwr -> pwr: Switch to lower freq domain
else Balanced Load
  rps -> freq: Maintain current frequency
  freq -> pwr: Keep voltage stable
end

== Feedback Loop ==

freq -> monitor: Frequency changed

monitor -> monitor: Continue monitoring\nnew workload

rps -> rps: Schedule next check\n(~130ms timeout)

@enduml
```

### RC6 Power Gating: Sleep & Wake Sequence

```plantuml
@startuml
title RC6 Power Gating: Entry and Exit Flow

participant "GPU Engine" as engine
participant "RC6 Controller" as rc6
participant "Power Domain" as pwr
participant "CPU/Wakeup" as cpu

== Idle Detection ==

engine -> engine: All contexts idle
engine -> engine: No pending work
engine -> rc6: Request power gating\n(RC6 entry)

== RC6 Entry ==

rc6 -> rc6: Prepare for sleep:
rc6 -> rc6: - Save context state
rc6 -> rc6: - Flush caches
rc6 -> rc6: - Set exit handler

rc6 -> pwr: Enter RC6\n(Clock/Power gate)

pwr -> pwr: Disable frequency scaling
pwr -> pwr: Gate clocks to engines
pwr -> pwr: Gate power to components

pwr -> pwr: Power state = RC6\nPower draw ~100mW

== Sleep Duration ==

pwr -> pwr: GPU in RC6\nMinimal power consumption

note right of pwr
Context retained in
on-die memory
No DDR access needed
end note

== Work Arrives ==

cpu -> cpu: New GPU work submitted\nor interrupt triggered

cpu -> rc6: Wakeup request\n(software or HW event)

== RC6 Exit ==

rc6 -> pwr: Exit RC6\nRestore power

pwr -> pwr: Enable clock gates
pwr -> pwr: Restore power domains

pwr -> pwr: Restore voltages\nfor operation

rc6 -> rc6: Restore context state\nfrom on-die memory

rc6 -> engine: Ready for execution

== Resume Execution ==

engine -> engine: Execute pending work

note right of rc6
Exit latency: 1-2ms
Wake time depends on
RC6 level (RC6p/RC6pp
may need DDR refresh)
end note

@enduml
```

### SLPC (Self-managed Low Power Controller) Operation

```plantuml
@startuml
title SLPC: GuC Self-managed Frequency Controller

participant "GPU Load" as load
participant "GuC SLPC" as slpc
participant "Frequency Table" as freq_table
participant "Voltage Regulator" as vreg

== Autonomous Control ==

load -> slpc: GuC monitors load\ndirectly (no CPU IRQ)

slpc -> slpc: Access performance\ncounters directly

slpc -> slpc: Calculate utilization\nwithout CPU help

== Frequency Decision ==

slpc -> slpc: Compare load vs thresholds:
slpc -> slpc: - RP0 (100%): highest freq
slpc -> slpc: - RP1 (95%): high freq
slpc -> slpc: - RP25 (50%): mid freq
slpc -> slpc: - RPe (0%): idle freq

alt High Load
  slpc -> freq_table: Look up high freq\n(RP0/RP1)
  slpc -> slpc: Ramp frequency up\ngradually
else Medium Load
  slpc -> freq_table: Look up mid freq
  slpc -> slpc: Hold at stable freq
else Low Load
  slpc -> freq_table: Look up low freq\n(RPe)
  slpc -> slpc: Power save mode
end

== Voltage/Frequency Adjustment ==

slpc -> vreg: Set frequency & voltage\nfrom lookup table

vreg -> vreg: Adjust PMIC/VDD\nto match frequency

vreg -> slpc: ACK - frequency set

== Continuous Optimization ==

slpc -> slpc: Monitor new load

slpc -> slpc: Feedback loop:\nevery ~1ms in GuC\n(no CPU wake!)

note right of slpc
Key benefit:
GuC adjusts frequency
without waking CPU
from sleep/idle
Autonomously optimizes
power/performance
end note

@enduml
```

### Power Well Hierarchy and Control

```plantuml
@startuml
title Power Well Hierarchy: Domain Dependencies

rectangle "Root Power Well" as root {
  rectangle "Always-On" as aon {
    database "System Agent\n(Always on)" as sa
  }
}

rectangle "Display Power Wells" {
  rectangle "DPLL Well" as dpll {
    database "DPLL0 (Display PLL)" as dpll0
    database "DPLL1" as dpll1
  }
  
  rectangle "Pipe Wells" {
    database "Pipe A Power\n(DDI A/B)" as pipe_a
    database "Pipe B Power\n(DDI C/D)" as pipe_b
  }
}

rectangle "GT (Render) Power Wells" {
  rectangle "GT Core" as gt_core {
    database "Render Engine" as rcs
    database "Blitter Engine" as bcs
  }
  
  rectangle "Media Wells" {
    database "Video Decode" as vcs
    database "Video Enhance" as vecs
  }
  
  rectangle "Slice/Subslice" as ss {
    database "Slice 0\n(variable power)" as s0
    database "Slice 1\n(can be gated)" as s1
  }
}

root --> dpll
root --> gt_core
root --> vcs

dpll --> pipe_a
dpll --> pipe_b

gt_core --> ss

note right of root
All other power wells
dependent on root
Gating root gates entire
GPU
end note

note right of ss
Slice 1 can be powered
down for power saving
depends on workload
end note

@enduml
```

### Runtime PM Device State Machine

```plantuml
@startuml
title Runtime PM: Device Suspend/Resume State Machine

state "ACTIVE" as active
state "AUTOSUSPEND_SCHEDULED" as autosched
state "SUSPENDING" as suspending
state "SUSPENDED" as suspended
state "RESUMING" as resuming

[*] --> active: Device in use

active --> autosched: GPU idle\nAutosuspend timeout\n(e.g., 200ms)

autosched --> suspending: Timeout expires\nor explicit suspend

suspending --> suspended: Suspend callbacks\nRC6 enabled\nClock gated

suspended --> resuming: GPU work arrives\nor explicit resume

resuming --> active: Resume callbacks\nClocks enabled\nRC6 exit

active --> suspending: Explicit suspend\n(e.g., sleep)

suspended --> suspended: Additional idle\n(stays suspended)

note right of suspending
During suspend:
1. Flush pending work
2. Enable RC6
3. Clock gates
4. Unmap GTT
5. Device quiescent
end note

note right of suspended
Power draw: ~1W (from GPU)\n+ interconnect power
All clocks gated
Minimal state retained
end note

@enduml
```

### Frequency Scaling Decision Tree

```plantuml
@startuml
title RPS Frequency Scaling: Adaptive Algorithm

start

:Monitor GPU load\nvia EI counter;

:Calculate utilization\nutil% = (EI_delta/time);

if (util > 90%?) then (yes)
  :High load detected;
  if (current_freq == RP0?) then (yes)
    :Already at max;
    :Hold frequency;
  else (no)
    :Increase frequency\ntoward RP0;
    note right
    Gradual increase
    up to 5 levels
    per adjustment
    end note
  endif
elseif (util > 50%?) then (yes)
  :Medium load;
  if (current_freq > RP1?) then (yes)
    :Decrease frequency\ntoward RP1;
  else (no)
    if (current_freq < RP1?) then (yes)
      :Increase frequency\ntoward RP1;
    else (no)
      :Maintain frequency;
    endif
  endif
else (no - low load)
  :Light load (<50%);
  if (current_freq > RPn?) then (yes)
    :Decrease frequency\ntoward RPn (min);
  else (no)
    :At minimum\nfrequency;
  endif
endif

:Schedule next check\n(~130ms);

:Return to monitoring;

stop

@enduml
```

### Power Budget Distribution

```plantuml
@startuml
title Power Budget Allocation Across GPU Components

rectangle "Total Power Budget (PL1)" {
  rectangle "Dynamic Power" as dyn {
    database "Render Engine\n(40%)" as render
    database "Media Engines\n(30%)" as media
    database "Display\n(20%)" as display
    database "Uncore/System\n(10%)" as uncore
  }
  
  rectangle "Static Power (Leakage)" as static {
    database "Always-on\nsubsystems\n(~5-10%)" as always_on
  }
}

note right of render
Dynamic power depends on:
- Frequency
- Voltage
- Switching activity
end note

note right of display
Display adds significant
power load
especially high refresh
or high resolution
end note

note right of static
Leakage current
proportional to
temperature & voltage
end note

@enduml
```

### Thermal Throttling & Temperature Control

```plantuml
@startuml
title Thermal Throttling: Temperature-based Frequency Capping

participant "Thermal Sensor" as thermal
participant "Throttle Manager" as throttle
participant "RPS" as rps
participant "Frequency Controller" as freq

== Temperature Monitoring ==

thermal -> thermal: Monitor GPU die temperature\nevery ~100ms

thermal -> throttle: Report temperature

== Throttle Decision ==

throttle -> throttle: Check temperature vs thresholds:
throttle -> throttle: - Normal: < 85°C
throttle -> throttle: - Caution: 85-95°C
throttle -> throttle: - Critical: > 95°C

alt Temperature Normal
  throttle -> rps: Allow full frequency range
  rps -> rps: Use standard RPS logic
else Temperature Elevated
  throttle -> rps: Cap maximum frequency\nto RP1 or RPn
  rps -> rps: Limit ceiling\nto cooler operation
else Temperature Critical
  throttle -> throttle: Trigger emergency\nfrequency reduction
  throttle -> freq: Set frequency to minimum\nto cool down
  freq -> freq: Ramp down to RPe\n(idle/emergency)
  
  note right of freq
  Prevent thermal damage
  prioritize cooling
  over performance
  end note
end

== Cooling ==

freq -> thermal: Thermal load decreases\nAs frequency drops

thermal -> thermal: Temperature drops\n(next sample)

thermal -> throttle: Report lower temperature

throttle -> rps: Remove throttle\nRestore normal operation

rps -> rps: Frequency scaling\nresumes normally

@enduml
```

---

## Sysfs Interface

### Power Management Parameters:

```bash
# Check current frequency
cat /sys/class/drm/card0/device/gt/0/freq_cur_mhz

# Check min/max frequency
cat /sys/class/drm/card0/device/gt/0/freq_min_mhz
cat /sys/class/drm/card0/device/gt/0/freq_max_mhz

# Check RC6 state
cat /sys/class/drm/card0/device/power_state

# Set boost mode
echo 1 > /sys/class/drm/card0/device/boost_enabled

# Runtime PM control
echo auto > /sys/bus/pci/devices/0000:00:02.0/power/control
```

---

## Performance Considerations

### 1. **Latency vs. Efficiency**
- Higher frequency = lower latency, higher power
- Lower frequency = more power efficient, higher latency
- RPS balances this dynamically

### 2. **RC6 Wake Latency**
- RC6 entry: ~100μs
- RC6pp exit: ~1-2ms
- Track operations sensitive to latency

### 3. **SLPC Efficiency**
- Reduces interrupt overhead
- Faster frequency response
- Generally preferred when available

---

## Related Components

- **Interrupt Handling:** `i915_irq.c` - RPS interrupt processing
- **Display Power:** `display/intel_display_power.c` - Display power wells
- **Uncore:** `intel_uncore.c` - Low-level hardware access
- **Telemetry:** `i915_pmu.c` - Power metrics monitoring

---

## References

- **Source:** `intel_pm.c`, `intel_runtime_pm.c`, `gt/intel_rps.c`, `gt/intel_rc6.c`
- **Headers:** `intel_pm_types.h`, `intel_runtime_pm.h`
- **Selftests:** `gt/selftest_rc6.c`, `gt/selftest_engine_pm.c`
- **Documentation:** `Documentation/gpu/i915.rst#pm`
