+++
title = "Visualize RISC-V Vector Memory Instructions"
date = "2025-01-05"
tags = ['riscv']
+++

RISC-V Vector (RVV) extension has several kinds of load / store instructions which access memory in different ways. Just as the memory access pattern might take a little more time to fully understand, it gets even more tricky when multiplexing with RVV's own concepts like variable element size (SEW), register groups (LMUL), number of elements (VL), masks and mask / tail policies.

Personally I found it easier to memorize them with visualization, hence this (relatively) short post!

The following content is going to put these instructions in two main categories by their memory access patterns: _strided_ and _segmented_ access. Each of them can be further divided into several sub categories.
We also assume `VLEN` -- the size of a single vector register -- to be 128 bits. And since RVV store instructions work nearly the same way as loads except going to other direction from registers to memory, we're only discussing load instructions here.

Without further ado, let's get started!

### Strided access

Strided memory access is meant to read individual elements from the memory into a _single_ vector register group. The sub variants of this mainly differ in how the "gap" between two in-memory elements is determined (or whether there is a gap at all).

#### Unit-stride

Consider this snippet:
```
vsetvli  zero, zero, e32, m4
vle32.v  v4, (a0)
```
We're loading a _continuous_ (meaning, no gap between two elements) chunk of data starting from the memory address pointed by `a0`.

This is the diagram for it:

<div style="text-align: center; min-width: 50vw;">
  <picture>
    <source srcset="/images/riscv-unit-stride-load.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/riscv-unit-stride-load.light.svg">
  </picture>
</div>

Here, I would like to spend some time on an important concept called **effective** LMUL and SEW, or _EMUL_ and _EEW_, respectively. Normally, we use VSETVL instructions, like `vsetvli` shown above, to select the current LMUL (register grouping) and SEW (element width) settings.

But case like the unit stride load we're discussing here uses their _own_ element width setting that is directly encoded into the opcode. And such element width setting, the one they actually use, is called EEW -- effective element width.

For unit stride load, EEW is placed in the opcode with this format: `vle<eew>.v`. So regardless of what `vsetvli` instruction right before it says, `vle32.v` always loads 32-bit elements from the memory.

But does that means we can ignore the SEW setting specified in the `vsetvli` instruction above? It turns out we cannot, because if the SEW (by `vsetvli`) is different from EEW (by `vle32.v`), we have to _scale_ the register grouping (LMUL) as well! The scaled register grouping -- effective LMUL or _EMUL_ -- is the one we actually use, and it's calculated by 

```
EMUL = (EEW / SEW) * LMUL
```

To give a concrete example, considered the following snippet:
```
vsetvli  zero, zero, e32, m4
vle64.v  v4, (a0)
```
EEW (e64) now differs from SEW (e32). Therefore, while it loads 64-bit elements from memory, the register grouping it actually uses (i.e. EMUL) is now 8, because `EMUL = (e64 / e32) * 4`.

The idea of scaling LMUL is to make sure `VLMAX` -- the maximum number of elements we can process, or the maximum `VL` value -- stays the same. 

#### Strided
For this, we're loading a stream of data in which elements are apart from each other in the memory by a certain distance or, _stride_.

```
li  t0, 8
vsetvli  zero, zero, e32, m4
vlse32.v  v4, (a0), t0
```
For instance, the snippet above loads the next element 8 bytes away from the starting address of the current element. Here is how it looks like:

<br>
<div style="text-align: center; min-width: 45vw;">
  <picture>
    <source srcset="/images/riscv-stride-load.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/riscv-stride-load.light.svg">
  </picture>
</div>

In this diagram, "Stride" is equal to 8, which is designated by `t0` in the original snippet.

It's worth noting that `VL` is still describing the total number of elements we want to load, or number of elements in the destination vector register group, rather than the "total range" of EEW-size elements on the memory. Same logic goes to the mask: it's applying on the vector register group rather on the memory.

#### Indexed

What if we don't want a _single_ constant stride, but different "strides" for each elements?
Allow me to introduce the indexed scheme, where the memory address offset -- or index value -- for each element is specified by yet another vector register group.

```
vsetvli  zero, zero, e32, m4
vluxei32.v  v4, (a0), v0
```
Here, the register group started with `v0` contains the offset values (in bytes) for each element it wants to load. This is how it looks:

<br>
<div style="text-align: center; min-width: 50vw;">
  <picture>
    <source srcset="/images/riscv-index-load.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/riscv-index-load.light.svg">
  </picture>
</div>

With a new vector register group as indices come into play, indexed load has a slightly different rule regarding EEW and EMUL. The element width encoded in the opcode -- 32 bits in the case of `vluxei32` -- becomes EEW for the _index_ register group, which consequently uses the scaled register grouping factor as its EMUL.
On the other hand, the data register group now uses SEW and LMUL specified by `vsetvli` as their element width and register grouping factor!

As shown in the diagram, these offset values from the index register group are always applied relative to the starting address -- the one pointed by `a0` -- rather than being relative to the address of the previous element.

Another thing worth noting is that the instruction we used, `vluxei32.v`, accesses elements on memory in _arbitrary_ order, just like unit-stride and constant-stride loads. RVV, however, does provide another variant of indexed load, `vloxei32.v`, that accesses memory in-order.

### Segmented access

Rather than dealing with individual in-memory elements, segmented memory access pattern reads a larger chunk of memory -- namely, a segment -- at a time and distributes the content of a segment into _multiple_ vector register groups.

It is designed for scenarios where there is an array of objects on memory, for instance:
``` cpp
struct Point {
  uint32_t X;
  uint32_t Y;
};

Point the_array[100];
```

We want to read `the_array` from memory _and_ effectively turn it into something like this:
```cpp
// Note: this is just a pseudo code.
struct NewPoint {
  uint32_t X[100]; // A vector register group.
  uint32_t Y[100]; // The other vector register group.
};

NewPoint the_array_new;
```

Where we use a vector register group to store all the `X` values, and use the other vector register group for `Y`.

In this case, `struct Point` is considered a **segment** with two _fields_ (i.e. `X` and `Y`). A segment is also known as a "sub-array", which means that individual fields in a segment need to have the _same_ data width. Therefore, this is NOT a segment:
```cpp
struct Foo {
  uint16_t A;
  uint32_t B;
};
```

For the following content, let's use `struct Point` as the segment.

#### Unit-stride segmented

Unit segmented load encodes the number of fields (NF) along with EEW in its opcode: `vlseg<NF>e<EEW>.v`.

Take following snippet as an example:
```
vsetvli  zero, zero, e32, m4
vlseg2e32.v  v4, (a0)
```
It's loading a _continuous_ sequence of two-field segments starting with the memory address pointed by `a0`. Here is the diagram:

<br>
<div style="text-align: center; min-width: 60vw;">
  <picture>
    <source srcset="/images/riscv-unit-segmented-load.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/riscv-unit-segmented-load.light.svg">
  </picture>
</div>

The most unique aspect here is of course how it deals with individual fields in a segment: in principle, values from the same field is store in a single vector register group.

The destination register group specified in the assembly instruction, `v4` in this case, tells the register group (i.e. `v4` ~ `v7`) for the first field. The next field will be stored in the _following_ register group (i.e. `v8` ~ `v11`), and so on and so forth.

So if we have something like `vlseg3e32.v  v4, (a0)` with EMUL = 2, a total of three vector register groups will be touched:  `v4` ~ `v5`, `v6` ~ `v7`, and `v8` ~ `v9`.

Similar to strided and indexed loads mentioned earlier, `VL` and mask are applied on each of the (destination) vector register groups, rather than applying on the memory.

#### Strided segmented

This is the segmented version of strided load we'd seen earlier.

```
li  t0, 16
vsetvli  zero, zero, e32, m4
vlsseg2e32.v  v4, (a0), t0
```

<br>
<div style="text-align: center; min-width: 50vw;">
  <picture>
    <source srcset="/images/riscv-stride-segmented-load.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/riscv-stride-segmented-load.light.svg">
  </picture>
</div>

The start address of the next segment is equal to that of the current segment plus stride.

#### Indexed segmented

Finally, indexed segmented is, you guess, the segmented version of indexed load! (duh...)

```
vsetvli  zero, zero, e32, m4
vluxseg2ei32.v  v4, (a0), v0
```

<br>
<div style="text-align: center; min-width: 60vw;">
  <picture>
    <source srcset="/images/riscv-index-segmented-load.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/riscv-index-segmented-load.light.svg">
  </picture>
</div>

Again, the offset values are always relative to the starting address (in this case, value of `a0`) rather than being relative to the address of the previous segment.

Similar to indexed load, `vluxseg` accesses memory in arbitrary order while `vloxseg` being its ordered variant. The EEW encoded in the opcode is for the index register group while all the data register groups are using SEW and LMUL.

----

And that's pretty much it! Once we categorized in this way it's actually not so hard to understand. Segmented access might look daunting at first glance, but once you're familiar with segments, rest of the concepts can just be carried over from their strided access counterparts.

### Comments
Feel free to leave comments at https://github.com/mshockwave/portfolio/discussions/8

Any feedback is much appreciated!