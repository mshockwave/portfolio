+++
title = "Machine Scheduler in LLVM"
date = "2025-08-31"
draft = true
tags = ['llvm', 'compiler-instruction-scheduling']
+++

By this point it's evidently that I won't shut up talking about scheduling model in LLVM.

I found the fact that it encodes so many low-level details pretty fanicated. It have so much potential and could arguably be used in more places -- but the latter is probably a story for another day, because there is still an elephant in the room:

What about **instruction scheduler**, which scheduling model was originally invented for?

In one of my previous _Scheduling Model in LLVM_ posts, I'd hinted that a certain scheduling model construct -- the `BufferSize` of individual processor resource, for example -- might have different _interpretations_ depending on the user of scheduling model.

Oh well, the instruction scheduler in LLVM happens to be one of the primary users of scheduling models, so learning how instruction scheduler works in LLVM will definitely help us to have a better understanding on the whole scheduling model framework. And this particular post is dedicated to explaining the principles of this fine component in LLVM. First, I'm starting with clarifying _which_ instruction schedulers we're going to talk about today -- because there are at least **two** of them that work in completely different ways.

### The inception of Machine Scheduler
The idea of the first instruction scheduler in LLVM was pretty straightforward and in some sense, pretty clever: the instruction selection was already performed on a DAG -- namely, SelectionDAG ISel -- so we might as well do the scheduling on our way to turning those DAGs into "linear" Machine IR. Because part of the linearization is resolving dependencies among nodes in the DAG, and scheduling is all about placing nodes in optimal places under the constraints of these dependencies.

This design -- sometimes known as DAG scheduler -- stayed until about 2012. Though I haven't been able to find the exact reasons of switching to a new design back then, my guess being that it was just too _early_ to do scheduling right at the beginning of the Machine IR pipeline: many optimizations run at the Machine IR level and even more of them run before register allocation, when most `MachineInstr` -- the instruction representation in Machine IR -- still operate in SSA form, which makes optimization development a lot easier. We would miss lots of scheduling opportunities if the scheduler couldn't account for those optimized Machine IR.

To be fair, about 4 years before that, LLVM added a scheduler that runs _after_ register allocation -- a **post-RA** scheduler, which of course operates on Machine IR. Yet scheduling physical registers poses many restrictions when it comes to moving instructions around. Besides, one of the main benefits of instruction scheduling is being able to reduce register pressure _before_ the main register allocation phase. Therefore, at the time it had become clear that scheduling Machine IR _right before_ register allocation was the right way to go forward.

And that was the story behind **Machine Scheduler** -- the protaganist of this post. Nowadays, almost all LLVM targets chooses to schedule instructions with Machine Scheduler: the _pre-RA_ Machine Scheduler runs right before the register allocation, while the _post-RA_ variant -- not enabled by every targets though -- usually runs relatively late in the codegen pipeline. Both pre- and post-RA variants share a similar codebase, so for simplification, we'll primarily focus on pre-RA scheduling in our discussions here.

Machine Scheduler has two primary objectives:
  - Reduce register pressure
  - Improve instruction level parallelism

For the first objective, we hope to rearrange instructions in a way that the degree of live range overlappings will be kept minimal, hence reduction in the overall register pressure and the number of potential _spills_.

The second objective is embodied by at least two items: hiding latency and preventing stalls caused by pipeline hazards in the processor. It is well known that issuing instructions with shorter latency independent to the current critical path -- another way to describe the chain of instructions with the longest latency -- to _hide_ its latency improves throughput and as a consequence, degree of (instruction level) parallelism. But it's only effective if the processor has enough resources, like free slots in the issue queues, to execute those parallel instructions, otherwise stalls and hazards are inevitable.

Compiler optimizations with multiple objectives like this could get complicated sometimes, so let's start with the high-level picture and look at Machine Scheduler's workflow.

### Machine Scheduler's Workflow
By now, we have learned that Machine Scheduler operates on Machine IR -- a "linear" IR where instructions are organized sequentially in a basic block, similar to LLVM IR. Despite being more intuitive, linear IR is probably not the most ideal way to express **dependencies**, including data dependencies (e.g. registers def-use chain) and control / side-effect dependenceis (e.g. partial orders imposed by memory operations). It's better to have a representation that could _explicitly_ spell out these dependencies.

Luckily we already have something in our inventory: the **DAG**. Not exactly the SelectionDAG from instruction selection, but the DAG originally designed for the legacy DAG scheduler mentioned earlier, which is known as _ScheduleDAG_. A ScheduleDAG that has been "repurposed" in this way is also known as `ScheduleDAGInstrs`, emphasizing its underlying `MachineInstr` content. Each `ScheduleDAGInstrs` represents a single `MachineBasicBlock` (basic block in MachineIR) and each of the `MachineInstr` within it is modeled by a data structure called `SUnit`. To give you a better idea, given the following LLVM IR:

``` llvm
define i64 @add(ptr %p0, ptr %p1) {
  store i64 0, ptr %p0
  %v = load i64, ptr %p1
  %r = add i64 %v, 87
  ret i64 %r
}
```

This is its ScheduleDAG: 

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-misched-scheddag.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-misched-scheddag.light.svg">
  </picture>
</div>

{{% details "How to generate this graph"%}}
~~~ session
$ llc -mtriple=riscv64 input.ll -view-misched-dags
~~~
This command will actually try to pop up a window to show the GraphViz result, don't worry if you're on a remote server without X server just like me -- it stores the *.dot file in a tmp dir whose path will be shown on screen, just copy that file to wherever you like.
{{% /details %}}

In this graph, the data dependency from `add` -- represented by `SUnit` node `SU(4)` -- to `load` -- `SU(3)` -- on its loaded value is shown as a solid arrow. On the other hand, the memory (side-effect) dependency between `SU(3)` and the store instruction -- `SU(2)` -- is represneted by a dashed blue arrow.

We can see more details in individual `SUnit` by adding `-misched-print-dags` to the `llc` command. For instance, here are the details for `SU(3)`:

~~~
SU(3):   %3:gpr = LD %1:gpr, 0 :: (load (s64) from %ir.p1)
  # preds left       : 2
  # succs left       : 1
  # rdefs left       : 0
  Latency            : 4
  Depth              : 1
  Height             : 5
  Predecessors:
    SU(2): Ord  Latency=1 Memory
    SU(0): Data Latency=0 Reg=%1
  Successors:
    SU(4): Data Latency=4 Reg=%3
~~~

Some important properties worth mentioning here:

  - **Predecessors** and **Successors**: a successor is a _dependant_ `SUnit` of the current `SUnit`. Conversly, a predecessor is a `SUnit` that the current node depends _on_. All predecessors have to be scheduled before the current `SUnit` becomes illegible for scheduling. From here we can see that `SU(3)` is successed by the add instruction, `SU(4)`, with data dependency and preceded by the store instruction `SU(2)` with memory dependency (the "Ord" stands for _ordering_).
  - **Latency**: latency of the current `SUnit`. This value comes from the scheduling model.
  - **Height**: this is effectively the critical path from the current `SUnit` to the bottom of this DAG. To put it more formally, it is the _maximal_ sum of latencies from the current node to any node that does not have any successors. In this particular case, it is the sum of latencies from `SU(3)`, `SU(4)`, and finally `SU(5)` (`SU(4)` has a latency of one, so 4 + 1 = 5).
  - **Depth**: this is an "inverse" of Height -- the maximal sum of latencies from any node that has no _predecessors_ to the current node. In this particular case, it is the sum of latencies from `SU(1)`, `SU(2)`, and `SU(3)` (`SU(1)` has zero latency and `SU(2)` has a latency of one, hence 0 + 1 = 1).

The reason there are both _height_ and _depth_ is because Machine Scheduler is able to schedule nodes from both bottom-up (starting from the _last_ instruction in the block) and top-down (starting from the _first_ instruction in the block), so we need some sort of critical path in both directions. For simplification, let's stick to bottom-up for now. That is, we'll focus on the _height_ in these nodes.

With these properties populated in each `SUnit`, a `ScheduleDAGInstrs` is ready to be scheduled by Machine Scheduler. The following diagram shows the high-level workflow of what happens next.

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-misched-important-components.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-misched-important-components.light.svg">
  </picture>
</div>

This flow can be seen as a three-phase process: pick a node from the **queue**, put that node onto a simulated **timeline**, update the **states**, and repeat. The keywords with boldface are three of the most components that we're diving deep into later. Let's actually start with the second one, _timeline_, first.

#### Scheduling timeline and boundary
In the core of Machine Scheduler's algorithm is a really simple simulation that _loosely_ keeps track of the cycle a node (hence its underlying `MachineInstr`) is expected to be issued. This helps the scheduler to avoid scheduling a node when either its input data still needs a few cycles to be ready, or the execution resource it needs (e.g. an execution pipe) is fully occupied. Either case might lead to a _stalling_ or more generally speaking, a _hazard_, in actual execution.

Therefore, Machine Scheduler is effectively keeping track of the current cycle on a virtual timeline while scheduling more nodes onto it. Here is a simple example: 

|   |    Assembly    | Latency | Predecessors |
|:-:|:--------------:|:-------:|:------------:|
| A | mul r2, r2, r5 |    3    |       -      |
| B | add r0, r1, r2 |    1    |       A      |
| C | addi r4, r4, 7 |    1    |       -      |

Assuming we only have three instructions A, B, and C with properties shown in the table above, and we're targeting an _infinite_ wide machine with _infinite_ number of execution pipes -- that is, we can issue however many instructions in the same cycle and they can all be executed.

On the virtual timeline, there is a pointer `CurrCycle` pointing to the cycle where the next instruction is subject to issue at. With a clean start, instruction A can naturally be issued at cycle 0 given it has no predecessor (dependency), so A will be the first instruction we schedule. On the other hand, instruction B could not be issued now since it has to wait for the result from instruction A, which is reflected by its predecessor field. That being said, nothing stops instruction C from being issued at the current cycle given there is no dependency it has to wait for, therefore we put C as the next instruction to schedule. Now, our timeline looks like this:

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-misched-timeline-1.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-misched-timeline-1.light.svg">
  </picture>
</div>

With both A and C scheduled, there is only B left now. Since B cannot be issued at cycle 0, we increment `CurrCycle` by one. But instruction A still hasn't finished at cycle 1, so we keep incrementing `CurrCycle` until B can be issued / scheduled, which is cycle 3 when A finally finishes. After we schedule B, here is the timeline looks like:

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-misched-timeline-2.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-misched-timeline-2.light.svg">
  </picture>
</div>

Our final schedule sequence would be A, C, B or C, A, B (A and C will be issued at the same cycle so it doesn't matter who goes first).

Now, let's make this scenario a little more spicy: we're no longer targeting a machine with infinite wide issue width and unlimited number of execution resource. Instead, the machine has a width of 2 and 2 integer execution pipes, `IEX0` and `IEX1`, where only `IEX0` can do the multiplication. Here are our instructions now:

|   |     Assembly    | Latency | Predecessors | Execution pipe | Occupancy |
|:-:|:---------------:|:-------:|:------------:|:--------------:|:---------:|
| A |  mul r2, r2, r5 |    4    |       -      |      IEX0      |     2     |
| B |  add r0, r1, r2 |    1    |       A      |   IEX0, IEX1   |     1     |
| C |  addi r4, r4, 7 |    1    |       -      |   IEX0, IEX1   |     1     |
| D |  mul r6, r6, r3 |    4    |       -      |      IEX0      |     2     |
| E | addi r7, r7, -5 |    1    |       -      |   IEX0, IEX1   |     1     |

Let's ignore the "Occupancy" column for a second. The first three instructions are the same and we add another 2 additional instructions that have no dependency on the results from previous instructions (i.e. no predecessors).

We handle instruction A and B the same way as before -- that is, only A is scheduled. Now, `CurrCycle` is still at cycle 0, we have three candidate instructions: C, D, and E -- none of them has dependency on previous instruction's result. One thing we notice is that A is already using `IEX0`, so instruction D, which is a multiplication, has to wait for it a little bit (we're going back to "how long it's gonna wait" shortly) and cannot be scheduled. With C and E, we cannot schedule both of them because we no longer has infinite issue width, and A already took one out of two issue slots at the current cycle -- we have to pick one. Let's pick C just like before, shall we? Now the timline looks exactly like one we saw before:

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-misched-timeline-1.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-misched-timeline-1.light.svg">
  </picture>
</div>

Moving `CurrCycle` to cycle 1, our candidates are again D and E (B is still waiting for A). For D, we face the same problem as before: A is using `IEX0`...but wait a second, _how long_ is A gonna use? Remember, most modern processors have _pipelined_ execution unit,[^1] meaning an instruction wouldn't normally "occupy" the whole execution pipe until it finishes. It only occupies for a duration shorter than its latency -- and such duration is the _occupancy_ column in the table above.

[^1]: that being said, complex instructions could be non-pipelined though. The most famous example being _divisions_ -- it's not pipelined in most processors nowadays.

According to this occupancy table, instruction A "occupies" `IEX0` for 2 cycles, which means that instruction D can only be issued after A releases its occupation. That leaves instruction E being the only option to schedule at cycle 1:

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-misched-timeline-3.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-misched-timeline-3.light.svg">
  </picture>
</div>

Moving forward to `CurrCycle = 2`, we can schedule insturction D as A now releases `IEX0`:

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-misched-timeline-4.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-misched-timeline-4.light.svg">
  </picture>
</div>

Finally, we schedule instruction B after A -- whose result B depends on -- finishes its execution at cycle 3:

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-misched-timeline-5.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-misched-timeline-5.light.svg">
  </picture>
</div>

**_Congratulation_**, you just learned the most important outline of Machine Scheduler's core scheduling algorithm!

This example also _foreshadows_ some of the topics we're covering later. For instance, how candidate instructions are grouped and how to select the best one among those candidates in order to avoid stallings and hazards.

But let's not get ahead of ourselves, because there is another interesting thing worth talking about on this `CurrCycle` pointer. This pointer is maintained and kept as part of a larger data structure called `SchedBoundary`. Personally I would like to call it scheduling _frontier_ instead -- the frontier where you schedule new instructions. This `SchedBoundary` also contains many other things like the queues carrying candidate instructions which we will talk about shortly, as well as information needed to keep track of critical path and processor resource pressures.

And the reason it packs these items into a dedicated data structure is because Machine Scheduler can schedule instructions from different _directions_ -- top-down or bottom-up -- as we briefly mentioned earlier. The `SchedBoundary` for these two schemes behave differently, for instance, in bottom-up scheduling an instruction is only illegible to schedule after all its _successors_ were scheduled. In a _bidirectional_ setup -- yes, Machine Scheduler is able to schedule instructions from both directions **at the same time** -- both "top" and "bottom" `SchedBoundary` exist simultaneously. The scheduler basically picks the single best instruction from these two `SchedBoundary` and schedules it accordingly.

In other words, `SchedBoundary` keeps a _state_ of the current scheduling in the direction it belongs to. And this state directly affects the decision of the next instruction to schedule, which is the main topic we're covering next.

### Picking the Best Candidate

