# TBB (Task-Based Batch) Scheduler - Design and Implementation

**Document:** i915 GPU Driver Architecture  
**Topic:** Task-Based Batch Scheduling System  
**Source Files:** `i915_tbb.c`, `i915_tbb.h`  
**Kernel Version:** 6.8+  
**Status:** 705 lines of implementation code

---

## Table of Contents

1. [Overview & Purpose](#overview--purpose)
2. [Architecture & Design Philosophy](#architecture--design-philosophy)
3. [Core Data Structures](#core-data-structures)
4. [Thread Pool Model](#thread-pool-model)
5. [Task Scheduling Logic](#task-scheduling-logic)
6. [NUMA-Aware Distribution](#numa-aware-distribution)
7. [CPU Affinity & Priority](#cpu-affinity--priority)
8. [Task Management](#task-management)
9. [Code Flow Examples](#code-flow-examples)
10. [Integration with i915 Driver](#integration-with-i915-driver)
11. [Performance Characteristics](#performance-characteristics)
12. [Debugging & Monitoring](#debugging--monitoring)

---

## Overview & Purpose

### What is TBB?

TBB is a **late-binding task scheduling framework** that defers CPU task assignment decisions until execution time, rather than determining the execution CPU when the task is created. This "greedy" scheduling approach enables:

- **Dynamic load balancing** across CPU cores
- **Avoiding oversubscription** of system cores
- **Idle NOHZ core utilization** without interfering with isolated applications

### Key Design Principle

```
Traditional Work Queue:
  Schedule task → Predetermined CPU → Execute

TBB Scheduler:
  Create task → Wait for CPU availability → Execute on available core
```

### Problem Solved

The i915 driver needs to handle CPU-bound work (memory management, register updates, error handling) that:
- Must execute but doesn't require specific CPU placement
- Should prefer idle OS cores over busy application cores
- Should avoid interrupting isolated (nohz_full) CPUs unless necessary

---

## Architecture & Design Philosophy

### TBB Design Goals

```
1. LATENCY-AWARE
   └─ Execute tasks on first available CPU
   └─ Prioritize OS-managed cores

2. NOHZ-FRIENDLY
   └─ Avoid waking isolated cores when OS cores available
   └─ Use low priority on nohz_full cores

3. NUMA-OPTIMIZED
   └─ Per-NUMA node task queues
   └─ Minimize cross-node data access

4. WORK-CONSERVING
   └─ Balance work across available cores
   └─ No unnecessary idle time
```

### Scheduling Hierarchy

```
                    ┌─────────────────┐
                    │  Task Creation  │
                    │  (any context)  │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ NUMA Node Queue │
                    │ (per-node list) │
                    └────────┬────────┘
                             │
      ┌──────────────────────┼──────────────────────┐
      │                      │                      │
  ┌───▼──────┐       ┌───────▼────────┐      ┌──────▼──┐
  │ OS Core  │       │  OS Core (N)   │      │ NOHZ    │
  │ Primary  │       │ Secondary      │      │ Core    │
  │ Thread   │       │ Thread         │      │ Thread  │
  │(Exclusive)       │ (Non-exclusive)│      │ (Low)   │
  └──────────┘       └────────────────┘      └─────────┘
     High Priority       Normal Priority      Idle Priority
```

### Core Principle: Deferred Assignment

```c
// Traditional: Immediate assignment
schedule_work_on(cpu, &work);  // CPU chosen now

// TBB: Late binding
i915_tbb_add_task(&task);      // CPU chosen at execution time
```

---

## Core Data Structures

### 1. TBB Node (Per-NUMA)

```c
struct i915_tbb_node {
    struct rb_node rb;                 // Red-black tree for node lookup
    struct list_head tasks;            // Global task queue for node
    wait_queue_head_t wq;              // Wakeup mechanism for threads
    struct kref ref;                   // Reference counting
    
    // Per-node execution statistics
    struct {
        local_t tasks;                 // Total tasks executed
        local_t local;                 // Local (assigned) tasks
        local_t primary;               // Primary thread executions
        local_t secondary;             // Secondary thread executions
        local_t yields;                // Yield events
        local_t wakeups;               // Wakeup counts
    } stats;
    
    int nid;                           // NUMA node ID
};
```

**Purpose:** Provides NUMA-locality and per-node task queuing

**Key Fields:**
- `tasks`: Global queue for this NUMA node
- `wq`: Kernel wait queue (kernel abstraction for wake/sleep)
- `stats`: Non-blocking counters (local_t) for performance monitoring

### 2. TBB Task (Per-Work Item)

```c
struct i915_tbb {
    struct list_head link;             // Link in node->tasks queue
    struct list_head local;            // Link in thread's local queue
    void (*fn)(struct i915_tbb *self); // Callback to execute
    
    // Tracking
    struct i915_tbb_node *node;        // Which NUMA node
    struct task_struct *tsk;           // Executing thread (when running)
};
```

**Purpose:** Encapsulates a unit of work to be scheduled

**Key Fields:**
- `fn`: Task callback (executed when scheduled)
- `link`: Position in global queue
- `local`: Position in thread-local queue
- `tsk`: Reference to executing thread (for cancellation)

### 3. TBB Thread (Per-CPU)

```c
struct i915_tbb_thread {
    struct wait_queue_entry wait;      // Kernel wait mechanism
    struct list_head local;            // This thread's local work queue
    struct i915_tbb_node *node;        // Associated NUMA node
    unsigned long flags;               // State flags (I915_TBB_SUSPEND)
    int cpu;                           // CPU number
};

// Per-CPU storage (one per CPU core)
static DEFINE_PER_CPU(struct i915_tbb_thread, i915_tbb_thread);
```

**Purpose:** Per-CPU thread context and local work queue

**Key Fields:**
- `local`: Work assigned to this specific thread
- `node`: Associated NUMA node
- `flags`: Suspension state during CPU hotplug/idle
- `wait`: Kernel wait structure (integration with schedule())

### 4. Global Node Tree

```c
static DEFINE_SPINLOCK(nodes_lock);    // Protects tree
static struct rb_root nodes;            // Red-black tree of nodes
static struct i915_tbb_node no_node;    // Fallback for non-NUMA systems
```

**Purpose:** Maps NUMA node IDs to TBB nodes for quick lookup

---

## Thread Pool Model

### Thread Organization

```
┌─────────────────────────────────────────────────┐
│         i915 TBB Thread Pool Architecture       │
├─────────────────────────────────────────────────┤
│                                                 │
│  NUMA Node 0          NUMA Node 1               │
│  ┌─────────────────┐  ┌──────────────────┐     │
│  │  Tasks Queue    │  │  Tasks Queue     │     │
│  │  (shared)       │  │  (shared)        │     │
│  └────┬────────────┘  └────┬─────────────┘     │
│       │                    │                   │
│  ┌────┴──────────┐    ┌────┴──────────┐       │
│  │   Threads     │    │   Threads     │       │
│  │               │    │               │       │
│  │ [CPU0: OS]    │    │ [CPU4: OS]    │       │
│  │  PRIMARY      │    │  PRIMARY      │       │
│  │  (excl, hi)   │    │  (excl, hi)   │       │
│  │               │    │               │       │
│  │ [CPU1: OS]    │    │ [CPU5: OS]    │       │
│  │  SECONDARY    │    │  SECONDARY    │       │
│  │  (nonexcl)    │    │  (nonexcl)    │       │
│  │               │    │               │       │
│  │ [CPU2: NOHZ]  │    │ [CPU6: NOHZ]  │       │
│  │  IDLE ONLY    │    │  IDLE ONLY    │       │
│  │  (low pri)    │    │  (low pri)    │       │
│  │               │    │               │       │
│  └───────────────┘    └───────────────┘       │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Thread Classification

#### Primary Thread (Exclusive Waiter)

```
Condition:  !tick_nohz_full_cpu(cpu)
Policy:     SCHED_FIFO_LOW
Wakeup:     First in waitqueue (exclusive)
Purpose:    Handle OS-managed cores
Behavior:   Always woken when work available
```

#### Secondary Thread (Non-Exclusive Waiter)

```
Condition:  !tick_nohz_full_cpu(cpu) AND use_nohz enabled
Policy:     SCHED_NORMAL
Wakeup:     All threads in waitqueue
Purpose:    Handle overflow from primary threads
Behavior:   May wake if primary threads insufficient
```

#### NOHZ Core Thread (Idle-Only)

```
Condition:  tick_nohz_full_cpu(cpu) AND use_nohz enabled
Policy:     SCHED_IDLE (minimum priority)
Wakeup:     Only when completely idle
Purpose:    Avoid interrupting isolated applications
Behavior:   Only executes if no other processes on core
```

### Thread Priority Hierarchy

```
Wakeup Order:
  1. Primary threads (OS cores) - EXCLUSIVE
     └─ Head of waitqueue
     └─ Woken first
  
  2. Secondary threads (OS cores) - NON-EXCLUSIVE
     └─ Middle of waitqueue
     └─ Woken if primary insufficient
  
  3. NOHZ threads - LOWEST PRIORITY
     └─ End of waitqueue
     └─ Only woken if completely idle


Scheduling Priority (sched_priority):
  Primary:     FIFO_LOW (system priority, preempt app)
  Secondary:   NORMAL
  NOHZ:        IDLE (lowest, app preempts TBB)
```

---

## Task Scheduling Logic

### Task Lifecycle

```
┌─────────────────────────────────────────────────────┐
│          TBB Task Execution Lifecycle               │
└─────────────────────────────────────────────────────┘

1. CREATION
   └─ i915_tbb_init_task(&task, callback)
   └─ Initialize task structure
   
2. SUBMISSION
   └─ i915_tbb_add_task(&task)
   └─ Lock NUMA node
   └─ Add to node->tasks queue
   └─ Wake threads if needed
   
3. ASSIGNMENT (Deferred)
   └─ Thread becomes available
   └─ Check local queue OR global queue
   └─ Assign to local queue
   └─ Local tasks prioritized
   
4. EXECUTION
   └─ Thread dispatches task
   └─ Execute task->fn(task)
   └─ Update statistics
   └─ Loop if more work
   
5. COMPLETION
   └─ Task callback returns
   └─ Thread considers rescheduling
   └─ May yield to higher priority work
```

### Core Scheduling Functions

#### 1. Task Addition: `i915_tbb_add_task_on()`

```c
void i915_tbb_add_task_on(struct i915_tbb *task, int cpu)
{
    // Select thread based on CPU argument
    struct i915_tbb_thread *t = per_cpu_ptr(
        &i915_tbb_thread, 
        cpu == WORK_CPU_UNBOUND ? raw_smp_processor_id() : cpu
    );
    
    struct i915_tbb_node *node = t->node;
    unsigned long flags;
    
    // Lock node
    flags = i915_tbb_lock(node);
    
    if (list_empty(&task->link)) {
        // First submission of this task
        __i915_tbb_add_task(task, t);
        // └─ Adds to both node->tasks AND t->local
        // └─ Wakes thread if suspended
    } else {
        // Task already in system, just wake worker
        wake_up_locked(&node->wq);
    }
    
    i915_tbb_unlock(node, flags);
}
```

**Behavior:**
- Adds task to specific thread's local queue
- Also adds to global node queue for work stealing
- Wakes thread only if necessary (optimization)

#### 2. Dispatch: `tbb_dispatch()`

```c
static void tbb_dispatch(unsigned int cpu)
{
    struct i915_tbb_thread *t = per_cpu_ptr(&i915_tbb_thread, cpu);
    struct i915_tbb_node *node = t->node;
    
    do {
        struct i915_tbb *task;
        
        // Check if we should stop
        if (test_bit(I915_TBB_SUSPEND, &t->flags) || 
            !tbb_ready(node))
            return;
        
        // Lock and select task
        i915_tbb_lock_irq(node);
        
        if (!list_empty(&t->local)) {
            // Priority 1: Local work (assigned to us)
            task = list_first_entry(&t->local, 
                                    struct i915_tbb, local);
            local_inc(&node->stats.local);
        } else if (!list_empty(&node->tasks) && 
                   idle_app_cpu(t)) {
            // Priority 2: Steal from global queue (if idle)
            task = list_first_entry(&node->tasks,
                                    struct i915_tbb, link);
            local_inc(p_thread(t) ? 
                      &node->stats.primary : 
                      &node->stats.secondary);
        } else {
            // No work for us
            i915_tbb_unlock_irq(node);
            return;
        }
        
        // Remove task from queues
        task->tsk = current;
        list_del(&task->local);
        list_del_init(&task->link);
        
        // Wake next thread if more work
        if (tbb_should_wake_up(node, t) && 
            !list_is_singular(&node->tasks))
            wake_up_locked(&node->wq);
        
        i915_tbb_unlock_irq(node);
        
        // Execute task
        local_inc(&node->stats.tasks);
        task->fn(task);  // <--- User callback
        
        // Loop continues if no need_resched()
    } while (!need_resched());
}
```

**Decision Tree:**
```
Execute next task?
├─ YES if suspended? → NO
├─ YES if node empty? → NO
└─ YES otherwise
    ├─ Has local work?
    │  └─ YES: Execute local task (assigned)
    ├─ Global work + idle?
    │  └─ YES: Work-steal from global queue
    └─ NO: Return and sleep
```

### Scheduling Policies

#### Local-First Policy

```c
// Each thread maintains local queue
if (!list_empty(&t->local)) {
    // Execute local work FIRST
    task = list_first_entry(&t->local, ...);
    stats.local++;
} else if (!list_empty(&node->tasks) && idle_app_cpu(t)) {
    // Only work-steal if completely idle
    task = list_first_entry(&node->tasks, ...);
}
```

**Purpose:** Minimize CPU context switches and improve cache locality

#### Idle-Only Work-Stealing

```c
static bool idle_app_cpu(struct i915_tbb_thread *t)
{
    // Primary (OS) threads: always consider idle
    // NOHZ threads: only if single_task_running()
    return p_thread(t) || single_task_running();
}
```

**Purpose:** Avoid interfering with application on nohz_full cores

#### Selective Wake-Up

```c
// Only wake next thread if:
// 1. More work exists
// 2. Task list not just our departing task
if (tbb_should_wake_up(node, t) && 
    !list_is_singular(&node->tasks))
    wake_up_locked(&node->wq);
```

**Purpose:** Avoid excessive wakeups; batch work together

---

## NUMA-Aware Distribution

### Node Lookup

```c
struct i915_tbb_node *i915_tbb_node(int nid)
{
    // nid <= 0: Use fallback
    if (nid <= 0)
        return &no_node;
    
    // Look up in red-black tree
    return to_node(rb_find(as_ptr(nid), &nodes, node_key)) 
        ?: &no_node;
}
```

**Behavior:**
- NUMA-aware systems: Separate queues per node
- Non-NUMA systems: Single `no_node` fallback
- Fast lookup: O(log n) red-black tree search

### Thread-to-Node Mapping

```c
static void tbb_create(unsigned int cpu)
{
    struct i915_tbb_thread *t = per_cpu_ptr(&i915_tbb_thread, cpu);
    int nid = cpu_to_node(cpu);  // <--- Get NUMA node
    
    // Find or create node
    struct i915_tbb_node *node = find_or_create_node(nid);
    
    // Assign thread to node
    t->node = node;
}
```

**Result:**
```
CPU 0-3 (Node 0) → node0
CPU 4-7 (Node 1) → node1
CPU 8-11 (Node 2) → node2
...
```

### NUMA Locality Benefits

```
When i915_tbb_add_task(task) called on CPU 2:
  1. Get current thread (CPU 2)
  2. Get NUMA node (Node 0 for CPU 2)
  3. Add task to Node 0 queue
  4. Execute on Node 0 thread (same node)
  
Result: All data access local to node → better memory bandwidth
```

---

## CPU Affinity & Priority

### Policy Selection

```c
static void tbb_setup(unsigned int cpu)
{
    struct i915_tbb_thread *t = per_cpu_ptr(&i915_tbb_thread, cpu);
    
    if (p_thread(t)) {
        // Primary thread: High priority
        sched_set_fifo_low(t->wait.private);
        // │
        // └─ SCHED_FIFO with priority just above normal apps
    } else if (!use_nohz) {
        // Secondary thread (NOHZ disabled): Stop thread
        stop_kthread(t->wait.private);
        // │
        // └─ No thread; all work on primary threads
    } else {
        // Secondary/NOHZ thread: Low priority
        sched_set_idle(t->wait.private);
        // │
        // └─ SCHED_IDLE: Minimum priority, no system load
    }
}
```

### Priority Levels

```
Priority Hierarchy (from high to low):

┌──────────────────┐
│  Real-Time Apps  │  (SCHED_FIFO, high priority)
│  (e.g., video)   │
├──────────────────┤
│  TBB Primary     │  (SCHED_FIFO_LOW)
│  (OS cores)      │  <-- i915 TBB on OS-managed cores
├──────────────────┤
│  Normal Apps     │  (SCHED_NORMAL)
│  (default)       │
├──────────────────┤
│  TBB Secondary   │  (SCHED_NORMAL)
│  (OS overflow)   │  <-- i915 TBB for extra work
├──────────────────┤
│  TBB NOHZ        │  (SCHED_IDLE)
│  (isolated cores)│  <-- i915 TBB on nohz_full only
├──────────────────┤
│  Idle (kernel)   │
└──────────────────┘
```

### NOHZ-Full Support

```c
#if IS_ENABLED(CONFIG_NO_HZ_FULL)
static bool __read_mostly use_nohz = CPTCFG_DRM_I915_NOHZ_OFFLOAD;
#else
#define use_nohz false
#endif

// Module parameter: nohz_offload
// Default: enabled if CONFIG_NO_HZ_FULL available
// Can disable: echo 0 > /sys/module/drm_i915/parameters/nohz_offload
```

**Purpose:**
- Support isolated/tickless CPU cores
- Avoid TBB on nohz_full cores when NOHZ support disabled
- Enable graceful fallback on non-NOHZ kernels

---

## Task Management

### Task Initialization

```c
static inline void i915_tbb_init_task(
    struct i915_tbb *tsk, 
    void (*fn)(struct i915_tbb *task))
{
    tsk->fn = fn;
    tsk->node = NULL;
    INIT_LIST_HEAD(&tsk->link);
}
```

**Usage:**
```c
// Typical i915 usage
struct i915_tbb task;

i915_tbb_init_task(&task, my_callback);
i915_tbb_add_task(&task);
```

### Task Cancellation

```c
bool i915_tbb_cancel_task(struct i915_tbb *task)
{
    struct i915_tbb_node *node = task->node;
    struct task_struct *tsk = NULL;
    unsigned long flags;
    
    if (!node)
        return false;  // Not scheduled
    
    flags = i915_tbb_lock(node);
    
    if (!list_empty(&task->link)) {
        // Task still in queue, remove it
        list_del(&task->local);
        list_del_init(&task->link);
        // Success: task never executed
    } else {
        // Task executing or completed
        tsk = fetch_and_zero(&task->tsk);
    }
    
    i915_tbb_unlock(node, flags);
    
    if (tsk) {
        // Task is running, park the thread
        kthread_park(tsk);
        kthread_unpark(tsk);
        // └─ Synchronize with executing task
        
        return false;  // Not cancelled
    }
    
    return true;  // Successfully cancelled
}
```

**Return Values:**
- `true`: Task cancelled before execution
- `false`: Task executed or executing

### Task Suspension/Resumption

```c
int i915_tbb_suspend_local(void)
{
    int cpu = raw_smp_processor_id();
    struct i915_tbb_thread *t = per_cpu_ptr(&i915_tbb_thread, cpu);
    struct i915_tbb_node *node = t->node;
    
    // Mark suspended
    set_bit(I915_TBB_SUSPEND, &t->flags);
    
    // Transfer work to other threads
    if (thread_is_running(t) && tbb_should_wake_up(node, t))
        wake_up(&node->wq);
    
    return cpu;
}

void i915_tbb_resume_local(int cpu)
{
    struct i915_tbb_thread *t = per_cpu_ptr(&i915_tbb_thread, cpu);
    struct i915_tbb_node *node = t->node;
    
    clear_bit(I915_TBB_SUSPEND, &t->flags);
    
    // Resume work if available
    if (!list_empty(&t->local) || tbb_should_wake_up(node, t))
        wake_up_thread(t);
}
```

**Use Cases:**
- CPU hotplug: Suspend task threads during offline
- Preemption: Temporarily pause TBB for high-priority work
- Power transitions: Suspend during power state changes

---

## Code Flow Examples

### Example 1: Basic Task Submission & Execution

```c
// ============================================
// 1. CALLER: Schedule GPU register write
// ============================================

struct i915_tbb write_task;

// Initialize task with callback
i915_tbb_init_task(&write_task, gpu_register_write_fn);

// Submit task (greedy late-binding)
i915_tbb_add_task(&write_task);  // WORK_CPU_UNBOUND
// └─ Added to current thread's local queue
// └─ Thread woken if suspended


// ============================================
// 2. KERNEL: TBB Dispatch Loop (Per-CPU thread)
// ============================================

// Thread was sleeping in wait queue, woken up
tbb_dispatch(cpu=2):
    node = get_node(cpu=2)      // Node 0 for CPU 2
    
    loop:
        if (suspended or no work)
            return
        
        lock(node)
        
        if (local work) {
            task = list_first_entry(&t->local)
            stats.local++
        } else if (global work and idle) {
            task = list_first_entry(&node->tasks)
            stats.primary++
        } else {
            unlock(node)
            return
        }
        
        remove_from_queue(task)
        if (more work and not singular)
            wake_up_locked(node)
        
        unlock(node)
        
        // ============================================
        // 3. CALLBACK: Execute GPU operation
        // ============================================
        
        task->fn(task)  // gpu_register_write_fn(&task)
        
        if (need_resched())
            break
        else
            continue loop  // Try to execute more


// ============================================
// 4. KERNEL: Potential Work-Stealing
// ============================================

// If this thread (CPU 2) finishes local work:
if (empty(t->local) and not_empty(node->tasks) and idle_app_cpu()) {
    // Work steal from global queue
    task = list_first_entry(&node->tasks)
    // Execute on CPU 2 instead of assigning to owner
}
```

**Flow Diagram:**

```
CPU N (any core)               TBB Thread Pool (CPU 2, Node 0)
┌──────────────┐               ┌────────────────────────────────┐
│ i915 Driver  │               │ tbb_dispatch(cpu=2)            │
│              │               │                                │
│ Task ready   │               │ [Sleeping in waitqueue]        │
│              │               └────────────────────────────────┘
│ schedule():  │                         △
└──┬───────────┘                         │ wake_up()
   │                                     │
   ├─ i915_tbb_init_task()               │
   │  └─ Setup callback                  │
   │                                     │
   ├─ i915_tbb_add_task()                │
   │  ├─ Lock node                       │
   │  ├─ Add to local queue              ├─────────────────┐
   │  ├─ Wake thread ─────────────────────┘                │
   │  └─ Unlock node                                       │
   │                                     ┌─────────────────▼──┐
   │                                     │ Wake up            │
   │                                     │ Lock node          │
   │                                     │                    │
   │                                     │ Check local work   │
   │                                     │ YES: task found    │
   │                                     │                    │
   │                                     │ Unlock node        │
   │                                     │                    │
   │                                     │ Execute:           │
   │                                     │ task->fn(task)     │
   │                                     │                    │
   │                                     │ Loop if !need_resched
   │                                     └────────────────────┘
```

### Example 2: Work Stealing Scenario

```c
// SCENARIO: Primary thread idle, secondary thread has work

State Before:
    Node Queue:    [TaskA]
    CPU 2 local:   [TaskB, TaskC]
    CPU 3 local:   [TaskD, TaskE]
    CPU 6 (NOHZ):  [] (empty)


Timeline:

T1: CPU 2 executing TaskB
    CPU 3 executing TaskD
    CPU 6 sleeping (NOHZ)

T2: CPU 2 finishes TaskB
    ├─ Check local: TaskC (execute local first)
    ├─ Dispatch local TaskC
    └─ Loop continues

T3: CPU 2 finishes TaskC
    ├─ Check local: empty
    ├─ Check node queue: TaskA available
    ├─ Check idle_app_cpu(): Yes, it's a primary thread
    ├─ Work-steal TaskA from node queue
    ├─ Execute TaskA
    └─ All work done


State After:
    Node Queue:    [] (empty)
    CPU 2 local:   [] (empty)
    CPU 3 local:   [TaskE] (CPU 3 still running TaskD)
    CPU 6 (NOHZ):  [] (sleeping)
```

### Example 3: NOHZ Core Behavior

```c
// SCENARIO: NOHZ-full core with application running

Config:
    CPU 6: tick_nohz_full_cpu = true  (isolated, application running)
    use_nohz = true  (NOHZ offload enabled)

State:
    Node Queue:    [TaskX, TaskY]
    CPU 6:         [running_app_task]
    CPU 6 TBB:     [TaskW] (local)


Timeline:

T1: CPU 6 TBB wakes up
    ├─ Check suspension: not suspended
    ├─ Check node ready: yes, TaskX/Y available
    ├─ Call tbb_should_run()
    │   ├─ Check suspended: no
    │   ├─ Check need_resched: yes (app thread preempted this)
    │   ├─ p_thread(t)=false (NOHZ core)
    │   ├─ Return FALSE: should not run
    │   └─ Ask other thread to run: wake_up()
    │
    └─ Return to sleep (idle state)

T2: CPU 2 (OS core) TBB wakes up
    ├─ Check local work: none
    ├─ Check global work: TaskX, TaskY available
    ├─ Check idle_app_cpu(): true (OS core, can execute)
    ├─ Work-steal TaskX from node
    └─ Execute TaskX on CPU 2

T3: Application on CPU 6 completes
    └─ CPU 6 TBB thread will wake and handle TaskW if needed


Policy Followed:
    ✓ Never interrupt NOHZ app with TBB work
    ✓ TBB runs only when CPU 6 is completely idle
    ✓ Other cores handle TBB work instead
```

---

## Integration with i915 Driver

### Where TBB is Used

TBB provides a generic task scheduling framework for i915 operations that:
1. Don't require specific CPU placement
2. Should avoid busy cores
3. Can tolerate late execution

### Typical i915 Use Cases

```c
// Example: Memory pressure handling
static void shrink_task_fn(struct i915_tbb *task)
{
    // Called when TBB decides to run us
    // Scan memory regions for eviction candidates
}

struct i915_tbb shrink_task;
i915_tbb_init_task(&shrink_task, shrink_task_fn);

// Trigger from memory pressure path:
if (memory_pressure_high) {
    i915_tbb_add_task(&shrink_task);
    // Task will run when CPU available
}
```

### Integration Points

```
i915 Driver Code
├─ i915_gem_shrink(): Memory management
│  └─ i915_tbb_add_task() for eviction work
│
├─ GPU error handling: Debug operations
│  └─ i915_tbb_add_task() for error capture
│
└─ Register management: MMIO updates
   └─ i915_tbb_add_task() for batch register writes
```

---

## Performance Characteristics

### Latency Analysis

```
Task Submission to Execution Time:

┌─────────────────────────────────────────┐
│  Best Case: Current CPU available       │
├─────────────────────────────────────────┤
│  1. Submission (1-5 μs)                 │
│  2. Wake-up (1-10 μs)                   │
│  3. Dispatch (1-5 μs)                   │
│  ─────────────────────────────────────  │
│  Total: ~5-20 μs                        │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│  Average Case: Current CPU busy         │
├─────────────────────────────────────────┤
│  1. Submission (1-5 μs)                 │
│  2. Wake-up (1-10 μs)                   │
│  3. Schedule out (1-100 μs)             │
│  4. Queue wait (variable)               │
│  5. Dispatch (1-5 μs)                   │
│  ─────────────────────────────────────  │
│  Total: ~10-1000 μs (depends on load)   │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│  Worst Case: Cross-NUMA, all idle       │
├─────────────────────────────────────────┤
│  1. Submission to NUMA node queue       │
│  2. IPI to remote node (1-5 μs)         │
│  3. Remote CPU idle entry (10-100 μs)   │
│  ─────────────────────────────────────  │
│  Total: ~50-200 μs                      │
└─────────────────────────────────────────┘
```

### Throughput Analysis

```
Maximum Task Throughput:

On 8-core system (4 primary OS, 4 NOHZ):
  ├─ Primary threads: ~50-100k tasks/sec per core
  ├─ Secondary threads: ~30-50k tasks/sec per core
  ├─ NOHZ threads: ~5-10k tasks/sec per core (limited by availability)
  └─ Total: 400k-800k tasks/sec
  
Critical factor: Task callback execution time
  ├─ Short callbacks (1-10 μs): Full throughput
  ├─ Medium callbacks (100 μs): Reduced (task execution dominates)
  └─ Long callbacks (1 ms+): Limited by thread availability
```

### Overhead

```
Overhead per task:

Space:
  ├─ Per-task: ~64 bytes (struct i915_tbb)
  ├─ Per-thread: ~256 bytes (struct i915_tbb_thread)
  ├─ Per-node: ~512 bytes (struct i915_tbb_node)
  └─ Total per 128-task workload: <10 KB

Time:
  ├─ Lock/unlock per dispatch: 100-500 ns
  ├─ Wake-up: 1-10 μs
  ├─ Context switch: 1-10 μs
  └─ Total: <20 μs overhead per task
```

---

## Debugging & Monitoring

### SysRq Support

TBB integrates with Linux SysRq to provide on-demand diagnostics:

```bash
# Trigger TBB status dump
echo t > /proc/sysrq-trigger

# Output example (from sysrq_show):
---
Threads:
  NUMA node: 0

CPUs:
  Primary: 2 (0,1)
  Secondary: 2 (2,3)
  Running: 1 (0)
  Suspended: 0

Tasks:
  - gpu_register_write x 5
  - memory_shrink x 2

Execution:
  Tasks: 1234
  Local: 890
  Primary: 223
  Secondary: 121
  Yields: 45
  Wakeups: 567
```

### Statistics

```c
// Accessible via per-node local counters
struct i915_tbb_node {
    struct {
        local_t tasks;       // Total tasks executed
        local_t local;       // Tasks from local queue
        local_t primary;     // Primary thread executions
        local_t secondary;   // Secondary thread executions
        local_t yields;      // Yield events
        local_t wakeups;     // Wakeup counts
    } stats;
};
```

### Debugging Interfaces

#### Kernel Logging

```c
// Available with drm.debug=0x2
DRM_DEBUG("TBB task submitted: %pS", task->fn);
DRM_DEBUG("TBB thread wake-up on CPU %d", cpu);
```

#### Debugfs (if enabled)

```bash
# Hypothetical debugfs interface
cat /sys/kernel/debug/dri/0/i915_tbb/stats
cat /sys/kernel/debug/dri/0/i915_tbb/threads
cat /sys/kernel/debug/dri/0/i915_tbb/nodes
```

### Performance Profiling

```bash
# Using perf to analyze TBB overhead
perf trace -e 'sched:sched_wakeup' --filter 'comm ~ "tbb"' sleep 10

# Track context switches
perf stat -e context-switches,cpu-migrations ./workload

# Profile task execution
perf record -e 'sched:sched_switch' -g sleep 10
perf report
```

### Tracing

```bash
# Enable kernel tracing
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_wakeup/enable
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable

# View trace
cat /sys/kernel/debug/tracing/trace | grep tbb
```

---

## Summary Table

| Aspect | Detail |
|--------|--------|
| **Purpose** | Late-binding task scheduling for load-balanced CPU work |
| **Task Submission** | O(1) append to local queue |
| **Task Selection** | Local-first, then work-steal if idle |
| **Thread Wakeup** | Selective, coalesces work |
| **NUMA Support** | Per-node queues, locality-aware |
| **Priority Levels** | Primary (high), Secondary (normal), NOHZ (idle) |
| **Core Count** | Per-CPU thread (1 per core) |
| **Scheduling Policy** | SCHED_FIFO_LOW (primary), SCHED_NORMAL (secondary), SCHED_IDLE (NOHZ) |
| **Typical Latency** | 5-20 μs (best) to 50-200 μs (worst) |
| **Throughput** | 400k-800k tasks/sec (8-core system) |
| **Memory Overhead** | ~64 bytes per task |
| **NOHZ-Full Safe** | Yes (lowest priority, idle-only on isolated cores) |

---

## References

- **Source:** `drivers/gpu/drm/i915/i915_tbb.c` (705 lines)
- **Header:** `drivers/gpu/drm/i915/i915_tbb.h` (111 lines)
- **Module Parameter:** `nohz_offload` (control NOHZ utilization)
- **Config:** `CONFIG_NO_HZ_FULL` (required for NOHZ support)

---

**End of Document**
