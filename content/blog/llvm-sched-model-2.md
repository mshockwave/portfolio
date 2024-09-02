+++
title = "Scheduling Model in LLVM - Part II"
date = "2024-08-17"
draft = true
tags = ['llvm', 'compiler-instruction-scheduling']
+++

In the [previous post](/llvm-sched-model-1), we covered the basics of scheduling model in LLVM. Specifically, per-operand tokens that connect an instruction with models that spell out processor-specific scheduling properties like instruction latency, and the concept of processor resources with different sizes of buffer.

In this post, I'll bring up some more advanced properties and focus on how scheduling models are actually _used_ in other parts of LLVM, specifically, **MCA (Machine Code Analyzer)** and **MachineScheduler**. It is particularly important to learn the usages of scheduling models, because LLVM actually doesn't have a formal spec for everything we have been talking so far, for example, the meaning of different buffer sizes. Those are coming from how other components _interpret_ these models[^1].

[^1]: There are, of course, code comments on scheduling models and without a doubt they are good source of knowledges. But they are neither formal specs nor always being consistent across the entire codebase.

By the end of this post, you'll get a better idea on how different scheduling model properties affect the instruction scheduling quality as well as the precision of other LLVM tools like `llvm-mca`. First, let's take a look at some more advanced properties in LLVM's scheduling model.

### Advanced scheduling model properties
Previously, we've learned `ProcResource` and `ProcResGroup`. The former more or less represents a single execution unit in a superscalar processor, while the latter is a logical group of `ProcResource` where an instruction can be assigned to **one of those** (sub-)resources. In other words, `ProcResGroup` represents a _heirarchical_ structure.

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

In short, it specifies the maximum number of uops a `ProcResource` can handle in a single cycle. Namely, the maximum throughput. For instance, previously we've seen that `Zn3LSU` can handle at most three memory uops. Similarly, in [SiFiveP600 scheudling model](https://github.com/llvm/llvm-project/blob/d4c519e7b2ac21350ec08b23eda44bf4a2d3c974/llvm/lib/Target/RISCV/RISCVSchedSiFiveP600.td#L75), its LSU, `SiFiveP600LDST` can handle at most two loads or stores:
```c++
// Two Load/Store ports that can issue either two loads, two stores, or one load
// and one store (P550 has one load and one separate store pipe).
def SiFiveP600LDST       : ProcResource<2>;
```
When an uop is dispatched to a resource of this kind, it is effectively dispatched to _one_ of its units.

Sounds familiar? That's right! in this sense, it is really similar to `ProcResGroup` we saw previously. In fact, `ProcResGroup` has an "implicit" number of units as well, and it's (unsurprisingly) equal to the number of sub-resource it contains.

The line between when to use `ProcResGroup` and when to use `ProcResource<N>` is somewhat blurred and in many cases they're interchangable. For example, in the following figure we have an integer execution unit with three pipes: two of them are capable of doing multiplications, while division and cryptography are each restricted to a single pipe. All three of them, however, can do basic ALU operations. Each pipe has a buffer size of 16. In other words, this is a **decoupled reservation station** style structure we talked about in the [previous post](/llvm-sched-model-1).
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
Highlights: how MachineScheduler abstract aways some details (as opposed to MCA's fidelity of actual microarchitectures) and focusing on the parts that actually affect instruction scheduling.

Also, visualizing instruction scheduling with MachineScheduler's debug output.
-->

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