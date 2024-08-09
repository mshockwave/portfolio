+++
title = "Legalizations in LLVM Backend"
date = "2024-05-15"
tags = ['llvm', 'compiler-codegen']
+++

Ideally, compilers can build a program for a wide variety of hardware without the need to change a single line of its source code.
While there are exceptions and corner cases, this holds in the majority of cases.
Which means that if the input code uses something that is not directly available on the hardware, the compiler has to figure out a way to effectively _emulate_ those
features.

This might sound a little distant to our typical software development experiences, but I'm not even talking about a problem that only happens on some exotic proprietary ML accelerator whatnots. There is a good
example of this regarding something you use almost everyday: **boolean variables**.
I'm pretty sure none of the modern processors provides 1-bit registers[^1] (or addressable memory space), yet we still use boolean variables extensively.
Other common examples include the lack of double-precision floating point operations, or even lacks floating point unit altogether in some embedded devices.

[^1]: Status registers like EFLAGS in X86 might qualify (I mean its individual status bits), but it's still far away from being generally usable.

The process that "reshapes" input programs into using what's available on the target hardware is called **legalization** in LLVM[^2], and it's done in LLVM's code generation (codegen)
pipeline a.k.a the **backend**.
In this post, I'm going to give an overview on how it works.

[^2]: Though frontends like Clang do generate target-specific LLVM IR, a lot of those target-specific bits are in regard to ABI conformance rather than legalizations.

### A sneak peek into SelectionDAG ISel
LLVM's codegen pipeline in the backend is consisted of LLVM Passes, same as the middle-end[^3]. This long long pipeline is usually partitioned by important events like register allocation and instruction scheduling, and the
legalization happens around one of the earlier events, the instruction selection (ISel).

[^3]: At the time of writing, the codegen pipeline hasn't migrated to using the _new_ PassManager yet, while the middle-end had wrapped up the migration years ago.

Several different ISel stratgies co-existing in LLVM at this moment. Here, we're focusing on **SelectionDAG ISel** first, which is the primary one implemented by every targets.
This ISel turns instrutions in each basic block into a DAG, different from the "linear" representation of instructions as we seen in LLVM IR.

We can roughly divide SelectionDAG ISel further into 4 steps, they are:
  1. Building SelectionDAG
  2. Type legalization
  3. Legalizing operations
  4. Instruction selection

There are actually lots of going on in between the steps, like optimizing the DAG (by DAGCombiner) and LegalizeVectorOps in the presence of vectors (which should really be called "scalarize vector ops"). But in any case, legalization is primarily consist of two separate steps, type legalization and legalizing the operations (i.e. instructions). Let's cover these two in order.

### Type legalization
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

#### Determining legal types and actions on illegal types
So how is a type to be considered legal or illegal in a target? In LLVM, the criteria is pretty straightforward: types that can natively fit into physical register are considered legal and deemed illegal otherwise. This information is setup by a target in its TargetLowering, which is usually put in `XXXXISelLowering.cpp` where "XXXX" is the target name. For instance, in RISC-V, it looks like:
```c
// Set up the register classes.
addRegisterClass(XLenVT, &RISCV::GPRRegClass);
```
In which `XLenVT` represents 32-bit integer in RV32 and 64-bit integer in RV64 (64-bit RISC-V). Take RV32 as an example, this line basically says that `GPRRegClass` -- a group of general-purpose registers -- can carry 32-bit integers. Since `GPRRegClass` is the only integer reigster class in RISC-V, 32-bit integer is the only legal integer type in RV32[^4].

[^4]: Floating point is optional in RISC-V, and we're not covering it here either.

What's a little more complicate is how we deal with illegal types. The type legalizer will determine the best action to turn them into legal types. What we've seen earlier, turning 16-bit or 1-bit integers into 32-bit integers, is _promotion_ (turning smaller types to larger ones). Other actions include expanding integer / floating point (split larger types into two smaller ones), soften floating point (turn into integer of equivalent size), split vector (cut vectors into shorter length) and widen vector (increase the vector length) etc.

The type legalizer has a pre-defined sequence to perform for each of these legalization actions, so a target doesn't really have to specify how to actually do the legalization in this part.


An important takeaway from type legalization is that it looks at every single value appears in the program, checks its type and tries legalizing it if needed. It doesn't care what **operation** the value came from. In other words, the concept of legal type in this phase is _global_ and indepdent from individual operations. This is different from what we're going to see in legalizing operation in the next section, where individual operation has different interpretations of its own legality.

### Legalizing operations
A SelectionDAG is generated from a single basic block of the source LLVM IR. Each instruction in the basic block is basically translated into a single SelectionDAG node[^5] called `SDNode`, representing an operation like `t16: i32 = add t2, Constant:i32<5>` we've seen previously. Initially, each SDNode has a **generic**, target-independent opcode. In a (heavily) hand-waving analogy, a SDNode in this stage is basically an one-to-one translation from its LLVM instruction which is equally target-independent.
Nearly all of the SDNodes we've seen so far in the examples, like `t20: i32 = and t17, Constant:i32<1>` and `t16: i32 = add t2, Constant:i32<5>` are SDNodes with generic opcodes[^6].

[^5]: Well, not always a single SDNode, since the SelectionDAG builder actually delegates lots of DAG building logics to each target and each target can definitely generate more than one SDNode from a LLVM instruction. The most notable example is function calls: each target implements `TargetLowering::LowerCall` to lower a `llvm::CallInst` to its native function call constructions consisting of one or more SDNodes.

[^6]: The full list of opcodes can be found under [llvm/include/llvm/CodeGen/ISDOpcodes.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/CodeGen/ISDOpcodes.h).

Not every of these SDNodes have its corresponding native instructions in the target hardware, though. For instance, some less-powerful processors do not natively support rotate  / funnel shifts. Meaning, in those cases we have to turn an operation rotating left by 3 bits (e.g. `t4: i32 = rotl t2, Constant:i32<3>`) into something else, for instance . For these targets, a `rotl` operation is considered _illegal_ and we're **legalizing** such operations into supported, legal operations in the phase follows type-legalization.

{{% details "What are funnel / rotate shifts"%}}
Funnel shift is a special variant of bit shifting that fills in the spaces left by bits shifted away with bits from another value. It's a function that takes two bit sequences (e.g. integers) and a constant value specifying the number of bits to shift.
For example, given two 5-bit integers A and B:
```
MSB                    LSB
| A4 | A3 | A2 | A1 | A0 |
--------------------------
| B4 | B3 | B2 | B1 | B0 |
```
Funnel left shift A and B by 2 bits, `funnel_left(A, B, 2)`, yields the following result:
```
| A2 | A1 | A0 | B4 | B3 |
```
You can imagine it being A shifts left by 2 bits while the empty space in the lower bits are filled in by the higher two bits of B.

If A and B are identical, for instance `funnel_left(A, A, 2)`, then it becomes a **rotate left** operation, as it yields the following result that looks like the higher bits in A that got shifted out are "wrapping around" to the lower bits:
```
| A2 | A1 | A0 | A4 | A3 |
```
{{% /details %}}

In fact, since RISC-V's rotate instructions are optional (they are defined in Zbb and Zbkb extensions), let's see how RISC-V handles rotate _in absence of_ native rotate instructions. Here is the input LLVM IR:

```llvm
define i32 @foo(i32 %v) {
  %r = call i32 @llvm.fshl.i32(i32 %v, i32 %v, i32 3)
  ret i32 %r
}
```
`llvm.fshl.*` is the [intrinsic](https://llvm.org/docs/LangRef.html#llvm-fshl-intrinsic) for funnel left shifts.
If we use the exactly same `llc` command as earlier to compile this snippet and dump the DAGs, this is the (optimized) DAG right after type-legalization:
```llvm
  t0: ch,glue = EntryToken
      t2: i32,ch = CopyFromReg t0, Register:i32 %0
    t4: i32 = rotl t2, Constant:i32<3>
  t6: ch,glue = CopyToReg t0, Register:i32 $x10, t4
  t7: ch = RISCVISD::RET_GLUE t6, Register:i32 $x10, t6:1
```
`rotl` on line 3 is the rotate left operation. It rotates its first operand (`t2`) by the number of bits specified in the second operand (i.e. 3).

After legalizing operations, we have the following DAG:
```llvm
  t0: ch,glue = EntryToken
  t2: i32,ch = CopyFromReg t0, Register:i32 %0
      t11: i32 = shl t2, Constant:i32<3>
      t13: i32 = srl t2, Constant:i32<29>
    t14: i32 = or t11, t13
  t6: ch,glue = CopyToReg t0, Register:i32 $x10, t14
  t7: ch = RISCVISD::RET_GLUE t6, Register:i32 $x10, t6:1
```
Line 3 to 5 in the post-legalized DAG show that we legalize `rotl` by _synthesizing_ it with bitwise OR on the extractions of higher-bit part (i.e. `t11: i32 = shl t2, Constant:i32<3>`) and lower-bit part (i.e. `t13: i32 = srl t2, Constant:i32<29>`).

Of course, there are more than one way to legalize an operation. For instance, on embedded devices that don't have multiplication instructions, we might legalize a `mul` by simply replacing it with a call to library functions that "emulate" multiplications with a sequence of additions.

This brings us to the next section, where we ask a similar question we had seen before: how is an operation considered legal or illegal in a specific target? How do we handle illegal operations?

#### Determine legal operations and actions on illegal operations
Contrary to what we've seen in type legalizer, each target has to declare its own illegal operations and specify the desired way to handle each of them. This information is also placed in each target's TargetLowering (put under `XXXXISelLowering.cpp`). For example, RISC-V uses the following lines from [here](https://github.com/llvm/llvm-project/blob/b7ed097f29d712b1cc839e15ab68d2c8a2ce07cc/llvm/lib/Target/RISCV/RISCVISelLowering.cpp#L353) (abridged for clarity) to declare multiplications being illegal in the absent of Zbb / Zbkb extensions, and how to handle it:
```c
if (!Subtarget.hasStdExtZbb() && !Subtarget.hasStdExtZbkb())
  setOperationAction({ISD::ROTL, ISD::ROTR}, XLenVT, Expand);
```
`setOperationAction` is the key here: for each opcode specified in its first argument (`ISD::ROTL` and `ISD::ROTR` in this case) that operates on value type specified by its second argument (i.e. `XLenVT`), we perform an action on the third argument (`Expand` in this case) to legalize it.

Let's look at the third argument first, here are the possible actions we can do to legalize an operation:
  - Expand
  - LibCall
  - Promote
  - Custom

**Expand** tries to synthesize an operation with other legal operations, which we had seen how it worked on rotate left. The "recipe" to expand an operation is pre-defined (rather than defined by each target). If you're interested in learning what these recipes look like, most of them are put under `llvm/lib/CodeGen/SelectionDAG/TargetLowering.cpp` and `llvm/lib/CodeGen/SelectionDAG/LegalizeDAG.cpp`.

If the legalizer fails to expand an operation (e.g. none of the sub-operations used in synthesis are legal), it will fallback to the next action, **LibCall**, which replaces the operation with calls to library functions. Of course, a target can just set an operation's legalizer action directly to this. For instance, RISC-V [always uses](https://github.com/llvm/llvm-project/blob/b7ed097f29d712b1cc839e15ab68d2c8a2ce07cc/llvm/lib/Target/RISCV/RISCVISelLowering.cpp#L1409) lock-free atomic builtin functions when the `forced-atomic` feature is enabled.

**Promote** will promote an operation to operate on the next larger legal type. For instance, in a target where 32-bit and 64-bit integers are the legal types, `t3:i32 = add t2, Constant:i32<5>` will be promoted to `t3:i64 = add t2, Constant:i64<5>`. The recipes of how to promote individual operations are pre-defined, too. In fact, they are the same routines that are also used by the type legalizer.
Now, since we mentioned type legalizer, you may wonder: didn't we finish type legalization already? Why do we need to deal with type legality again?

Recall our takeaway at the end of last section: type legalization only cares about "globally illegal" types. That are, types which can't natively fit into any of the physical registers. It turns arbitrary types that can be as crazy as `i17` or `i87` into a small subset of legal types and it does this in an operation-agnostic fashion.
But within this small subset of legal types, some operations might only capable of handling an even _smaller_ number of (legal) types!

An interesting example in RISC-V is the _Zfhmin_ extension. Zfhmin is designed for a special scenarios where floating point values are stored in 16-bit format (i.e. F16), but majority of the arithmetics still operates on normal 32-bit floating points (i.e. F32). Therefore, in Zfhmin only data conversion / type casting operations like `fcvt.s.h` (convert from F16 to F32) support F16 while rest of the floating point operations are same as the F extension, which operate on F32. To deal with this type mixing, RISC-V backend declares F16 as legal type when Zfhmin is present, but mandates that all non-conversion F16 arithmetic instructions have to be promoted to F32 in this scenario:
```c
static const unsigned ZfhminZfbfminPromoteOps[] = {
    ISD::FMINNUM,      ISD::FMAXNUM,       ISD::FADD,
    ISD::FSUB,         ISD::FMUL,          ISD::FMA,
    ISD::FDIV,         ISD::FSQRT,         ISD::FABS,
    ISD::FNEG,         ...};
...
if (Subtarget.hasStdExtZfhminOrZhinxmin() && !Subtarget.hasStdExtZfhOrZhinx())
  setOperationAction(ZfhminZfbfminPromoteOps, MVT::f16, Promote);
```
(The above snippet is adapted from [here](https://github.com/llvm/llvm-project/blob/b7ed097f29d712b1cc839e15ab68d2c8a2ce07cc/llvm/lib/Target/RISCV/RISCVISelLowering.cpp#L457))

Now, onto the last legalization action: **Custom**.
As the name suggested, this is basically a wildcard action that allows a target to do whatever it wants on a generic SDNode. For each operation assigned to this action, `XXXXTargetLowering::LowerOperation` implements the actual legalization.

Take `ISD::FMINIMUM` and `ISD::FMAXIMUM` as an example, these two are floating point min/max operations conforming to IEEE-754-**2019** standard. The F extension in RISC-V, however, mostly[^7] conforms to IEEE-754-**2008** standard. Biggest difference between these two? Only IEEE-754-2019 propagates the NaN (Not-A-Number): If _either_ A or B is a NaN, `fmaximum(A, B)` in IEEE-754-2019 returns a NaN; RISC-V's `fmaximum(A, B)`, on the other hand, returns NaN only if _both_ A and B are NaNs.

[^7]: Except the fact that RISC-V's F extension does make -0.0 smaller than +0.0, which is a IEEE-754-2019 feature.

So for RISC-V, doing custom legalization on `ISD::FMINIMUM` and `ISD::FMAXIMUM` would be an easier option to overcome this semantic mismatch.
```c
if (Subtarget.hasStdExtFOrZfinx() && !Subtarget.hasStdExtZfa())
  setOperationAction({ISD::FMAXIMUM, ISD::FMINIMUM}, MVT::f32, Custom);
```
Then, in `lowerFMAXIMUM_FMINIMUM`, the [function](https://github.com/llvm/llvm-project/blob/b7ed097f29d712b1cc839e15ab68d2c8a2ce07cc/llvm/lib/Target/RISCV/RISCVISelLowering.cpp#L5725) `RISCVTargetLowering::LowerOperation` calls to legalize those two operations, RISC-V adds an additional check that conditionally returns the result from its native fminimum/fmaximum instructions only when _neither_ of the operands is NaN; otherwise, it returns a NaN.

Another thing worth noting is that, if a custom legalize action failed (e.g. the custom handler doesn't recognize/support the code it's looking at), it falls back to _Expand_. So the complete "chain of legalizer fallbacks" would be: Custom -> Expand -> LibCall.

#### The value type to legalize
Before wrapping up this section, I would like to spend some time on the second argument of `setOperationAction`: value type of the operation it's trying to legalize.
Theoritically, it represent the subset of legal types -- given the fact that we have finished type legalization at this stage -- that are not supported in a specific operation and demanded further legalizations.

As it turns out, _illegal_ types can also be used on this argument!

For instance, in RV64 where 32-bit integer is considered an illegal type[^8], we have the following line (abridged for clarity) for even the most basic operations like `ISD::ADD`:
```c
if (Subtarget.is64Bit())
  setOperationAction({ISD::ADD, ISD::SUB, ISD::SHL, ISD::SRA, ISD::SRL},
                      MVT::i32, Custom);
```
[^8]: Unless you flip an experimental flag `-riscv-experimental-rv64-legal-i32` to say otherwise.

The truth is that, the relationship between type and operation legalizer is more...interwinded than we thought. Type legalizer mostly runs in an operation-agnostic fashion, but when it sees an illegal type, it actually asks the operation legalizer if there is a `Custom` action handler attached on this operation with the said illegal type, and tries to run that custom (operation) legalization preemptively.

The idea is that we want to have a leeway to legalize an operation with its _original_ value type. Because unlike LLVM IR where we have explicit `zext`, `sext`, or `trunc` instructions to specify type conversions, in SelectionDAG all these extensions / truncations might be lowered into operations like `and t2, <bit mask>` (for zero extension) anytime before we actually legalize the operation, which increases the difficulties to recover those information.
Therefore, `Custom` legalizer action is allowed to handle illegal types.

Back to our RV64 example, the reason it wants custom legalization on `i32` is due to RV64's unique _widening_ instructions, like `ADDW`, which takes two 32-bit integers and sign-extends them into 64-bit integers before the actual (64-bit) arithmetic addition. By replacing the original operations with these widening instructions preemptively in the [custom handler](https://github.com/llvm/llvm-project/blob/b7ed097f29d712b1cc839e15ab68d2c8a2ce07cc/llvm/lib/Target/RISCV/RISCVISelLowering.cpp#L11971), we could avoid extra sign-extension instructions that would have been created (by type legalizer) for each of its operands otherwise.

Since we have brought up `ADDW`, an instruction with 32-bit operands and 64-bit result, another interesting question related to `setOperationAction`'s second argument is: _whose_ type does this argument refer to? result type(s)? operand types? _which_ operand's type?

For most of the instructions in majority of architectures, this is barely a question, since operands and results in simple arithmetics like ADD, SUB, and MUL usually have the same data type. But as we've seen in `ADDW`'s example, that's not always the case. Even worse, many operations don't even have a uniform data type for all their operands!

Take **vector reduction** as an example, it is a common vector operation that aggregates vector elements by a specific action (e.g. add, mul, and). For instance, the `llvm.vector.reduce.add` [intrinsic](https://llvm.org/docs/LangRef.html#llvm-vector-reduce-add-intrinsic) produces an integer result that is the _summation_ of all its elements. Some of its variant, `llvm.vector.reduce.fadd` which performs floating point add reduction, [has](https://llvm.org/docs/LangRef.html#llvm-vector-reduce-fadd-intrinsic) a _scalar_ start value as its first operand and the vector to sum up as the second operand.
```llvm
declare float @llvm.vector.reduce.fadd.v4f32(float %start_value, <4 x float> %v)
```
In this case, which type should we specify in `setOperationAction` for `ISD::VECREDUCE_FADD` (the opcode of `llvm.vector.reduce.fadd`)?
```c
setOperationAction(ISD::VECREDUCE_FADD, ???, Custom);
```
The answer for this particular question is the scalar operand's type (i.e. `float`). But can we use the vector operand's type instead to determine the legality of this operation? 

Unfrotunately, no.

The legalizer has already set the rule on which operand type or result type to use. This might not be a huge inconvenient in most cases, yet it still causes confusions and ambiguities sometimes, largely because these rules are not written in any documentations! (or any TableGen or .inc / .def files for easier lookups[^9]) They only appear in legalizer's codebase, specifically in the `SelectionDAGLegalize::LegalizeOp` function for most rules.

[^9]: One exception might be VP (Vector Predicated) intrinsics, whose operand for legalization can be found in `llvm/include/llvm/IR/VPIntrinsics.def`.

In addition to this issue, so far we've seen several shortcomings on how legalization is designed in SelectionDAG ISel. In the last section of this post, I'm going to briefly show you how another instruction selection framework in LLVM, **GlobalISel**, addresses some of these problems.

### Legalization in GlobalISel: a comparison
GlobalISel is a relatively new instruction selection framework designed to improve compilation time while producing code with a decent quality. It deserves its own blog posts (or even series!) so we're not going into the details here, but covering only its legalization component.

But even before switching the topic to GlobalISel, let's jump back to SelectionDAG ISel and take a moment to think about its overall flow:

  1. At the beginning, values from LLVM IR can have arbitrary types so crazy types like `i17` and `i87` might sprinkle here and there.
  2. Type legalizer goes all the way to turn _every_ of these illegal types into a small set of legal types.
  3. But then, you found out: "Oops, each operation might have their own preference on the types it supports". Namely, these types are what an operation _actually_ wants.
  4. We legalizes individal operations to iron out those unsupported legal types as well as unsupported operations.
  5. But if our focus has always been the types supported by individual operations...

_Then why can't we just jump from Step (1) to Step (4)?_

Why can't we **consolidate** type and operation legalization into a single stage?

And that is basically what GlobalISel does: for each operation, we specify its legal types and the "recipes" for legalizing it _at the same place_.
The interface to describe these information has a similar look to the `setOperationAction` function we've seen previously in SelectionDAG ISel. Let's see an example from RISC-V's GlobalISel legalizer:
```c
getActionDefinitionsBuilder({G_ADD, G_SUB, G_AND, G_OR, G_XOR})
    .legalFor({s32, sXLen})
    .legalIf(typeIsLegalIntOrFPVec(0, IntOrFPVecTys, ST))
    .widenScalarToNextPow2(0)
    .clampScalar(0, s32, sXLen);
```
`getActionDefinitionsBuilder` here specifies the legality of operations listed in its first argument: `G_ADD`, `G_SUB`, `G_AND` etc. They are opcodes for (generic) operations in GlobalISel, similar to `ISD::ADD`, `ISD::SUB`, and `ISD::AND` in SelectionDAG ISel.

The next line describes the legal types of its operands. More specifically, `legalFor({s32, sXLen})` says that operand 0 is considered legal if it's a 32-bit scalar or a `XLen` type. Note that the first operand of an operation in GlobalISel, called a [generic Machine IR (gMIR)](https://llvm.org/docs/GlobalISel/GMIR.html) instruction, actually represents the instruction's **result**. So we're specifying the legal result type here; the line after is doing a similar thing, but calling out to another predicate function `typeIsLegalIntOrFPVec` to determine the legality.

After declaring the legal types, it's time to describe how to deal with the _illegal_ ones (e.g. crazy types like `i17` and `i87`). If we look at the lines follows, `widenScalarToNextPow2(0)` will make illegal types at operand 0 widen to the next type with power-of-two size, before the resulting types being clamped by `clampScalar` into a type range bounded by 32-bit scalar and `XLen`.

The aforementioned function calls compose a chain of legalization steps consisting of _checks_ (e.g. `legalFor`) and _actions_ (e.g. `clampScalar`) that are executed in sequence. There are also some familiar actions that we've seen earlier, for example:
```c
getActionDefinitionsBuilder(G_SEXT_INREG)
    .customFor({sXLen})
    .maxScalar(0, sXLen)
    .lower();
```
`customFor` has the same effect as the `Custom` action we've seen in `setOperationAction`: delegating the legalization to custom handlers reside in each target's `LegalizerInfo::legalizeCustom`. If this step fails, it falls to the next action, `maxScalar`, which sets an upper bound on the type size and goes to the final action, `lower`, which is basically the `Expand` action in SelectionDAG's legalizer.

In the previous section, we mentioned that SelectionDAG legalizer uses one of the operand types to check against the second argument of `setOperationAction` for determining the operation's legality. As of which operand types to pick, it's predefined and sometimes causing some confusions and ambiguities.

GlobalISel's legalizer, on the other hand, has more flexibility on which operand you want to legalize. We've already got a hint from our previous examples, where we can designate a specific operand index subject to the legalizer action. So for instructions without a uniform operand types, like interger-to-pointer, it becomes easy to specify the action for its integer operand only:
```c
getActionDefinitionsBuilder(G_INTTOPTR)
    .legalFor({{p0, sXLen}})
    .clampScalar(1, sXLen, sXLen);
```
To summarize, GlobalISel's legalizer expresses legalities -- especially type legalities -- in a more _explicit_ way. Consolidating all legalizations into one phase also helps people to understand them better.

### Epilogue
Without a doubt, we need to create more learning resources for LLVM backend development. This post is my humble effort to shed some lights on a really important backend subsystem, which we're not even able to get to instruction selection without it. I hope you learn how SelectionDAG ISel's type and operation legalizer interacts with each other and how to specify the action to legalize illegal types or operations. I also hope you enjoy the last section on a more modern legalizer design.

Thanks for reading!