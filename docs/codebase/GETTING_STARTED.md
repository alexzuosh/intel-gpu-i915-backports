# Intel i915 GPU Driver Documentation - Getting Started Guide

## Quick Navigation

### Core Architecture Documents (01-05)

#### 1. [01-Memory-Management.md](01-Memory-Management.md)
**Focus:** GPU Memory Allocation, Management, and Coherency

**Read This If You Need To:**
- Understand GEM (Graphics Execution Manager) object lifecycle
- Learn about VRAM allocation and eviction strategies
- Understand CPU-GPU cache coherency issues
- Work on memory pressure and shrinker mechanisms
- Debug memory-related performance issues

**Key Diagrams:**
- Memory type hierarchy (DDR, VRAM, caches)
- GEM object lifecycle (create → allocate → bind → evict → destroy)
- Buddy allocator fragmentation management
- VMA (Virtual Memory Area) binding with page tables
- Memory coherency synchronization flows

**Time Investment:** 30-45 minutes for overview, 1-2 hours for deep dive

---

#### 2. [02-GuC-Firmware.md](02-GuC-Firmware.md)
**Focus:** Graphics Unified Controller (GuC) Firmware, Submission, and Scheduling

**Read This If You Need To:**
- Understand GPU work submission via GuC
- Learn H2G (Host-to-GuC) message protocols
- Understand context scheduling and preemption
- Debug GuC submission failures
- Optimize GPU scheduling and throughput

**Key Diagrams:**
- H2G/G2H message communication protocol
- GuC internal architecture and event processing
- Context registration, scheduling, and deregistration flows
- Request lifecycle through GuC (6-phase timing)
- Priority-based preemption and concurrent scheduling
- Error handling and recovery mechanisms

**Time Investment:** 45 minutes for overview, 2-3 hours for complete understanding

---

#### 3. [03-Context-Management.md](03-Context-Management.md)
**Focus:** GPU Execution Contexts, Address Spaces, and Per-Process Isolation

**Read This If You Need To:**
- Understand per-application GPU context creation
- Learn about multi-engine context management
- Work on Per-Process Page Tables (PPGTT)
- Configure SSEU (Slice/Subslice/EU) allocation
- Debug context creation or isolation issues

**Key Diagrams:**
- Context creation and lifecycle (allocation → programming → execution → cleanup)
- LRC (Logical Ring Context) memory layout
- Multi-engine context state synchronization
- PPGTT per-process address space management
- Context priority and SSEU configuration

**Time Investment:** 30-45 minutes for overview, 1.5-2 hours for deep dive

---

#### 4. [04-Power-Management.md](04-Power-Management.md)
**Focus:** Frequency Scaling, Power Gating, and Thermal Management

**Read This If You Need To:**
- Understand RPS (Render P-state) frequency scaling
- Learn about RC6 power gating and wake latency
- Work on SLPC (Smart Low Power Controller) autonomous control
- Debug thermal throttling issues
- Optimize power consumption vs performance trade-offs

**Key Diagrams:**
- RPS frequency scaling workflow and decision trees
- RC6 power gating entry/exit sequences
- SLPC autonomous controller operation
- Power well hierarchy and control signals
- Thermal throttling and temperature management

**Time Investment:** 30-45 minutes for overview, 1.5-2 hours for deep dive

---

#### 5. [05-Request-Scheduling.md](05-Request-Scheduling.md)
**Focus:** GPU Request Lifecycle, Scheduling, and Execution

**Read This If You Need To:**
- Understand complete request lifecycle from IOCTL to GPU execution
- Learn about priority-based scheduling with preemption
- Work on request dependency tracking and batching
- Optimize batch submission and throughput
- Debug scheduling or execution issues

**Key Diagrams:**
- Complete request lifecycle (IOCTL → driver → GuC → GPU → completion)
- Request scheduler state machine and transitions
- Priority-based scheduling with preemption
- Request dependency chains and batch coalescing
- Engine queue management and context switching
- Fence completion tracking and retirement

**Time Investment:** 45 minutes for overview, 2-3 hours for complete understanding

---

### Supporting Documentation

#### [06-Virtual-Memory.md](06-Virtual-Memory.md)
MMU (Memory Management Unit) architecture, TLB management, and address translation.

#### [06c-GGTT-PPGTT-Deep-Dive.md](06c-GGTT-PPGTT-Deep-Dive.md)
Detailed analysis of Global GTT and Per-Process GTT management.

#### [07-TBB-Task-Scheduling.md](07-TBB-Task-Scheduling.md)
Task-Based Batching (TBB) for optimizing work submission.

#### [08-Debugger-Support.md](08-Debugger-Support.md)
Debugging infrastructure and state capture mechanisms.

#### [09-i915-Fence-Timeline-Study.md](09-i915-Fence-Timeline-Study.md)
DMA fence lifecycle, timeline management, and completion signaling.

---

## Reading Paths for Different Roles

### For GPU Driver Developers

**Essential Foundation (Week 1):**
1. [00-OUTLINE.md](00-OUTLINE.md) - 30 minutes
2. [02-GuC-Firmware.md](02-GuC-Firmware.md) - 2 hours
3. [03-Context-Management.md](03-Context-Management.md) - 1.5 hours
4. [05-Request-Scheduling.md](05-Request-Scheduling.md) - 2 hours

**Deep Dive (Week 2-3):**
5. [01-Memory-Management.md](01-Memory-Management.md) - 1.5 hours
6. [04-Power-Management.md](04-Power-Management.md) - 1.5 hours
7. [09-i915-Fence-Timeline-Study.md](09-i915-Fence-Timeline-Study.md) - 1 hour
8. [06-Virtual-Memory.md](06-Virtual-Memory.md) - 1 hour

**Specialization:**
- Submission path: [06c-GGTT-PPGTT-Deep-Dive.md](06c-GGTT-PPGTT-Deep-Dive.md), [07-TBB-Task-Scheduling.md](07-TBB-Task-Scheduling.md)
- Debugging: [08-Debugger-Support.md](08-Debugger-Support.md)

---

### For Performance Engineers

**Critical Path (Week 1):**
1. [00-OUTLINE.md](00-OUTLINE.md) - 30 minutes
2. [05-Request-Scheduling.md](05-Request-Scheduling.md) - 2 hours
3. [04-Power-Management.md](04-Power-Management.md) - 1.5 hours
4. [01-Memory-Management.md](01-Memory-Management.md) - 1.5 hours

**Detailed Study:**
5. [02-GuC-Firmware.md](02-GuC-Firmware.md) - Sections on optimizations
6. [07-TBB-Task-Scheduling.md](07-TBB-Task-Scheduling.md) - 1 hour

---

### For Hardware Integration Engineers

**Essential Reading:**
1. [00-OUTLINE.md](00-OUTLINE.md) - 30 minutes
2. [02-GuC-Firmware.md](02-GuC-Firmware.md) - 2 hours (focus on firmware interaction)
3. [03-Context-Management.md](03-Context-Management.md) - 1 hour
4. [04-Power-Management.md](04-Power-Management.md) - 1.5 hours

**Reference Materials:**
5. [06-Virtual-Memory.md](06-Virtual-Memory.md)
6. [08-Debugger-Support.md](08-Debugger-Support.md)

---

### For Graphics Application Developers

**Recommended Reading:**
1. [00-OUTLINE.md](00-OUTLINE.md) - 30 minutes
2. [03-Context-Management.md](03-Context-Management.md) - 1 hour (understanding contexts)
3. [05-Request-Scheduling.md](05-Request-Scheduling.md) - 1 hour (submission and priority)
4. [04-Power-Management.md](04-Power-Management.md) - 30 minutes (power impact understanding)

---

## How to Use the Diagrams

### Understanding PlantUML Diagrams

Each diagram in this documentation includes:

1. **Title & Description**: What the diagram shows
2. **Participants/Components**: Main actors in the system
3. **Flow/States**: Sequences, state transitions, or hierarchies
4. **Annotations**: Important notes and decision points
5. **Legend**: Color/shape meanings (if applicable)

### Viewing Diagrams

**Online (Recommended):**
- Copy diagram code (between @startuml and @enduml)
- Paste into [PlantUML online editor](http://www.plantuml.com/plantuml/uml/)
- View and export as PNG/SVG

**Locally:**
- Install PlantUML: `brew install plantuml` (macOS) or `apt install plantuml` (Linux)
- Run: `plantuml diagram.puml` to generate PNG

**IDE Integration:**
- VS Code: Install PlantUML extension
- IntelliJ: Built-in PlantUML support
- Vim/Neovim: Use PlantUML plugins

---

## Key Concepts Explained

### GPU Execution Pipeline
```
User Application
    ↓
Kernel Driver (i915)
    ↓
GuC Firmware (if enabled) OR Direct Engine Submission
    ↓
GPU Hardware Execution
    ↓
Completion & Interrupts
```

### Memory Hierarchy
```
System RAM ←→ GPU VRAM ←→ GPU Caches
     ↕           ↕           ↕
  NUMA-aware   Local memory  L3, LLC
```

### Context Isolation
```
Per-Process PPGTT (Page Tables)
    ↓
Logical Ring Context (LRC)
    ↓
Hardware Context Descriptor
    ↓
GPU Engine Execution State
```

---

## Troubleshooting Guide

### Problem: "GuC submission timeout"
- Check: [02-GuC-Firmware.md](02-GuC-Firmware.md) - Error Handling section
- Review: Context Registration and Scheduling flows
- Verify: H2G message protocol compliance

### Problem: "Out of GPU memory"
- Check: [01-Memory-Management.md](01-Memory-Management.md) - Shrinker and eviction sections
- Review: Memory pressure handling diagrams
- Verify: Allocation policies and fragmentation

### Problem: "Frequency/power throttling"
- Check: [04-Power-Management.md](04-Power-Management.md) - RPS and thermal sections
- Review: Power well and frequency scaling workflows
- Verify: Thermal monitoring and limits

### Problem: "Request execution stuck"
- Check: [05-Request-Scheduling.md](05-Request-Scheduling.md) - Lifecycle and dependencies
- Review: Fence tracking and completion signaling
- Verify: [09-i915-Fence-Timeline-Study.md](09-i915-Fence-Timeline-Study.md)

### Problem: "Context isolation/security issue"
- Check: [03-Context-Management.md](03-Context-Management.md) - Context isolation
- Review: PPGTT management in [06c-GGTT-PPGTT-Deep-Dive.md](06c-GGTT-PPGTT-Deep-Dive.md)
- Verify: Address space separation

---

## Related Resources

### Official Intel Documentation
- [i915 Kernel Documentation](https://www.kernel.org/doc/html/latest/gpu/i915/)
- [GuC Firmware Documentation](https://www.kernel.org/doc/html/latest/gpu/i915/uapi.html)

### Linux Kernel Source
- [i915 Driver Source](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/i915/)
- [DRM Subsystem](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/)

### Useful Commands for Investigation
```bash
# Check i915 module parameters
cat /sys/module/i915/parameters/

# Monitor GPU usage
sudo intel_gpu_top

# Check power states
cat /sys/kernel/debug/dri/0/i915_frequency_info

# Review driver logs
dmesg | grep i915

# Check GuC status
cat /sys/kernel/debug/dri/0/guc_info
```

---

## Contributing to Documentation

Found an issue or have improvements?

1. **Update the relevant document** with clear, concise information
2. **Include a diagram** using PlantUML syntax (compatible format)
3. **Cross-reference** related sections and documents
4. **Test diagram rendering** before committing
5. **Update ENHANCEMENT-SUMMARY.md** if adding new sections

### Documentation Style Guide
- Use technical but accessible language
- Start with "why" before "how"
- Include concrete examples from source code
- Use PlantUML for complex interactions
- Keep sections focused and concise
- Link to related documentation

---

## Document Statistics

| Document | Lines | Diagrams | Focus |
|----------|-------|----------|-------|
| 00-OUTLINE.md | 400 | 2 | High-level overview |
| 01-Memory-Management.md | 1,085 | 9 | GEM, allocation, coherency |
| 02-GuC-Firmware.md | 1,388 | 19 | GuC submission, scheduling |
| 03-Context-Management.md | 1,051 | 8 | Context lifecycle, isolation |
| 04-Power-Management.md | 1,088 | 7 | RPS, RC6, SLPC, thermal |
| 05-Request-Scheduling.md | 1,093 | 8 | Request lifecycle, scheduling |
| 06-Virtual-Memory.md | 625 | 3 | MMU, TLB, address translation |
| 06c-GGTT-PPGTT-Deep-Dive.md | 450 | 4 | GTT architecture |
| 07-TBB-Task-Scheduling.md | 380 | 2 | Task-based batching |
| 08-Debugger-Support.md | 420 | 2 | Debugging infrastructure |
| 09-i915-Fence-Timeline-Study.md | 850 | 5 | Fence, timeline, completion |

**Total: ~8,000+ lines | 60+ diagrams | Comprehensive coverage**

---

## Questions or Feedback?

This documentation represents the i915 GPU driver architecture as of kernel version 6.0+.

For discussions or improvements, reference the specific document and section in your communication.

---

**Last Updated:** February 2026  
**Version:** 2.0  
**PlantUML Compatibility:** All versions with standard syntax support  

