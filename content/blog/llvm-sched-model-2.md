+++
title = "(Archive) Scheduling Model in LLVM - Part II"
date = "2024-08-17"
draft = true
tags = ['llvm', 'compiler-instruction-scheduling']
+++

In the [previous post](/llvm-sched-model-1), we covered the basics of scheduling model in LLVM. Specifically, per-operand tokens that connect an instruction with models that spell out processor-specific scheduling properties like instruction latency, and the concept of processor resources with different sizes of buffer.

In this post, I'll bring up some more advanced properties and focus on how scheduling models are actually _used_ in other parts of LLVM, specifically, **MCA (Machine Code Analyzer)** and **MachineScheduler**. It is particularly important to learn the usages of scheduling models, because LLVM actually doesn't have a formal spec for everything we have been talking so far, for example, the meaning of different buffer sizes. Those are coming from how other components _interpret_ these models[^1].

[^1]: There are, of course, code comments on scheduling models and without a doubt they are good source of knowledge. But they are neither formal specs nor always being consistent across the entire codebase.

By the end of this post, you'll get a better idea on how different scheduling model properties affect the instruction scheduling quality as well as the precision of other LLVM tools like `llvm-mca`. First, let's take a look at some more advanced properties in LLVM's scheduling model.

### Advanced scheduling model properties
Previously, we've learned `ProcResource` and `ProcResGroup`. The former more or less represents a single execution unit in a superscalar processor, while the latter is a logical group of `ProcResource` where an instruction can be assigned to **one of those** (sub-)resources. In other words, `ProcResGroup` represents a _hierarchical_ structure.

Here, we're going to introduce another kind of hierarchical structure: super resource.
Let's explain it with an example from [AMD Zen3's Load / Store Unit (LSU)](https://github.com/llvm/llvm-project/blob/c503758ab6a4eacd3ef671a4a5ccf813995d4456/llvm/lib/Target/X86/X86ScheduleZnver3.td#L368):
```c++
def Zn3LSU : ProcResource<3>;

let Super = Zn3LSU in
def Zn3Load : ProcResource<3> {
  ...
}

let Super = Zn3LSU in
def Zn3Store : ProcResource<2> {
  ...
}
```
`Zn3Load` and `Zn3Store` are processor resources representing the load and store units, respectively. Both of them designate `Zn3LSU`, which represents the _entire_ LSU, as their **super resource** via the `Super` field.

At this point, you might be wondering: "why do we need another `Zn3LSU` when we've already covered the load and store parts separately through `Zn3Load` and `Zn3Store`?"

To clarify this, let's first look at the actual LSU in [Zen3's microarchitecture](https://chipsandcheese.com/2022/11/05/amds-zen-4-part-1-frontend-and-execution-engine/zen3-drawio/) shown below.

<figure style="text-align: center;">
  <img src="/images/zen3-uarch-lsu.png">
  <figcaption>Image source: <a href="https://chipsandcheese.com/2022/11/05/amds-zen-4-part-1-frontend-and-execution-engine/">Chip and Cheese</a>. Captured from the <a href="https://chipsandcheese.com/2022/11/05/amds-zen-4-part-1-frontend-and-execution-engine/zen3-drawio/"> original image </a>.</figcaption>
</figure>

In the figure above, there are three arrows between load & store queues and L1 Data Cache, along with an equal number of AGUs (Address Generation Unit) positioned above the queues.
Now, you might notice that among all three arrows between the queues and L1 Data Cache, only two of them goes down (indicating stores) while there are three going up (indicating loads).

This reveals that all three available "channels" are capable of loading data, while only two of them (don't care which two of them though) can store data. Importantly, each channel can either load or store data at any given time, but not both simultaneously.
To put it differently, a LSU channel can either be _allocated_ as a load or store channel, while no more than two store channels are allowed to exist at any given time.

Ooor, we can say that a load or store channel is simply an **alias** for one of the three total channels in LSU!

This alias relationship is precisely what a super resource is meant to describe. `Zn3LSU` represents all the available channels in the entire load / store unit, with the number of channels indicated by the number of units in its `ProcResource` declaration (i.e. the template parameter in `ProcResource<3>`). `Zn3Load` and `Zn3Store`, on the other hand, serve as aliases for the load and store channels, respectively. The number of load and store channels is also reflected in their `ProcResource` declarations, which is why `Zn3Store` was declared with `ProcResource<2>`.

When an instruction consumes `Zn3Store`, it effectively allocates one of the two units from `Zn3LSU`. Similarly, when an instruction consumes `Zn3Load`, it can choose from any of the three units in `Zn3LSU`. Of course, once a unit is allocated, it cannot be used for other loads or stores until it’s released.

The concept of super resource might sound really similar (and maybe confusing at first glance) to that of `ProcResGroup`: both of them are kind of describing a group; both of them are effectively picking a single unit / resource from a pool of available options upon consumed. The biggest difference is that these "options" are _separate_ and largely independent `ProcResource` instances in `ProcResGroup`'s case. In super resource's case, however, options are a limited number of units from the _same_ `ProcResource` (e.g. `Zn3LSU`), and the resource consumption of one alias (e.g. `Zn3Store`) will affect other aliases derived from the same super resource (e.g. `Zn3Load`).

#### Number of units in a ProcResource
Despite seeing things like `ProcResource<1>` or `ProcResource<2>` multiple times, so far we've been really hand-waving on the actual meaning of `ProcResource`'s first template argument -- the number of units in this resource.

In short, it specifies the maximum number of uops a `ProcResource` can handle in a single cycle. Namely, the maximum throughput. For instance, previously we've seen that `Zn3LSU` can handle at most three memory uops. Similarly, in [SiFiveP600 scheduling model](https://github.com/llvm/llvm-project/blob/d4c519e7b2ac21350ec08b23eda44bf4a2d3c974/llvm/lib/Target/RISCV/RISCVSchedSiFiveP600.td#L75), its LSU, `SiFiveP600LDST` can handle at most two loads or stores:
```c++
// Two Load/Store ports that can issue either two loads, two stores, or one load
// and one store (P550 has one load and one separate store pipe).
def SiFiveP600LDST       : ProcResource<2>;
```
When a uop is dispatched to a resource of this kind, it is effectively dispatched to _one_ of its units.

Sounds familiar? That's right! in this sense, it is really similar to `ProcResGroup` we saw previously. In fact, `ProcResGroup` has an "implicit" number of units as well, and it's (unsurprisingly) equal to the number of sub-resource it contains.

The line between when to use `ProcResGroup` and when to use `ProcResource<N>` is somewhat blurred and in many cases they're interchangeable. For example, in the following figure we have an integer execution unit with three pipes: two of them are capable of doing multiplications, while division and cryptography are each restricted to a single pipe. All three of them, however, can do basic ALU operations. Each pipe has a buffer size of 16. In other words, this is a **decoupled reservation station** style structure we talked about in the [previous post](/llvm-sched-model-1).
<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-sched-model-hierarchy-example.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-sched-model-hierarchy-example.light.svg">
  </picture>
</div>

The first option we have is using `ProcResGroup`:
```c++
let BufferSize = 16 in {
  // ALU, Mul
  def IEX0 : ProcResource<1>;
  // ALU, Div
  def IEX1 : ProcResource<1>;
  // ALU, Mul, Crypto
  def IEX2 : ProcResource<1>;
}

def IntegerMul   : ProcResGroup<[IEX0, IEX2]>;

def IntegerArith : ProcResGroup<[IEX0, IEX1, IEX2]>;
```
Then we have the following resource assignments:
|          **Operations**          |  **Resources** |
|:--------------------------------:|:--------------:|
|          Multiplications         |  `IntegerMul`  |
|             Divisions            |     `IEX1`     |
|           Cryptography           |     `IEX2`     |
| Every other (simple) arithmetics | `IntegerArith` |


Option two, on the other hand, uses the super resource to build this hierarchy:
```c++
def IEX : ProcResource<3> {
  let BufferSize = !mul(16, 3);
}

let Super = IEX in {
  def IntegerMul : ProcResource<2> {
    let BufferSize = !mul(16, 2);
  }

  def IntegerDiv : ProcResource<1> {
    let BufferSize = 16;
  }

  def Crypto : ProcResource<1> {
    let BufferSize = 16;
  }
}
```
And here is its resource assignments:
|          **Operations**          |  **Resources** |
|:--------------------------------:|:--------------:|
|          Multiplications         |  `IntegerMul`  |
|             Divisions            |  `IntegerDiv`  |
|           Cryptography           |    `Crypto`    |
| Every other (simple) arithmetics |     `IEX`      |

### MCA

<!-- The narration goes by visualizing the scheduling models. -->

### MachineScheduler

<!--
Highlights: how MachineScheduler abstracts away some details (as opposed to MCA's fidelity of actual microarchitectures) and focusing on the parts that actually affect instruction scheduling.

Also, visualizing instruction scheduling with MachineScheduler's debug output.
-->

So far, we have learned that LLVM's scheduling model is essentially a high-level _abstraction_ over the microarchitecture of modern processors. The decision on what properties to capture (e.g. instruction latency, resource buffer size etc.) is largely dictated by its users, for instance, LLVM MCA from the previous section. In the last section of this series, we are going to cover MachineScheduler, LLVM's instruction scheduler that majority of the scheduling model was originally designed for. Unlike LLVM MCA which only analyzes machine codes, MachineScheduler actually **changes** them. Thus, it plays an even more instrumental role in the codegen quality and of course, the performance.

While we're not interested in the details of MachineScheduler's instruction scheduling algorithm here, it's useful to know the outline of its workflow, which is depicted in the diagram below.

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-sched-model-machinescheduler.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-sched-model-machinescheduler.light.svg">
  </picture>
</div>

At the beginning of the scheduling, MachineScheduler uses a DAG to model all of the instructions in a single (machine) basic block. Each node in this DAG, `ScheduleDAG` shown on the left-hand-side of the figure, is known as `SUnit`. In MachineScheduler, it contains a single machine instruction (`MachineInstr`)[^2] and other metadata.

[^2]: `SUnit` is used in both SelectionDAG-based schedulers (e.g. `ScheduleDAGRRList`) and MachineScheduler. In the former, a `SUnit` stores `SDNode` / `SDValue`; in the latter, it stores `MachineInstr`.

On the right-hand-side of this figure, we have the target machine basic block (MBB) in which MachineScheduler may schedule instructions from both top-down and bottom-up directions. Each direction is controlled by a _schedule boundary_, which keep tracks of all the instructions that have been scheduled so far and their states in the corresponding "regions". It works with `tryCandidate` at the center of the figure to pick the optimal `SUnit` from the DAG. The states in each boundary are then updated by the `schedule` function once a `SUnit` is selected and scheduled into the target MBB.

Here, we're especially interested in how `tryCandidate` selects a `SUnit`, and hopefully an _optimal_ one. This `SUnit` selection process is _dynamic_ and is depending on the current state of the schedule boundary. For instance, when the register pressure within this boundary is too high we'll select a `SUnit` that doesn't stretch the live intervals; if a certain processor resource has been fed with too many instructions, we pick a `SUnit` that balanced the overall resource usages. Sometimes, it's also affected by the code we're scheduling right now, like having a different selection strategy in loop bodies.

To give you a better idea on how this works and more importantly, the role of scheduling model in this process, we're going to use the following RISC-V assembly snippet to explain how MachineScheduler schedule the enclosing instructions to optimize their overall **latency** and **resource usages**.
```
sample:
  mul     s1, s1, s0                # SU(0)
  vsetvli a0, a0, e64, m2, ta, ma   # SU(1)
  vadd.vv v8, v8, v14               # SU(2)
  vadd.vv v10, v10, v8              # SU(3)
  vadd.vv v8, v14, v10              # SU(4)
  add     a0, s1, s1                # SU(5)
  add     a0, a0, s1                # SU(6)
  vadd.vx v8, v8, a0                # SU(7)
  ret
```
In this snippet, three simple vector additions (`vadd.vv`) produce a vector is summed with three scalar instructions -- consisting of `add` and `mul` -- using a `vadd.vx` instruction (let's just ignore the `vsetvli` instruction here...).
Each of these instructions is also assigned with a `SUnit` shown in the `SU(N)` comment on the right. To make the relationships between these instructions / `SUnit` more clear, here is a diagram of the data flows between them.

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-sched-model-misched-dataflow.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-sched-model-misched-dataflow.light.svg">
  </picture>
</div>

Now, as much as I would like to use this snippet as the input to our scheduler, we actually have to use the _MachineIR_ version of it instead as the input, which unfortunately looks a bit scary:
{{% details "The MachineIR snippet"%}}
```
---
name:            sample
tracksRegLiveness: true
body:             |
  bb.0.entry:
    liveins: $x8, $x9, $x10, $v8m2, $v10m2, $v12m2, $v14m2

    # SU(0)
    renamable $x9 = MUL killed renamable $x9, killed renamable $x8
    # SU(1)
    renamable $x10 = PseudoVSETVLI killed renamable $x10, 217 /* e64, m2, ta, ma */, implicit-def $vl, implicit-def $vtype
    # SU(2)
    renamable $v8m2 = PseudoVADD_VV_M2 undef renamable $v8m2, killed renamable $v8m2, renamable $v14m2, $noreg, 6 /* e64 */, 0 /* tu, mu */, implicit $vl, implicit $vtype
    # SU(3)
    renamable $v10m2 = PseudoVADD_VV_M2 undef renamable $v10m2, killed renamable $v10m2, renamable $v8m2, $noreg, 6 /* e64 */, 0 /* tu, mu */, implicit $vl, implicit $vtype
    # SU(4)
    renamable $v8m2 = PseudoVADD_VV_M2 undef renamable $v8m2, killed renamable $v14m2, killed renamable $v10m2, $noreg, 6 /* e64 */, 0 /* tu, mu */, implicit $vl, implicit $vtype
    # SU(5)
    renamable $x10 = ADD renamable $x9, renamable $x9
    # SU(6)
    renamable $x10 = ADD killed renamable $x10, killed renamable $x9
    # SU(7)
    renamable $v8m2 = PseudoVADD_VX_M2 undef renamable $v8m2, killed renamable $v8m2, killed renamable $x10, $noreg, 6 /* e64 */, 0 /* tu, mu */, implicit $vl, implicit $vtype
    PseudoRET implicit $v8m2

...
```
{{% /details %}}

Save this as `input.mir` and run the following command
```
$ /path/to/bin/llc -mtriple=riscv64 -mcpu=sifive-p670 \
    -run-pass=postmisched -misched-topdown \
    -misched-dump-schedule-trace \
    -debug-only=machine-scheduler \
    input.mir -o output.mir &> dump.txt
```
In this command we run the (PostRA) MachineScheduler with SiFive P670's scheduling on `input.mir`. The `ouptut.mir` carries the scheduling result, which is a MachineIR equivalent to the following assembly snippet:
```
sample:
  mul     s1, s1, s0                # SU(0)
  vsetvli a0, a0, e64, m2, ta, ma   # SU(1)
  vadd.vv v8, v8, v14               # SU(2)
  vadd.vv v10, v10, v8              # SU(3)
  add     a0, s1, s1                # SU(5)
  vadd.vv v8, v14, v10              # SU(4)
  add     a0, a0, s1                # SU(6)
  vadd.vx v8, v8, a0                # SU(7)
  ret
```
Notably, `SU(4)` and `SU(5)` are swapped.

This command also prints MachineScheduler's _trace_ into `dump.txt`. This trace allows us to peak inside MachineScheduler's internal, including the selection process of `SUnit` we're interested in.
Each step in this selection process emits trace starts with _"** ScheduleDAGMI::schedule picking next node"_, like the line below:
```
** ScheduleDAGMI::schedule picking next node
Queue TopQ.P:
Queue TopQ.A: 5 2
  TopQ.A RemainingLatency 0 + 0c > CritPath 6
  TopQ.A ResourceLimited: SiFiveP600IEXQ1
  Cand SU(5) ORDER
  Cand SU(2) TOP-DEPTH
Pick Top TOP-DEPTH
Scheduling SU(2) renamable $v8m2 = PseudoVADD_VV_M2 undef renamable $v8m2(tied-def 0), renamable $v8m2, renamable $v14m2, $noreg, 6, 0, implicit $vl, implicit $vtype
  Ready @0c
  SiFiveP600VectorArith +2x2u
TopQ.A @0c
  Retired: 3
  Executed: 1c
  Critical: 1c, 1 SiFiveP600IEXQ1
  ExpectedLatency: 0c
  - Resource limited.
```
This is the step where the scheduler is deciding the next `SUnit` to pick after `SU(0)` and `SU(1)` are scheduled.

There's a lot going on here, but let's focus on a few things first: `Queue TopQ.A` reflects the current state of the _available queue_ -- a queue that stores `SUnit` without any hazard to prevent it from being issued. Here we can see that `SU(5)` and `SU(2)` are both available at this moment. In other words, the scheduler has to make a choice between the first `vadd.vv` instruction (i.e. `vadd.vv v8, v8, v14`) and the first scalar addition (i.e. `add a0, s1, s1`)

To make that decision, it compares each candidate in the queue using `tryCandidate` as a custom comparison function. Think `tryCandidate` like the `std::sort` function's comparator -- it performs pairwise comparisons to determine if one candidate is better than the other.

And the "Cand SU(X) ..." lines show the details, notably the _reason_ why a candidate got picked, of each decision `tryCandidate` made:
  - _"Cand SU(5) ORDER"_ essentially says that `SU(5)` is considered the optimal choice at this moment because no other `SUnit` presents before it
  - _"Cand SU(2) TOP-DEPTH"_ says that `SUnit(2)` is considered better than the previous selection, `SUnit(5)`, because its *depth* is smaller than that of `SUnit(5)`. This is also the final decision in this step (i.e. the "Pick Top TOP-DEPTH" line that follows).

So what does it means by picking the candidate with smaller depth, and why do we do that?

The thing we tried to do here is keeping serialized instructions with long latency at bay. Serialized instructions are a sequence of instructions that have data dependency with each other. The depth is the maximum number of cycles to execute all its predecessors in this sequence, which is equal to the summation of *latency* along its longest data path. The concept of depth here is also known as **critical path**.

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-sched-model-misched-latency.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-sched-model-misched-latency.light.svg">
  </picture>
</div>

The idea is that if we keep scheduling instructions that extend the critical path, there will always be one instruction that runs sequentially for its entire latency[^3]. This reduces the overall degree of _parallelism_. Ideally, we want to keep the critical path as short as possible while scheduling as many instructions as we can without extending it.

[^3]: Ignoring pipeline bypass / read advanced here.

And that's why we pick `SUnit(2)` over `SUnit(5)`: from the illustration above we can learn that `SUnit(2)`'s depth is zero, since it doesn't have predecessor (in this block). On the other hand, `SUnit(5)` has a depth equal to `SUnit(0)`'s latency.

You can also find the depth of each `SUnit` in the same trace. Just scroll up to the beginning, where the lines under the _"MI Scheduling"_ banner detail each `SUnit`:
```
SU(0):   renamable $x9 = MUL renamable $x9, renamable $x8
  # preds left       : 0
  # succs left       : 2
  # rdefs left       : 0
  Latency            : 3
  Depth              : 0
  Height             : 6
  Successors:
    SU(6): Data Latency=3 Reg=$x9
    SU(5): Data Latency=3 Reg=$x9
...
SU(2):   renamable $v8m2 = PseudoVADD_VV_M2 undef renamable $v8m2(tied-def 0), renamable $v8m2, renamable $v14m2, $noreg, 6, 0, implicit $vl, implicit $vtype
  # preds left       : 2
  # succs left       : 3
  # rdefs left       : 0
  Latency            : 1
  Depth              : 0
  Height             : 4
  Predecessors:
    SU(1): Data Latency=0 Reg=$vl
    SU(1): Data Latency=0 Reg=$vtype
  Successors:
    SU(4): Out  Latency=0
    SU(4): Data Latency=1 Reg=$v8m2
    SU(3): Data Latency=1 Reg=$v8m2
...
SU(5):   renamable $x10 = ADD renamable $x9, renamable $x9
  # preds left       : 2
  # succs left       : 2
  # rdefs left       : 0
  Latency            : 1
  Depth              : 3
  Height             : 3
  Predecessors:
    SU(1): Out  Latency=0
    SU(0): Data Latency=3 Reg=$x9
  Successors:
    SU(6): Out  Latency=0
    SU(6): Data Latency=1 Reg=$x10
```
Here, `SUnit(2)`'s depth is unsurprisingly equal to zero due to the absent of predecessors; `SUnit(5)`'s depth, as expected, is equal to `SUnit(0)`'s latency -- 3 cycles.

And where does this 3 cycle come from? That's right, it is the latency assigned to the `mul` instruction [in P600 processor's scheduling model](https://github.com/llvm/llvm-project/blob/9ade4e2646bd52b49e50c1648301da65de90ffa9/llvm/lib/Target/RISCV/RISCVSchedSiFiveP600.td#L122), using the method we introduced in the previous post.
```cpp
let Latency = 3 in {
// Integer multiplication
def : WriteRes<WriteIMul, [SiFiveP600MulI2F]>;
def : WriteRes<WriteIMul32, [SiFiveP600MulI2F]>;
...
}
```
More specifically, from the [instruction definition](https://github.com/llvm/llvm-project/blob/9ade4e2646bd52b49e50c1648301da65de90ffa9/llvm/lib/Target/RISCV/RISCVInstrInfoM.td#L28) of `mul` we learn that it has `WriteIMul` associated with its output operand. Therefore, the TableGen snippet above effectively assigns a latency of 3 cycles to this instruction.

So far, we have seen how MachineScheduler uses **latency** information from the scheduling model to alleviate the pressure on (instruction-level) parallelism caused by long latency chains. Next, we're going to see how scheduler uses the **processor resource** info to achieve a more balanced resource usage.

-------

Recall that the whole idea of superscalar processor is to run instructions of different types _in parallel_, distributed among several different execution units like integer and floating point pipes we saw earlier. If an instruction stream is poorly organized such that a single type of instructions are executed consecutively in a large number, for instance 1000 integer ADDs running back-to-back, then a few execution units (processor resources) might be disproportionality overwhelmed while leaving others idle. 

It is true that out-of-order processors can address this problem by looking ahead and distributing different instruction types evenly among all resources. Nevertheless, the number of instructions it can look ahead, which is usually equal to the size of the scheduler buffer, is limited -- less than 30 instructions in most cases. Therefore, MachineScheduler has to shoulder some of the works here to ensure uniform distribution of workloads among different processor resources.

In the previous section, we picked the next SUnit to schedule based on their critical path, which is related to an instruction's **latency**. When scheduling for processor resource, however, all we care about is the instruction's **occupancy**. Occupancy is how long (in terms of cycles) an instruction "holds" a processor resource until the next instruction can use the same resource. Put it differently, since most processor resources / execution units are also pipelined, you can think of occupancy being the time an instruction spends on the first stage of this pipeline.

Does this concept sound familiar? That's right, actually we have already seen occupancy in the previous post already! Specifically, when we're explaining in-order processor resource:

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-sched-model-in-order.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-sched-model-in-order.light.svg">
  </picture>
</div>

In an in-order processor resource (`BufferSize = 0`), a younger uop (i.e. uOp2) has to wait until `ReleaseAtCycle` after its predesessor (i.e. uOp1) was issued -- and the occupancy of uOp1, as it turns out, is roughly equal to uOp1's `ReleaseAtCycle`.

More precisely speaking, the occupancy of uOp1 is equal to `ReleaseAtCycle - AcquireAtCycle`. `AcquireAtCycle` is yet another scheduling property of an instruction, representing the (relative) cycle when the instruction starts to hold a processor resource. This property is default to zero (making the occupancy equal to `ReleaseAtCycle`) unless a model opts in the latest interval-based scheduling strategy on in-order cores. Because the P670 core we've been studying here is an out-of-order core, we only look at `ReleaseAtCycle`.

Let's go back to the MachineScheduler trace. Previously, we saw how the MachineScheduler chose between `SU(2)` and `SU(5)` based on their critical paths. Here are the instructions we have scheduled so far:
```
  mul     s1, s1, s0                # SU(0)
  vsetvli a0, a0, e64, m2, ta, ma   # SU(1)
  vadd.vv v8, v8, v14               # SU(2)
  # We are here
```

Now, we're focusing on the next entry in the trace that explains how it picks the next SUnit:
```
** ScheduleDAGMI::schedule picking next node
Queue TopQ.P:
Queue TopQ.A: 5 3
  TopQ.A RemainingLatency 0 + 0c > CritPath 6
  TopQ.A ResourceLimited: SiFiveP600IEXQ1
  Cand SU(5) ORDER
  Cand SU(3) TOP-DEPTH                 1 cycles
Pick Top TOP-DEPTH
Scheduling SU(3) renamable $v10m2 = PseudoVADD_VV_M2 undef renamable $v10m2(tied-def 0), renamable $v10m2, renamable $v8m2, $noreg, 6, 0, implicit $vl, implicit $vtype
  Ready @1c
  SiFiveP600VectorArith +2x2u
  *** Critical resource SiFiveP600VectorArith: 2c
  TopQ.A TopLatency SU(3) 1c
  *** Max MOps 4 at cycle 0
Cycle: 1 TopQ.A
TopQ.A @1c
  ...
  Critical: 2c, 4 SiFiveP600VectorArith
  ExpectedLatency: 1c
  - Resource limited.
```
Based on what we learned in the previous section, we know from this entry that `SU(5)` and `SU(3)` are now the candidates (because _"Queue TopQ.A: 5 3"_).

We also know that MachineScheduler eventually picked `SU(3)` as the next SUnit to schedule due to shorter critical path (_"Cand SU(3) TOP-DEPTH"_) -- previously we saw that `SU(5)` has a critical path (depth) of 3 cycles, while `SU(3)` only has a depth of 1 cycle:
{{% details "SU(3)'s detailed properties"%}}
```
SU(3):   renamable $v10m2 = PseudoVADD_VV_M2 undef renamable $v10m2(tied-def 0), renamable $v10m2, renamable $v8m2, $noreg, 6, 0, implicit $vl, implicit $vtype
  # preds left       : 3
  # succs left       : 2
  # rdefs left       : 0
  Latency            : 1
  Depth              : 1
  Height             : 3
  Predecessors:
    SU(2): Data Latency=1 Reg=$v8m2
    SU(1): Data Latency=0 Reg=$vl
    SU(1): Data Latency=0 Reg=$vtype
  Successors:
    SU(4): Data Latency=1 Reg=$v10m2
    SU(4): Anti Latency=0
```
{{% /details %}}

But what's more interesting is this line sending an important message _after_ we scheduled `SU(3)`:
```
*** Critical resource SiFiveP600VectorArith: 2c
```
It says that `SiFiveP600VectorArith` becomes the **critical resource** now. In plain English, a critical resource is a processor resource that has been loaded with too many instructions.

The exact definition of "too many instructions" here is that a single processor resource, like `SiFiveP600VectorArith`, has been _occupied_ for more cycles than the expected latency from instructions that have been scheduled so far[^4]. Again, for a pipelined processor resource, all it matters is how long a resource is (exclusively) blocked or held by a single instruction until it can handle the next one. That is, an instruction's **occupancy** we introduced earlier.

[^4]: To be precise, a resource is considered critical only when it's occupied for more cycles than the sum of latency _by a certain factor_, `MCSchedModel::getLatencyFactor`, which is default to one.

Thus, we compare the total occupancy of all instructions using this resource with the expected latency from all instructions scheduled so far: when we scheduled `SU(3)`, we're at cycle 1. The instructions that have been scheduled at that point are `mul`, `vsetvli`, and a `vadd`. The multiplication instruction (i.e. `mul`) has a latency of 3 cycles, so it hasn't finished at cycle 1. On the other hand, both `vsetvli` and `vadd` has a latency of a single cycle. 

Now let's look at the total occupancy of instructions running on `SiFiveP600VectorArith` at cycle 1, which are `SU(2)` and `SU(3)`, both of them are `vadd`. In RISC-V, a `vadd` instruction uses `WriteVIALU` as its SchedWrite token. 

```
** ScheduleDAGMI::schedule picking next node
Queue TopQ.P:
Queue TopQ.A: 5 4
  TopQ.A RemainingLatency 0 + 1c > CritPath 6
  TopQ.A ResourceLimited: SiFiveP600VectorArith
  Cand SU(5) ORDER
Pick Top RES-REDUCE
Scheduling SU(5) renamable $x10 = ADD renamable $x9, renamable $x9
  Ready @3c
  SiFiveP600IntArith +1x1u
  TopQ.A TopLatency SU(5) 3c
...
```

### Notes
  - Latency
    - How: Described by critical path. `tryCandidate` prioritizes latency reduction by default.
  - Resources
    - In-order
      - Goal: Reduce stalls
      - How: Use `ReservedCycles` to keep track of reservation status. Each uop describes its reservation via `AcquireAtCycle` & `ReleaseAtCycle`.
    - Out-of-order
      - Goal: Increase parallelism & prevent a single resource from being saturated
      - How: Record the total execution time (i.e. "counts"), described by `AcquireAtCycle` & `ReleaseAtCycle`, of a resource. Keep track of the critical resource, which has the maximum counts. Declared resource limited when the critical resource count exceeds critical path (i.e. latency) by a single latency factor (default to 1). When it's resource limited, the `tryCandidate` will prioritize resource reduction instead.
        - The reason we don't track the reservation status just like in-order cores is because there is no way we can know the exact issue order. Plus, one of the main reasons we track resource reservation in in-order cores is because we want to reduce stall, which is something assumed to be done by out-of-order hardware.