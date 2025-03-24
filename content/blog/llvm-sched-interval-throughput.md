+++
title = "Calculate Throughput with LLVM's Scheduling Model"
description = "Compiler, uArch, and a little bit of...jigsaw puzzle?"
date = "2025-03-23"
tags = ['llvm', 'compiler-instruction-scheduling', 'performance']
+++

From Cambridge Dictionary:
> **Throughput** /ˈθruː.pʊt/ (noun)
> 
>   - an amount of work done in a particular period of time.
 
In architecture-level performance analysis, throughput is usually measured by IPC -- Instruction Per Cycle. The inverse of this property, namely, inverse or reciprocal throughput, is also commonly used to describe the performance characteristics of a sinlge instruction.
It's not the time an instruction spends on to finish from start to end -- that is _latency_ -- but more closed to the amount of time it takes to finish a bunch of instructions amortized by their degree of (instruction-level) _parallelism_.

LLVM's scheduling model -- which we'd covered in [several posts](/llvm-sched-model-1) previously -- is a huge database describing the performance characteristics of instructions in a specific processor. While it specifies the instruction latency, a scheduling model does not spells out the inverse throughput of each instruction. We are, however, able to derive this property from other metrics in the model, and this short post is dedicated to show you how to do it.

#### Scheduling model in a nutshell

A scheduling model in LLVM describes an instruction with three primary properties:
  1. Latency
  2. Hardware resources it uses
  3. Number of cycles it "holds" on each of these hardware resources

A hardware resource can be thought as an _execution pipe_ in a superscalar processor. Let's say we have an instruction `BLAH` which uses three pipes, Pipe0 to Pipe2 (`P0` ~ `P2`), during its execution. We can describe the number of cycles it holds on each of these three pipes with a pair of numbers: `AcquireAtCycle` and `ReleaseAtCycle`.

`AcquireAtCycle` equals to the cycle where `BLAH` grabs a certain pipe and start working, **relative** to the cycle when instruction was issued, this value is usually zero, meaning this instruction starts using this resource as soon as it was issued -- we'll talk about the scenario where it's NOT (spoiler alert: it has more fun), but right now let's just assume it's always zero; similarly, `ReleaseAtCycle` is the cycle where `BLAH` releases this pipe for other instructions to use, also relative to the cycle instruction was issued.

To give a more concrete example, let's say `BLAH` has the following `AcquireAtCycle` and `ReleaseAtCycle` for each of the pipes it uses:

|    | AcquireAtCycle | ReleaseAtCycle |
|:--:|:--------------:|:--------------:|
| P0 |        0       |        1       |
| P1 |        0       |        3       |
| P2 |        0       |        2       |

We can put it on the timeline like this:

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-sched-interval-throughput-basic.dark.drawio.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-sched-interval-throughput-basic.light.drawio.svg">
  </picture>
</div>

As you probably also noticed, `ReleaseAtCycle` is the cycle where a resource had _already_ been released, which means that the time an instruction spends on a certain resource is an interval closed on the left and opened on the right:

`[AcquireAtCycle, ReleaseAtCycle)`

Making it easier to calculate the number of cycles in this interval by subtracting these two fields.

If we have two `BLAH` issued back-to-back:

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-sched-interval-throughput-basic2.dark.drawio.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-sched-interval-throughput-basic2.light.drawio.svg">
  </picture>
</div>

An instruction is allowed to be issued only if _all_ the hardware resources it needs are available. That's why the second instruction is issued on cycle 3 -- issued at any earlier cycles would result in at least one resource being unavailable.

It is also worth noting that `AcquireAtCycle` / `ReleaseAtCycle` only accounts for the time an instruction **exclusively** holds on a single resource[^2]. When we say `BLAH` holds `P1` for 3 cycles according to the table above, `BLAH` might take longer than 3 cycles to completely finish the entire instruction -- 3 cycles are just the duration it blocks every other instructions from using `P1`. After these 3 cycles `BLAH` might keep running until it finishes. And the time it takes to finish from start to end, is **latency**.

[^2]: This is why metrics like `AcquireAtCycle` / `ReleaseAtCycle` sometimes are also called **occupancy** -- the number of cycles it occupies a resource.

So assuming `BLAH` has a latency of 5 cycles, this is what it looks like when we zoom into `P1`:

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-sched-interval-throughput-ilp.dark.drawio.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-sched-interval-throughput-ilp.light.drawio.svg">
  </picture>
</div>

Each row is an instruction in this diagram. Boxes with solid edges are the duration `BLAH` holds onto `P1`; dashed boxes mark the extra time `BLAH` spends on to actually finish. As you can see, the combined duration of solid and dashed boxes is equal to the latency of `BLAH` -- 5 cycles.

Pretty straightforward, right? Now let's see how to calculate the (inverse) throughput in this scenario.

#### Basic throughput calculation

Earlier we mentioned that inverse throughput is the time to execute a group of instructions amortized by their degree of instruction-level parallelism. On the hind sight, it sounds like we need to know how long it takes to run `N` instructions, and divided by the number of instructions on the fly in this duration -- which are both something compiler cannot easily figure out statically.

But in reality, it's actually pretty easy to know the throughput with `ReleaseAtCycle`. Recall the diagram we just saw, but increase the number of instructions on the fly:

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-sched-interval-throughput-ilp2.dark.drawio.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-sched-interval-throughput-ilp2.light.drawio.svg">
  </picture>
</div>

The total number of cycles it takes to finish these instructions is `ReleaseAtCycle x N + (Latency - ReleaseAtCycle)` (or `ReleaseAtCycle x (N - 1) + Latency` if you prefer), where `N` is the number of instructions we see here, which means we're going to spend `3 x 4 + (5 - 3)`, 14 cycles in total.

If we increase the value of `N` to, let's say _infinity_, then `(Latency - ReleaseAtCycle)` -- which is a constant -- will actually become really insignificant! What actually dominates the total number of cycles becomes `ReleaseAtCycle x N`. 
So eventually, the inverse throughput with regards to this particular resource, `P1`, equals to `ReleaseAtCycle x N / N = ReleaseAtCycle`.

But hold on a second what about _other_ resources like `P0` and `P2`? Applying the same diagram on these two pipes and you'll find out their total time to finish `N` instructions are shorter than 14 cycles imposed by `P1`. Meaning `P1`'s total time dominates the inverse throughput of this instruction, making other two's insignificant (even they finish ealier, they have to wait for `P1`). And this is caused by the fact that `P1` has the _largest_ `ReleaseAtCycle`, which finally leads to a neat formula for calculating inverse throughput of the **entire instruction**:

```
max(ReleaseAtCycle_0, ReleaseAtCycle_1, ..., ReleaseAtCycle_N)
```
or
```
max(ReleaseAtCycles)
```

This formula is also what `MCSchedModel::getReciprocalThroughput` [uses](https://github.com/llvm/llvm-project/blob/616737c386776b0cfbda888a4d52e6036ccf1af8/llvm/lib/MC/MCSchedule.cpp#L107) to calculate the inverse throughput of an instruction -- modulo some differences like accounting for number of units of a resource[^1], which we assume it to be one in our discussions.

[^1]: We covered this concept in [here](/llvm-sched-model-1.5/).

Now, I won't spend hours wrestling with grammar, go all the way around to **just** write a blog post about something you can look up from code in a couple of minutes -- the things we've discussed so far are just a prelude to the **_fun_** part:

How to calculate throughput when `AcquireAtCycle` is not zero.

#### Calculating throughput with resource segments

Let's look at another instruction, `BLOB`, with the following `AcquireAtCycle` and `ReleaseAtCycle` numbers on `P0` ~ `P2`:

|    | AcquireAtCycle | ReleaseAtCycle |
|:--:|:--------------:|:--------------:|
| P0 |        0       |        2       |
| P1 |        2       |        5       |
| P2 |        1       |        3       |

When `AcquireAtCycle` is greater than zero, the instruction will not seize this resource right after being issued, but until another `AcquireAtCycle` cycles later. Which means the duration `BLOB` holds on each resources -- also knownw as **resource segments** -- now looks like:

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-sched-interval-throughput-segment.dark.drawio.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-sched-interval-throughput-segment.light.drawio.svg">
  </picture>
</div>

The biggest implication of having non-zero `AcquireAtCycle` is that the _second_ `BLOB` instruction that comes after will be issued and arranged like this:

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-sched-interval-throughput-segment2.dark.drawio.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-sched-interval-throughput-segment2.light.drawio.svg">
  </picture>
</div>

Meaning, we can issue the second `BLOB` as early as cycle 3 -- as opposed to cycle 5 when all of the `AcquireAtCycle` are zero. It's safe to "run ahead" and issue early because although `BLOB` seizes `P0` right away, there is a slack before it tries to acquire `P1` and `P2` a few cycles later.

This is how an instructions with resource segments is scheduled. Now, how do we calculate its inverse throughput?

There are several insights we learned about calculating inverse throughput of non-resource-segment instructions from the previous section:
  - Compared to `ReleaseAtCycle` (and `AcquireAtCycle`), latency doesn't really matter -- it'll eventually be a single constant appended at the end of the formula that can be ignored when number of instructions (i.e. `N`) is approaching infinity
  - The whole calculation can be simplified to dividing the cycle at which the last instruction releases the last resource -- or the _right-most_ cycle, which is cycle 7 in the last diagram -- by `N`. For non-resource-segment instructions, the right-most cycle is always `max(ReleaseAtCycles) x N`

So our task can really boil down to finding the right-most cycle here.
Looking at the last diagram, we might considering using `max(ReleaseAtCycles) x N` as the right-most cycle for instructions with resource segments here again.

Because first, the resource with `max(ReleaseAtCycles)` -- `P1` in this case -- will always be the resource where righ-most cycle happens. Now the question would be the quantity of this right-most cycle: _Intuitively_, resource with `max(ReleaseAtCycles)` will always be the one that concatenates with its counterpart in the next instruction, back to back, without any "gap" in between. So the quantity of right-most cycle can be easily calculated as `max(ReleaseAtCycles) x N`.

As you might have noticed, having no gap is an important prerequsite here, otherwise if there is a gap, it will becomes part of the _recurring_ factor. Namely, the right-most cycle will be something like `(max(ReleaseAtCycles) + C) x N` where `C` is a constant factor of gap.

But is it always the case for resources with `max(ReleaseAtCycles)` to have no gaps in between?

Sadly, here is a counter example:

|    | AcquireAtCycle | ReleaseAtCycle |
|:--:|:--------------:|:--------------:|
| P0 |        0       |        2       |
| P1 |        4       |        5       |
| P2 |        1       |        4       |

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-sched-interval-throughput-segment3.dark.drawio.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-sched-interval-throughput-segment3.light.drawio.svg">
  </picture>
</div>

There is still a silver lining in this though: regardless of gap, the resource with `max(ReleaseAtCycles)` is still the resource where righ-most cycle happens -- as we can observe from this counter example as well. So if we can figure out `C` -- the constant factor of gap -- then we still can calculate the inverse throughput in terms of `max(ReleaseAtCycles)`.

For that, let's play a liiiitle bit of jigsaw puzzle.

Let's step back a little bit and think about how we placed our second instruction in the first place which eventually led to the last diagram: starting from cycle 0, we effectively "shift" the second instruction right by `M` cycles, so far beyond that there is no overlap between the two instructions.

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-sched-interval-throughput-segment4.dark.drawio.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-sched-interval-throughput-segment4.light.drawio.svg">
  </picture>
</div>

Then, we shift the second instruction left by `D` until one of the resources _touches_ its counterpart in the first instruction.

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-sched-interval-throughput-segment4-2.dark.drawio.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-sched-interval-throughput-segment4-2.light.drawio.svg">
  </picture>
</div>

To simplify, we can set `M`, the right shift amount, to be `max(ReleaseAtCycles)` (`R_max` in the diagram below):

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-sched-interval-throughput-segment4-3.dark.drawio.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-sched-interval-throughput-segment4-3.light.drawio.svg">
  </picture>
</div>

With `M = max(ReleaseAtCycles)`, the distance between the right edges of both `P1` boxes becomes `max(ReleaseAtCycles) - D` after we shift the second instruction left by `D`:

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-sched-interval-throughput-segment4-4.dark.drawio.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-sched-interval-throughput-segment4-4.light.drawio.svg">
  </picture>
</div>

This distance, `max(ReleaseAtCycles) - D`, is especially important because it's the recurring factor `max(ReleaseAtCycles) + C` mentioned earlier (i.e. `C = -D`). In other words, the right-most cycle we've been looking for is equal to `(max(ReleaseAtCycles) - D) x N` when `N` is sufficiently big:

<div style="text-align: center;">
  <picture>
    <source srcset="/images/llvm-sched-interval-throughput-segment4-5.dark.drawio.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/llvm-sched-interval-throughput-segment4-5.light.drawio.svg">
  </picture>
</div>

So now we only have one thing left: finding the value of `D`.

During the left shifting, if we look at each hardware resource in the second instruction _individually_, each of them needs to move a different distance before touching their counterpart in the first instruction. Using the last setup as example, `P0` needs to shift 3 cycles left, 6 cycles for `P1`, and 2 cycles for `P2`. The value of `D` would be the minimum among them (i.e. `D = 2`). Furthermore, the individual shifting amount can be expressed as
```
(AcquireAtCycle + M) - ReleaseAtCycle
```
Because `AcquireAtCycle + M` is effectively the "new" `AcquireAtCycle` after we shift it right by `M`, and subtracting it with `ReleaseAtCycle` (of the first instruction), would give you the distance to close up.

Therefore:
```
D = min((AcquireAtCycle_i + M) - ReleaseAtCycle_i), ∀ i = P0 ~ P2
  = min(M - (ReleaseAtCycle_i - AcquireAtCycle_i))
```

Because `M` is constant, to get the minimum of `M - (ReleaseAtCycle_i - AcquireAtCycle_i)` we need the largest `ReleaseAtCycle_i - AcquireAtCycle_i`...which is the interval / segment with **longest** length! (i.e. `i = P2` in this case)

But let's not stop here: because `M = max(ReleaseAtCycles)`, the recurring factor becomes...
```
  max(ReleaseAtCycles) - D
= max(ReleaseAtCycles) - M + (ReleaseAtCycle_i - AcquireAtCycle_i), i = P2
= ReleaseAtCycle_i - AcquireAtCycle_i, i = P2
```

Which makes the right-most cycle (of all `N` instructions) we've been looking for equals to:
```
(ReleaseAtCycle_i - AcquireAtCycle_i) x N, i = P2
```
As it turns out, the inverse throughput of `N` instructions with resource segments -- right-most cycle divided by `N` -- simply equals[^3] to the **longest** segment length among all hardware resources!

```
Inverse throughput =
  max(ReleaseAtCycle_i - AcquireAtCycle_i), ∀ i = hardware resources
```

This formula also covers both the basic cases from section 2 and cases with resource segments.

[^3]: Again, assuming number of units in each resource is equal to 1. If it's greater than 1, then the inverse throughput should be further divided by the number of units.

#### Discussions

Here is a little background of why I wrote this post: at the time of writing, `MCSchedModel::getReciprocalThroughput` doesn't handle `AcquireAtCycle` at all, this [comment](https://github.com/llvm/llvm-project/pull/130574#discussion_r2005110229) from a code review raised a question on how to calculate it, which nerd-sniped me and as a consequence, this post.

In that thread, [@jvillette38](https://github.com/jvillette38) also proposed using longest resource segment as inverse throughput. However, it was slightly counter-intuitive to me at that time as inverse throughput should be calculated from the right-most cycle of all `N` instructions, yet is the resource with the longest segment _always_ be the one where right-most cycle happens? And in general, what's the connection between the longest segment length and the right-most cycle?

We've shown the answer of the first question to be "No". Though as it turns out, inverse throughput is still dominated by the longest segment. I think this can partially be explained by  `D`'s formula, which suggests that the longest hardware resource segment in the second instruction is always the first to touch its counterpart in the first instruction during left shifting, with **no gap** in between. Implying that the total length (i.e total cycles) is equal or larger than the longest segment length times `N`. And in the case where resource with the longest segment is not where right-most cycle happens, the "delta" between the release cycle of _last_ longest segment and the actual right-most cycle will just be a constant which we can happily ignore when `N` is sufficiently large.

Anyway, I hope I'm not overthinking this whole time 😛 and I hope this post provides a stronger argument on using largest segment length as inverse throughput. 

### Comments
Feel free to leave comments at https://github.com/mshockwave/portfolio/discussions/12