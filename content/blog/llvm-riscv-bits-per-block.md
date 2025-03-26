+++
title = "When LLVM scalable vector meets RISC-V"
description="And a mistery constant called \"RVVBitsPerBlock\""
date = "2024-10-05"
tags = ['llvm', 'riscv']
+++

There is a [nice page](https://llvm.org/docs/RISCV/RISCVVectorExtension.html) about how LLVM handles RISC-V Vector Extension (RVV). It primarily covers how the RISC-V backend lowers vector types and vector operations. Right [at the beginning of the page](https://llvm.org/docs/RISCV/RISCVVectorExtension.html#mapping-to-llvm-ir-types) lies this table:

<figure style="text-align: center;">
  <img src="/images/llvm-rvv-ir-types.png">
</figure>

It shows the LLVM IR types we use to represent RVV's _dynamically_ sized vectors: each row is an element type, while each column is a **LMUL** -- the register grouping factor, or "how many vector registers should we slap together and treat it as a single _logical_ vector register".

For instance, when LMUL = 4, each vector instruction effectively operates on a (gigantic) logical vector register that is four-time the size of a normal vector register. Under this LMUL setting, a RVV vector of 64-bit integer (i.e. `i64`) is represented by IR type `<vscale x 4 x i64>` according to the table. 

Both LMUL and the element type can be changed at any point during the runtime, hence the dynamically sized vectors.

The `<vscale x 4 x i64>` is a **scalable vector type** in LLVM. It looks similar to a normal (fixed) vector type like `<4 x i64>` -- a vector of four 64-bit integers -- but the "vscale" keyword gives it the ability to scale the "base" vector type -- namely, the `4 x i64` part, where 4 here is the <u>minimum number of elements</u> -- by a certain factor, vscale, that is only known during runtime. So if vscale equals to 2 during runtime, we have an equivalent vector of `<8 x i64>`; `<16 x i64>` when vscale is 4. Simple, right?

Well...it looks simple until you squint a little harder at the table we just showed and start wonder: _how_ exactly does RVV LMUL map to different scalable vector IR types? how do we calculate the minimum number of elements?

Using LMUL = 4 again as an example, but this time we want to use `i32` as the element type. Intuitively, we would have thought the corresponding scalable vector type to be `<vscale x 4 x i32>` because the minimum number of elements is equal to LMUL, right? -- except it is not, it's actually `<vscale x 8 x i32>`.

To find the answer, we navigate to a line sitting just above the table in the same page: 

> ...vscale is defined as VLEN/64 (see RISCV::RVVBitsPerBlock).

It suggests that we can calculate the minimum number of scalable vector elements for a RVV type using `RISCV::RVVBitsPerBlock`, a magic constant that is [set to 64](https://github.com/llvm/llvm-project/blob/bf895c714e1f8a51c1e565a75acf60bf7197be51/llvm/include/llvm/TargetParser/RISCVTargetParser.h#L36).
The code comment for `RISCV::RVVBitsPerBlock` doesn't explain a lot, nor is clear what "block" here means.

In this short post, I'm going to explain how the table is created and the actual purpose of this mystery `RISCV::RVVBitsPerBlock` constant.

### ARM SVE -- the origin of scalable vector types
I know it's weird to start a RISC-V blog post with ARM, but I think it's imperative to understand the origin of scalable vector types in LLVM first.

ARM introduced SVE (Scalable Vector Extension) about 8 years ago. SVE allows vector registers to scale their size by a certain runtime factor, `LEN`. The figure below[^1] shows the structure of vector registers in SVE.

<div style="text-align: center;">
  <picture>
    <source srcset="/images/arm-sve.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/arm-sve.light.svg">
    <figcaption>Scalable vector registers in ARM SVE</figcaption>
  </picture>
</div>

[^1]: I redrew [this figure](https://developer.arm.com/documentation/102476/0100/SVE-architecture-fundamentals/Scalable-vector-registers-z0-z31?lang=en) from ARM's official document. Because the original figure missed an important component -- it's "LEN x 128", not just "LEN 128".

When `LEN` is set to 3, each vector registers `Z0` ~ `Z31` has a size of 512 bits (128 \* 3 + 128).

Sounds familiar, right? That's because the scalable vector type in LLVM we just talked about was introduced by ARM folks for SVE, and it's using the exact same principle.

In the most basic setting, the scalable vector type for a SVE vector has minimum number of elements equal to 128 divided by element size, like `<vscale x 4 x i32>` or `<vscale x 2 x i64>`. And during runtime, vscale is equal to `LEN + 1`.

If we think of a single SVE vector register being multiple 128-bit vector registers slapped together -- which I believe it is indeed what happens in hardware -- the runtime factor (i.e. `LEN`) dictates the _number_ of those fixed-size, 128-bit registers.

Now we learned that LLVM's scalable vector type is basically a 1:1 mapping to ARM SVE vectors, let's see how well (or bad) it maps to RISC-V vectors.

### Scalable vector types meet RVV

In RVV, each vector register, `v0` ~ `v31`, has a size of `VLEN` bits. `VLEN` is a hardware-defined constant. Though each RISC-V processor has its own fixed-value `VLEN`, the software doesn't know it during compile time -- assuming you're building a portable binary.

As introduced earlier in the post, RVV instructions operate on logical vector register, which is a group of LMUL actual vector registers, as depicted by the following figure.

<div style="text-align: center;">
  <picture>
    <source srcset="/images/rvv-registers.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/rvv-registers.light.svg">
    <figcaption>RISC-V vector registers</figcaption>
  </picture>
</div>

To update the current LMUL and element type -- which is also known as **SEW** (Selected Element Width) -- during runtime, we can use instructions like `vsetvl` and `vsetvli`.

For instance, `vsetvl rd, rs1, rs2` updates the current LMUL and SEW settings according to the (runtime) value stored in scalar register `rs2`. This LMUL + SEW combination -- part of a RVV setting called **vtype** -- will continue to hold until another `vsetvl`-family instruction change the vtype.

Here is the catch: compilers do NOT generate `vsetvl` in like 99% of the cases.

Instead, we use one of its siblings, `vsetvli`, which encodes the desired vtype with _immediate value_ operands. For instance, `vsetvli x2, x0, e32, m4` sets the new LMUL into 4 because of `m4`, an immediate value, and the new SEW to be 32-bit wide because of `e32`, also an immediate value.

With `vsetvli`, we can actually know the exact LMUL of a certain region of code _ahead of time_. For example, the following snippet uses different vtype settings in three different regions, but we're able to statically determine the exact LMUL (and SEW) values in each region.

``` asm
vsetvli t0, zero, e32, m2, ta, ma
# === operate on LMUL=2 & SEW of 32 bits ====
vadd.vv v2, v2, v4
vadd.vv v2, v2, v6
# ===========================================

vsetvli t0, zero, e32, m4, ta, ma
# === operate on LMUL=4 & SEW of 32 bits ====
vadd.vv v0, v0, v4
vadd.vv v0, v0, v8
# ===========================================

vsetvli t0, zero, e64, m4, ta, ma
# === operate on LMUL=4 & SEW of 64 bits ====
vadd.vv v0, v0, v4
vadd.vv v0, v0, v8
# ===========================================
ret
```

The bottom line is that in most cases, LMUL is considered to be "semi-dynamic" -- it might change during runtime, but in a more _deterministic_ way.

And that is one of the reasons why RISC-V backend assigns each LMUL (and SEW) a unique scalable vector type -- because it is possible to determine it during compile time. In comparison, ARM SVE folds all the dynamic bits into a single parameter, vscale.

But now a new problem emerges: `VLEN` is unknown during compile time, because different RISC-V processors might have different `VLEN` values. Without knowing the exact value of `VLEN` we're unable to know the number of elements in a single vector register, which is different from SVE's case because the latter effectively has a "base" vector register of 128 bits.

Let's try to solve this problem with one of our (incorrect) intuitions mentioned earlier: use LMUL -- the "known" value -- as the minimum number of elements in a scalable vector type, and fold every unknowns -- including `VLEN` -- into vscale. So LMUL=4 + SEW=32 becomes `<vscale x 4 x i32>` or `<vscale x 4 x f32>`.

To illustrate this solution, let's rearrange our RVV LMUL figure earlier so that the unknown part, `VLEN`, is put along the horizontal axis, while individual vector registers are arranged vertically with the respective LMUL quantity.

<div style="text-align: center;">
  <picture>
    <source srcset="/images/rvv-sve-registers-naive.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/rvv-sve-registers-naive.light.svg">
    <figcaption>A naive way to map RVV vectors to scalable vector types</figcaption>
  </picture>
</div>

In this solution, RVV's vscale value is equal to the number of (vertical) dotted boxes, `VLEN / SEW`. This means that we might have different vscale at different code regions since SEW might change (recall the snippet shown above) -- and that, causes lots of inconveniences.

The value of vscale is essential to estimating the number of vector elements, which directly relates to many things like _vectorization factors_ (i.e. how many items can we process in a single iteration of a vectorized loop). Having a more predictable range of vscale is always preferrable to the optimizers.

To have a vscale value that doesn't depend on `SEW`, there is a simple trick: create a fake, _fixed-size_ "base vector" similar to what SVE has.

<div style="text-align: center;">
  <picture>
    <source srcset="/images/rvv-register-final.dark.svg" media="(prefers-color-scheme: dark)">
    <img src="/images/rvv-register-final.light.svg">
  </picture>
</div>

The figure above used a fake base vector of 64 bits, which contains 2 elements in it when SEW=32. Combining with LMUL=4, we get eight 32-bit elements in a single dotted box. Again, vscale value is equal to the number of dotted boxes, but this time, we can easily evaluate it with `VLEN / 64` -- vscale no longer depends on `SEW`.

The size of the fake base vector, 64 bits in this example, is `RISCV::RISCVBitsPerBlock`. To put it differently, in RISC-V backend, the vscale value is always equal to `VLEN / RISCV::RISCVBitsPerBlock`.

It sound a little bit like magic where the `SEW` factor just suddenly disappears from the equation. But what it really does was simply using the _least common multiple (LCM)_ of all the supported SEW (8, 16, 32, and 64 bits) to "tile" `VLEN`. The downside of this design is that we can't support 32-bit `VLEN` out of the box, which is a known limitation in RISC-V LLVM backend. It sounds a little unusual to have such a small vector size of 32 bits, but apparently `Zve32*` extensions were created to put vectors into embedded devices, in which case a smaller vector sizes makes more sense.

Granted, it's not really common to use `RISCV::RISCVBitsPerBlock` directly, but I thought it's fun to know where it came from and why it's created in the first place. And that's all for this short post! Hope you enjoy.