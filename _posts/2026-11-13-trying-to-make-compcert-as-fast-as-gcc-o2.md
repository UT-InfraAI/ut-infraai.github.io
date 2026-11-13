---
layout: post
title: "Can a formally-verified C compiler be as performant as gcc -O2?"
title_markdown: "Can a formally-verified C compiler be as performant as `gcc -O2`?"
date: 2026-11-13
categories: compilers systems
description: "What happens when agents write about 14,800 lines of Rocq code trying to close a measured 28% runtime gap between CompCert 3.18 and GCC 11.4 -O2."
author: Aananth V
tags: [Compilers, Formal verification]
image: /assets/posts/compcert-gcc-o2/cover.png
image_alt: "A C program passes through CompCert to become x86-64 assembly, with machine-checked proofs."
math: true
---

<style>
.tab-nav {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  border-bottom: 1px solid #e1e4e8;
  padding-bottom: 8px;
  margin: 16px 0 12px 0;
}
.tab-btn {
  background: none;
  border: 1px solid transparent;
  border-radius: 4px;
  padding: 5px 12px;
  font-size: 13px;
  font-weight: 500;
  color: #586069;
  cursor: pointer;
  transition: all 0.15s ease;
}
.tab-btn:hover {
  color: #24292e;
  background: #f6f8fa;
}
.tab-btn.active {
  color: #0366d6;
  background: #f1f8ff;
  border-color: #c8e1ff;
  font-weight: 600;
}
.tab-subnav {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  margin: 10px 0;
}
.tab-subbtn {
  background: #f6f8fa;
  border: 1px solid #d1d5da;
  border-radius: 4px;
  padding: 4px 10px;
  font-size: 12px;
  color: #586069;
  cursor: pointer;
}
.tab-subbtn:hover {
  background: #e1e4e8;
}
.tab-subbtn.active {
  background: #24292e;
  color: #ffffff;
  border-color: #24292e;
  font-weight: 600;
}
.fig-box {
  background: #fafbfc;
  border: 1px solid #e1e4e8;
  border-radius: 6px;
  padding: 16px;
  margin: 20px 0;
}
.fig-title {
  font-size: 15px;
  font-weight: 600;
  color: #24292e;
  margin-bottom: 2px;
}
.fig-subtitle {
  font-size: 12px;
  color: #586069;
  margin-bottom: 12px;
}
.fig-caption {
  font-size: 12px;
  color: #586069;
  margin-top: 12px;
  padding-top: 10px;
  border-top: 1px solid #e1e4e8;
  line-height: 1.45;
}
.bar-row {
  margin-bottom: 8px;
}
.bar-labels {
  display: flex;
  justify-content: space-between;
  font-size: 12px;
  margin-bottom: 2px;
  color: #444d56;
}
.bar-labels.is-bold {
  font-weight: 600;
  color: #24292e;
}
.bar-track {
  width: 100%;
  height: 10px;
  background: #e1e4e8;
  border-radius: 999px;
  overflow: hidden;
}
.bar-fill {
  height: 100%;
  border-radius: 999px;
  transition: width 0.3s ease;
}
.asm-code-wrap pre {
  margin: 0;
  padding: 12px;
  font-size: 12px;
  line-height: 1.45;
  background: #ffffff;
  border: 1px solid #e1e4e8;
  border-radius: 4px;
  overflow-x: auto;
  font-family: ui-monospace, Menlo, Monaco, Consolas, monospace;
}
.data-table-wrap {
  overflow-x: auto;
  max-height: 420px;
  overflow-y: auto;
  border: 1px solid #e1e4e8;
  border-radius: 4px;
  background: #ffffff;
}
.data-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 12px;
  text-align: left;
}
.data-table th {
  background: #f6f8fa;
  padding: 8px 10px;
  font-weight: 600;
  color: #24292e;
  border-bottom: 1px solid #e1e4e8;
  position: sticky;
  top: 0;
  z-index: 1;
}
.data-table td {
  padding: 7px 10px;
  border-bottom: 1px solid #eaecef;
  color: #444d56;
}
.data-table tr:hover {
  background: #f6f8fa;
}
.badge {
  display: inline-block;
  padding: 1px 5px;
  border-radius: 3px;
  font-family: ui-monospace, Menlo, Monaco, monospace;
  font-size: 11px;
  font-weight: 600;
}
.badge-green {
  background: #dcffe4;
  color: #155724;
}
.badge-blue {
  background: #dbedff;
  color: #0366d6;
}
.badge-gray {
  background: #f1f2f4;
  color: #586069;
}
/* Assembly Syntax Highlighting */
.highlight pre code.language-asm, pre code.language-asm {
  color: #24292e;
}
.asm-comment { color: #6a737d; font-style: italic; }
.asm-label   { color: #6f42c1; font-weight: 600; }
.asm-op      { color: #d73a49; font-weight: 600; }
.asm-reg     { color: #005cc5; }
.asm-num     { color: #0086b3; }
</style>

CompCert ([GitHub](https://github.com/AbsInt/CompCert)) is a *formally verified* C compiler written in Rocq and OCaml. Like `gcc` or `clang`, it compiles C to assembly and optimizes the program along the way. Its verified compilation pipeline comes with machine-checked proofs of *semantic preservation*: for a source program without undefined behavior under CompCert's semantics, the generated assembly exhibits one of the source program's allowed observable behaviors. These semantics are mathematical definitions of the supported C language and target instructions.

The main theorem connects a CompCert C abstract syntax tree to an assembly abstract syntax tree under the formal models. Applying that guarantee to a running executable also relies on trusted components, including front-end elaboration, assembly printing, the assembler, and the linker; it does not verify every step from source text to execution.[^compcert-guarantee] Within that scope, it makes CompCert particularly suitable for safety-critical software, since mainstream compilers such as GCC and LLVM have been shown to silently generate wrong code.[^csmith][^emi][^sun]

However, code produced by CompCert can have a substantial runtime cost. In our measurements, the CompCert benchmarks run roughly 28% slower with CompCert 3.18 than with GCC 11.4 `-O2`, and the PolyBench loop-nest benchmarks run about 55% slower.[^bench]

One common theory for this gap is CompCert's strict and fairly simple semantic models of C and x86-64, which rule out several assumptions that GCC uses to optimize:
- **Signed integer overflow.** The C standard leaves it undefined, so GCC assumes it never happens; for example, it can treat `i + 1 > i` as always true and simplify loop counters. CompCert's semantics defines overflow to wrap around, so it can't make that assumption.
- **Type-based aliasing.** C's aliasing rules let GCC assume that an `int *` and a `float *` never point to the same memory, so it can keep a value in a register across a store through the other pointer. CompCert's memory model doesn't include these rules.
- **Fused multiply-add.** GCC may combine `a * b + c` into a single FMA instruction, which is faster and rounds only once. CompCert rounds every floating-point operation separately.
- **SIMD.** GCC vectorizes loops to process several elements per instruction. CompCert's x86-64 model has no vector instructions.

I wanted to know whether these semantic restrictions actually explain why CompCert is slow, or whether CompCert is simply missing many standard compiler optimizations and backend code generation polish. And if the gap is mostly missing passes, 
1. Is proof generation the bottleneck? (CompCert was first released in 2008...)
2. Could AI agents find, implement, and *prove* those transformations in Rocq without any human intervention? (Recent progress in math suggests yes...)

All of our passes, proofs, and benchmarking scripts are available on the [`v3.18` branch of our CompCert fork](https://github.com/UT-InfraAI/CompCert/tree/v3.18). The pass names in Section 4 link to their implementations on that branch.

---

## 1. Testing the Folk Theory: Is It the Semantic Model?

Before touching CompCert, we tested four parts of the semantic-model hypothesis on GCC 11.4. The following flags disable optimizations associated with four differences between the compilers; they do not make GCC's semantics equivalent to CompCert's:

| Restriction | In CompCert | GCC flag used |
|---|---|---|
| Signed integer overflow | Defined to wrap around | `-fwrapv` |
| Type-based aliasing | Rules not in the memory model | `-fno-strict-aliasing` |
| Fused multiply-add | Every operation rounded separately | `-ffp-contract=off` |
| SIMD | No vector instructions in the x86-64 model | `-fno-tree-vectorize` |

If optimizations enabled by these four differences explained most of the measured gap, disabling them in GCC should remove most of its lead.

However, restricted `gcc -O2` still takes only `0.795x` the CompCert runtime on the CompCert benchmarks and `0.648x` on PolyBench: 20.5% and 35.2% lower runtime, respectively. The restrictions change the corresponding full-GCC ratios only from `0.784x` to `0.795x` and from `0.646x` to `0.648x`.

There is an important limit to this experiment: GCC 11.4 does not enable auto-vectorization by default at `-O2`; that changed in GCC 12.[^gcc12] Our commands also do not explicitly enable FMA instructions with target flags. Thus, this comparison does not measure the benefit of removing active SIMD and FMA optimizations, or establish the gap against newer GCC releases. It shows that these four flags explain little of the gap on this setup. The assembly experiments below then identify specific missing transformations, some permitted by the existing model and some requiring changes to it.

<div class="fig-box">
  <div class="fig-title">Figure 1: Geometric Mean Runtime Across Compiler Configurations</div>
  <div class="fig-subtitle">Normalized to CompCert = 1.000 (lower is faster). Interleaved round-robin timing.</div>
  <div class="tab-nav" id="fig1-tabs">
    <button onclick="setFig1Suite('testc')" data-suite="testc" class="tab-btn active">CompCert benchmarks (24 programs)</button>
    <button onclick="setFig1Suite('polybench')" data-suite="polybench" class="tab-btn">PolyBench (30 kernels)</button>
    <button onclick="setFig1Suite('big')" data-suite="big" class="tab-btn">big (5 programs)</button>
  </div>
  <div id="fig1-bars"></div>
  <div id="fig1-caption" class="fig-caption"></div>
</div>

---

## 2. Assembly Surgery: So where does the time actually go?

We first needed to know which missing transformations actually contribute to the performance gap on real hardware. The idea is simple: we take CompCert's `.s` assembly output for a slow benchmark and manually edit it to apply one missing optimization at a time. Then we re-assemble the binary, check its output against the reference results, and time it. Once we know which optimizations actually help, we can prove and implement them in CompCert. Thanks to coding agents, this was not as painful as it sounds. 

The following are some of the missing optimizations we found this way:

### Loop-Carried Reloads in `spectral`

One of the CompCert benchmarks, [`spectral`](https://github.com/AbsInt/CompCert-small-tests/blob/main/c/spectral.c), spends most of its time in an inner loop that accumulates products into an array element `Au[i]`:

```c
/* spectral.c: inner product loop */
for (j = 0; j < n; j++)
  Au[i] += u[j] / (double)((i + j) * (i + j + 1) / 2 + i + 1);
```

Normally, CompCert's common subexpression elimination (CSE) pass ([`backend/CSE.v`](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/CSE.v)) would avoid reloading a value the program already has. But CSE only remembers values within a stretch of code that has a single way in, and the top of a loop has two: from the code before the loop, and from the end of the previous iteration. So CSE forgets everything it knew at the start of every iteration. As a result, CompCert reloads `Au[i]` from memory, adds to it, and stores it back on every single inner-loop iteration.

The following schematic fragments show the accumulator update; the rest of the arithmetic, index update, and loop control are omitted:

```asm
# Before: reload and store on every iteration
movsd   0(%rdx,%rax,8), %xmm1   # reload Au[i] from memory
mulsd   %xmm4, %xmm0            # multiply matrix term by u[j]
addsd   %xmm0, %xmm1            # add product to accumulator
movsd   %xmm1, 0(%rdx,%rax,8)   # store Au[i] back to memory
```

Keeping the accumulator available in a register across iterations avoids the reload. The initial assembly-surgery experiment reduced runtime from 783 ms to 464 ms, a 41% reduction.[^surgery] The verified [`Loadfwd` pass](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/Loadfwd.v) later implemented load forwarding across loop back-edges, reducing `spectral` from 126 ms to 75.5 ms in that evaluation. These are separate measurement runs.

```asm
# Loadfwd's effect: the register carries Au[i] across iterations
# It is initialized from the preceding load or store.
mulsd   %xmm4, %xmm0            # multiply matrix term by u[j]
addsd   %xmm0, %xmm1            # accumulate without reloading Au[i]
movsd   %xmm1, 0(%rdx,%rax,8)   # keep the store on each iteration
```

`Loadfwd` checks that the register still represents the memory value across intervening operations and retains the stores. Moving those stores after the loop would require additional aliasing and control-flow conditions. Our [LICM implementation](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/LICM.v) instead hoists memory-independent computations; loop rotation does not by itself move these memory accesses.

### Over-Conservative Stack Aliasing in `sha3`

Another CompCert benchmark, [`sha3`](https://github.com/AbsInt/CompCert-small-tests/blob/main/c/sha3.c), spends most of its time in `keccakf`, the function that scrambles SHA-3's internal state over 24 rounds. It writes temporary values into a local array `bc[5]` on the stack:

```c
/* sha3.c: temporary slice buffer in keccakf */
uint64_t bc[5];
for (i = 0; i < 5; i++)
  bc[i] = st[i] ^ st[i + 5] ^ st[i + 10] ^ st[i + 15] ^ st[i + 20];
```

By the end of each round, 34 stores to `bc[]` are dead because those slots are overwritten before they are ever read again.

CompCert's dead-code elimination pass ([`backend/Deadcode.v`](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/Deadcode.v)) should remove them, but it didn't. `keccakf` reads its state through a pointer `st` passed in by the caller. CompCert's analysis ([`backend/NeedDomain.v`](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/NeedDomain.v)) already knows that `st` cannot point into `keccakf`'s own stack frame, yet it treated every load through `st` as if it might read anything on that stack frame, including `bc`. So every store to `bc` looked like it might be needed later:

```asm
# CompCert: stores to bc[] that are never read
movq    %rax, 0(%rsp)           # store bc[0] to the stack
movq    %rbx, 8(%rsp)           # store bc[1] to the stack
movq    %rcx, 16(%rsp)          # store bc[2] to the stack
# ... 31 more like these per round; each is overwritten before it is read ...
# CompCert keeps them all: it assumes later loads through st might read them
```

Removing these dead stores eliminated 80% of the stack stores per round and cut `sha3` runtime by 18%.

### Redundant Zero-Extensions and False Register Dependencies

When we inspected [`aes`](https://github.com/AbsInt/CompCert-small-tests/blob/main/c/aes.c) and [`nbody`](https://github.com/AbsInt/CompCert-small-tests/blob/main/c/nbody.c) we found two issues in CompCert's x86-64 backend model and code emission.

First, on x86-64, writing a 32-bit register automatically zeroes the upper 32 bits of the 64-bit register. CompCert's instruction selector did not model this hardware rule for shifts and bitwise masks. When [`aes`](https://github.com/AbsInt/CompCert-small-tests/blob/main/c/aes.c) indexed a lookup table using `(u64)(x >> 24)`, CompCert emitted an explicit zero-extension:

```asm
# CompCert: redundant movl zero-extension
shrl    $24, %edx               # logical right shift table index by 24 bits
movl    %edx, %edx              # redundant! 32-bit shift already zeroes upper 32 bits
movl    0(%r9,%rdx,4), %r10d    # load 32-bit word from lookup table Te0
```

In the assembly-surgery report, 23 redundant extensions per round were removed, cutting `aes` runtime from 85.2 ms to 77.6 ms, a 9% runtime reduction.[^surgery] With further optimizations including LICM, the final all-flags configuration reached 67.2 ms versus 83.7 ms for baseline CompCert, a 20% runtime reduction. This was slightly faster than `gcc -O2` in that run.

Second, we found a suboptimal lowering in CompCert's emission of scalar SSE instructions like `cvtsi2sd` and `sqrtsd`. While these instructions write only the lower 64 bits, the hardware considers the full 128 bits of the `%xmm` register when executing and must keep the upper bits intact. These upper bits are never used in scalar code (no SIMD). This creates a false read-after-write dependency where the CPU waits for the previous write to that register to complete, stalling the instruction pipeline.

A compiler can break this dependency by zeroing the destination before the partial-register write, provided that doing so does not overwrite a live input (see LLVM's [`BreakFalseDeps.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/CodeGen/BreakFalseDeps.cpp)). On the tested Xeon, xor-with-self can break the dependency without adding execution latency, though it still consumes instruction bytes and front-end resources.

The `nbody` experiment used a variant with `sqrt` replaced by `__builtin_sqrt`. Its square-root instruction had different source and destination registers, leaving a false dependency on the old destination. The relevant pattern is shown below with XMM register names normalized; the rest of the function is omitted:

```asm
# Builtin-sqrt variant before the fix (184 ms)
sqrtsd  %xmm5, %xmm1            # low result depends on xmm5; upper bits retain xmm1

# Same variant with destination zeroing (112 ms)
xorpd   %xmm1, %xmm1            # safe because the input is in xmm5
sqrtsd  %xmm5, %xmm1            # no dependency on the old xmm1 contents
```

This is a 39% runtime reduction (a `1.64x` speedup) for that modified benchmark, not for the original `nbody` configuration.[^surgery] In the final measurements, original `nbody` takes 131.5 ms with baseline CompCert, 130.7 ms with our defaults, and 105.4 ms with all opt-in flags.

AVX makes the merge source explicit: `vcvtsi2sd` copies bits 127:64 from its XMM source operand and zeroes bits above 127. It does not automatically zero the upper 64 bits of the XMM result. Choosing an independent merge source can avoid dependence on the old destination.[^intel-cvt]

---

## 3. How CompCert Proves Compiler Passes

Before asking AI agents to write new transformations, let's try to understand how CompCert proves that the transformations are correct. Each pass in the verified pipeline has a Rocq proof of semantic preservation under the formal models. There are two fundamentally different approaches that CompCert uses, and each comes with distinct engineering trade-offs.

### Approach 1: Direct Verified Dataflow

In direct verification, the optimization passes are directly written in Rocq's purely functional language (Gallina) and extracted to OCaml to build the compiler binary. A pass proves a simulation theorem relating executions of the input program $$P$$ and the transformed program $$P'$$. These theorems compose into the semantic-preservation guarantee described above. Since the optimization pass is the formal object under proof, the algorithm and the proof are fused together. CompCert uses this design for its core passes like constant propagation ([`Constprop.v`](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/Constprop.v)), common subexpression elimination ([`CSE.v`](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/CSE.v)), and dead code elimination ([`Deadcode.v`](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/Deadcode.v)). 

Most passes in this category are dataflow analyses. To optimize code, the compiler tracks what facts are true at every point in the function:

1. **Define what the compiler knows**: You define a set of facts about variables and memory. A variable might hold a known number (`r1 = 42`) or an unknown runtime value. These facts propagate through the control flow graph of the function.
2. **Update facts across each instruction**: You write a function that updates these facts for every operation. An assignment like `r1 = 5` adds a known constant to the current state. A pointer store like `*p = r3` updates the state and wipes out any cached load that might share memory with `p`.
3. **Loop until facts stop changing**: When two blocks merge after an `if-else` or when iterating over a loop, the compiler combines their facts. For example, if one branch says `r1 = 42` and the other says `r1 = 10`, the compiler drops the specific number and marks `r1` as unknown. CompCert runs a solver (Kildall's algorithm) that walks the control flow graph repeatedly until the facts stabilize.

Direct verification works well for textbook passes with clean, predictable rules. However, many optimizations rely on complex heuristics and edge cases. Since the optimization algorithm and its mathematical proof are inseparable, you cannot simply tweak how a pass behaves without also re-proving its underlying theorems. A change to a single analysis rule can invalidate proof chains across hundreds of lines of Rocq. This tight coupling makes iterating on complex optimizations slow.

### Approach 2: Translation Validation

While direct verification proves the transformation algorithm itself, translation validation proves a checker.

Think of it like solving a Sudoku puzzle versus checking one. Writing a solver that never makes a wrong guess is difficult to verify. Writing a checker that inspects a completed grid to confirm numbers 1 through 9 appear without duplicates is much simpler. 

Instead of proving that an optimization algorithm produces the correct output for every conceivable input program, translation validation splits the pass into two halves:
1. An **untrusted oracle** (written in standard OCaml) that runs the complicated heuristics and performs the transformation.
2. A **verified validator** (written and proved in Rocq, and then extracted to OCaml) that checks whether the transformation is correct for that specific function.

CompCert already uses this design for passes with complex graph algorithms:
* **Register Allocation ([`Allocation.v`](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/Allocation.v)):** An untrusted OCaml solver runs iterated register coalescing graph coloring, and a compact Rocq validator checks that no interfering variables share a physical register.
* **Basic Block Layout ([`Linearize.v`](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/Linearize.v)):** An untrusted OCaml heuristic orders blocks to minimize jumps, and a verified Rocq validator checks that all control flow paths remain intact.

This architecture does not weaken the pass's mathematical guarantee or require trusting the oracle. You prove the validator's soundness once, relative to the formal semantics. Schematically:

$$\begin{gathered}\texttt{check_function}(f, f') = \texttt{OK}\\ \implies \operatorname{Preserves}(f, f')\end{gathered}$$

Here, $$\operatorname{Preserves}(f, f')$$ denotes the pass's required semantic-preservation relation.

When you compile a C program, CompCert runs the validator to guarantee that the transformation is correct for that specific program. Since the validator is mathematically proven never to return `OK` on an invalid rewrite, an untrusted oracle cannot compromise correctness. If the oracle suggests a transformation that violates the required semantic relation, the validator cannot accept it. Our optional passes can keep the original function when validation fails; other passes, such as register allocation, can report a compilation error. Most of the passes we implemented in this project used this architecture.

---

## 4. Scaling Verified Passes with Coding Agents

Once we identified which passes we needed to implement ([Section 2](#2-assembly-surgery-so-where-does-the-time-actually-go)), we used coding agents to scale the generation and verification of optimization passes. Each optimization was assigned to an agent in an isolated git worktree. Every change made by the agent had to compile cleanly across all target architectures and pass CompCert's regression test suite. The proofs also had to introduce zero `Admitted` proofs or unapproved `Axiom` declarations.

While Rocq verifies that the transformations themselves are mathematically sound, it cannot verify changes to the hardware model. If an agent incorrectly alters the definition of an x86 instruction, Rocq will still accept the proof, but the compiler will emit broken machine code. Due to this, agents were barred from modifying the **trusted computing base** (C semantics, x86 assembly instructions, and the memory model) without explicit human approval. Unfortunately, the monkey in the loop (me) also approved incorrect changes to the hardware model, but we'll go over that in [Section 6](#6-the-human-gate-when-proofs-cannot-save-you).

We implemented new passes as well as improvements to existing passes. The names below link to the implementations on our `v3.18` branch; the percentages are runtime reductions from the corresponding experiments, not additive contributions to the final result:

| Pass & Intermediate Layer | Transformation | Example runtime reduction |
|---|---|---|
| **[IVwiden](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/IVwiden.v)** (RTL, new) | Widens 32-bit IVs to 64 bits; folds affine array offsets | [`gemm`](https://github.com/MatthiasJReisinger/PolyBenchC-4.2.1/blob/master/linear-algebra/blas/gemm/gemm.c) 26%, [`sha1`](https://github.com/AbsInt/CompCert-small-tests/blob/main/c/sha1.c) 13% |
| **[LICM](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/LICM.v) + [Looprot](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/Looprot.v)** (RTL, new) | Hoists invariant chains (row strides and base addresses) | [`gemm`](https://github.com/MatthiasJReisinger/PolyBenchC-4.2.1/blob/master/linear-algebra/blas/gemm/gemm.c) 28%, [`syrk`](https://github.com/MatthiasJReisinger/PolyBenchC-4.2.1/blob/master/linear-algebra/blas/syrk/syrk.c) 27% |
| **[Loadfwd](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/Loadfwd.v)** (RTL, new) | Forwards loads across join points and loop back-edges | [`spectral`](https://github.com/AbsInt/CompCert-small-tests/blob/main/c/spectral.c) 40%, [`3mm`](https://github.com/MatthiasJReisinger/PolyBenchC-4.2.1/blob/master/linear-algebra/kernels/3mm/3mm.c) 47% |
| **[Tailacc](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/Tailacc.v) + [Inlining(2)](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/Inlining.v)** (RTL) | Rewrites accumulator-style recursion to loops; second inlining round | [`integr`](https://github.com/AbsInt/CompCert-small-tests/blob/main/c/integr.c) 54%, [`fib`](https://github.com/AbsInt/CompCert-small-tests/blob/main/c/fib.c) 46% |
| **[CSE](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/CSE.v) + [Deadcode](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/Deadcode.v)** (RTL, refined) | Disjoint alias analysis; byte-load merging (`Mint64u`) | [`siphash24`](https://github.com/AbsInt/CompCert-small-tests/blob/main/c/siphash24.c) 21%, [`sha3`](https://github.com/AbsInt/CompCert-small-tests/blob/main/c/sha3.c) 18% |
| **[Loadop](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/Loadop.v) + [x86 Asm](https://github.com/UT-InfraAI/CompCert/blob/v3.18/x86/Asm.v)** (Linear/Asm) | Fuses loads into memory operands; zero-extending shifts | [`gemm`](https://github.com/MatthiasJReisinger/PolyBenchC-4.2.1/blob/master/linear-algebra/blas/gemm/gemm.c) 9%, [`nsievebits`](https://github.com/AbsInt/CompCert-small-tests/blob/main/c/nsievebits.c) 6% |
| **[Pure libm Builtins](https://github.com/UT-InfraAI/CompCert/blob/v3.18/common/Builtins0.v) + [front-end rewrites](https://github.com/UT-InfraAI/CompCert/blob/v3.18/cfrontend/C2C.ml)** (C/common) | Rewrites `sqrt`/`fabs`; treats 50 math calls as pure for CSE | [`nbody`](https://github.com/AbsInt/CompCert-small-tests/blob/main/c/nbody.c) 17%, [`perlin`](https://github.com/AbsInt/CompCert-small-tests/blob/main/c/perlin.c) 15% |

The corresponding proof files are [IVwidenproof.v](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/IVwidenproof.v), [LICMproof.v](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/LICMproof.v), [Looprotproof.v](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/Looprotproof.v), [Loadfwdproof.v](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/Loadfwdproof.v), [Tailaccproof.v](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/Tailaccproof.v), and [Loadopproof.v](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/Loadopproof.v). Changes to models and unverified components are recorded separately in the [trusted-base ledger](https://github.com/UT-InfraAI/CompCert/blob/v3.18/faster-compcert/TRUSTED_BASE.md).

---

## 5. From Isolated Wins to a Full Pipeline

Writing Rocq proofs was usually straightforward. Using Opus 5.5 in Claude Code, most passes took a few hours and about 10 to 40 compile-and-fix iterations to complete. However, the work didn't stop there and some real challenges showed up after the proofs were done, with held-out code, with passes interacting with each other, and with real hardware.

### 5.1 Pass Interactions and the Case for Autotuning

Every single pass we implemented made benchmarks faster in isolation. However, merging all of them into a single *static* pipeline revealed subtle interactions across passes. On some programs, combining all optimizations even made them slower than applying a small subset. We used `-fno-*` flags to pinpoint bad interactions, for example:

* In `sha3`, `Loadfwd` inserted shadow copies after memory operations. While `Deadcode` normally eliminated them, `CSE` used those shadow registers as canonical replacements, merging 27 distinct values and increasing register pressure until `keccakf` spilled to the stack.
* In `nsieve`, rotating a loop copied the exit test to the bottom, turning an unconditional jump into a conditional branch. That disqualified the block from CompCert's latch-placement heuristic in `Linearizeaux.ml`, adding extra jumps on the hot path.

While we were able to resolve both cases by refining local heuristics, this pattern points to a broader reality: no single fixed pass order or heuristic is optimal across all codebases. Pass interactions like these make a strong case for **profile-driven autotuning** and/or **agentic optimization**. This is especially true for *verified compilers* since each optimization pass is guaranteed to be correct (or no-op) - so we only need to worry about the interactions! This is very different from traditional compilers where the wrong order of passes might not only be suboptimal but could also introduce bugs.

### 5.2 Evaluating Beyond the CompCert Benchmarks: PolyBench and Larger Programs

After addressing these interactions, the initial version of our modified compiler, `faster-compcert` (v0), was able to beat `gcc -O1` and achieve `0.871x` the baseline CompCert runtime on the CompCert benchmarks. This closed 60% of the gap to `gcc -O2` (restricted) (`0.794x`).

To test whether these speedups generalized beyond the CompCert benchmarks, we initially evaluated the compiler on two held-out suites: `big` (five larger programs: `raytracer`, `spass`, `arcode`, `lzw`, and `lzss`) and `polybench` (all 30 loop-nest kernels from PolyBench/C 4.2.1). The results were modest and both lagged behind `gcc -O1`.

- On `big`, the compiler reached `0.982x` of baseline (vs `0.949x` for `gcc -O1`).
- On `polybench`, the compiler reached `0.845x` of baseline (vs `0.739x` for `gcc -O1`).

Inspecting the assembly of [`gemm`](https://github.com/MatthiasJReisinger/PolyBenchC-4.2.1/blob/master/linear-algebra/blas/gemm/gemm.c) (`C[i][j] += alpha * A[i][k] * B[k][j]`) showed immediately why PolyBench was still slow. The displayed inner-loop fragment contains 22 instructions, compared with 8 in the `gcc -O1` version:

**Before (`faster-compcert` v0: 22 instructions in inner loop):**
```asm
.L114:  movq    %rsi, %r8
        imulq   $8800, %r8              # C[i] row base recomputed every j!
        leaq    0(%rbp,%r8,1), %r8
        movslq  %edx, %r11              # 32-to-64 sign extension of j
        movq    %r11, %rcx
        movsd   0(%r8,%rcx,8), %xmm0
        movq    %rsi, %r10
        imulq   $9600, %r10             # A[i] row base recomputed every j!
        leaq    0(%r12,%r10,1), %r10
        movslq  %r9d, %rax
        movsd   0(%r10,%rax,8), %xmm6   # A[i][k]
        movapd  %xmm2, %xmm3
        mulsd   %xmm6, %xmm3
        imulq   $8800, %rax             # B[k] row base recomputed every j!
        leaq    0(%rbx,%rax,1), %r10
        movsd   0(%r10,%rcx,8), %xmm4
        mulsd   %xmm4, %xmm3
        addsd   %xmm3, %xmm0
        movsd   %xmm0, 0(%r8,%r11,8)
        leal    1(%edx), %edx
        cmpl    $1100, %edx
        jl      .L114
```

**After (`faster-compcert` v1, with dependent-chain LICM + IVwiden: 11 instructions in inner loop):**
```asm
.L114:  movsd   0(%rdi,%rcx,8), %xmm6   # C[i][j]: row base hoisted, 64-bit index
        movsd   0(%rdx,%r11,8), %xmm7   # A[i][k]: row base hoisted
        movapd  %xmm3, %xmm4
        mulsd   %xmm7, %xmm4
        movsd   0(%rsi,%rcx,8), %xmm8   # B[k][j]: row base hoisted
        mulsd   %xmm8, %xmm4
        addsd   %xmm4, %xmm6
        movsd   %xmm6, 0(%rdi,%rcx,8)
        leaq    1(%rcx), %rcx           # 64-bit induction variable, no movslq
        cmpq    $1100, %rcx
        jl      .L114
```

Why did `LICM` initially fail to hoist the row base addresses `C[i]`, `A[i]`, and `B[k]`? In 2D arrays, computing a row base takes a chain of three dependent instructions: sign-extend the outer index (`movslq %esi, %r8`), multiply by the row stride (`imulq $8800, %r8`), and add the array base (`leaq 0(%rbp,%r8,1), %r8`). Our first `LICM` pass only hoisted single operations whose arguments were already defined outside the loop.

For v1, we extended [`LICM.v`](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/LICM.v) to hoist dependent chains of invariant operations, and we built [`IVwiden.v`](https://github.com/UT-InfraAI/CompCert/blob/v3.18/backend/IVwiden.v) to widen 32-bit loop indices into 64-bit induction variables. Together, those two passes cut `gemm` runtime by nearly half (from `1.96 s` down to `1.05 s`) and dropped our PolyBench geomean from `0.845x` to `0.738x`.

Once we used these results to guide implementation, PolyBench became a development suite. Its initial evaluation was held out; the final results are not an independent held-out test. We label it accordingly below.

---

## 6. The Human Gate: When Proofs Cannot Save You

In CompCert, the transformations are formally verified using Rocq. If an agent makes a mistake in the transformations, say `Loadfwd` or `LICM`, `make proof` will reject it. However, that safety net vanishes the moment we touch the **trusted base**: the C semantics, the memory model, the x86 assembly semantics, or the code that prints the final assembly. A wrong assumption in the trusted base invalidates the correctness guarantee while Rocq happily reports `Qed`.

Even with our rule requiring human sign-off on trusted files, **twice I approved a trusted-base proposal that turned out to be mathematically false**.

In one of those instances, an agent was trying to improve the performance of `siphash24`, one of the CompCert benchmarks. `siphash24` reads little-endian 64-bit words from an unaligned `uint8_t *` pointer. In C, each word is built from eight byte loads shifted and ORed together:

```c
#define U8TO64_LE(p) \
  (((uint64_t)((p)[0]))       | ((uint64_t)((p)[1]) <<  8) | \
   ((uint64_t)((p)[2]) << 16) | ((uint64_t)((p)[3]) << 24) | \
   ((uint64_t)((p)[4]) << 32) | ((uint64_t)((p)[5]) << 40) | \
   ((uint64_t)((p)[6]) << 48) | ((uint64_t)((p)[7]) << 56))
```

GCC folds those eight byte loads into a single unaligned `movq (%rdi), %rax`. CompCert could not do this because its 64-bit memory chunk (`Mint64`) requires 8-byte alignment in the memory model.

To fix this, an agent suggested adding an unaligned 8-byte chunk (`Mint64u`) to the memory model, and claimed a simple lemma: *if 8 consecutive `Mint8unsigned` loads at `ptr + 0 .. ptr + 7` all succeed, then a single `Mint64u` load at `ptr` also succeeds and returns the combined 64-bit word.*

That sounded obviously true to me. I approved it.

Then the implementing agent sat down to prove the lemma in Rocq, and **proved a counterexample instead.**

In CompCert's memory model, pointer offsets are 64-bit modular integers (`Ptrofs.int`), which `Mem.loadv` interprets as unsigned integers in $$[0, 2^{64})$$. `Mem.load` checks permissions over the contiguous range $$[\text{ofs}, \text{ofs} + \text{size})$$ using mathematical integers; it does not wrap that range. These are offsets within an abstract memory block, not physical addresses. Suppose the block has readable permissions only at offsets $$2^{64} - 4$$ through $$2^{64} - 1$$ and $$0$$ through $$3$$, and `ptr` sits at offset $$2^{64} - 4$$:
1. All eight individual byte loads `ptr + 0` through `ptr + 7` succeed because pointer addition `Ptrofs.add` wraps modulo $$2^{64}$$ before each 1-byte check.
2. The merged 8-byte load at `ptr` fails (`None`) because it needs permissions at offsets $$2^{64}$$ through $$2^{64} + 3$$, which this block does not have.

Had this claim gone into the trusted base as an assumption rather than a lemma to be proved, every proof would still have passed, and the compiler would have been unsound. Because the agent was instructed to halt rather than improvise around a trusted-base flaw, we caught the error immediately. We replaced the flawed proposal with a change to the memory model itself ([`common/Memtype.v`](https://github.com/UT-InfraAI/CompCert/blob/v3.18/common/Memtype.v) and [`common/Memory.v`](https://github.com/UT-InfraAI/CompCert/blob/v3.18/common/Memory.v)). Specifically, we added a global invariant `access_bounded` stating that no memory block ever has permissions at offsets $$\ge 2^{64} - 8$$ (`perm_limit := Ptrofs.modulus - 8`). No real x86-64 user-space object can have size $$\ge 2^{64} - 8$$ anyway. With `access_bounded` in place, 8 consecutive byte loads cannot wrap around $$2^{64}$$. The merge lemma went through cleanly, and `siphash24` runtime dropped by 21% (`86.2 ms` down to `68.4 ms`).

> **Lesson from the Human Gate:** Human sign-off on formal specifications is necessary, but human intuition alone is not reliable enough. After this, we changed our process: every trusted-base proposal had to be mechanically prototyped in a scratch branch (with the key soundness lemma proved and zero admits) *and* differentially tested against the real CPU or glibc before coming to me for approval.

---

## 7. Where Performance Stands Today

Here is where `faster-compcert` stands in the final measurement session, expressed as geometric mean runtime relative to CompCert (lower is faster).

**Measurement setup.** These results were collected on 2026-10-25 with CompCert 3.18 and GCC 11.4 on an Intel Xeon Silver 4314. The [harness](https://github.com/UT-InfraAI/CompCert/blob/v3.18/faster-compcert/harness/bench.py) uses `gcc -O1`, `gcc -O2`, and `gcc -O3`; restricted `-O2` adds `-fwrapv -fno-strict-aliasing -fno-tree-vectorize -ffp-contract=off`. It adds no explicit `-march`, `-mtune`, or FMA-enabling flag. Programs are pinned to one core and compiler configurations run in rotating, interleaved rounds. We report each configuration's minimum wall-clock time over at least seven runs and at least 1.5 seconds of accumulated execution time, then take the geometric mean of per-program ratios.

PolyBench uses its standard LARGE dataset except for the MEDIUM and EXTRALARGE exceptions listed in [`suites.py`](https://github.com/UT-InfraAI/CompCert/blob/v3.18/faster-compcert/harness/suites.py). Its output check uses a separate MEDIUM-size binary compared with GCC `-O0`; the CompCert tests are checked against their reference outputs. The [raw timing files](https://github.com/UT-InfraAI/CompCert/tree/v3.18/faster-compcert/results) contain the reported minima. These are single-session measurements on a shared machine, not confidence intervals. Some memory-bound kernels varied by up to 40% across runs, so small differences should not be read as established wins. Earlier assembly-surgery measurements came from separate sessions.

| Compiler configuration | CompCert benchmarks (24 progs, tuned) | `big` (5 progs, held out) | `PolyBench` (30 kernels, used in development) |
|---|---:|---:|---:|
| CompCert (baseline) | 1.000 | 1.000 | 1.000 |
| `gcc -O1` | 0.893 | 0.950 | 0.756 |
| **faster-compcert (defaults)** | **0.831** | **0.967** | **0.738** |
| **faster-compcert (all opt-in flags)** | **0.815** | **0.961** | **0.733** |
| `gcc -O2` (restricted) | 0.795 | 0.887 | 0.648 |
| `gcc -O2` (full) | 0.784 | 0.886 | 0.646 |
| `gcc -O3` | 0.790 | 0.913 | 0.583 |

*"Defaults" includes the verified middle-end passes plus `-fshift-mask`, `-fzext-load`, and `-fmerge-bytes`. These results include approved changes to the assembly and memory models, not only new passes under unchanged semantics. "All opt-in flags" adds `-fload-op` and `-ftrust-libm`. The latter assumes the documented libm behavior and requires programs not to use `errno` to detect errors from the affected math functions; it was validated against glibc 2.35 on x86-64. See the [trusted-base ledger](https://github.com/UT-InfraAI/CompCert/blob/v3.18/faster-compcert/TRUSTED_BASE.md) for the exact assumptions.*

Relative to full GCC 11.4 `-O2` (`1.00x` = `gcc -O2`):
1. On the CompCert benchmarks, CompCert started **28% slower**. Our defaults are now **6.1% slower**, and all opt-in flags are **4.0% slower**.
2. On PolyBench, CompCert started **55% slower**. Our defaults are now **14.2% slower**, and all opt-in flags are **13.5% slower**. Both configurations have lower measured runtime than `gcc -O1`.
3. On the held-out `big` programs (`spass`, `raytracer`, `arcode`, `lzw`, `lzss`), CompCert started **13% slower**. Our defaults are now **9.1% slower**, and all opt-in flags are **8.5% slower**.

We also report the fraction of the *logarithmic runtime gap* closed against **restricted** GCC `-O2`. If $$r$$ is our runtime ratio to baseline CompCert and $$g$$ is restricted GCC's ratio, that fraction is $$\log(r) / \log(g)$$. Using the unrounded measurements, it is 81% to 89% on the CompCert benchmarks, 70% to 72% on PolyBench, and 28% to 33% on `big`. These percentages use a different reference and definition from the slowdowns against full GCC above.

<div class="fig-box">
  <div class="fig-title">Figure 2: Full Per-Benchmark Explorer (All 59 Programs)</div>
  <div class="fig-subtitle">All faster-compcert results below use all opt-in flags, including -ftrust-libm under the assumptions above. Values &lt; 1.00x mean lower runtime than the reference compiler. Select a suite tab to inspect measured execution times.</div>
  
  <div class="tab-nav" id="fig3-tabs">
    <button onclick="setFig3Suite('testc')" data-suite="testc" class="tab-btn active">CompCert benchmarks (24 progs, tuned)</button>
    <button onclick="setFig3Suite('polybench')" data-suite="polybench" class="tab-btn">PolyBench (30 kernels, used in development)</button>
    <button onclick="setFig3Suite('big')" data-suite="big" class="tab-btn">big (5 progs, held out)</button>
  </div>

  <div class="data-table-wrap">
    <table class="data-table" id="fig3-table">
      <thead>
        <tr>
          <th>Benchmark</th>
          <th>CompCert</th>
          <th>faster-compcert (all opt-in flags)</th>
          <th>gcc -O2</th>
          <th>Runtime reduction vs CompCert</th>
          <th>Runtime vs gcc -O2</th>
        </tr>
      </thead>
      <tbody id="fig3-tbody"></tbody>
    </table>
  </div>
  <div class="fig-caption">Positive runtime reductions mean less time than CompCert; negative values indicate regressions. Ratios to gcc -O2 describe measured minima, not statistical significance.</div>
</div>

---

## 8. What Still Blocks Us from Matching GCC -O2

While we have made significant progress in closing the performance gap in this pilot, there is still a long way to go. The gap between the CompCert benchmarks and the held-out `big` programs reflects where we focused our optimization effort (and token spend). We focused on low hanging fruit like loop-level optimizations inside individual functions. That strategy works well for small numerical kernels. But in larger programs like `spass` and `raytracer`, runtime was driven by function call overhead, pointer indirection, and memory layout. These are things we haven't even begun to address. 

Closing the remaining gap to `gcc -O2` could involve both better optimization within functions and analysis across function boundaries. The following are some examples of optimizations that we identified but did not implement.

### 1. Coarse Alias Analysis and Loop-Invariant Loads

In `gemm` (`C[i][j] += alpha * A[i][k] * B[k][j]`), the final default configuration takes 1.049 s versus 0.950 s for `gcc -O2`, a 10.5% gap. With all opt-in flags, it takes 0.959 s, a 1.0% measured gap. One remaining inefficiency in the assembly is the repeated invariant load. In the inner `j` loop, the value `A[i][k]` never changes. GCC hoists it once into a register before the loop starts. But CompCert currently lacks heap points-to analysis, so it cannot prove that storing into `C[i][j]` leaves `A[i][k]` untouched. Consequently, CompCert reloads `A[i][k]` from memory on every single iteration.

**What it will take:** Either a verified points-to analysis for the heap, or translation validation for C99 `restrict` annotations so the compiler can prove the arrays do not overlap.

### 2. Interprocedural Register Allocation (IPA-RA)

In call-heavy code like `spass` and `arcode`, functions call other functions constantly. CompCert compiles one function at a time. When a function executes a `call` instruction, it assumes all caller-saved registers are wiped out. Even if a small callee function touches only two registers, the caller still saves its own live variables to stack memory beforehand and reloads them afterward.

**What it will take:** Whole-program analysis across the compilation unit, letting the compiler customize calling conventions so callers only save registers the callee actually touches.

### 3. Stack Frame Back-Links

On x86-64, an optimized function return simply adds a constant offset to the stack pointer (`addq $sz, %rsp; ret`).

CompCert instead implements its stack frame teardown (`Pfreeframe`) by reading a stored back-link from memory. Small, non-inlined functions that could otherwise run entirely in CPU registers are forced to round-trip stack pointers through memory on every return.

**What it will take:** Either more aggressive inlining to remove the frame altogether, or refactoring `Mach` and `Asm` frame emission to replace the memory read with a constant `addq` offset.

---

## 9. Takeaways

We started with two questions: why is CompCert slower than `gcc -O2`, and can LLM agents close the gap?

First, the four GCC restrictions we tested explain little of the measured gap against GCC 11.4 `-O2` on these workloads. Assembly experiments and verified implementations show substantial opportunities in missing passes and instruction selection. Some gains also required approved changes to the formal models. The experiment does not establish the contribution of SIMD and FMA against newer or differently configured GCC versions.

Second, LLM agents can write and prove serious compiler optimizations at scale with minimal human supervision. The compiler changes added about 14,800 lines of Rocq code, including implementations and proofs, plus about 2,600 lines of OCaml. Most individual passes took a few hours and 10 to 40 compile-and-fix iterations. Classical performance engineering (catching negative pass interactions, preventing quadratic compile-time blow-ups, evaluating beyond the initial suite, and fixing shared-machine timing noise) involved more human input than proof engineering.

Finally, formal verification works only when the model of the real system is accurate. Agents automate traditionally difficult parts of the job like pass-building and proof generation, but human guardrails are still needed for defining the scope of the system and verifying that the model is accurate. Methods like differential testing can help bridge this gap in the future to further reduce the human effort required.

<script>
// --- Figure 1 Data ---
const fig1Suites = {
  testc: {
    caption: "On the CompCert benchmarks (24 programs), restricting GCC -O2 with the four flags above costs only 1.1 points (0.784 vs 0.795).",
    rows: [
      { label: "CompCert (baseline)", val: 1.000, color: "#94a3b8" },
      { label: "gcc -O1", val: 0.893, color: "#f59e0b" },
      { label: "gcc -O2 (restricted)", val: 0.795, color: "#0284c7" },
      { label: "gcc -O3", val: 0.790, color: "#6366f1" },
      { label: "gcc -O2 (full)", val: 0.784, color: "#0369a1" }
    ]
  },
  polybench: {
    caption: "On PolyBench (30 loop-nest kernels), restricting GCC -O2 with the four flags above costs only 0.2 points (0.646 vs 0.648).",
    rows: [
      { label: "CompCert (baseline)", val: 1.000, color: "#94a3b8" },
      { label: "gcc -O1", val: 0.756, color: "#f59e0b" },
      { label: "gcc -O2 (restricted)", val: 0.648, color: "#0284c7" },
      { label: "gcc -O2 (full)", val: 0.646, color: "#0369a1" },
      { label: "gcc -O3 (with SIMD vectorization)", val: 0.583, color: "#6366f1" }
    ]
  },
  big: {
    caption: "On the big programs (raytracer, spass, arcode, lzw, lzss), restricting GCC -O2 with the four flags above costs only 0.1 points (0.886 vs 0.887).",
    rows: [
      { label: "CompCert (baseline)", val: 1.000, color: "#94a3b8" },
      { label: "gcc -O1", val: 0.950, color: "#f59e0b" },
      { label: "gcc -O3", val: 0.913, color: "#6366f1" },
      { label: "gcc -O2 (restricted)", val: 0.887, color: "#0284c7" },
      { label: "gcc -O2 (full)", val: 0.886, color: "#0369a1" }
    ]
  }
};

function setFig1Suite(suiteKey) {
  document.querySelectorAll('#fig1-tabs .tab-btn').forEach(btn => {
    btn.classList.toggle('active', btn.dataset.suite === suiteKey);
  });
  const data = fig1Suites[suiteKey];
  const container = document.getElementById('fig1-bars');
  container.innerHTML = data.rows.map(r => {
    const pct = Math.round(r.val * 100);
    const isBold = r.label.startsWith("faster-compcert");
    return `
      <div class="bar-row">
        <div class="bar-labels ${isBold ? 'is-bold' : ''}">
          <span>${r.label}</span>
          <span style="font-family: monospace;">${r.val.toFixed(3)}x</span>
        </div>
        <div class="bar-track">
          <div class="bar-fill" style="width: ${pct}%; background-color: ${r.color};"></div>
        </div>
      </div>
    `;
  }).join('');
  document.getElementById('fig1-caption').textContent = data.caption;
}

// --- Figure 2: Full Per-Benchmark Table with Tabs ---
const benchmarkData = {
  testc: [
    { name: "fib", cc: 0.0577, fc: 0.0245, gcc: 0.0193 },
    { name: "integr", cc: 0.0359, fc: 0.0167, gcc: 0.0168 },
    { name: "spectral", cc: 0.1259, fc: 0.0703, gcc: 0.0574 },
    { name: "perlin", cc: 0.0539, fc: 0.0359, gcc: 0.0384 },
    { name: "sha3", cc: 0.0342, fc: 0.0246, gcc: 0.0221 },
    { name: "vmach", cc: 0.0690, fc: 0.0503, gcc: 0.0464 },
    { name: "siphash24", cc: 0.0902, fc: 0.0683, gcc: 0.0632 },
    { name: "sha1", cc: 0.0587, fc: 0.0446, gcc: 0.0442 },
    { name: "nbody", cc: 0.1315, fc: 0.1054, gcc: 0.0855 },
    { name: "aes", cc: 0.0837, fc: 0.0672, gcc: 0.0724 },
    { name: "nsieve", cc: 0.1180, fc: 0.1012, gcc: 0.1105 },
    { name: "binarytrees", cc: 0.0294, fc: 0.0260, gcc: 0.0269 },
    { name: "nsievebits", cc: 0.0953, fc: 0.0862, gcc: 0.0830 },
    { name: "fft", cc: 0.0628, fc: 0.0576, gcc: 0.0576 },
    { name: "chomp", cc: 0.1622, fc: 0.1525, gcc: 0.1540 },
    { name: "fannkuch", cc: 0.1983, fc: 0.1876, gcc: 0.1935 },
    { name: "fftsp", cc: 0.0334, fc: 0.0321, gcc: 0.0271 },
    { name: "fftw", cc: 0.0522, fc: 0.0512, gcc: 0.0493 },
    { name: "almabench", cc: 0.2271, fc: 0.2237, gcc: 0.2182 },
    { name: "bisect", cc: 0.0733, fc: 0.0730, gcc: 0.0749 },
    { name: "qsort", cc: 0.1225, fc: 0.1221, gcc: 0.1249 },
    { name: "mandelbrot", cc: 0.1129, fc: 0.1127, gcc: 0.1084 },
    { name: "lists", cc: 0.0740, fc: 0.0741, gcc: 0.0713 },
    { name: "knucleotide", cc: 0.1162, fc: 0.1172, gcc: 0.1134 }
  ],
  polybench: [
    { name: "3mm", cc: 8.6009, fc: 3.8529, gcc: 3.8432 },
    { name: "doitgen", cc: 1.6574, fc: 0.7470, gcc: 0.7251 },
    { name: "2mm", cc: 5.2756, fc: 2.5131, gcc: 2.5065 },
    { name: "trisolv", cc: 0.1217, fc: 0.0587, gcc: 0.0586 },
    { name: "gemm", cc: 1.9781, fc: 0.9593, gcc: 0.9496 },
    { name: "floyd-warshall", cc: 0.2459, fc: 0.1383, gcc: 0.0957 },
    { name: "atax", cc: 0.0568, fc: 0.0323, gcc: 0.0292 },
    { name: "jacobi-2d", cc: 4.1933, fc: 2.7041, gcc: 2.1561 },
    { name: "syrk", cc: 1.3270, fc: 0.8592, gcc: 0.7528 },
    { name: "bicg", cc: 0.0528, fc: 0.0363, gcc: 0.0335 },
    { name: "heat-3d", cc: 0.0600, fc: 0.0415, gcc: 0.0285 },
    { name: "fdtd-2d", cc: 3.0836, fc: 2.1350, gcc: 1.9288 },
    { name: "ludcmp", cc: 0.1269, fc: 0.0887, gcc: 0.0885 },
    { name: "cholesky", cc: 0.1280, fc: 0.0904, gcc: 0.0751 },
    { name: "lu", cc: 0.1607, fc: 0.1192, gcc: 0.0913 },
    { name: "mvt", cc: 0.3163, fc: 0.2482, gcc: 0.2089 },
    { name: "gemver", cc: 0.3418, fc: 0.2693, gcc: 0.2290 },
    { name: "nussinov", cc: 6.2777, fc: 5.3942, gcc: 3.4427 },
    { name: "symm", cc: 2.0533, fc: 1.7662, gcc: 1.7174 },
    { name: "gesummv", cc: 0.1316, fc: 0.1147, gcc: 0.1042 },
    { name: "durbin", cc: 0.0263, fc: 0.0237, gcc: 0.0181 },
    { name: "deriche", cc: 0.3995, fc: 0.3610, gcc: 0.3592 },
    { name: "syr2k", cc: 3.1813, fc: 2.9311, gcc: 2.7998 },
    { name: "correlation", cc: 5.0584, fc: 4.7055, gcc: 4.6925 },
    { name: "covariance", cc: 5.0437, fc: 4.7053, gcc: 4.6792 },
    { name: "jacobi-1d", cc: 0.0104, fc: 0.0101, gcc: 0.0075 },
    { name: "seidel-2d", cc: 0.2381, fc: 0.2346, gcc: 0.2326 },
    { name: "trmm", cc: 2.0619, fc: 2.0312, gcc: 1.5356 },
    { name: "adi", cc: 0.1109, fc: 0.1101, gcc: 0.1038 },
    { name: "gramschmidt", cc: 5.7080, fc: 5.9490, gcc: 5.8013 }
  ],
  big: [
    { name: "lzw", cc: 2.5998, fc: 2.3516, gcc: 2.3218 },
    { name: "raytracer", cc: 0.8245, fc: 0.7654, gcc: 0.6696 },
    { name: "arcode", cc: 0.4922, fc: 0.4835, gcc: 0.3730 },
    { name: "spass", cc: 5.6808, fc: 5.6504, gcc: 5.1145 },
    { name: "lzss", cc: 3.7454, fc: 3.7475, gcc: 4.1275 }
  ]
};

function formatTime(sec) {
  if (sec < 0.1) {
    return (sec * 1000).toFixed(1) + " ms";
  } else if (sec < 1.0) {
    return (sec * 1000).toFixed(0) + " ms";
  } else {
    return sec.toFixed(2) + " s";
  }
}

function setFig3Suite(suiteKey) {
  document.querySelectorAll('#fig3-tabs .tab-btn').forEach(btn => {
    btn.classList.toggle('active', btn.dataset.suite === suiteKey);
  });

  const list = benchmarkData[suiteKey].slice().sort((a, b) => (a.fc / a.cc) - (b.fc / b.cc));
  const tbody = document.getElementById('fig3-tbody');
  
  tbody.innerHTML = list.map(item => {
    const fcRatio = item.fc / item.cc;
    const reductionPct = (1 - fcRatio) * 100;
    const badgeText = Math.abs(reductionPct) < 0.05 ? '0.0%' : `${reductionPct.toFixed(1)}%`;
    const speedBadge = reductionPct >= 10 ? 'badge-green' : 'badge-gray';
    const gccRatio = item.fc / item.gcc;

    return `
      <tr>
        <td><strong>${item.name}</strong></td>
        <td>${formatTime(item.cc)}</td>
        <td><strong>${formatTime(item.fc)}</strong> (${fcRatio.toFixed(2)}x)</td>
        <td>${formatTime(item.gcc)}</td>
        <td><span class="badge ${speedBadge}">${badgeText}</span></td>
        <td>${gccRatio.toFixed(3)}x</td>
      </tr>
    `;
  }).join('');
}

function highlightAsm() {
  const opRegex = /^(mov[a-z0-9]*|imul[a-z0-9]*|mul[a-z0-9]*|add[a-z0-9]*|sub[a-z0-9]*|div[a-z0-9]*|xor[a-z0-9]*|and[a-z0-9]*|shr[a-z0-9]*|shl[a-z0-9]*|lea[a-z0-9]*|cmp[a-z0-9]*|test[a-z0-9]*|jmp|je|jne|jl|jle|jg|jge|ja|jb|call|ret|push[a-z0-9]*|pop[a-z0-9]*|cvtsi2sd|sqrtsd)\b/i;

  document.querySelectorAll('code.language-asm').forEach(code => {
    const raw = code.textContent;
    const lines = raw.split('\n');
    const highlighted = lines.map(line => {
      const commentIdx = line.indexOf('#');
      let codePart = commentIdx !== -1 ? line.slice(0, commentIdx) : line;
      let commentPart = commentIdx !== -1 ? line.slice(commentIdx) : '';

      let commentHtml = '';
      if (commentPart) {
        const safeComment = commentPart.replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;');
        commentHtml = `<span class="asm-comment">${safeComment}</span>`;
      }

      if (!codePart.trim()) {
        return commentHtml;
      }

      let labelHtml = '';
      const labelMatch = codePart.match(/^(\s*)([a-zA-Z0-9_.]+:)(.*)$/);
      if (labelMatch) {
        labelHtml = labelMatch[1] + `<span class="asm-label">${labelMatch[2]}</span>`;
        codePart = labelMatch[3];
      }

      const opMatch = codePart.match(/^(\s*)([a-zA-Z0-9_]+)(.*)$/);
      let opHtml = '';
      let rest = codePart;
      if (opMatch && opRegex.test(opMatch[2])) {
        opHtml = opMatch[1] + `<span class="asm-op">${opMatch[2]}</span>`;
        rest = opMatch[3];
      }

      rest = rest.replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;');
      rest = rest.replace(/(%[a-zA-Z0-9]+)/g, '<span class="asm-reg">$1</span>');
      rest = rest.replace(/(\b[0-9]+\b(?=\s*[\(,]))/g, '<span class="asm-num">$1</span>');

      return (labelHtml || '') + opHtml + rest + commentHtml;
    });

    code.innerHTML = highlighted.join('\n');
  });
}

// Initialize on page load
setFig1Suite('testc');
setFig3Suite('testc');
highlightAsm();
</script>

*Written by [Aananth V](https://www.linkedin.com/in/aananthv), a first-year master’s student working with InfraAI on AI for systems and formal verification. Edited by [Rohit Dwivedula](https://dwivedula.dev). This work was supported by AWS credits from Amazon.*

[^csmith]: Xuejun Yang, Yang Chen, Eric Eide, and John Regehr. [Finding and Understanding Bugs in C Compilers](https://doi.org/10.1145/1993498.1993532). PLDI 2011.
[^emi]: Vu Le, Mehrdad Afshari, and Zhendong Su. [Compiler Validation via Equivalence Modulo Inputs](https://doi.org/10.1145/2594291.2594334). PLDI 2014.
[^sun]: Chengnian Sun, Vu Le, Qirun Zhang, and Zhendong Su. [Toward Understanding Compiler Bugs in GCC and LLVM](https://doi.org/10.1145/2931037.2931074). ISSTA 2016.
[^bench]: CompCert benchmarks: the programs in CompCert 3.18's `test/c` directory ([CompCert-small-tests](https://github.com/AbsInt/CompCert-small-tests)). PolyBench: the 30 loop-nest kernels of [PolyBench/C 4.2.1](https://sourceforge.net/projects/polybench/). All measurements in this post were taken on an Intel Xeon 4314; the benchmarking harness is in [`faster-compcert/`](https://github.com/UT-InfraAI/CompCert/tree/v3.18/faster-compcert) in our fork.

[^compcert-guarantee]: [CompCert manual, semantic preservation and verification scope](https://compcert.org/man/manual001.html).
[^gcc12]: [GCC 12 release notes](https://gcc.gnu.org/gcc-12/changes.html): auto-vectorization became enabled by default at `-O2` in GCC 12.
[^surgery]: [Assembly-surgery measurements](https://github.com/UT-InfraAI/CompCert/blob/v3.18/faster-compcert/REPORT.md#13-causal-attribution-by-assembly-surgery) and [initial verified-pass results](https://github.com/UT-InfraAI/CompCert/blob/v3.18/faster-compcert/REPORT.md#23-effect-of-each-change-alone).
[^intel-cvt]: Intel, [Instruction Set Reference, CVTSI2SD / VCVTSI2SD](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-2a-manual.pdf).
