# Performance Bugs

A program's performance can decline due to various reasons, from excessive memory access to inefficient loops. It is essential to diagnose the bug that is causing the issue before making any modifications to the program. There are various tools that can be utilized to diagnose the issues.

---

## Performance Monitoring Units

 PMUs (Performance Monitoring Units) are parts that are integrated into the hardware to track hardware events such as cache misses or branch mispredictions. However, on its own, it tends to be a challenge to identify which of the events actually causes the performance declines. That is why tools such as Intel Vtuner are used to give a predefined set of events to use as a reference. Through these events, it is possible to narrow down on the hotspot functions.

Current modern CPUs implement pipelining to use hardware resources in the most effective way possible. Unfortunately, there are situations where the pipeline is completely idle without any instructions being executed, which causes performance decline. By using TMA (Top Down Microarchitecture Analysis), it is possible to find the dominant performance bottlenecks in the application. It is also possible to find out how many of the CPU pipelines are being used for the application.

Modern CPU pipelining is divided into frontend and backend components. The frontend is what fetches the program code and decodes it into instructions. It also breaks down the code into lower-level hardware operations called micro operations. A micro operation (or µop) is the CPU’s internal mini-instruction. A majority of x86 instructions decode into one or more micro operations that the core is able to schedule, execute, or retire. The frontend then feeds the backend the micro operations, which is a process called “allocation”. The backend executes the operation on an available unit. Once the operation is executed, it is considered “retired.” Most of these processes end up retired, but some get cancelled due to branch mispredictions or other related issues.

On average, Intel’s CPU frontend is able to allocate, and the backend is able to execute 4 micro operations per cycle. TMA makes the assumption that there are 4 pipelines for each core of the CPU.

![alt text](./images/pmu.png)

### **Pipeline Stalling**

When a pipeline slot is empty during a cycle, it is considered a stall. PMU events are able to show if the stall is due to a frontend or backend failure. 

- If the frontend is the cause of the stall, it is referred to as frontend bound. It is due to the frontend’s inability to fill a slot with a micro operation. 

- If a backend is the cause of the stall, it is referred to as a backend stall. It usually indicates the backend running out of resources, resulting in a stall.

- If a micro operation does not retire at all, it is called bad speculation. It can be caused by branch mispredictions. 

- Lastly, if a micro operation is complete, it is considered retired.

- In cases when both the frontend and backend are causing the stall, it is considered a backend bound. This is due to the fact that fixing the frontend will not alleviate the situation, since the backend bottlenecks the performance.

![alt text](./images/pipeline.png)

As seen above in the diagram, pipelining can be divided into 5 different stages. 

- Fetch (IF)
- Decode (ID)
- Execute (EX)
- Memory (MEM)
- Writeback WB)

Fetching indicates the processor receiving instructions. The processor then decodes this instruction into micro operations in the decode stage. This micro operation is then executed in the execute stage. Data is read during the memory stage and, if required, the memory is then committed in the writeback stage. The writeback and the memory stage can technically be combined into a single process, since it can be generalized to load and store memory.

![alt text](./images/writeback.png)

A pipeline can be fetching instructions while the other pipeline decodes or executes another instruction. In essence, multiple processes can instructions can happen at once. Non-pipelined systems would result in one instruction waiting for another instruction to complete, which results in significant delays. Having multiple instructions run concurrently results in better utilization of the resources.

In cases where there are branches in the code, branch instructions can take a significant amount of time to execute. Often, the branch is determined later in the processes, such as decode or execute. A branch prediction is made based on previous executions to avoid waiting for the branch before fetching the instruction. Issues might arise if the branch hasn’t been determined until the execution state. This is due to the fact that memory is read and committed in the following stages. A clear decision has to be made whether the branch is mispredicted or not.
  
---
## Micro ops

One detail that matters when you start looking at PMU counters is that the CPU does not directly execute your x86 instructions. The frontend first decodes them into micro-operations (µops), which are the smaller internal steps the core actually schedules and runs.

A single x86 instruction can expand into multiple µops. For example, a memory add like:

```
add dword ptr [rdi], eax
```

is effectively doing a load, an add, and a store, so it may decode into 3 micro ops:

```
µop 1: load [rdi]
µop 2: add with eax
µop 3: store back to [rdi]
```

---

## TMA Hierarchy / Slot Buckets

TMA (Top-Down Microarchitecture Analysis) uses PMU events to estimate how the CPU’s pipeline slots are being used, then reports the result as fractions of total throughput. The idea is simple: every cycle has a limited number of pipeline slots where useful work could have happened, and TMA tells you where those slots went.

![alt text](./images/hierarchy.png)

At the top level, TMA groups pipeline slots into four buckets (If you need more detail, you travel down the TMA hierarchy to break a broad category into more specific causes):

- **Frontend bound:** slots are empty because the frontend is not delivering µops fast enough (fetch, decode, or instruction supply problems).

- **Backend bound:** µops are available, but the backend cannot make progress (often waiting on memory or limited by execution resources).

- **Bad speculation:** the CPU did work that got thrown away, most commonly due to branch mispredictions, so those slots did not produce retired results.
Retiring: useful work that is actually completed and committed to the CPU’s architectural state.

This breakdown immediately tells you whether you should be thinking about instruction supply (frontend), execution and memory behavior (backend), branch behavior (bad speculation), or whether the core is mostly doing productive work (retiring).

![alt text](./images/tma.png)

---
## Causes for Each Pipeline Stall Category

### **Backend Bound**

Backend bound issues are usually due to the backend running out of resources to execute, causing latency. Backend bound is divided into memory-bound and core-bound issues. For the most part, backend bound issues are under memory bound. Cache misses and memory accesses usually cause a high memory-bound. 

### **Front-End bound**

Frontend bound is mostly due to the frontend’s inability to fill a slot with a micro operation. It is broken down into latency and bandwidth. Latency occurs when there are no micro operations being issued by the frontend while the backend is waiting for any instructions. Bandwidth is from having less than 4 micro operations issued per cycle, which is an inefficient use of the frontend. This is because it is assumed that 4 micro operations are being issued per cycle, and under 4 micro operations per cycle indicates resources being underutilized. 

### **Bad speculation**

Bad speculation is mostly by micro operations not retiring. It is usually caused by branch mispredictions. The pipeline is busy fetching and executing operations that end up being useless. Branch mispredictions can also happen due to error handling. This would be cases such as try-catch statements to catch errors that might happen in the code. These cases are uncommon and often unintended, and aren't necessarily negatively affecting the performance.

### **Retiring**

Lastly, once a micro operation is executed, it falls under the retiring category. The results of the micro operation are committed to the architectural state, which are either CPU registers or main memory. In ideal cases, a majority of the pipelines fall under this category, since it means that useful work is actually being committed to memory.

The question is how much of each category is considered “good” for a program. Intel recommends around 50% retiring with 20% backend bound for client or desktop applications, with lower ranges of 10 ~ 30% retiring for server, database, or distributed applications.

![alt text](./images/ranges.png)

---

## Resolving Bugs in Each Pipeline Stall Category

### **Backend Bound Issues**

The majority of issues will be back-end bound. To begin, you want to first determine if the issue is core-bound or memory-bound, which can be determined using the various tools mentioned later in this week's summary.

For memory-bound issues, which are the most common, there are several solutions depending on the issue.

- Latency Bottleneck
  - Use prefetching to bring data into the cache before the CPU asks for it.
- Improving Throughput
  - Use batching/tiling to process data into chunks that fit into the L1/L2 cache to avoid constant trips to RAM.
  - Have more spatial locality by ensuring that data is stored sequentially in memory
- Data Layout
  - Make hot data smaller by using SoA, which is a structure of arrays rather than an array of structures. This is better for vectorization.
  - Use smaller data types if possible
  - Split structs into hot and code structs. Rarely used data (cold) should be kept out of the main structure to keep frequently accessed data (hot) compact and cache-friendly.

Core-bound issues typically occur when the CPU is limited by its use of execution units or the speed of the instruction dependency chain.

- Compute Limits:
  - Reduce complexity by moving expensive operations out of hot code blocks
  - Vectorization: use SIMD (single instruction, multiple data) to perform the same operation on multiple data points simultaneously 
- Dependency Chains
  - Use multiple accumulators to break up dependency chains for expensive operations. This can be done through using separate variables to store intermediate results of a calculation. 
  - Use the -O3 flag

> **A note about the -O3 flag:**
> The compiler will perform helpful optimizations like loop unrolling and vectorization **ONLY** when it can ensure that it is fault-proof. If it cannot guarantee that its optimization will not change how the program functions, it will not optimize. Therefore, when using the -O3 flag you should ensure that your code is optimizable (or do the optimizations you expect yourself) 

