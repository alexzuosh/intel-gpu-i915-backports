# Intel i915 GPU Driver Codebase Documentation - Complete Index

**Last Updated:** 2026-02-06  
**Total Documentation:** 10,000+ lines  
**Coverage:** 8 major components + 3 deep-dives + utilities

---

## 📂 Complete File Listing

### Master Documents

| File | Size | Purpose | Status |
|------|------|---------|--------|
| **README.md** | 17 KB | Master index, navigation guide, FAQ | ✅ Ready |
| **00-OUTLINE.md** | 14 KB | Architecture overview, component list | ✅ Ready |

### Detailed Component Documentation

| File | Size | Component | Lines | Status |
|------|------|-----------|-------|--------|
| **01-Memory-Management.md** | 19 KB | GEM, LMEM, Buddy Allocator | 571 | ✅ Complete |
| **02-GuC-Firmware.md** | 17 KB | Firmware, Communication, Submission | 576 | ✅ Complete |
| **03-Context-Management.md** | 16 KB | Contexts, LRC, Address Spaces | 577 | ✅ Complete |
| **04-Power-Management.md** | 21 KB | RPS, RC6, SLPC, Runtime PM | 627 | ✅ Complete |
| **05-Request-Scheduling.md** | 17 KB | Requests, Scheduler, Execution | 552 | ✅ Complete |

### Memory & Virtual Memory Deep-Dives

| File | Size | Topic | Lines | Status |
|------|------|-------|-------|--------|
| **06-Virtual-Memory.md** | 41 KB | GGTT, PPGTT, VMA Binding, TLB | 1,800+ | ✅ Complete |
| **06b-Memory-Migration.md** | 24 KB | Migration, Eviction, Coherency | 700+ | ✅ Complete |
| **06c-GGTT-PPGTT-Deep-Dive.md** | 45 KB | GGTT/PPGTT Comparison & Analysis | 1,600+ | ✅ Complete |
| **06d-GGTT-PPGTT-Implementation.md** | 25 KB | GGTT/PPGTT Practical Guide | 850+ | ✅ Complete |

### CPU Task Scheduling & Breadcrumbs

| File | Size | Topic | Lines | Status |
|------|------|-------|-------|--------|
| **07-TBB-Task-Scheduling.md** | 35 KB | CPU Task-Based Batch Scheduler | 1,200+ | ✅ Complete |
| **07b-TBB-Implementation-Guide.md** | 24 KB | TBB API & Patterns | 800+ | ✅ Complete |

### Debugger Support & Error Handling

| File | Size | Topic | Lines | Status |
|------|------|-------|-------|--------|
| **08-Debugger-Support.md** | 38 KB | Error Capture, Hang Detection, Debugging | 1,400+ | ✅ Complete |
| **08b-Debugger-Implementation.md** | 30 KB | Debugger API & Practical Guide | 1,200+ | ✅ Complete |

### Support & Summary Files

| File | Size | Purpose | Status |
|------|------|---------|--------|
| **MEMORY-DOCUMENTATION-SUMMARY.md** | 11 KB | Memory subsystem overview | ✅ Complete |
| **TBB-DOCUMENTATION-SUMMARY.md** | 12 KB | TBB scheduler overview | ✅ Complete |
| **GGTT-PPGTT-DOCUMENTATION-SUMMARY.md** | 14 KB | Address space overview | ✅ Complete |
| **DEBUGGER-SUPPORT-SUMMARY.md** | 18 KB | Debugger support overview | ✅ Complete |
| **QUICKSTART.md** | 6 KB | Quick reference guide | ✅ Complete |
| **QUICKSTART-DEBUGGER.md** | 2.7 KB | Debugger quick start | ✅ Complete |

### Planned Documentation

| # | Component | File | Priority | Status |
|---|-----------|------|----------|--------|
| 9 | Interrupt Handling | 09-Interrupt-Handling.md | HIGH | 📋 Next |
| 10 | Reset & Error Handling | 10-Reset-Error.md | HIGH | 📋 Next |
| 11 | Display Subsystem | 11-Display.md | MEDIUM | 📋 Planned |
| 12 | Protected Execution (PXP) | 12-Protected-Execution.md | MEDIUM | 📋 Planned |
| 13 | Performance Monitoring | 13-Performance-Monitoring.md | MEDIUM | 📋 Planned |
| 14 | Firmware Management | 14-Firmware-Management.md | LOW | 📋 Planned |
| 15 | SR-IOV Virtualization | 15-SR-IOV-Virtualization.md | LOW | 📋 Planned |

---

## 🎯 Quick Access by Task

### For New Driver Developers
1. **Start:** README.md → "Quick Navigation" section
2. **Architecture:** 00-OUTLINE.md
3. **Foundations:** 01-Memory-Management.md
4. **Execution Model:** 03-Context-Management.md → 05-Request-Scheduling.md
5. **Advanced Topics:** 06-Virtual-Memory.md → 07-TBB-Task-Scheduling.md → 08-Debugger-Support.md

### For Memory System Work
- **Primary Documentation:**
  - 01-Memory-Management.md - GEM, regions, buddy allocator
  - 06-Virtual-Memory.md - GGTT, PPGTT, VMA, page tables
  - 06b-Memory-Migration.md - Migration, eviction, coherency
  - 06c-GGTT-PPGTT-Deep-Dive.md - Architecture & comparison
  - 06d-GGTT-PPGTT-Implementation.md - Practical guide

### For Power Optimization
- **Primary:** 04-Power-Management.md
  - RPS (frequency scaling)
  - RC6 (power gating)
  - SLPC (firmware control)
  - Runtime PM
  - Thermal management

### For Request/Workload Handling
- **Primary:** 05-Request-Scheduling.md
  - GPU requests
  - Scheduler
  - Ring buffers
  - Submission (GuC/Execlists)
  - Preemption

- **Secondary:** 02-GuC-Firmware.md (GuC submission path)

### For CPU Task Scheduling
- **Primary:** 07-TBB-Task-Scheduling.md
  - CPU task design
  - Thread pool model
  - NUMA awareness
  - Work stealing
  - Integration with GPU work

- **Secondary:** 07b-TBB-Implementation-Guide.md (API reference)

### For GPU Hang/Error Investigation
- **Primary Documentation:**
  - 08-Debugger-Support.md - Error capture, hang detection, recovery
  - 08b-Debugger-Implementation.md - Practical API guide
  - QUICKSTART-DEBUGGER.md - 5-minute setup

### For Display Work
- **Will need:** 11-Display.md (planned)

---

---

## 📊 What Each Document Covers

### 01-Memory-Management.md (19 KB)
GEM objects, memory regions, buddy allocator, eviction, LMEM
- Object lifecycle and VMA binding
- Region-specific memory management
- Pressure-based eviction
- Cache coherency considerations

### 02-GuC-Firmware.md (17 KB)
GPU firmware system, communication protocol, submission
- Firmware loading and initialization
- Command transport layer
- Work queue mechanism
- GuC submission path vs. Execlists

### 03-Context-Management.md (16 KB)
GPU execution contexts, logical ring context, address spaces
- Context lifecycle management
- LRC (Logical Ring Context) structure
- Per-context address space (PPGTT)
- Context switching and preemption

### 04-Power-Management.md (21 KB)
Frequency scaling, power gating, firmware-based power control
- RPS (Render Power States) control loop
- RC6 power gating (classic, RC6p, RC6pp)
- SLPC (Self-Learning Power Control)
- Runtime suspend/resume

### 05-Request-Scheduling.md (17 KB)
GPU request objects, scheduling, ring buffer execution
- Request lifecycle and priority scheduling
- Ring buffer management
- Batch execution pipeline
- Request retirement and signaling

### 06-Virtual-Memory.md (41 KB)
Global and per-process graphics translation tables, virtual memory binding
- GGTT (Global Graphics Translation Table) - shared, 256MB-2GB
- PPGTT (Per-Process Graphics Translation Table) - per-context, 48-bit
- VMA (Virtual Memory Address) objects and binding lifecycle
- Page table management and TLB handling

### 06b-Memory-Migration.md (24 KB)
Memory region migration, eviction, and cache coherency
- Migration triggers and pipeline
- Eviction algorithms under memory pressure
- Cache coherency strategies
- Region-specific optimizations

### 06c-GGTT-PPGTT-Deep-Dive.md (45 KB)
Comprehensive comparison of address space systems
- GGTT characteristics and use cases
- PPGTT characteristics and use cases
- Architecture comparison and decision matrix
- Hardware constraints and platform differences
- Advanced scenarios and edge cases
- Debugging techniques

### 06d-GGTT-PPGTT-Implementation.md (25 KB)
Practical guide to using GGTT and PPGTT
- GGTT operations (init, bind, access, cleanup)
- PPGTT operations (create, bind, switch)
- VMA lifecycle management
- Common patterns and best practices
- Real-world implementation examples

### 07-TBB-Task-Scheduling.md (35 KB)
CPU task-based batch scheduler for kernel work
- Design philosophy and use cases
- Thread pool model (primary, secondary, NOHZ)
- Scheduling logic and work-stealing
- NUMA awareness and CPU affinity
- Integration with GPU submission

### 07b-TBB-Implementation-Guide.md (24 KB)
Practical API reference for TBB task scheduler
- API functions and signatures
- Common patterns (periodic work, async tasks, etc.)
- Real-world examples (power control, memory management)
- Debugging and performance tips
- Pitfalls and best practices

### 08-Debugger-Support.md (38 KB)
GPU error capture, hang detection, firmware debugging
- Error state capture and coredump structure
- GPU hang detection via heartbeat mechanism
- Recovery escalation (preemption → engine reset → full reset)
- Userspace debugger protocol (event-driven)
- Firmware debugging (GuC logging and capture)
- Debug interfaces (debugfs, sysfs, ioctl)

### 08b-Debugger-Implementation.md (30 KB)
Practical guide to debugging GPU issues
- Quick start (5 minutes)
- API reference for error capture and hang detection
- Error state access patterns
- Debugger protocol usage examples
- Common debugging scenarios
- Real-world code examples (bash scripts, C code)
- Test frameworks and monitoring daemons

---

## 🏆 Documentation Quality Features

### Each File Includes:
✓ Clear section hierarchy  
✓ Conceptual overview with diagrams  
✓ Key data structures documented  
✓ State machines & flow diagrams  
✓ Real code examples (not pseudo-code)  
✓ Complete execution paths  
✓ Configuration & parameter details  
✓ Debugging & inspection methods  
✓ Related component references  
✓ Source file locations  

### Diagrams Present:
✓ System architecture  
✓ Component hierarchies  
✓ Data/control flow  
✓ State transitions  
✓ Memory layouts  
✓ Communication protocols  
✓ Scheduling queues  
✓ Ring buffer structure  

### Code Examples:
✓ Real function signatures  
✓ Actual data structures  
✓ Traced execution paths  
✓ Function call chains  
✓ Pseudocode for complex logic  

---

## 📈 Learning Progression

### Level 1: Foundations (Day 1-2)
- README.md (navigation)
- 00-OUTLINE.md (big picture)

### Level 2: Core Systems (Day 3-5)
- 01-Memory-Management.md (foundational)
- 03-Context-Management.md (execution model)
- 05-Request-Scheduling.md (workload)

### Level 3: Advanced Systems (Day 6-7)
- 02-GuC-Firmware.md (firmware control)
- 04-Power-Management.md (optimization)
- 06-Virtual-Memory.md (memory subsystem)

### Level 4: Specialized (Day 8-10)
- 06b-Memory-Migration.md (migration/eviction)
- 06c-GGTT-PPGTT-Deep-Dive.md (address spaces)
- 07-TBB-Task-Scheduling.md (CPU task scheduling)

### Level 5: Debugging & Diagnostics (Day 11+)
- 08-Debugger-Support.md (error capture)
- 08b-Debugger-Implementation.md (practical debugging)

### Level 6: Reference (As-Needed)
- 07b-TBB-Implementation-Guide.md (TBB API)
- 06d-GGTT-PPGTT-Implementation.md (practical GGTT/PPGTT)
- Summary documents for quick overview

---

## 🔗 Cross-Reference Map

```
README.md (Master Index)
    ↓
00-OUTLINE.md (Architecture)
    ├→ 01-Memory-Management.md (GEM, regions, buddy)
    ├→ 02-GuC-Firmware.md (Firmware system)
    ├→ 03-Context-Management.md (GPU contexts)
    ├→ 04-Power-Management.md (Power control)
    ├→ 05-Request-Scheduling.md (GPU scheduling)
    │
    ├─ 06-Virtual-Memory.md (GGTT, PPGTT, VMA)
    │   ├→ 06b-Memory-Migration.md (Migration/eviction)
    │   ├→ 06c-GGTT-PPGTT-Deep-Dive.md (Design analysis)
    │   └→ 06d-GGTT-PPGTT-Implementation.md (Practical guide)
    │
    ├─ 07-TBB-Task-Scheduling.md (CPU task scheduler)
    │   └→ 07b-TBB-Implementation-Guide.md (API reference)
    │
    ├─ 08-Debugger-Support.md (Error capture & hang detection)
    │   └→ 08b-Debugger-Implementation.md (Practical guide)
    │
    └─ [Future: 09-14 Interrupt, Reset, Display, PXP, Perf, Firmware, VF]

Internal Cross-References:
    01 ↔ 06 (VMA binding and memory objects)
    01 ↔ 06b (Eviction and migration)
    02 ↔ 05 (GuC-based submission path)
    03 ↔ 06 (Context PPGTT address space)
    04 ↔ 05 (Power and scheduling interaction)
    05 ↔ 07 (Kernel work scheduling)
    06 ↔ 08 (Memory in error state dumps)
    07 ↔ 08 (Heartbeat uses task scheduler)
    08 ↔ [All] (Error capture includes state from all components)
```

---

## 📍 File Locations

**Documentation Root:**
```
/home/alex/code/intel-gpu-i915-backports/docs/codebase/
```

**Key Source Directories Referenced:**
```
drivers/gpu/drm/i915/
    ├── gem/                 # Memory & execution
    ├── gt/                  # Graphics technology
    │   ├── uc/             # Firmware (GuC, HUC, GSC)
    │   └── [engine files]  # Context, RPS, RC6, reset
    ├── display/            # Display subsystem
    ├── pxp/                # Protected execution
    └── [core files]        # Driver, IRQ, request, scheduler
```

---

## ✅ Verification Checklist

- [x] All 22 files created and documented
- [x] Total 10,000+ lines
- [x] ~500 KB documentation
- [x] 8 major components documented (+ 3 deep-dives)
- [x] Architecture diagrams included
- [x] Code examples included
- [x] Cross-references complete
- [x] Source locations mapped
- [x] Master index complete
- [x] Navigation guide present
- [x] Quick start guides (2 versions)
- [x] Learning paths defined
- [x] Summary documents for each major topic

---

## 🎯 Next Steps

### Immediate (Critical Path):
1. ✅ **06-Virtual-Memory.md** - GGTT, PPGTT, VMA binding
2. ✅ **07-TBB-Task-Scheduling.md** - CPU task scheduling
3. ✅ **08-Debugger-Support.md** - Error capture & hang detection

### Short Term (High Priority):
4. **09-Interrupt-Handling.md** - IRQ processing, breadcrumbs, fence completion
5. **10-Reset-Error.md** - Detailed reset choreography, error recovery
6. **13-Performance-Monitoring.md** - OA metrics, profiling, PMU

### Medium Term (Important):
7. **11-Display-Subsystem.md** - Display pipeline, GGTT usage
8. **12-Protected-Execution.md** - PXP, security, TEE integration
9. **14-Firmware-Management.md** - UC framework, versioning

### Long Term (Specialized):
10. **15-SR-IOV-Virtualization.md** - VF management, resource partitioning

---

## 📞 Using This Documentation

### For Reference:
- Bookmark README.md as starting point
- Use 00-OUTLINE.md as roadmap
- Refer to specific docs as needed

### For Learning:
- Follow recommended learning path
- Study code examples
- Cross-reference related components
- Use source code locations to verify concepts

### For Development:
- Find relevant doc for your task
- Understand component architecture
- Review code examples
- Check debugging section for tools

### For Problem-Solving:
- README.md FAQ section
- Search cross-references
- Follow related components links
- Check source file locations

---

## 📊 Document Metadata

| Metric | Value |
|--------|-------|
| Total Files | 22 |
| Core Docs | 8 (01-08) |
| Deep-Dive Docs | 4 (06b-07b) |
| Support/Summary | 6 (README, INDEX, 4x summaries, 2x quickstart) |
| Total Lines | 10,000+ |
| Total Size | ~500 KB |
| Components Documented | 8/14 (57%) |
| Code Examples | 100+ |
| Diagrams | 100+ |
| Cross-references | 200+ |
| Source Locations | 100+ |

---

**Generated by: GPU Driver Expert Assistant**  
**For: Intel i915 GPU Driver Backports**  
**Scope: Comprehensive Codebase Documentation**

---

*See README.md for full navigation guide and quick start.*
