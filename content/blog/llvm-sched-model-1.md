+++
title = "Scheduling Model in LLVM - Part I"
draft = true
tags = ['llvm', 'compiler-instruction-scheduling']
+++

Instruction scheduling is essential to modern compilers. It tries to hide latencies and increases the throughput of a straight line code by reordering the enclosing instructions.
In order to do that, compilers have to know a whole bunch of information, ranging from individal instruction's latency to microarchitecture details. The system that describes these is called a scheduling model. In this post, I'm going to briefly talk about LLVM's scheduling model.

In LLVM, scheduling models are not only used by the instruction schedulers, but also target-specific optimizations like MachineCombiner and components like MCA (Machine Code Analyzer). It should have been used in more places like instruction selection and cost models in the middle end, but that's a story for another day.
Here, I'm going to focus on the scheduling model itself. It'll be dull to talk about their abstractions so let's look at a really basic example first.

### The Basics
Scheduling models are part of the target definition. These models are associated with one or more target processors. Take RISC-V as an example, we can find the scheduling model for SiFive's P600 series processors, `SiFiveP600Model`, in `llvm/lib/RISCV/RISCVSchedSiFiveP600.td`.
Multiple processors can also share the same scheduling model.
For instance, in RISC-V, CPU `sifive-u74` and `sifive-x280` both use a scheduling model called `SiFive7Model`, whose definition resides in `llvm/lib/Target/RISCV/RISCVSchedSiFive7.td`.

To describe per-instruction scheduling information, like latency, naively we can express such information with descriptions like _"opcode ADD has latency of X"_. It's all fun and games until you realize that there are tens of thousands of opcodes in some of the targets[^1] so it doesn't scale well. More importantly, many instructions share the same scheduling characteristics.
For example, simple arithmetic operations like add, sub, and shifts usually have the same latency in most modern architectures.

[^1]: Interestingly, RISC-V has one of the biggest sets of opcodes. A large portion of them are pseudo instructions for RVV.

Instead of associating to opcodes, LLVM chooses a different path that characterizes an instruction's scheduling properties with **operand reads and writes**:
each operand in an instruction is assigned with a "token" that represents the hardware resources (e.g. ALU, FPU) this operand uses, along with info like the number of cycles it will consume.

Take the [ADD instruction](https://github.com/llvm/llvm-project/blob/e70f376b25ea96f3b0db75ff77ae1a58d53f2119/llvm/lib/Target/RISCV/RISCVInstrInfo.td#L666) in RISC-V shown below as an example, `WriteIALU` is the token assigned to the first operand, which is also the destination register hence a _write_ operand (also called **definition**), while the two `ReadIALU` follow are tokens for the two source registers, both are _read_ operands (or **use**).
```c++
// From llvm/lib/Target/RISCV/RISCVInstrInfo.td.
def ADD  : ALU_rr<0b0000000, 0b000, "add", Commutable=1>,
           Sched<[WriteIALU, ReadIALU, ReadIALU]>;
```
A write token is a `SchedWrite` TableGen instance, while a read token is a `SchedRead` instance.

Now we know that each instruction has their own `SchedWrite` and `SchedRead` tokens.
What a scheduling model does is assigning these tokens processor-specific information, like _latency_ and the _hardware resources_ they use.

Take `SiFive7Model` we saw earlier as an example, in the file where the model is defined (i.e. `llvm/lib/Target/RISCV/RISCVSchedSiFive7.td`), `WriteIALU` is assigned with a latency of 3 cycles:
```c++
// Integer arithmetic and logic
let Latency = 3 in {
def : WriteRes<WriteIALU, [SiFive7PipeAB]>;
...
}
```
In addition to latency, the snippet above also shows a _mapping_ from `WriteIALU` to the (abstract) hardware resource it uses -- `SiFive7PipeAB`.

What about `ReadIALU`, the `SchedRead` instances we saw earlier? By default, LLVM's scheduling model assumes that operand reads finish instantly, so a `SchedRead` cannot be assigned a latency property nor consuming any cycle the same way as a `SchedWrite`.

Which means that effectively, write operands are expected to repersent majority of the changes made by its enclosing instruction on processor states, and thus dictate the instruction's scheduling properties. So, it's safe to say that in this case, an `ADD` instruction has a latency of 3 cycles.

Alright, what we've covered so far is how you specify the latency of an instruction in the most basic way.
I have intentionally left many details aside, like what exactly is a hardware resource (i.e.g `SiFive7PipAB`) or other instruction properties like `AcquireAtCycle`. These details are strongly related to what we're going to cover in the next section: the **microarchitecture** of a processor.

### Modeling microarchitecture
Recall the goals of instruction scheduling are to minimize latency, maximize throughput, and eliminate as many potential hazards in the processor's pipeline as possible.
These goals have a lot to do with how instructions _move_ within the processor's pipeline. For instance, extra cycles might be spent on fulfilling RAW (Read after Write) data dependencies, or certain units, like an integer division unit, might be overwhelmed and stall the entire pipeline.

And this is where the processor's microarchitecture comes into play. Specifically, we're focusing on **superscalar** architecture here, which has a typical structure shown below:
<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-sched-model-basic-superscalar.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-sched-model-basic-superscalar.light.svg">
  </picture>
</div>

The above structure has an issue width of 3, meaning it can feed at most 3 instructions to later stages at a time; micro-codes, or _uops_, are generated at the decode stage; at this moment we're a little hand-waving on the definition of dispatch and issue, as we'll see several of their variants later; there are three different **functional units** in the execution stage for integer, floating point, and memory instructions (load / store unit). All of them are fully pipelined.

Despite the fact that scheduling model has strong connections with microarchitectures, we don't want our model to cover _too_ many microarchitecture details -- it's simply too expensive! What we want is a model that can raise red flags when we come up with a bad scheduling, yet abstract away all the nitty-gritty details and present just the information we care.

"What kind of red flags?" you may ask. The most important thing is probably whether we have enough _resources_ to even run an instruction. Remember, the whole idea of superscalar is to run instructions of different types in different function units **in parallel**: integer uops go into integer units and floating point uops go into another floating point unit, where these units can run independently.
Each of these units has one or more _pipes_ that actually run the uops. In many places "functional units" and "pipes" are interchangable.
If there is no enough resource, we won't be able to **dispatch** an uop into any of the pipes. After all, processors have only a limited number of pipes for a specific type of instructions.

In LLVM, a `ProcResource` instance in TableGen represents a unit that can executes uops, hence an equivalent to a single pipe in most cases. But it's not the only instance you can _dispatch_ uops to, because sometimes you can dispatch uops into a _collection_ of pipes in which any of the pipes in that group can execute the dispatched uop. Such groups are represented by `ProcResGroup` instances in TableGen. Here is an example in `llvm/lib/Target/RISC/RISCVSchedSiFiveP400.td`:
```c++
def SiFiveP400IEXQ0       : ProcResource<1>;
def SiFiveP400IEXQ1       : ProcResource<1>;
def SiFiveP400IEXQ2       : ProcResource<1>;
...
def SiFiveP400IntArith    : ProcResGroup<[SiFiveP400IEXQ0, SiFiveP400IEXQ1, SiFiveP400IEXQ2]>;
```
Here, it shows that SiFive's P400 series processors have 3 pipes for executing integer operations, denoted by `SiFiveP400IEXQ0`, `SiFiveP400IEXQ1`, and `SiFiveP400IEXQ2`. In most cases we don't care which of these three pipes we actually dispatch uops into. So majority of the simple integer instructions in RISC-V, like `ADD` we've seen earlier, assign `SiFiveP400IntArith` -- a group of **all** integer pipes -- to `WriteIALU`, the `SchedWrite` token for the said simple integer instructions.

Here is the assignment of `SiFiveP400IntArith` to `WriteIALU` in the same `RISCVSchedSiFiveP400.td` file:
```c++
let Latency = 1 in {
// Integer arithmetic and logic
def : WriteRes<WriteIALU, [SiFiveP400IntArith]>;
...
}
```

When there is no available pipe, a stall or a hazard occurs, in this case it is considered a _dispatch_ hazard since you can't dispatch more instructions. To avoid such hazards, each pipe usually has a buffer or a queue to store a certain number of candidates. Knowing the **size** of that buffer is crucial to our scheduling model, because then we can make a wiser decision on distributing the instructions _evenly_ across all pipes, rather than jamming them into a single one just like freeways in LA.

Most[^2] processors with buffers / queues like this have a more well-known functionality: reordering instructions. In an our-of-order processor, each of these buffers is used as some sort of staging area for the scheduler to preemptively **issue** instructions with no dependencies on the others, in the hope of hiding latencies.

[^2]: Superscalar and out-of-order-ness are two independent concepts. Superscalar is about increasing the "width" of your execution units; out-of-order is about the order of executions. You can have in-order superscalar processors, which are pretty common, as well as out-of-order non-superscalar processors, which barely exist but no one can stop you from making it.

Aside from the size of the buffer, how to _organize_ these buffers (e.g. several pipes might share a buffer) is just as important as their sizes. In the next section, we're gonna go through processor resource buffers of different sizes and layouts.

#### Processor resource buffers
Let's sort out some terminologies first. The buffer, or **scheduler queue** we're talking about here, is often known as a **reservation station** too.
Reservation station stores instructions whose _operands_ might not be ready yet, until those operands are available. **Reorder buffer (ROB)**, on the other hand, is a kind of unit with slightly confusing name. ROB reorders the _results_ of an instruction; it's usually depicted at the end of the execution stage, as shown in the previous diagram. Nevertheless, we need to reserve a slot in the ROB _before_ we can even dispatch an instruction, therefore, to some extent, the size of ROB also controls how many instructions are allowed into the execution stage.

Back to the buffers, in LLVM's scheduling model, we use the `BufferSize` field in a `ProcessorResource` or a `ProcResGroup` instance to specify the size of the associated buffer.

Special values are also used to denote unconventional buffer layouts, or even the _absent_ of buffers. Here is the list of them.

##### Decoupled reservation stations
<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-sched-model-decoupled-reservation-stations.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-sched-model-decoupled-reservation-stations.light.svg">
  </picture>
</div>

When we assign a buffer size to an individual processor resource or processor resource group, we're modeling processor resources with their own scheduler queues. Here is an example from PowerPC POWER9's [scheduling model](https://github.com/llvm/llvm-project/blob/22c06aa5e94e30fb1333ecaf46ce33c65d148634/llvm/lib/Target/PowerPC/PPCScheduleP9.td#L123):
```c++
// Only one Branch unit.
def BR : ProcResource<1> {
  let BufferSize = 16;
}
```
It shows that the execution unit for branch instructions has a scheduler queue of 16 entries.

A more common arrangement is assigning `BufferSize` on a processor resource _group_.
For instance, in AMD Zen2 CPU, there are total of 4 general integer execution units with their own schedulers; each scheduler has 16 buffer entries. In Zen2's LLVM scheduling model, these schedulers are represented by 4 `ProcessorResource` instances: `Zn2ALU0` ~ `Zn2ALU3`.
Despite the fact that there are 4 separate schedulers in real hardware, Zen2' scheduling model doesn't assign `BufferSize = 16` to each `Zn2ALUX` instance, but instead group all 4 instances into a single `ProcResGroup` called `Zn2ALU` and [assign](https://github.com/llvm/llvm-project/blob/22c06aa5e94e30fb1333ecaf46ce33c65d148634/llvm/lib/Target/X86/X86ScheduleZnver2.td#L75) a buffer size of 16 * 4:
```c++
// 64 Entry (16x4 entries) Int Scheduler
def Zn2ALU : ProcResGroup<[Zn2ALU0, Zn2ALU1, Zn2ALU2, Zn2ALU3]> {
  let BufferSize=64;
}
```
The rationale behind this is that most integer instructions map `Zn2ALU`, rather than a specific `Zn2ALUX`, to their `SchedWrite` token. Because they simply don't care which `Zn2ALUX` they are going to run on. Therefore, it makes more sense to specify the buffer size on the processor resource group.

##### Unified reservation station
This is when `BufferSize` equals to **-1**, which is also the default resource buffer kind. As illustrated in the diagram below, all the processor resources effectively share the same buffer.

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-sched-model-unified-reservation-station.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-sched-model-unified-reservation-station.light.svg">
  </picture>
</div>

In this case, the actual size of such buffer is dictated by `MicroOpBufferSize`, which is a global attribute to the entire scheduling model rather than individual processor resource. To be more specific, it's a field in the `SchedMachineModel` TableGen class.

This `MicroOpBufferSize` attribute is primarily used to throttle the number of uops to be _dispatched_. There are, however, many factors that control this throttle at the same time. Notably, ROB and register renaming. We've explained the effect of ROB earlier; for register renaming, if there aren't enough physical registers for us to rename, a dispatch hazard also occurs. Consequently, the value of `MicroOpBufferSize` should be the minimal of reorder buffer size, size of register renaming pool, and the actual size of unified reservation station.

##### In-order core
So far we have been discussing out-of-order cores, which is one of the primary reasons why we need buffers in the first place. But in-order cores are still a thing, because it dramatically simplifies the chip design and reduces the area, which are top on the list for products like embedded devices. It's also getting more popular in cases where you want save areas for specialized units like BIG vector units or even matrix multiplication units. SiFive's X280, which we've seen its scheduling model previously, and X390 are good examples, where they save area by adopting in-order design and enjoying a whopping 512- and 1024-bit vector, respectively.

For in-order cores, you simply set `BufferSize` to zero. The same goes for the (global) `MicroOpBufferSize` attribute.
When an uop is handed to an in-order unit in LLVM's scheduling model, _dispatch_ and _issue_ are conisdered happening at the same time.

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-sched-model-in-order.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-sched-model-in-order.light.svg">
  </picture>
</div>

The uop then holds the processor resource until `ReleaseAtCycle` before another uop can be dispatched. In other words, the younger uop would encounter a dispatch hazard until `ReleaseAtCycle` has passed in the older uop.

##### Latency resource
When `BufferSize` equals to 1, we create a unique kind of resource also known as "latency device". Resources of this kind act just like an in-order pipeline, except two things:
  1. Since there is still a buffer, albiet being a really small one, a younger uop waits in the buffer until the older uop finishes, rather than encounters a dispatch hazard.
  2. LLVM's instruction scheduler always treats two adjacent uops that use this resource as producer and consumer (even if there is no data dependency between them). Meaning, the younger uop always waits `Latency` cycles after the old uop was issued -- as opposed to waiting until `ReleaseAtCycle` has passed in a normal in-order pipeline -- before it can be issued.

This kind of resource was designed to model in-order units within an out-of-order core. It's most commonly used for modeling **un-pipelined** units nowadays (which, of course, is in-order). Yes, it's 2024 and there are still computations that are difficult to be pipelined, most notably integer -- sometimes even floating point -- _divisions_. It is possible to make integer divisions pipelined, but many of the times it's not worth the chip area.

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-sched-model-latency-device.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-sched-model-latency-device.light.svg">
  </picture>
</div>

Some examples: Samsung Exynos M5's serialized (i.e. un-pipelined) integer division unit, represented by `M5UnitD` from [here](https://github.com/llvm/llvm-project/blob/5262865aac683b72f3e66de7a122e0c455ab6b9b/llvm/lib/Target/AArch64/AArch64SchedExynosM5.td#L41), has a buffer size of 1:
```c++
let Super =  M5UnitC, BufferSize = 1 in
def M5UnitD  : ProcResource<1>; // Integer division (inside C0, serialized)
```

RISC-V's Rocket chip [marks](https://github.com/llvm/llvm-project/blob/fdb9f96fa2a926425bdf8315048db7623d63547d/llvm/lib/Target/RISCV/RISCVSchedRocket.td#L41) their integer and floating point division units, `RocketUnitIDiv` and `RocketUnitFPDivSqrt`, as latency devices too:
```c++
let BufferSize = 1 in {
def RocketUnitIDiv       : ProcResource<1>; // Int Division
def RocketUnitFPDivSqrt  : ProcResource<1>; // FP Divide/Sqrt
}
```