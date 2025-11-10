+++
title = "Machine Scheduler in LLVM - Part II"
date = "2025-11-01"
tags = ['llvm', 'compiler-instruction-scheduling']
+++

In the [first part](/llvm-machine-scheduler) of this series, we covered the basic workflow of Machine Scheduler -- LLVM's predominated instruction scheduling framework -- and learned that an instruction could go through three phases of checks before it finally got scheduled: **legality** check, **feasibility** check, and **profitibility** check.

The first two phases -- which were explained in details in that post -- have direct connections with program correctness and avoiding potential processor hazard, respectiviely. The last phase tries to pick the _optimal_ candidate that'll hopefully reduce the register pressures and increase the instruction level parallelism (ILP) -- the two primary goals for instruction scheduling in LLVM. In this post, we're going to dive deep into this profitibility check phase.

### Profitibility Checks
Making the optimal choice has always been a difficult problem in computer science (as in real life). There is a whole big field telling you how to optimize for a specific set of constraints -- usually with a cost of non-trivial amount of runtime, however. Machine Scheduler, just like other parts of LLVM, prioritizes speed and perhaps maintainability over finding the absolute optimal instruction.

And that, is the rationale behind the design of `tryCandidate` -- specifically its fixed set of comparisons done on candidates -- we've shown in the previous post. Among those heuristics and comparisons, we are particularly interested in two of them: favor the candidate with a lower _register pressure_ and pick the instruction with lower _resource pressure_. As they have a more direct connection with the goals of instruction scheduling mentioned earlier. Plus, both out-of-order and in-order cores put attentions on these items. So, without further ado, let's look at the register pressure heuristics first.

#### Register pressure
I always like to describe register pressure as a synonym for the number of (concurrent) **live intervals** at a given point in the program.

{{% picture "llvm-misched2-reg-pressure" %}}

In the diagram above, each vertical bars on the left is an live interval for a specific register. A live interval starts from a register definition and ends at its _last_ use that appears right before another register (re)definition. In this simplified scenario, counting the number of overlapping live intervals at a certain point -- either before or after an instruction -- gives you its register pressure.

Live range is basically the duration where a register is _reserved_ and blocked out from any other purposes. Therefore, if too many registers are being booked at the same time, the likelyhood a register spilling happens increases -- that's why register pressure is a useful tool to gauge the degree of register spillings.

In reality, LLVM tracks register pressure by groups: registers with similar characteristics are put in the same **pressure set** and maintains its own pressure value. Take _scalar_ and _vector_ registers as an example, they are usually assigned into different pressure sets. The pressure induced by any scalar registers are aggregated into a same pressure value; a similar logics applies to vector registers:

{{% picture "llvm-misched2-reg-pressure2" %}}

The number of concurrent live intervals, and hence register pressure, obviously depends on the structure of instructions and their _ordering_ -- but then back to what we're doing here, how should we make an estimation on the prospective register pressure even _before_ all the instructions are scheduled?

As it turns out, in most scenarios Machine Scheduler cares more about the register pressure **difference** made by a single instruction. The scheduler initializes each instruction with a rought "guess" on how it might impact the overall pressure, which is stored in a data structure called `PressureDiff`. It is an array indexed by pressure set ID in which individual element expresses the degree or the quantity of pressure change.

Each `PressureDiff` associated with the instruction might get updated during the scheduling, but ultimately when an instruction is compared against others on register pressure inside `tryCandidate`, its `PressureDiff` is applied on top of the _current_ register pressure at that moment. For example, if instruction A could increase scalar registers' pressure by 2, and the current scalar pressure set already has a pressure of 4, then the prospective new pressure in the scalar pressure set will be 6, easy! This gives scheduler an idea on whether this instructions increases or decreases the _overall_ pressure (again, among the scheduled instructions at that moment) and how much does it change. More specifically, the scheduler wants to know the change on three metrics:

##### Excess pressure
How much does the new pressure goes overboard compare to the old pressure. While this sounds exactly like the pressure difference of an individual instruction we just talked about -- and in some cases, it is -- there is a catch here: excess pressure is "filtered" by a per-pressure set _threshold_ value.

The idea is that if we care about every tiny bit of register pressure changes, we might lose the big picture because those are just noise. By setting a lower bound threshold, we now calculate the excess pressure with only the pressure values that go over that line:

```
ExcessP = max(NewP, Threshold) - max(OldP, Threshold)
```

By this formula, if both new and old pressures are below the threshold, excess pressure would be zero -- because both of them are considered noise in this case; if only _one_ of the pressures goes over the threshold, the difference taken in this case will be confined by the threshold -- either `Threshold - OldP` or `NewP - Threshold`. In other words, this formula is effectively a _damper_ that prevents the pressure difference from oscillating too much.

##### Critical Max pressure
The next register pressure related heuristic compares the new register pressure (i.e. `NewP`) with the maximal pressure we have seen so far in a specific pressure set. To be more specific, this is the maximal pressure from _both_ the top and bottom `SchedBoundary` that are going on at this moment, so critical max pressure -- as it's called in Machine Scheduler -- is truely the maximal pressure of the entire scheduling region at the time.

The value we're interested in here, similar to excess pressure, is the difference between `NewP` and critical max pressure. But unlike excess pressure where the pressure difference can be negative -- namely, decreasing the pressure -- we only consider the difference when `NewP` is larger than critical max pressure.

##### Current Max pressure
As confusing as the naming can go, _current_ max pressure represents the maximal pressure of a pressure set in the **original**, unscheduled program. Seriously, people should just call it "original max pressure". Aside from that, we also take the difference between `NewP` and current max pressure, if the former is larger than the latter.

These are the three metrics `tryCandidate` really cares about when it comes to the per-instruction register pressure impact. The execess pressure is always used before other two, as it represents the before vs. after pressure of a specific pressure set, which is much more meaningful in most cases.

Before wrapping up this part, I would like to dive a little more into the technical details and breakdown the meanings of some key data structures -- most of them were already covered in the previous passages -- that have even _more_ confusing names than "current max pressure" vs. "critical max pressure" in this part of the Machine Scheduler 🙃:

|     Class Name     |                                                                                  Description                                                                                 |
|:------------------:|:----------------------------------------------------------------------------------------------------------------------------------------------------------------------------:|
| `PressureChange`   | The quantity of (potential) register pressure change on a specific pressure set                                                                                              |
| `PressureDiff`     | An array of `PressureChange` , indexed by _pressure set ID_, made by a single instruction                                                                                    |
| `PressureDiffs`    | A data structure that maps from an _instruction_ to its `PressureDiff`                                                                                                         |
| `RegPressureDelta` | This specialized data structure only carries the three metrics Machine Scheduler cares about. Namely, _excess pressure_, _critical max pressure_, and _current max pressure_ |
| `RegisterPressure` | Despite its name, this data structure only carries the maximal register pressure of each pressure set[^1] in the current region                                                            |

[^1]: As well as the live-in and live-out registers in this region.

Next, we're looking at how `tryCandidate` picks the candidate with a lower **resource pressure**.

#### Resource pressure
To explain resource pressure, we can actually draw an analogy from register pressure we just learned: _pressure sets_ are the smallest units to keep track of registers pressures in LLVM. Machine Scheduler, as we just saw, tries really hard to prevent the pressure within a single pressure set from exceeding a certain _threshold_.

Here, individual processor resource (i.e. pipe) is like a register pressure set. And the goal of reducing resource pressure here, is to prevent the accumulated **occupancy cycles** within a processor resource from exceeding a certain _threshold_.

This threshold is the total latency we have scheduled so far -- which is roughly equal to the **critical path length**. Critical path, by definition, is the number of cycles in the longest serial execution path. In a perfectly parallelized system, the total execution time of the current region will be bounded by the critical path length.

Well, **IF** we have a perfectly parallelized system, that is. But such a theoretical upperbound is the _exact_ thing we need here to gauge how well we're doing ILP-wise in the region we're currently scheduling -- whenever the occupancies of a single resource go over this value, there might still be rooms for running more instructions in parallel.

In other words, conceptually, we want to avoid _over-concentrated_ pipes like this:

{{% picture "llvm-misched2-resource-pressure1" %}}

And prefer more uniform _distribution_ among all the available pipes:

{{% picture "llvm-misched2-resource-pressure2" %}}

Without a doubt, the accumulated occupancies in each pipes plays a key role here. However, let's not forget that each pipe might be built differently in terms of its throughput so that we have to _normalize_ these occupancy values to get a fair comparison.

To be more specific, we're talking about the number of units in a `ProcResource`:

``` c++
def PipeA : ProcResource<1>;
def PipeB : ProcResource<4>;
def PipeC : ProcResource<2>;
```

Recall in [one of my previous blog posts](/llvm-sched-model-1.5/#number-of-units-in-a-procresource) about scheduling model, a processor resource that has more than one unit, like

``` c++
def PipeB : ProcResource<4>;
```

can be thought as having multiple identical execution pipes:

``` c++
def PipeB_0 : ProcResource<1>;
def PipeB_1 : ProcResource<1>;
def PipeB_2 : ProcResource<1>;
def PipeB_3 : ProcResource<1>;
```

An instrucion consumes one of the execution pipes for a duration of its occupancy. The unit of occupancy is cycle per instruction (hence the name of _inverse_ throughput), but take this case as an example, the occupancy value describes not cycle per instruction but cycle per _four_ instructions, as we can run another 3 instructions at the same time. In other word, if an instruction has a **nominal** occupancy of `N` cycles, its **effective** occupancy on `PipeB` will be `N / 4`.

As a consequence, the effective occupancy of the same instruction on `PipeA`, `PipeB`, and `PipeC` would be `N / 1`, `N / 4`, `N / 2`, respectively. This would be the occupancy value we use to check whether a single resource is over concentrated as introduced earlier. However, compiler itself -- as a software -- tries to avoid floating point numbers as much as possible, so we're gonna normalize it by multiplying them with their least common multiple (LCM), yielding `4 * N`, `N`, and `2 * N`, respectively.

Now, in addition to the number of units, which is proportional to the result throughput in each pipe, we also need to consider the throughput on the other end of the pipe -- the _input_ rate, which is modeled by the **issue width**. Issue width represents how many instrutions / uops could be pumped into these pipes per cycle. Combining issue width with the number of units per hardware reousrce, we can assign an _ingress_ and _egress_ ratio for each resource. For example, given an issue width of 3, `PipeB` has an ingress vs. egress ratio of `3 : 4`. This ratio is an important invariant we have to keep across normalization.

The easiest way to preserve this ratio is by taking issue width into the LCM calculation as well:

```
LCM(IssueWidth, NumOfUnits_0, NumOfUnits_1, ..., NumOfUnits_N)
```

This LCM value is also known as **latency factor**. From the latency factor we can further derive **resource factor**, the factor we use to normalize occupancies. For example, for an instruction that runs on resource `X`, its _normalized_ occupancy can be calculated as:

```
ResourceFactor      = LatencyFactor / NumOfUnits_X
NormalizedOccupancy = Occupancy * ResourceFactor
```

The intuition behind this is that issue width represents the number of ingress instructions per cycle, so for an issue width of 3, feeding in an instruction takes `1 / 3` cycle -- this is the equivalent of effective occupancies like `N / 4` or `N / 2` we saw earlier, except that instead of describing how many cycles it occupies, it describes how many cycles it takes to ingest. As a consequence, by taking issue width into the LCM, we can effectively scale the ingress _and_ egress throughputs at the same time.

Let's wrap up this section with an example to show how every components comes together. First, let's spell out the hardare resource definitions we've been using so far:

``` c++
def FooSchedMachineModel : SchedMachineModel {
    let IssueWidth = 3;
}

let SchedModel = FooSchedMachineModel in {
    def PipeA : ProcResource<1>;
    def PipeB : ProcResource<4>;
    def PipeC : ProcResource<2>;
}
```

The latency factor is equal to

```
LCM(/*issue width=*/3,
    /*PipeA units=*/1,
    /*PipeB units=*/2,
    /*PipeC units=*/4) = 12
```

And the resource factor of each pipes are:

|       | Resource Factor |
|:-----:|:---------------:|
| PipeA |   12 / 1 = 12   |
| PipeB |    12 / 4 = 3   |
| PipeC |    12 / 2 = 6   |

On the instruction side, here are the resource usages, occupancies, and latencies we will use in this example:

| Instruction | Resources | Occupancy | Latency |
|:-----------:|:---------:|:---------:|:-------:|
|     `ld`    |   PipeC   |     1     |    3    |
|    `mul`    |   PipeA   |     2     |    3    |
|    `add`    |   PipeB   |     1     |    1    |
|    `slli`   |   PipeB   |     1     |    1    |

Given these conditions, assuming we already have the following instructions scheduled:

```
ld  a0, 0(s0)
mul t0, s0, s1
add a1, a0, a0
```

The critical path in this case would be dominated by the load instruction, `ld  a0, 0(s0)`, therefore, the **nominal** critical path length is equal to 3 cycles[^2]. But remember, we have to scale this number with the latency factor, so the **normalized** critical path length is `12 * 3 = 36` cycles.

[^2]: In a top-down schedule, the critical path length is effectively defined as the number of cycles up to the point where the last instruction was _issued_, so the latency of `add a1, a0, a0` is not accounted for.

Similarly, we have to normalize the occupancy of individual instructions with their corresponding resource factor:

| Instruction | Resources | Normalized Occupancy |
|:-----------:|:---------:|:--------------------:|
|     `ld`    |   PipeC   |       6 * 1 = 6      |
|    `mul`    |   PipeA   |      12 * 2 = 24     |
|    `add`    |   PipeB   |       3 * 1 = 3      |
|    `slli`   |   PipeB   |       3 * 1 = 3      |

As a consequence, the _accumulated_ occupancies for each pipe at this moment are:

|       |       Executed Instruction(s)      | Accumulated Occupancies |
|:-----:|:----------------------------------:|:-----------------------:|
| PipeA |          `mul t0, s0, s1`          |            24           |
| PipeB |          `add a1, a0, a0`          |            3            |
| PipeC |           `ld a0, 0(s0)`           |            6            |

Now, assuming we have two candidates to pick from for the next instruction to schedule:
  - `mul t1, s2, s1`
  - `slli a2, a2, 4`

If we pick `mul t1, s2, s1` as the next instruction to schedule, the accumulated occupancy of `PipeA` becomes 48 cycles, which exceeds the critical path length, 36 cycles, by a large margin. On the other hand, if we pick `slli a2, a2, 4`, none of the three pipes' accumulated occupancies will be greater than 36 cycles.

Therefore, it's more desired to pick `slli a2, a2, 4` over the other.

If we try to picture what will actually happen in the hardware, for the case where we pick `mul t1, s2, s1`, this instruction has no choice but stacks on top of the previous muliplication instruction, hence increasing the occupancy of `PipeA`:

{{% picture "llvm-misched2-resource-pressure-timeline1" %}}

On the other hand, in the case where we pick `slli a2, a2, 4`, hardware can actually **distribute** it into one of the four pipes within `PipeB`. Therefore, the effective occupancy in `PipeB` does not increase: 

{{% picture "llvm-misched2-resource-pressure-timeline2" %}}

So all the normalization processes we discussed early are actually just trying to model this distrbution capability (or the lack of it) in a _numeric_ way -- "penalize" resources of small number of units (i.e. low distribution capability) with high resource factor and reward them with low resource factor otherwise.

### So, what's next?
Perhaps surprisingly, I think there are actually many works we can do for in-order processors in this area!

Compared to other parts of LLVM backend, Machine Scheduler hasn't received many improvements or even just changes in the past decade. I am convinced that this is caused by the fact that out-of-order processors have dominated the market and hence the compiler development space in the past few decades or so -- and out-of-order processors are much more agnostic or resilient to the instruction scheduling quality, as the whole premises of out-of-order-ness is to do the exactly same scheduling in hardware, during runtime, yet with more precision. In other words -- it's usually not worth the efforts to improve Machine Scheduler for out-of-order processors. To be fair, reducing the number of register spillings (through instruction scheduling) still helps, but even on that regards, missing a few register spillings generally won't make a huge difference performance-wise on modern out-of-order processors.

It's a different story for in-order processors, of course. Stallings and register spillings both make huge impacts on performance, and therefore the importance of instruction scheduling quality for these processors. Sadly in the past few decades, most in-order processors are embedded ones, where either the hardware is not complicated enough to warrant more advanced scheduling techniques, or the fact that performance altogether is just not the first priority to them. All of these factors contribute to the relatively staggering development in Machine Scheduler and to some extent, the scheduling model framework.

That brings us to the first item I think Machine Scheduler can do better on in-order cores -- it's a little _too pessimistic_ on stallings. As we learned in the [first part](/llvm-machine-scheduler) of this series, scheduling candidates will stay in the pending queue until all of its hazards are cleared, at which point they'll migrate to the available queue. On one hand, this matches the expectations of hardware: hazards _are_ devastating for in-order cores; yet on the other hand, we might be missing out many opportunities where we rather take the penalty from stallings because doing so can dramatically reduce the register pressure!

Let's say an instruction might stall for 2 cycles due to data hazards, in the current design it'll stay in pending queue until 2 cycles later. But as it turns out, if we put it into the available queue right away, the scheduling algorithm will realize that it can help us to lower the register pressure and effectively eliminate a register spill (usually a vector register spill) that would have costed us **10 cycles**! In this case we definitely want to trade hazard preventions for less register spills.

This shortcoming could be circumvented by using `BufferSize = 1`. As we introduced earlier this configuration only imposes "soft" stalls that still allow instruction candidates to move from the pending queue to the available queue in the presence of hazards. In other words, Machine Scheduler schedules these in-order instructions "out-of-order-ly".

Though `BufferSize = 1` does give us more flexibility, foregoing _any_ kinds of hazard detection while some of them still brings, as we mentioned earlier, devastating performance impacts on in-order processors is hardly a general solution -- it's going a little too _optimistic_. Just like everything in real life, we have to strike a balance here.

And that brings us to the second topic -- potential improvements for `tryCandidate`. As we learned from the previous post, `tryCandidate` decides whether an instruction is more profitable over the other by going through a list of criterias -- register pressure, resource pressure, latency etc. -- in **a fixed order**. That fixed order could cause some problem. Similar to the trade off between a 2-cycle stalling and a 10-cycle reduction due to eliminating a register spill we just mentioned, what if we have two instructions `A` and `B`, where instruction `A` has a slightly better register pressure than `B` but might cause serious trouble on resource pressure. If register pressure is checked before resource pressure -- which is actually what we're doing at this moment -- then `tryCandidate` will fail to see the resource pressure red flag on `A`!

Of course, one can argue that this issue can boil down into the (in)famous phase-ordering problem that has plagued compiler development since the beginning of the time. Yet I argue that this is not the end of the world: we can improve this by assigning each of these factors -- register pressure, resource pressure, latency etc. -- a **cost**, and comparing against the _aggregated_ (and probably weighted) cost of each instruction candidate. The idea is that we can account for _every_ factors we care about, rather than just some of them, with this method.

Last but not the least, Machine Scheduler is pretty complicated -- but it certainly doesn't help when the debug messages (i.e. `-debug-only=machine-scheduler`) are cryptic and relatively unstructured. Granted, the amount of information it tries to convey is _massive_, yet I think we should invent a better framework to **trace** development messages in this scale, as neither the current debug message framework nor the optimization remarks frameowrk seem fit here.

### Epilogue
This is the second and last part of this series. I hope I shed some lights on how Machine Scheduler decides which instruction is more profitable and the rationale behind it. Just like everything else in LLVM, all the things I've been talking here in this series are just the tip of an iceberg -- I unfortunately has to omit lots of great topics due to the length and deep beneath this iceberg, there are still many areas in Machine Scheduler we can improve as I illustrated in the last section.

### Comments
Feel free to leave comments at https://github.com/mshockwave/portfolio/discussions/14