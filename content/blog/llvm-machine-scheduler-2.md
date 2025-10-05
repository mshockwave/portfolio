+++
title = "Machine Scheduler in LLVM - Part II"
date = "2025-09-26"
draft = true
tags = ['llvm', 'compiler-instruction-scheduling']
+++

In the [first part](/llvm-machine-scheduler) of this series, we covered the basic workflow of Machine Scheduler -- LLVM's predominated instruction scheduling framework -- and learned that an instruction could go through three phases of checks before it finally got scheduled: **legality** check, **feasibility** check, and **profitibility** check.

The first two phases -- which were explained in details in that post -- have direct connections with program correctness and avoiding potential processor hazard, respectiviely. The last phase tries to pick the _optimal_ candidate that'll hopefully reduce the register pressures and increase the instruction level parallelism -- the two primary goals for instruction scheduling in LLVM. In this post, we're going to start with diving deep into this profitibility check phase.

### Profitibility Checks
Making the optimal choice has always been a difficult problem in computer science (as in real life). There is a whole big field telling you how to optimize for a specific set of constraints -- usually with a cost of non-trivial amount of runtime, however. Machine Scheduler, just like other parts of LLVM, prioritizes speed and perhaps maintainability over finding the absolute optimal instruction.

And that, is the rationale behind the design of `tryCandidate` -- specifically its fixed set of comparisons done on candidates -- we've shown in the previous post. Among those heuristics and comparisons, we are particularly interested in two of them: favor the candidate with a lower _register pressure_ and pick the instruction with lower _resource pressure_. As they have a more direct connection with the goals of instruction scheduling mentioned earlier. Plus, both out-of-order and in-order cores put attentions on these items. So, without further ado, let's look at the register pressure heuristics first.

#### Register pressure
I always like to describe register pressure as a synonym for the number of (concurrent) **live intervals** at a given point in the program.

{{% picture "llvm-misched2-reg-pressure" %}}

In the diagram above, each vertical bars on the left is an live interval for a specific register. A live interval starts from a register definition and ends at its _last_ use that appears right before another register (re)definition. In this simplified scenario, counting the number of overlapping live intervals at a certain point -- either before or after an instruction -- gives you its register pressure.

Live range is basically the duration where a reigster is _reserved_ and blocked out from any other purposes. Therefore, if too many registers are being booked at the same time, the likelyhood a register spilling happens increases -- that's why register pressure is a useful tool to gauge the degree of register spillings.

In reality, LLVM tracks register pressure by groups: registers with similar characteristics are put in the same **pressure set** and maintains its own pressure value. Take _scalar_ and _vector_ registers as an example, they are usually assigned into different pressure sets. The pressure induced by any scalar registers are aggregated into a same pressure value; a similar logics applies to vector registers:

{{% picture "llvm-misched2-reg-pressure2" %}}

The number of concurrent live intervals, and hence register pressure, obviously depends on the structure of instructions and their _ordering_ -- but then back to what we're doing here, how should we make an estimation on the prospective register pressure even _before_ all the instructions are scheduled?

As it turns out, in most scenarios Machine Scheduler cares more about the register pressure **difference** made by a single instruction. The scheduler initializes each instruction with a rought "guess" on how it might impact the overall pressure, which is stored in a data structure called `PressureDiff`. It is an array indexed by pressure set ID in which individual element expresses the degree or the quantity of pressure change.

Each `PressureDiff` associated with the instruction might get updated during the scheduling, but ultimately when an instruction is compared against others on register pressure inside `tryCandidate`, its `PressureDiff` is applied on top of the _current_ register pressure at that moment. For example, if instruction A could increase scalar registers' pressure by 2, and the current scalar pressure set already has a pressure of 4, then the prospective new pressure in the scalar pressure set will be 6, easy! This gives scheduler an idea on whether this instructions increases or decreases the _overall_ pressure (again, among the scheduled instructions at that moment) and how much does it change. More specifically, the scheduler wants to know the change on three metrics:

##### Excess pressure
How much does the new pressure goes overboard compare to the old pressure. While this sounds exactly like the pressure difference of an individual instruction we just talked about -- and in some cases, it is -- there is a catch here: excess pressure is "filtered" by a per-pressure set _threshold_ value.

The idea is that if we care about every tiny bit of reigster pressure changes, we might lose the big picture because those are just noise. By setting a lower bound threshold, we now calculate the excess pressure with only the pressure values that go over that line:

```
ExcessP = max(NewP, Threshold) - max(OldP, Threshold)
```

By this formula, if both new and old pressures are below the threshold, excess pressure would be zero -- because both of them are considered noise in this case; if only _one_ of the pressures goes over the threshold, the difference taken in this case will be confined by the threshold -- either `Threshold - OldP` or `NewP - Threshold`. In other words, this formula is effectively a _damper_ that prevents the pressure difference from oscillating too much.

##### Critical Max pressure
The next register pressure related heuristic compares the new register pressure (i.e. `NewP`) with the maximal pressure we have seen so far in a specific pressure set. To be more specific, this is the maximal pressure from _both_ the top and bottom `SchedBoundary` that are going on at this moment, so critical max pressure -- as it's called in Machine Scheduler -- is truely the maximal pressure of the entire scheduling region at the time.

The value we're interested in here, similar to excess pressure, is the difference between `NewP` and critical max pressure. But unlike excess pressure where the pressure difference can be nagative -- namely, decreasing the pressure -- we only consider the difference when `NewP` is larger than critical max pressure.

##### Current Max pressure
As confusing as the naming can go, _current_ max pressure represents the maximal pressure of a pressure set in the **original**, unscheduled program. Seriously, people should just call it "original max pressure". Aside from that, we also take the difference between `NewP` and current max pressure, if the former is larger than the latter.

These are the three metrics `tryCandidate` really cares about when it comes to the per-instruction register pressure impact. The execess pressure is always used before other two, as it represents the before v.s. after pressure of a specific pressure set, which is much more meaningful in most cases.

Before wrapping up this part, I would like to dive a little more into the technical details and breakdown the meanings of some key data structures -- most of them were already covered in the previous passages -- that have even _more_ confusing names than "current max pressure" v.s. "critical max pressure" in this part of the Machine Scheduler:

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

> Mention the downside of how `tryCandidate` compares each heuristics.

### Which scheduling model properties are used?

> Summarize what we've covering so far

### Future

> We should use some sort of cost model instead of an order list of heuristics as used by `tryCandidate`.