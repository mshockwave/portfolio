+++
title = "Legalizations in LLVM Backend"
date = "2024-04-28"
draft = true
tags = ['llvm', 'compiler-codegen']
+++

In an ideal world, compilers can build a program for a wide variety of hardware without the need to change a single line of its source code.
While there are exceptions and corner cases, this holds in the majority of cases.
Which means that if the input code uses something that is not directly available on the hardware, the compiler has to figure out a way to effectively _emulate_ that
specific feature.

This sounds distant and even a little scary, but I'm not even talking about a problem that only happens on some exotic proprietary ML accelerator whutnuts. There is a good
example of this regarding something you use almost everyday: **boolean variables**.
I'm pretty sure none of the modern processors provides 1-bit registers[^1] (or addressable memory space), yet we still use boolean variables extensively.

[^1]: Status registers like EFLAGS in X86 might qualify (I mean its individual status bits), but it's still far away from being generally usable.

Other common examples include the lack of double-precision floating point operations, or even lacks floating point unit altogether in some embedded devices.
The process that "reshapes" input programs into using what's available on the target hardware is called **legalization** in LLVM, and it's done in LLVM's code generation (codegen)
pipeline a.k.a the backend[^2].
In this post, I'm going to give an overview on how it works.

[^2]: Though frontends like Clang do generate target-specific LLVM IR, a lot of those target-specific bits are in regard to ABI conformance rather than legalizations.

### A sneak peek into SelectionDAG ISel
LLVM's codegen pipeline in the backend is consisted of LLVM Passes, same as the middle-end[^3]. This long long pipeline is usually segmented by important events, and the
legalization happens around one of the earlier ones, instruction selection (ISel).

[^3]: At the time of writing, the codegen pipeline hasn't migrated to using the _new_ PassManager yet, while the middle-end had wrapped up the migration years ago.

LLVM has several different ISel stratgies co-existing at this moment. Here, we're focusing on **SelectionDAG ISel** first, which is the primary one implemneted by every targets.
This ISel turns instrutions in each basic block into a DAG, different from the "linear" representation of instructions as we seen in LLVM IR.

Though this DAG is not so relevant in our discussion here, we can roughly divide SelectionDAG ISel further into 4 steps, they are:
  1. Building SelectionDAG
  2. Type legalization
  3. Legalizing operations
  4. Instruction selection

There are actually lots of going on in between the steps, like optimizing the DAG (by DAGCombiner) and LegalizeVectorOps in the presence of vectors (which should really be called "scalarize vector ops"). But in any case, legalization is primarily consist of two separate steps, type legalization and legalizing the operations (i.e. instructions). Let's cover these two in order.

#### Type legalization
Type legalization tries to turn unsupported types into the ones supported by the target architecture. To give you a better idea on how this works in action, let's send the following
LLVM IR snippet into SelectionDAG ISel and see how it got legalized.
```llvm
define i32 @foo(i32 %v) {
  %lo = trunc i32 %v to i16
  %p = add i16 %lo, 5
  %c = icmp ugt i16 %p, 6
  %r = select i1 %c, i32 %v, i32 9
  ret i32 %r
}
```
In order to print the trace of SelectionDAG ISel, please follow the instructions here to make sure the tool we're about to use, `llc`, has the right capability.
{{% details "Notes on llc" %}}
Printing the trace of SelectionDAG ISel requires either a debug build of LLVM or an LLVM with assertions enabled. Unfortunately, prebuilt LLVM provided by major Linux / BSD / MacOSX
distributions meet none of the requirements. So you might have to build LLVM from source.
  1. Please checkout the build instructions [here](https://llvm.org/docs/CMake.html#quick-start)
  2. During cmake configuration phase, either you set it to debug build by passing `-DCMAKE_BUILD_TYPE=Debug`, or passes `-DLLVM_ENABLE_ASSERTIONS=ON` to enable assertions on a release build
  3. We're about to use RISC-V as the target throughput the examples in this post, please make sure it's built, which is the default.
  4. Run `cmake --build . --target llc` to build only the `llc`.
{{% /details %}}
Then, please run the following command with `input.ll` being the snippet we saw previously.
{{% details "llc command" %}}
`llc -mtriple riscv32 -debug-only=isel-dump input.ll -o /dev/null`
{{% /details %}}

You'll see an output partitioned into several sections, starting with sentences like _"Initial selection DAG: ..."_ or _"Optimized lowered selection DAG: ..."_

```
Initial selection DAG: %bb.0 'foo:'
...
Optimized lowered selection DAG: %bb.0 'foo:'
...
Type-legalized selection DAG: %bb.0 'foo:'
...
Optimized type-legalized selection DAG: %bb.0 'foo:'
...
Legalized selection DAG: %bb.0 'foo:'
...
Optimized legalized selection DAG: %bb.0 'foo:'
...
Selected selection DAG: %bb.0 'foo:'
...
```
These sections correspond to the 4 steps we've discussed previously. Each of these section shows the SelectionDAG after the step, like this:
```llvm
Initial selection DAG: %bb.0 'foo:'
SelectionDAG has 14 nodes:
  t0: ch,glue = EntryToken
  t2: i32,ch = CopyFromReg t0, Register:i32 %0
          t3: i16 = truncate t2
        t5: i16 = add t3, Constant:i16<5>
      t8: i1 = setcc t5, Constant:i16<6>, setugt:ch
    t10: i32 = select t8, t2, Constant:i32<9>
  t12: ch,glue = CopyToReg t0, Register:i32 $x10, t10
  t13: ch = RISCVISD::RET_GLUE t12, Register:i32 $x10, t12:1
```
Again, we're not going into details of the DAG. But here are some quick primers on reading this DAG:
{{% details "How to read SelectionDAG in 30 seconds or less"%}}
  - SelectionDAG stills keeps the SSA form, so an operation like `t3: i16 = truncate t2` defines value `t3`, which is used by `t5: i16 = add t3, Constant:i16<5>` as its first operand
  - `t3: i16` means that value `t3` has a 16-bit integer type
    - Don't worry about types like `ch` (chain) and `glue`. They're used to express dependencies stem from control flow or side effects.
  - Most of the operations here, like `truncate`, `add`, and `select` are pretty easy to understand. `setcc` is basically a comparison operation, which compares its first and second operand (in this case `t5` and constant 6) according to the conditional code in the third operand(in this case `setugt` -- unsigned greater than). `CopyFromReg`, as its name suggested, copies values from a certain physical register to a value like `t2`.
    - Don't worry about the rest of the operations like `RISCVISD::RET_GLUE`. We're not going to need them here
{{% /details %}}

What we're really interested in here is the differences before and after the type legalization step.
Here is the DAG before:
```llvm
Optimized lowered selection DAG: %bb.0 'foo:'
SelectionDAG has 14 nodes:
  t0: ch,glue = EntryToken
  t2: i32,ch = CopyFromReg t0, Register:i32 %0
          t3: i16 = truncate t2
        t5: i16 = add t3, Constant:i16<5>
      t8: i1 = setcc t5, Constant:i16<6>, setugt:ch
    t10: i32 = select t8, t2, Constant:i32<9>
  t12: ch,glue = CopyToReg t0, Register:i32 $x10, t10
  t13: ch = RISCVISD::RET_GLUE t12, Register:i32 $x10, t12:1
```
And this is the type-legalized DAG:
```llvm
Type-legalized selection DAG: %bb.0 'foo:'
SelectionDAG has 17 nodes:
  t0: ch,glue = EntryToken
  t2: i32,ch = CopyFromReg t0, Register:i32 %0
            t16: i32 = add t2, Constant:i32<5>
          t22: i32 = and t16, Constant:i32<65535>
        t17: i32 = setcc t22, Constant:i32<6>, setugt:ch
      t20: i32 = and t17, Constant:i32<1>
    t10: i32 = select t20, t2, Constant:i32<9>
  t12: ch,glue = CopyToReg t0, Register:i32 $x10, t10
  t13: ch = RISCVISD::RET_GLUE t12, Register:i32 $x10, t12:1
```
First, let's look at line 4 ~ 6 in the **pre**-type-legalized DAG:
```llvm
t2: i32,ch = CopyFromReg t0, Register:i32 %0
    t3: i16 = truncate t2
  t5: i16 = add t3, Constant:i16<5>
```
The `t5: i16 = add t3, Constant:i16<5>` obviously corresponds to the `%p = add i16 %lo, 5` instruction in our original LLVM IR, in which both of them are _16-bit_ arithmetic summation. However, a physical register in 32-bit RISC-V (RV32) is always 32 bits, therefore after copying values from physical register `%0` via `CopyFromReg`, we have to truncate its value to 16 bits before feeding into the `add`, hence `t3: i16 = truncate t2`.

But wait a second, RV32 doesn't have any 16-bit arithmetic add instruction either! Actually, at this moment, none of the RISC-V instructions is capable of processing 16-bit data natively. That means we can never lower `t5: i16 = add t3, Constant:i16<5>` to a single RISC-V instruction. What we can do is _synthesizing_ it with 32-bit arithmetic instructions, therefore we get this in the **post**-type-legalized DAG:
```llvm
t2: i32,ch = CopyFromReg t0, Register:i32 %0
    t16: i32 = add t2, Constant:i32<5>
  t22: i32 = and t16, Constant:i32<65535>
```
`t22: i32 = and t16, Constant:i32<65535>` effectively zeros out the higher 16 bits in the result produced by the now-32-bit arithmetic add instruction, `t16: i32 = add t2, Constant:i32<5>`, which makes sure the result has the same precision as before. With this transformation, we not only ensure that all operations are only using types supported by RV32, the calculation result is also correct. In other words, we turn operations that use 16-bit integers -- an _illegal type_ in RISC-V -- into legal ones, hence the name of type legalization.

Let's look at another similar example: line 7 and 8 in the pre-type-legalized DAG.
```llvm
  t8: i1 = setcc t5, Constant:i16<6>, setugt:ch
t10: i32 = select t8, t2, Constant:i32<9>
```
As we mentioned at the beginning of this post, none of the modern processors really supports 1-bit type natively, and RISC-V is not an exception, which means 1-bit integer/boolean is considered an illegal type. Here, the type legalization did a similar thing we've seen previously: turning boolean into 32-bit integers and apply a proper mask:
```llvm
    t17: i32 = setcc t22, Constant:i32<6>, setugt:ch
  t20: i32 = and t17, Constant:i32<1>
t10: i32 = select t20, t2, Constant:i32<9>
```
The result of `setcc` is changed from `i1` to `i32`, whose higher 31 bits are cleared by the mask before feeding into the `select` operation.

##### Determining legal types and actions on illegal types
So how is a type to be considered legal or illegal in a target? In LLVM, the criteria is pretty straightforward: types that can natively fit into physical register are considered legal and deemed illegal otherwise. This information is setup by a target in its TargetLowering, which is usually put in `XXXXISelLowering.cpp` where "XXXX" is the target name. For instance, in RISC-V, it looks like:
```cpp
// Set up the register classes.
addRegisterClass(XLenVT, &RISCV::GPRRegClass);
```
In which `XLenVT` represents 32-bit integer for RV32 and 64-bit integer for RV64 (64-bit RISC-V). Take RV32 as an example, this line basically says that `GPRRegClass` -- a group of general-purpose registers -- can carry 32-bit integers. Since `GPRRegClass` is the only integer reigster class in RISC-V, 32-bit integer is the only legal integer type in RV32[^4].

[^4]: Floating point is optional in RISC-V, and we're not covering it here either.

What's a little more complicate is how we deal with illegal types. The type legalizer will determine the best action to turn them into legal types. What we've seen earlier, turning 16-bit or 1-bit integers into 32-bit integers, is _promotion_ (turning smaller types to larger ones). Other actions include expanding integer / floating point (split larger types into two smaller ones), soften floating point (turn into integer of equivalent size), split vector (cut vectors into shorter length) and widen vector (increase the vector length) etc.

The type legalizer has a pre-defined sequence to perform for each of these legalization actions, so a target doesn't really have to specify how to actually do the legalization in this part.


An important takeaway from type legalization is that it looks at every single value appears in the program, checks its type and tries to legalize it if needed. It doesn't care what **operation** the value came from. In other words, legal type in this phase is a _global_ concept, indepdent from individual operations. This is different from what we're going to see in legalizing operation in the next section, where individual operation has different interpretations of its own legality.