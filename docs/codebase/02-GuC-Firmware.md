# Graphics Unified Controller (GuC) Firmware System

## Overview

The **GuC (Graphics Unified Controller)** is a microcontroller embedded in modern Intel GPUs that handles GPU command submission, scheduling, and workload management. It provides a more efficient alternative to the older execlists submission mechanism and reduces CPU interrupt overhead.

**Key Responsibilities:**
- Firmware loading and initialization
- Work queue (command buffer) management
- Bi-directional communication with CPU driver
- Workload scheduling and context switching
- Error handling and recovery

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│           User Application / Kernel Driver          │
│            (i915 kernel module)                     │
└────────────────────┬────────────────────────────────┘
                     ↓
        ┌────────────────────────────┐
        │   intel_guc_submission.c   │
        │  (GuC Submission Backend)  │
        └────────────┬───────────────┘
                     ↓
        ┌────────────────────────────┐
        │    intel_guc_ct.c          │
        │ (Communication Transport)  │
        │  Bi-directional Protocol   │
        └────────────┬───────────────┘
                     ↓
        ┌────────────────────────────┐
        │  intel_guc.c / intel_guc.h │
        │  (GuC Main Driver)         │
        │  - Firmware loading        │
        │  - Initialization          │
        │  - State management        │
        └────────────┬───────────────┘
                     ↓
        ┌────────────────────────────┐
        │  GPU Hardware Registers    │
        │  (GuC Doorbell, Queues)    │
        └────────────┬───────────────┘
                     ↓
            ┌────────────────────┐
            │  GuC Microcontroller│
            │  (Hardware Firmware)│
            └────────────┬───────┘
                         ↓
                GPU Execution Engines
```

---

## Core Components

### 1. **GuC Main Driver (intel_guc.c)**

**Purpose:** Manages GuC firmware lifecycle and overall GuC state

**Key Structures:**
```c
struct intel_guc {
    struct intel_uc *uc;                 // Parent UC (microcontroller)
    struct intel_uc_fw fw;               // Firmware management
    
    struct intel_guc_ct ct;              // Communication transport
    struct intel_guc_log log;            // Logging system
    struct intel_guc_ads ads;            // Address descriptor
    
    struct intel_guc_slpc slpc;          // Self-managed Performance Scaling
    
    /* Work queue (submission queues) */
    struct i915_guc_client *execbuf_client;
    struct guc_wq_item *wq;
    u32 wq_tail;
    
    /* State tracking */
    unsigned long flags;
    #define GUC_INIT_IN_PROGRESS  0
    #define GUC_LOADED            1
    #define GUC_RUN_PARAM_FAILED  2
    #define GUC_CT_ENABLED        3
    
    /* Error tracking */
    struct {
        u32 last_error;
        struct work_struct error_work;
    } error;
};
```

**GuC Initialization Flow:**
```
┌──────────────────────────────────┐
│  GuC Firmware Loading            │
│  (intel_guc_fw_load)             │
└────────────┬─────────────────────┘
             ↓
    ┌────────────────────────────┐
    │ Validate firmware binary   │
    │ - Check version            │
    │ - Verify signature         │
    └────────────┬───────────────┘
                 ↓
    ┌────────────────────────────┐
    │ Allocate GuC data memory   │
    │ (ADS - Address Descriptor) │
    └────────────┬───────────────┘
                 ↓
    ┌────────────────────────────┐
    │ Load firmware to WOPCM     │
    │ (Writes Program Counter)   │
    └────────────┬───────────────┘
                 ↓
    ┌────────────────────────────┐
    │ Enable GuC                 │
    │ (Set run bit)              │
    └────────────┬───────────────┘
                 ↓
    ┌────────────────────────────┐
    │ Wait for handshake         │
    │ (GuC → CPU mailbox)        │
    └────────────┬───────────────┘
                 ↓
    ┌────────────────────────────┐
    │ Initialize GuC CT (if used)│
    │ Setup communication channel│
    └────────────┬───────────────┘
                 ↓
    ┌────────────────────────────┐
    │ GuC Ready for operation    │
    └────────────────────────────┘
```

---

### 2. **Communication Transport (intel_guc_ct.c)**

**Purpose:** Implements bi-directional communication protocol between CPU driver and GuC firmware

**Protocol Overview:**
```
CPU Driver ←→ Doorbell Registers ←→ GuC Firmware
                    ↓
            Command/Response Buffers
                    ↓
            Shared Memory Queues
```

**Key Structures:**
```c
struct intel_guc_ct {
    struct i915_vma *vma;              // Virtual memory for buffers
    
    struct guc_ct_buffer_desc *ctbs;   // CT buffer descriptors
    
    /* Command queue (CPU → GuC) */
    struct {
        u32 head;
        u32 tail;
        void *cmds;                    // Command ring buffer
    } cmd;
    
    /* Response queue (GuC → CPU) */
    struct {
        u32 head;
        u32 tail;
        void *cmds;                    // Response ring buffer
    } rsp;
    
    /* Pending request tracking */
    struct {
        struct list_head pending;
        spinlock_t lock;
        u32 next_id;
    } requests;
    
    /* Tasklet for processing responses */
    struct tasklet_struct tasklet;
    struct work_struct worker;
};
```

**Communication Protocol Flow:**

```
CPU Driver (Kernel)                GuC Firmware (Microcontroller)
        │                                    │
        │  1. Prepare command buffer         │
        ├─ Write CT command (4 DWORDs)       │
        │                                    │
        │  2. Send doorbell                  │
        ├───────────────────────────────────→│
        │                                    │
        │                        3. Firmware reads command
        │                        4. Execute command
        │                        5. Write response to response queue
        │                                    │
        │  6. Firmware asserts interrupt    │
        │←───────────────────────────────────┤
        │                                    │
        │  7. IRQ handler wakes tasklet     │
        │  8. Read response from queue      │
        │  9. Notify waiting sender         │
        │                                    │
```

**Command Format (CT Command):**
```c
struct guc_ct_header {
    u32 type : 4;           // Command type
    u32 action : 16;        // Specific action
    u32 reserved : 4;       
    u32 dlen : 8;           // Data length (DWORDs)
};

// Full command = header + data DWORDs
// Total size = header (1 DWORD) + data (dlen DWORDs)
```

---

### 3. **Work Queue & Submission (intel_guc_submission.c)**

**Purpose:** Implements GPU command submission via GuC work queues

**Submission Mechanism:**

```c
struct i915_guc_client {
    struct intel_guc *guc;
    
    /* Work queue (command buffer) */
    struct i915_vma *vma;
    struct guc_wq_item *wq;            // Work queue items
    u32 wq_head;
    u32 wq_tail;
    u32 wq_size;
    
    /* Queue state */
    struct {
        u32 doorbell_id;               // Assigned doorbell
        u64 doorbell_offset;           // Register offset
    } guc_id;
    
    /* Context tracking */
    struct list_head active_contexts;
    
    spinlock_t lock;
};
```

**Work Queue Item:**
```c
struct guc_wq_item {
    u32 header;
    u32 context_desc;                  // Context reference
    u32 batch_addr_lo;                 // Batch buffer address (low)
    u32 batch_addr_hi;                 // Batch buffer address (high)
    u32 fence_id;                      // Fence identifier
    u32 reserved;
};
```

**Submission Code Flow:**
```c
// Submit work to GuC queue
int intel_guc_submit_request(struct i915_request *rq)
{
    struct intel_guc *guc = rq->engine->gt->uc.guc;
    
    // Step 1: Prepare work queue item
    struct guc_wq_item *wqi = &guc->wq[guc->wq_tail];
    wqi->context_desc = rq->context->guc_id;
    wqi->batch_addr_lo = lower_32_bits(rq->batch_addr);
    wqi->batch_addr_hi = upper_32_bits(rq->batch_addr);
    
    // Step 2: Update work queue tail
    guc->wq_tail = (guc->wq_tail + 1) % guc->wq_size;
    
    // Step 3: Write work queue item to memory (ordering barrier)
    wmb();
    
    // Step 4: Ring doorbell to notify GuC
    intel_guc_notify(guc);
    
    return 0;
}
```

---

### 4. **GuC Logging (intel_guc_log.c)**

**Purpose:** Capture GuC firmware debug logs

**Logging Mechanism:**
```
GuC Firmware writes logs
        ↓
Shared Log Buffer
        ↓
CPU reads when full
        ↓
Parse and store in ring buffer
        ↓
Export via debugfs
```

**Key Structures:**
```c
struct intel_guc_log {
    struct i915_vma *vma;              // Shared log buffer
    struct intel_uncore *uncore;
    
    /* Logging state */
    struct {
        u32 sampled_tail;
        u32 flush_count;
    } stats;
    
    /* Log levels */
    u32 level;                         // Verbosity level
    
    /* Ring buffer for storing logs */
    struct circ_buf relayed_logs;
};
```

---

### 5. **Address Descriptor Structure (intel_guc_ads.c)**

**Purpose:** Provides GuC firmware with essential driver data structures

**ADS Contents:**
```c
struct guc_ads {
    struct guc_ads_engine_usage engine_usage;
    struct guc_ads_payload payload;
    
    u32 golden_context_lrca;           // Golden context reference
    u32 uc_flag;
    
    struct guc_ads_rlc_state {
        u32 head;
        u32 tail;
        u32 status;
    } rlc_state;
    
    /* ... more fields */
};
```

**ADS Update Flow:**
```
Driver modifies scheduling parameters
        ↓
Update ADS structure in shared memory
        ↓
Notify GuC via CT message
        ↓
GuC reads updated ADS
        ↓
Apply new parameters
```

---

## Code Flow Examples

### GuC Request Submission (Simplified Path)

```
User Executes Batch
        ↓
i915_gem_execbuffer_ioctl()
        ↓
i915_request_create()
        ↓
intel_engine_cs_prepare_request()
        ↓
[GuC Submission Path]
        ↓
intel_guc_submit_request()
        ├─ Prepare work queue item
        ├─ Add to work queue
        ├─ Ring doorbell
        └─ Return
        ↓
GuC Firmware Processes
        ├─ Fetch from work queue
        ├─ Parse work queue item
        ├─ Load context
        └─ Submit to engine
        ↓
Engine Executes Batch
        ↓
Interrupt on completion
        ↓
CPU Notifies Application
```

### GuC Communication Flow (CT Message)

```c
int intel_guc_send(struct intel_guc *guc, u32 *action, u32 len)
{
    struct intel_guc_ct *ct = &guc->ct;
    
    // Step 1: Find space in command queue
    u32 head = READ_ONCE(ct->cmd.head);
    u32 space = available_space(ct->cmd.tail, head);
    
    if (space < len)
        return -EBUSY;  // Queue full
    
    // Step 2: Copy command to queue
    u32 *cmd = (u32 *)ct->cmd.cmds + ct->cmd.tail;
    memcpy(cmd, action, len * sizeof(u32));
    
    // Step 3: Update tail pointer
    ct->cmd.tail = (ct->cmd.tail + len) % ct->cmd_size;
    
    // Step 4: Memory barrier to ensure writes visible to GuC
    wmb();
    
    // Step 5: Notify GuC via doorbell
    intel_guc_notify(guc);
    
    // Step 6: Wait for response (or timeout)
    return wait_for_response(ct, timeout);
}
```

---

## Key GuC Features

### 1. **Self-Managed Performance Scaling (SLPC)**
- GuC autonomously adjusts GPU frequency based on workload
- Reduces CPU overhead vs. driver-based RPS
- Configurable frequency targets via CT messages

### 2. **Context Scheduling**
- GuC handles context switching on engine
- Preemption via GuC command
- Priority-based scheduling

### 3. **Doorbell Mechanism**
```
Work Queue → Doorbell Register (Memory-Mapped)
       ↓
    Ring doorbell (write register)
       ↓
    GuC Firmware Interrupt
       ↓
    Process work queue items
```

### 4. **Error Recovery**
```
GuC detects error
        ↓
Sets error bit in status
        ↓
Sends error notification to CPU
        ↓
CPU driver triggers reset
```

---

## GuC vs. Execlists: Comparison

| Feature | GuC Submission | Execlists |
|---------|--------------|-----------|
| Submission Method | Work Queue Doorbell | CPU-driven submission |
| CPU Overhead | Low (GuC handles scheduling) | High (CPU submits each batch) |
| IRQ Frequency | Lower | Higher |
| Complexity | More complex (firmware) | Simpler (SW scheduling) |
| Preemption | GuC-controlled | CPU-controlled |
| Power Efficiency | Better | Baseline |
| Hardware Support | Modern GPUs (Gen11+) | All generations |

---

## Initialization & Configuration

### GuC Enable Sequence:

```
1. Check firmware availability
2. Load firmware binary from filesystem
3. Initialize GuC memory structures (ADS)
4. Upload firmware to WOPCM
5. Set run bit to start GuC
6. Wait for handshake
7. Initialize CT communication (if enabled)
8. Load HUC if present
9. Ready for submission
```

### Module Parameters:
```bash
# Enable GuC submission (vs. execlists)
# i915.enable_guc=3          # Enable GuC + HUC
# i915.guc_log_level=X       # Firmware log verbosity
```

---

## Performance Optimizations

### 1. **Batch Submission Optimization**
- Submit multiple requests per doorbell ring
- Reduces context switches
- Amortizes doorbell overhead

### 2. **SLPC Benefits**
- Eliminates CPU overhead of RPS
- Tighter feedback loop (GuC sees load directly)
- Lower power consumption

### 3. **Interrupt Reduction**
- GuC signals completion via CT messages
- Batches notifications
- Reduces CPU interrupt latency

---

## Debugging & Troubleshooting

### Debugfs Interface:
```bash
# Check GuC status
cat /sys/kernel/debug/dri/0/guc_info

# View GuC logs
cat /sys/kernel/debug/dri/0/guc_log

# Check submission stats
cat /sys/kernel/debug/dri/0/guc_stats
```

### Common Issues:

| Issue | Cause | Solution |
|-------|-------|----------|
| GuC load failure | Missing/corrupt firmware | Update kernel/firmware |
| CT timeout | GuC hung | Check logs, may need reset |
| Submission stalls | Work queue full | Reduce batch size |
| High latency | CT message delays | Enable GuC logging |

---

## Related Components

- **Firmware Loading:** `intel_uc_fw.c` - General UC firmware framework
- **HUC:** `intel_huc.c` - Hardware Utility Controller (video encode)
- **Communication:** `gt/uc/abi/` - CT message definitions
- **Scheduling:** `i915_scheduler.c` - CPU-side scheduling

---

## References

- **Source:** `drivers/gpu/drm/i915/gt/uc/`
- **Headers:** `intel_guc.h`, `intel_guc_fwif.h`
- **Selftests:** `selftest_guc.c`, `selftest_doorbells.c`
- **Documentation:** GuC firmware ABI in `abi/guc_communication_mmio.h`
- **Kernel Docs:** `Documentation/gpu/i915.rst#guc`
