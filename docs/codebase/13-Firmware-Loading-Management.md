# Firmware Loading and Management

**Document ID:** 13 | **Status:** Complete | **Last Updated:** February 2026

---

## Executive Summary

Modern i915 GPUs require firmware for GuC (Graphics Unified Compute) and HuC (HEVC Unified Codec). This document explains how i915 loads, verifies, and manages GPU firmware, including error handling and runtime management.

### Key Topics
- **Firmware sources:** Kernel releases, external files, version management
- **Loading mechanisms:** DMA transfer, initial handshake, capability negotiation
- **Verification:** Signature validation, checksum verification, compatibility checks
- **Runtime management:** Firmware status monitoring, recovery mechanisms

### Performance Impact
- **Load time:** 50-200ms for firmware initialization
- **Memory overhead:** 2-5MB per firmware image
- **Recovery time:** 100-500ms if firmware recovery needed

---

## Table of Contents

1. [Firmware Architecture](#firmware-architecture)
2. [Firmware Sources and Distribution](#firmware-sources-and-distribution)
3. [Firmware Loading Process](#firmware-loading-process)
4. [Signature Verification](#signature-verification)
5. [GuC Initialization and Handshake](#guc-initialization-and-handshake)
6. [HuC Codec Engine Setup](#huc-codec-engine-setup)
7. [Firmware Error Recovery](#firmware-error-recovery)
8. [Monitoring and Debugging](#monitoring-and-debugging)
9. [Summary & Best Practices](#summary--best-practices)

---

## Firmware Architecture

### GuC and HuC Overview

```plaintext
GPU Firmware Stack
┌─────────────────────────────────────┐
│ HuC (HEVC Unified Codec)            │
│ - Handles video decoding            │
│ - Hardware accelerated codec engine │
│ - Loads after GuC                   │
└─────────────┬───────────────────────┘
              │
┌─────────────▼───────────────────────┐
│ GuC (Graphics Unified Compute)      │
│ - Schedules GPU work                │
│ - Manages contexts                  │
│ - Loads first, required base        │
└─────────────┬───────────────────────┘
              │
┌─────────────▼───────────────────────┐
│ Kernel Driver (i915)                │
│ - Loads firmware from filesystem    │
│ - Initializes GPU                   │
│ - Communicates with firmware        │
└─────────────────────────────────────┘
```

### Firmware File Structure

```c
// GuC firmware file format
struct uc_fw_header {
    u32 header_size;           // Bytes before binary
    u32 header_version;        // Format version
    u32 binary_size;           // Microcode binary size
    u32 major_version;         // GuC version major
    u32 minor_version;         // GuC version minor
    u32 patch_version;         // GuC version patch
    u32 build_version;         // Build number
    u32 flags;                 // Capabilities/features
    u32 checksum;              // Data integrity
    u32 xvt_version;           // XVT version
};

// Data structure in firmware file
struct uc_fw {
    enum uc_fw_type type;      // GuC or HuC
    const char *path;          // Filesystem path
    const struct firmware *fw; // Loaded firmware
    size_t size;               // File size
    u32 header_size;
    u32 header_version;
    u32 major_version;
    u32 minor_version;
    u32 patch_version;
    u32 build_version;
};
```

---

## Firmware Sources and Distribution

### Firmware Search Paths

```plaintext
Firmware Loading Search Order
1. Kernel firmware directory
   └─ /lib/firmware/i915/<cpu_type>/
      ├─ guc_<version>.bin
      └─ huc_<version>.bin

2. Staging firmware
   └─ /lib/firmware/i915/staging/
      └─ [Pre-release versions]

3. Fallback locations
   └─ /lib/firmware/
      └─ [Legacy locations for older kernels]
```

### Firmware Version Management

```c
// Firmware version definitions by generation
#define GEN8_GUC_FW_VERSION     "8.90"
#define GEN9_GUC_FW_VERSION     "9.0"
#define GEN10_GUC_FW_VERSION    "10.0"
#define GEN12_GUC_FW_VERSION    "49.0"
#define DG1_GUC_FW_VERSION      "49.0"
#define TGL_HUC_FW_VERSION      "7.0"

// Firmware path construction
static const char *get_firmware_path(struct i915_drm_private *i915,
                                      enum uc_fw_type type)
{
    const char *cpu = i915->platform.cpu_name;
    static char path[PATH_MAX];
    
    switch (type) {
    case INTEL_UC_FW_TYPE_GUC:
        snprintf(path, sizeof(path),
                 "i915/%s/guc_%s.bin",
                 cpu, GUC_FW_VERSION);
        break;
    
    case INTEL_UC_FW_TYPE_HUC:
        snprintf(path, sizeof(path),
                 "i915/%s/huc_%s.bin",
                 cpu, HUC_FW_VERSION);
        break;
    
    default:
        return NULL;
    }
    
    return path;
}
```

---

## Firmware Loading Process

### Overall Loading Flow

```plaintext
┌──────────────────────────────────┐
│ Driver Initialization            │
│ (i915_driver_load)               │
└──────┬───────────────────────────┘
       │
       ├─→ Request firmware from kernel
       │   └─ request_firmware(&fw, path, dev)
       │
       ├─→ Validate firmware header
       │   ├─ Check header size
       │   ├─ Verify version compatibility
       │   └─ Validate checksum
       │
       ├─→ Allocate GPU memory for firmware
       │   └─ Create GEM object for binary
       │
       ├─→ DMA transfer to GPU memory
       │   ├─ GPU reads from system RAM
       │   └─ Verify checksum at destination
       │
       ├─→ Initialize GuC
       │   ├─ Reset GuC microprocessor
       │   ├─ Set entry point address
       │   ├─ Release from reset
       │   ├─ Wait for microcode load
       │   └─ Handshake to confirm ready
       │
       ├─→ Negotiate capabilities
       │   ├─ Kernel tells GuC what features needed
       │   ├─ GuC reports capabilities
       │   └─ Both agree on feature set
       │
       └─→ Continue to HuC (if supported)
           └─ Similar process for HuC
```

### Loading Firmware Binary

```c
// Load firmware file into GPU memory
static int uc_fw_load_binary(struct intel_uc_fw *uc_fw,
                             struct i915_drm_private *i915)
{
    struct intel_guc *guc = &i915->gt.uc.guc;
    struct i915_vma *vma;
    const struct firmware *fw;
    void *src, *dst;
    size_t size;
    int ret = 0;
    
    // 1. Get firmware file from kernel
    ret = request_firmware(&fw, uc_fw->path, i915->drm.dev);
    if (ret) {
        dev_err(i915->drm.dev,
                "Failed to load %s firmware\n",
                uc_fw->path);
        return ret;
    }
    
    uc_fw->fw = fw;
    size = fw->size;
    
    // 2. Create GPU memory for firmware
    vma = i915_gem_object_create_shmem(i915, size);
    if (IS_ERR(vma)) {
        ret = PTR_ERR(vma);
        goto release;
    }
    
    // 3. Map kernel and GPU memory
    src = (void *)fw->data;
    dst = i915_vma_pin_iomap(vma);
    if (IS_ERR(dst)) {
        ret = PTR_ERR(dst);
        goto unpin;
    }
    
    // 4. Copy firmware to GPU memory
    memcpy_toio(dst, src, size);
    
    // 5. Store vma for later use
    uc_fw->vma = vma;
    
    i915_vma_unpin_iomap(vma);
    
release:
    release_firmware(fw);
    return ret;

unpin:
    i915_vma_unpin_iomap(vma);
    return ret;
}
```

---

## Signature Verification

### Firmware Signature Validation

```plaintext
Firmware File Layout
┌─────────────────────────┐
│ RSA Signature (256 bytes)│  ← Driver verifies
├─────────────────────────┤
│ Header (100+ bytes)      │  ← Version, build, etc
├─────────────────────────┤
│ Microcode Binary         │  ← Loaded to GPU
│ (Variable size)          │
├─────────────────────────┤
│ Build Info String        │
└─────────────────────────┘
```

### Signature Checking

```c
// Verify firmware RSA signature
static int uc_fw_verify_signature(struct intel_uc_fw *uc_fw,
                                  const struct firmware *fw)
{
    struct key *keyring;
    int ret;
    
    // 1. Get kernel keyring with public keys
    keyring = get_builtin_keyring("i915_gpu_firmware");
    if (!keyring) {
        dev_warn(uc_fw->drm->dev,
                 "No i915 firmware keyring found\n");
        return 0;  // Skip signature check if no keys
    }
    
    // 2. Verify signature on firmware binary
    ret = verify_pkcs7_signature(fw->data,
                                fw->size,
                                fw->data + SIGNATURE_OFFSET,
                                SIGNATURE_SIZE,
                                keyring,
                                VERIFYING_UNSPECIFIED_SIGNATURE,
                                NULL, NULL);
    
    if (ret) {
        dev_err(uc_fw->drm->dev,
                "Failed to verify %s firmware signature\n",
                uc_fw->path);
        return -ENOEXEC;
    }
    
    dev_dbg(uc_fw->drm->dev,
            "%s firmware signature verified\n",
            uc_fw->path);
    
    return 0;
}

// Validate header and version compatibility
static int uc_fw_validate_header(struct intel_uc_fw *uc_fw,
                                 const struct firmware *fw)
{
    struct uc_fw_header *header;
    u32 version, compatible_version;
    
    if (fw->size < sizeof(*header)) {
        return -ENODATA;
    }
    
    header = (struct uc_fw_header *)fw->data;
    
    // 1. Check header version
    if (header->header_version != EXPECTED_HEADER_VERSION) {
        dev_err(uc_fw->drm->dev,
                "Incompatible header version %u\n",
                header->header_version);
        return -EINVAL;
    }
    
    // 2. Check firmware version compatibility
    version = header->major_version;
    compatible_version = get_compatible_guc_version(uc_fw->drm);
    
    if (version < compatible_version) {
        dev_err(uc_fw->drm->dev,
                "GuC firmware too old: %u.%u.%u (need >= %u)\n",
                header->major_version,
                header->minor_version,
                header->patch_version,
                compatible_version);
        return -EINVAL;
    }
    
    // 3. Validate checksum
    if (!uc_fw_validate_checksum(header, fw->data)) {
        dev_err(uc_fw->drm->dev, "Bad firmware checksum\n");
        return -EBADMSG;
    }
    
    return 0;
}
```

---

## GuC Initialization and Handshake

### GuC Reset and Wake Sequence

```plaintext
GuC Initialization Sequence
┌────────────────────────────┐
│ 1. Reset GuC               │
│    - Hold in reset         │
│    - Clear all state       │
└────────┬───────────────────┘
         │
         ├─→ 2. Configure MMIO Registers
         │   - Set interrupt masks
         │   - Configure memory base
         │   - Setup command queue
         │
         ├─→ 3. Load Firmware
         │   - DMA copy binary
         │   - Verify checksum
         │
         ├─→ 4. Release from Reset
         │   - Allow microprocessor to execute
         │   - Wait for boot notification
         │
         ├─→ 5. Handshake Protocol
         │   - GuC sends HELLO message
         │   - Driver sends features request
         │   - GuC confirms ready
         │
         └─→ 6. Verify Readiness
             - Check GuC status register
             - Confirm communication works
```

### GuC Reset Implementation

```c
// Reset and initialize GuC
static int guc_reset(struct intel_guc *guc)
{
    struct intel_uncore *uncore = guc->uncore;
    u32 guc_status;
    int ret;
    
    // 1. Force GuC into reset
    intel_uncore_rmw(uncore, GUC_CTRL,
                     GUC_CTRL_RUN, 0);
    
    // Wait for GuC to acknowledge reset
    ret = intel_wait_for_register(uncore,
                                  GUC_STATUS,
                                  GUC_STATUS_MIA,
                                  GUC_STATUS_MIA,
                                  100);
    if (ret) {
        dev_err(guc_to_i915(guc)->drm.dev,
                "GuC MIA after reset\n");
        return ret;
    }
    
    // 2. Program GuC memory location
    intel_uncore_write(uncore, GUC_WOPCM_SIZE,
                       guc->fw.vma->size);
    intel_uncore_write(uncore, GUC_WOPCM_OFFSET,
                       guc->fw.vma->node.start);
    
    // 3. Program GuC initial instruction pointer
    intel_uncore_write(uncore, GUC_INST_RST_FAILSAFE, 0);
    intel_uncore_write(uncore, GUC_BASE_ADDR,
                       guc->fw.vma->node.start);
    
    // 4. Release from reset
    intel_uncore_rmw(uncore, GUC_CTRL,
                     0, GUC_CTRL_RUN);
    
    // 5. Wait for GuC to come out of reset
    ret = intel_wait_for_register(uncore,
                                  GUC_STATUS,
                                  GUC_STATUS_MIA,
                                  0,  // Should clear MIA bit
                                  200);
    
    if (ret) {
        guc_status = intel_uncore_read(uncore, GUC_STATUS);
        dev_err(guc_to_i915(guc)->drm.dev,
                "GuC failed to come out of reset: 0x%x\n",
                guc_status);
        return ret;
    }
    
    dev_info(guc_to_i915(guc)->drm.dev,
             "GuC reset complete\n");
    
    return 0;
}
```

### Handshake Protocol

```c
// GuC handshake - exchange capabilities
static int guc_init_handshake(struct intel_guc *guc)
{
    struct intel_guc_msg_hdr *msg;
    u32 features_to_request;
    int ret;
    
    // 1. Wait for GuC to signal readiness
    ret = guc_wait_for_hello(guc);
    if (ret) {
        dev_err(guc_to_i915(guc)->drm.dev,
                "GuC did not respond to HELLO\n");
        return ret;
    }
    
    dev_dbg(guc_to_i915(guc)->drm.dev,
            "GuC HELLO received\n");
    
    // 2. Send feature request to GuC
    features_to_request = 0;
    
    if (INTEL_UC_FEATURE_GUC_SUBMISSION)
        features_to_request |= GUC_FEATURE_SUBMISSION;
    
    if (INTEL_UC_FEATURE_GUC_PREEMPTION)
        features_to_request |= GUC_FEATURE_PREEMPTION;
    
    if (INTEL_UC_FEATURE_GUC_SLPC)
        features_to_request |= GUC_FEATURE_SLPC;
    
    // Build and send feature request message
    msg = guc_alloc_message(sizeof(*msg) + 
                           sizeof(features_to_request));
    msg->action = GUC_ACTION_REQUEST_FEATURE;
    *(u32 *)(msg + 1) = features_to_request;
    
    ret = guc_send_message(guc, msg);
    if (ret) {
        dev_err(guc_to_i915(guc)->drm.dev,
                "Failed to send feature request\n");
        return ret;
    }
    
    // 3. Wait for feature response
    ret = guc_wait_for_response(guc);
    if (ret) {
        dev_err(guc_to_i915(guc)->drm.dev,
                "GuC did not respond to feature request\n");
        return ret;
    }
    
    dev_dbg(guc_to_i915(guc)->drm.dev,
            "GuC handshake complete\n");
    
    return 0;
}
```

---

## HuC Codec Engine Setup

### HuC Loading

```c
// Load and initialize HuC (HEVC codec)
static int huc_init(struct intel_huc *huc)
{
    struct intel_uc_fw *fw = &huc->fw;
    int ret;
    
    // 1. Load firmware
    ret = uc_fw_load_binary(fw, huc_to_i915(huc));
    if (ret) {
        dev_warn(huc_to_i915(huc)->drm.dev,
                 "HuC firmware load failed\n");
        return ret;  // HuC is optional
    }
    
    // 2. Verify signature
    ret = uc_fw_verify_signature(fw, fw->fw);
    if (ret) {
        dev_err(huc_to_i915(huc)->drm.dev,
                "HuC signature verification failed\n");
        return ret;
    }
    
    // 3. Wait for GuC to authenticate HuC
    // (GuC must load and authenticate HuC firmware)
    ret = guc_huc_authenticate(huc);
    if (ret) {
        dev_err(huc_to_i915(huc)->drm.dev,
                "HuC authentication failed\n");
        return ret;
    }
    
    dev_info(huc_to_i915(huc)->drm.dev,
             "HuC loaded and authenticated\n");
    
    return 0;
}
```

---

## Firmware Error Recovery

### Firmware Load Failure Handling

```plaintext
Firmware Load Failure Recovery
┌──────────────────────────────┐
│ Firmware load fails          │
│ (file not found or corrupted)│
└──────┬───────────────────────┘
       │
       ├─ GuC required?
       │  ├─ YES → Fatal error
       │  │   └─ Abort driver load
       │  │
       │  └─ NO → Continue without firmware
       │      └─ Proceed with legacy path
       │
       └─ HuC required?
           ├─ YES → Fatal error
           │   └─ Abort HuC features
           │
           └─ NO → Continue without HuC
               └─ Video decode disabled
```

### Retry Mechanism

```c
// Retry firmware load with exponential backoff
static int uc_fw_load_with_retry(struct intel_uc_fw *fw,
                                 struct i915_drm_private *i915)
{
    int max_retries = 3;
    int retry_delay_ms = 100;
    int ret;
    int i;
    
    for (i = 0; i < max_retries; i++) {
        ret = uc_fw_load_binary(fw, i915);
        
        if (ret == 0) {
            dev_info(i915->drm.dev,
                    "Firmware loaded successfully (attempt %d/%d)\n",
                    i + 1, max_retries);
            return 0;
        }
        
        if (ret == -ENOENT) {
            // File not found - don't retry
            dev_warn(i915->drm.dev,
                    "Firmware file not found: %s\n",
                    fw->path);
            return ret;
        }
        
        if (i < max_retries - 1) {
            dev_warn(i915->drm.dev,
                    "Firmware load failed (attempt %d/%d), "
                    "retrying in %dms\n",
                    i + 1, max_retries, retry_delay_ms);
            msleep(retry_delay_ms);
            retry_delay_ms *= 2;  // Exponential backoff
        }
    }
    
    dev_err(i915->drm.dev,
            "Firmware load failed after %d attempts\n",
            max_retries);
    
    return ret;
}
```

### Firmware Corruption Detection

```c
// Detect and recover from firmware corruption
static int guc_verify_loaded(struct intel_guc *guc)
{
    struct intel_uncore *uncore = guc->uncore;
    u32 status, checksum;
    int ret;
    
    // 1. Check GuC reports as loaded
    status = intel_uncore_read(uncore, GUC_STATUS);
    
    if (!(status & GUC_STATUS_LOADED)) {
        dev_err(guc_to_i915(guc)->drm.dev,
                "GuC firmware not marked as loaded\n");
        return -EIO;
    }
    
    // 2. Verify firmware checksum at GPU location
    checksum = compute_fw_checksum(guc->fw.vma);
    
    if (checksum != guc->fw.checksum) {
        dev_err(guc_to_i915(guc)->drm.dev,
                "GuC firmware corrupted in GPU memory\n");
        
        // Try reloading firmware
        ret = guc_reset(guc);
        if (ret) {
            return ret;
        }
        
        ret = uc_fw_load_binary(&guc->fw, 
                                guc_to_i915(guc));
        if (ret) {
            return ret;
        }
    }
    
    return 0;
}
```

---

## Monitoring and Debugging

### Firmware Status Monitoring

```bash
# Check GuC firmware status
cat /sys/kernel/debug/dri/0/i915_guc_info

# Monitor GuC-HuC version
cat /sys/kernel/debug/dri/0/i915_guc_version

# Check HuC loaded status
cat /sys/kernel/debug/dri/0/i915_huc_info
```

### Debug Output

```c
// Log firmware loading
#define FW_DEBUG(uc_fw, fmt, ...)                   \
    do {                                            \
        const char *name;                           \
        if ((uc_fw)->type == INTEL_UC_FW_TYPE_GUC) \
            name = "GuC";                           \
        else                                        \
            name = "HuC";                           \
        pr_debug("i915[%s]: " fmt,                 \
                 name, ##__VA_ARGS__);             \
    } while (0)

// Usage
FW_DEBUG(&guc->fw, "Loading GuC version %u.%u.%u\n",
         major, minor, patch);
```

### Firmware Compatibility Information

| Generation | GuC Version | HuC Version | Notes |
|-----------|------------|------------|-------|
| **Gen8 (BDW)** | 8.x | N/A | GuC submission optional |
| **Gen9 (SKL)** | 9.x | N/A | GuC submission optional |
| **Gen10 (CNL)** | 10.x | N/A | GuC submission optional |
| **Gen11 (ICL)** | 33.x | 4.x | GuC required for submission |
| **Gen12 (TGL)** | 49.x | 7.x | GuC/HuC both required |
| **DG1/DG2** | 49.x/52.x | 7.x | Modern arch, most capable |

---

## Summary & Best Practices

### Key Takeaways

1. **Two-firmware system:** GuC must load before HuC
2. **Signature verification:** Critical for security
3. **Version compatibility:** Must match GPU generation
4. **Graceful degradation:** Continue if HuC fails (optional)
5. **Robust retry:** Handle transient failures

### Best Practices

**For Deployment:**
- Always include latest firmware in kernel tree
- Verify firmware signatures before distribution
- Test firmware loading on target platform
- Monitor firmware version in production
- Keep fallback paths for missing firmware

**For Debugging:**
- Check firmware file exists before loading
- Verify file permissions and checksums
- Monitor GuC/HuC status registers
- Log all firmware-related errors
- Test with error injection

**For Reliability:**
- Use exponential backoff for retries
- Implement firmware corruption detection
- Verify loaded firmware with checksums
- Handle missing firmware gracefully
- Test failure scenarios

---

## References

- [GuC Firmware](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/i915/gt/uc)
- [Firmware Loading](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/i915/gt/uc/intel_uc_fw.c)
- Related: [02-GuC-Firmware.md](02-GuC-Firmware.md), [10-Hardware-Discovery-Initialization.md](10-Hardware-Discovery-Initialization.md)

---

**Next Steps:**
- Study firmware loading in intel_uc_fw.c
- Analyze GuC/HuC handshake messages
- Test firmware loading failure scenarios
- Review firmware version compatibility matrices

---

**Document Status:** ✅ Complete  
**Version:** 1.0  
**Last Updated:** February 8, 2026
