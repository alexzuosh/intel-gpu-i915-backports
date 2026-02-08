# Intel i915 GPU Driver Documentation - Complete Index

## 📚 Documentation Structure

This directory contains comprehensive documentation for the Intel i915 GPU driver architecture, including 60+ PlantUML diagrams and 8,000+ lines of detailed technical content.

---

## 🚀 Quick Start

1. **New to i915?** → Start with [GETTING_STARTED.md](GETTING_STARTED.md)
2. **Want an overview?** → Read [00-OUTLINE.md](00-OUTLINE.md) 
3. **Need specific info?** → See the table below for direct navigation

---

## 📖 Core Documentation

| # | Document | Lines | Diagrams | Key Topics |
|---|----------|-------|----------|------------|
| 00 | [OUTLINE.md](00-OUTLINE.md) | 400 | 2 | System overview, architecture |
| 01 | [Memory-Management.md](01-Memory-Management.md) | 1,085 | 9 | GEM, VRAM, allocation, coherency |
| 02 | [GuC-Firmware.md](02-GuC-Firmware.md) | 1,388 | 19 | Submission, scheduling, H2G messages |
| 03 | [Context-Management.md](03-Context-Management.md) | 1,051 | 8 | Context creation, PPGTT, isolation |
| 04 | [Power-Management.md](04-Power-Management.md) | 1,088 | 7 | RPS, RC6, SLPC, thermal throttling |
| 05 | [Request-Scheduling.md](05-Request-Scheduling.md) | 1,093 | 8 | Request lifecycle, preemption, batching |
| 06 | [Virtual-Memory.md](06-Virtual-Memory.md) | 625 | 3 | MMU, TLB, address translation |
| 06c | [GGTT-PPGTT-Deep-Dive.md](06c-GGTT-PPGTT-Deep-Dive.md) | 450 | 4 | GTT architecture, page tables |
| 07 | [TBB-Task-Scheduling.md](07-TBB-Task-Scheduling.md) | 380 | 2 | Task-based batching, optimization |
| 08 | [Debugger-Support.md](08-Debugger-Support.md) | 420 | 2 | Debug infrastructure, state capture |
| 09 | [i915-Fence-Timeline-Study.md](09-i915-Fence-Timeline-Study.md) | 850 | 5 | Fence lifecycle, completion signaling |

---

## 🎯 Reference Documents

| Document | Purpose |
|----------|---------|
| [ENHANCEMENT-SUMMARY.md](ENHANCEMENT-SUMMARY.md) | Summary of all documentation enhancements (45 diagrams, 6,253 lines) |
| [GETTING_STARTED.md](GETTING_STARTED.md) | Reading guide for different roles (developers, engineers, etc.) |
| [INDEX.md](INDEX.md) | This file - complete navigation guide |

---

## 🔍 Topic-Based Navigation

### Memory Management
- **[01-Memory-Management.md](01-Memory-Management.md)** - Complete memory system
  - GEM object lifecycle
  - Buddy allocator
  - Memory shrinker & eviction
  - VMA binding
  - Cache coherency

- **[06-Virtual-Memory.md](06-Virtual-Memory.md)** - MMU details
  - TLB management
  - Address translation
  - Page fault handling

- **[06c-GGTT-PPGTT-Deep-Dive.md](06c-GGTT-PPGTT-Deep-Dive.md)** - Global & per-process GTT
  - GGTT architecture
  - PPGTT per-process page tables
  - GTT entries and management

### GPU Submission Pipeline
- **[02-GuC-Firmware.md](02-GuC-Firmware.md)** - GuC submission mode
  - H2G/G2H messages
  - Context registration & scheduling
  - Preemption & priority
  - Error handling

- **[07-TBB-Task-Scheduling.md](07-TBB-Task-Scheduling.md)** - Work batching
  - Task-based batching
  - Ring submission optimization
  - Batch coalescing

### Request Execution & Scheduling
- **[03-Context-Management.md](03-Context-Management.md)** - GPU contexts
  - Context creation & lifecycle
  - Multi-engine support
  - Per-process isolation
  - SSEU configuration

- **[05-Request-Scheduling.md](05-Request-Scheduling.md)** - Request scheduling
  - Complete request lifecycle
  - Priority scheduling
  - Preemption mechanics
  - Dependency tracking
  - Fence completion

- **[09-i915-Fence-Timeline-Study.md](09-i915-Fence-Timeline-Study.md)** - Synchronization
  - DMA fence signaling
  - Timeline management
  - Breadcrumb detection
  - Request retirement

### Power & Performance
- **[04-Power-Management.md](04-Power-Management.md)** - Power states
  - RPS frequency scaling
  - RC6 power gating
  - SLPC autonomous control
  - Power wells
  - Thermal throttling

### Debugging & Development
- **[08-Debugger-Support.md](08-Debugger-Support.md)** - Debug infrastructure
  - State capture
  - Error reporting
  - Debug registers

---

## 📊 Statistics

### Coverage
- **Total Documents:** 11 (core + supporting)
- **Total Lines of Code:** 8,000+
- **Total Diagrams:** 60+
- **Total Commits:** 5 (enhancement commits)

### Diagram Distribution
| Component | Diagrams | Focus |
|-----------|----------|-------|
| GuC Firmware | 19 | Submission, scheduling, messages |
| Context Management | 8 | Lifecycle, multi-engine, isolation |
| Request Scheduling | 8 | Lifecycle, preemption, batching |
| Power Management | 7 | Frequency, RC6, thermal |
| Memory Management | 9 | GEM, allocation, coherency |
| Virtual Memory | 3 | MMU, TLB, translation |
| GTT Architecture | 4 | GGTT, PPGTT, page tables |
| Task Scheduling | 2 | Batching, optimization |
| Debugger | 2 | Debug infrastructure |
| Synchronization | 5 | Fence, timeline, completion |
| **Total** | **60+** | **Comprehensive coverage** |

---

## 🎓 Learning Paths

### Path 1: GPU Driver Development (Comprehensive)
```
Week 1:
  └─ 00-OUTLINE.md (30 min)
  └─ 02-GuC-Firmware.md (2 hours)
  └─ 03-Context-Management.md (1.5 hours)
  └─ 05-Request-Scheduling.md (2 hours)

Week 2-3:
  └─ 01-Memory-Management.md (1.5 hours)
  └─ 04-Power-Management.md (1.5 hours)
  └─ 09-i915-Fence-Timeline-Study.md (1 hour)
  └─ 06-Virtual-Memory.md (1 hour)

Specialization:
  └─ 06c-GGTT-PPGTT-Deep-Dive.md
  └─ 07-TBB-Task-Scheduling.md
  └─ 08-Debugger-Support.md
```

### Path 2: Performance Engineering (Focused)
```
  └─ 00-OUTLINE.md (30 min)
  └─ 05-Request-Scheduling.md (2 hours)
  └─ 04-Power-Management.md (1.5 hours)
  └─ 01-Memory-Management.md (1.5 hours)
  └─ 02-GuC-Firmware.md (optimization sections)
  └─ 07-TBB-Task-Scheduling.md (1 hour)
```

### Path 3: Debugging & Troubleshooting (Quick)
```
  └─ GETTING_STARTED.md (30 min)
  └─ Specific document based on issue
  └─ 08-Debugger-Support.md
```

---

## 🔗 Cross-References

### Memory System Interactions
```
User App
    ↓
GEM Object (01-Memory)
    ↓
VMA Binding (01-Memory, 06-Virtual-Memory)
    ↓
Page Tables (06-Virtual-Memory, 06c-GGTT-PPGTT)
    ↓
GPU Memory Hierarchy (01-Memory)
```

### Submission Pipeline
```
IOCTL Request
    ↓
Request Object (05-Scheduling)
    ↓
Submission Mode (02-GuC or Direct)
    ↓
Context (03-Context)
    ↓
GuC Scheduling (02-GuC) or Ring (07-TBB)
    ↓
GPU Execution
    ↓
Fence Completion (09-Fence-Timeline)
```

### Power Management
```
GPU Load Detection (04-Power)
    ↓
RPS Decision (04-Power)
    ↓
Frequency Scaling (04-Power)
    ↓
Power Wells (04-Power)
    ↓
RC6 Gating (04-Power)
    ↓
Thermal Throttling (04-Power)
```

---

## 💡 Quick Reference

### Common Questions → Document

| Question | Document |
|----------|----------|
| How does GPU work submission work? | 02-GuC-Firmware.md, 07-TBB |
| How is GPU memory allocated? | 01-Memory-Management.md |
| How does process isolation work? | 03-Context-Management.md |
| How is work prioritized/scheduled? | 05-Request-Scheduling.md |
| How does power management work? | 04-Power-Management.md |
| How does address translation work? | 06-Virtual-Memory.md |
| How do I debug GPU issues? | 08-Debugger-Support.md |
| How do fences and timelines work? | 09-i915-Fence-Timeline-Study.md |
| How is VRAM managed? | 01-Memory-Management.md |
| How are contexts created? | 03-Context-Management.md |

---

## 🛠️ Using the Documentation

### For Code Review
1. Identify the subsystem being modified
2. Check the relevant documentation (01-05)
3. Verify changes against the diagrams
4. Cross-reference with related systems

### For Debugging
1. Use GETTING_STARTED.md troubleshooting section
2. Study the relevant state machine diagram
3. Review error handling flows
4. Check fence/completion mechanisms

### For Architecture Design
1. Review relevant section in core documents
2. Study interaction diagrams
3. Check dependencies with other systems
4. Refer to state machine transitions

### For Performance Optimization
1. Check 04-Power-Management.md for overhead
2. Review 05-Request-Scheduling.md for bottlenecks
3. Study 01-Memory-Management.md for allocation patterns
4. Reference 07-TBB-Task-Scheduling.md for batching

---

## 📋 Document Checklist

Core Components:
- [x] Memory Management (GEM, VRAM, allocation, coherency)
- [x] GuC Firmware (submission, scheduling, messaging)
- [x] Context Management (creation, isolation, PPGTT)
- [x] Power Management (RPS, RC6, SLPC, thermal)
- [x] Request Scheduling (lifecycle, preemption, batching)
- [x] Virtual Memory (MMU, TLB, translation)
- [x] GTT Architecture (GGTT, PPGTT)
- [x] Task Scheduling (TBB, batching)
- [x] Synchronization (fence, timeline)
- [x] Debugger Support

Navigation & Reference:
- [x] Core outline (00-OUTLINE.md)
- [x] Getting started guide (GETTING_STARTED.md)
- [x] Enhancement summary (ENHANCEMENT-SUMMARY.md)
- [x] Complete index (INDEX.md)

Quality Assurance:
- [x] All diagrams PlantUML compatible
- [x] Cross-references validated
- [x] Code examples current
- [x] Git history maintained

---

## 🚀 Repository Information

**Repository:** intel-gpu/intel-gpu-i915-backports  
**Branch:** fix/comprehensive-bug-fixes  
**Location:** `/home/alex/code/intel-gpu-i915-backports/docs/codebase/`

### Recent Enhancement Commits
```
2f445c2 Add comprehensive Getting Started guide with reading paths for different roles and troubleshooting guide
193dee6 Add comprehensive enhancement summary documenting 45 diagrams and 6,253 lines of documentation across 5 core files
6f72308 docs: Add deep-dive analysis and 9 new diagrams to Memory Management documentation
6eec074 docs: Add comprehensive diagrams and deep-dive analysis to codebase documentation
7a4eef8 Add comprehensive i915 fence and timeline study documentation
```

---

## 📞 Support & Feedback

### Documentation Issues
- Broken links? Check the file path matches the markdown references
- Diagram not rendering? Copy code to PlantUML online editor
- Outdated content? Cross-check with kernel source (kernel.org)

### Contributing Improvements
1. Identify the relevant document
2. Update content with clear explanations
3. Add/update diagrams using PlantUML
4. Test diagram rendering
5. Update INDEX.md and ENHANCEMENT-SUMMARY.md
6. Create detailed commit message

### Recommended Tools
- **PlantUML Viewer:** http://www.plantuml.com/plantuml/uml/
- **Markdown Editor:** VS Code with PlantUML extension
- **Git:** For tracking documentation history

---

## 📅 Version History

| Version | Date | Changes |
|---------|------|---------|
| 2.0 | Feb 2026 | Added 45 diagrams, comprehensive deep-dives, getting started guide |
| 1.0 | Jan 2026 | Initial documentation structure |

---

## 🎯 Next Steps

1. **Start Reading:** Choose your role in GETTING_STARTED.md
2. **Explore Diagrams:** Pick a document and view its diagrams
3. **Cross-Reference:** Use INDEX.md to navigate between related topics
4. **Deep Dive:** Study specific subsystems in detail
5. **Contribute:** Share improvements and insights

---

**Last Updated:** February 2026  
**Version:** 2.0  
**Maintainers:** Intel GPU Driver Documentation Team  

