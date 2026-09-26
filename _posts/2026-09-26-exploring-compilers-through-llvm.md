---
layout: post
title: "Exploring Compilers Through LLVM"
date: 2026-09-26
author: "Tejas Gaikwad"
categories:
  - LLVM
  - Compilers
tags: [LLVM, Compiler Engineering, Optimization, GlobalISel, InstCombine]
---
Over the past several months, I have been exploring compiler engineering through LLVM, with a particular focus on optimization and code generation.

I started by studying LLVM's architecture and understanding how its different components interact. This gradually developed into hands-on investigation of LLVM issues, reduced test cases, compiler transformations, and the implementation behind them.

My work so far has included GlobalISel, SelectionDAG, InstCombine, ValueTracking, KnownBits, loop optimization, and vectorization. Along the way, I have investigated missed optimizations and contributed changes upstream.

One of my first contributions involved GlobalISel and KnownBits, where I worked on reasoning about the parity of `ctpop` results. I later explored an InstCombine transformation involving `copysign`, investigated ValueTracking and KnownBits behavior, and worked through several other LLVM optimization and code-generation issues.

These contributions have given me an opportunity to work beyond the surface level of compiler behavior — reading LLVM's C++ implementation, reducing test cases, tracing transformations, writing regression tests, debugging with GDB, and working through upstream review.

This post is a short overview of that work so far, the problems I have investigated, and the areas of LLVM I am continuing to explore.

## LLVM Work and Areas of Exploration

My work with LLVM has covered several parts of the optimization and code-generation pipeline.

I initially focused on **GlobalISel**, particularly its value-tracking and `KnownBits` infrastructure. This led me to investigate how LLVM reasons about properties of values and how those properties can be used during instruction selection and optimization.

I also explored **InstCombine**, where I worked on identifying and implementing a transformation involving `copysign`. This involved understanding an existing transformation, identifying an additional equivalent pattern, and adding a regression test for it.

Beyond these contributions, I have investigated several LLVM issues involving **ValueTracking, KnownBits, SelectionDAG, loop optimization, and vectorization**. Some of these investigations resulted in upstream changes, while others helped me understand potential optimization opportunities and how LLVM's existing analyses and transformations handle them.

Working across these areas has also given me experience with the practical workflow of LLVM development — reproducing issues, reducing test cases, locating the relevant implementation, debugging the compiler, writing tests, and working through upstream review.
## Contribution 1 — GlobalISel and KnownBits

One of my first upstream contributions involved the interaction between **GlobalISel** and LLVM's `KnownBits` analysis.

### The Problem

The problem I worked on involved reasoning about the parity of the result of the `ctpop` operation.

`ctpop` returns the population count of a value — the number of set bits. An important property of this result is that its least significant bit represents the parity of the population count:

```text
ctpop(x) & 1
```

is `1` when `x` contains an odd number of set bits and `0` when it contains an even number.

The existing GlobalISel value-tracking logic did not fully capture this property. This meant that information that could be derived from the `ctpop` result was not always available to later optimization or code-generation decisions.

### The Approach

I started by tracing how GlobalISel performs known-bit analysis and comparing the relevant behavior with LLVM's existing value-tracking infrastructure.

The main challenge was not simply identifying the mathematical property, but understanding where that information belonged in the GlobalISel implementation and how it should be represented using the existing matching and `KnownBits` mechanisms.

I used a small MIR test case to isolate the behavior and verify the expected known-bit information.

### The Implementation

The change was made in:

```text
llvm/lib/CodeGen/GlobalISel/GISelValueTracking.cpp
```

The implementation recognizes the relevant `ctpop` pattern and uses the existing GlobalISel instruction-matching infrastructure to reason about the result.

One detail that became particularly useful during the implementation was learning to work with LLVM's matcher utilities rather than introducing custom logic for simple instruction patterns.

This also became part of the upstream review process. The review feedback encouraged simplifying the implementation, including using `mi_match` directly rather than introducing a lambda for the matching logic.

That was a useful lesson in LLVM development: a solution that is functionally correct can often be improved further by fitting it more closely into the existing LLVM coding patterns.

### Testing

I added a dedicated MIR regression test:

```text
llvm/test/CodeGen/GlobalISel/knownbits-ctpop-parity.mir
```

The test checks that GlobalISel's known-bit reasoning captures the expected information for the `ctpop` parity case.

I then ran the corresponding LLVM test using `llvm-lit` and verified that the test passed.

### Upstream Review

The contribution also gave me my first experience working through the LLVM upstream review process.

The review was useful beyond the specific change because it exposed me to LLVM's expectations around implementation style, instruction matching, and keeping changes focused.

The final result was a small change, but the process involved several parts of LLVM development that I had previously only studied separately: understanding the existing implementation, creating a minimal regression test, refining the code based on review feedback, and validating the change with LLVM's test infrastructure.

This contribution was an important step in moving from studying LLVM internals to actively modifying and contributing to the project.
## Contribution 2 — InstCombine and `copysign`

My next contribution involved **InstCombine**, LLVM's instruction-combining optimization pass.

The work focused on extending an existing transformation in `foldSelectToCopysign`.

### The Problem

While investigating InstCombine transformations, I came across an existing optimization that recognizes certain `select` patterns and transforms them into the `copysign` intrinsic.

The existing transformation handled specific relationships between the selected values. I investigated whether the same semantic operation could also be recognized when the sign information was expressed through a bitwise XOR operation.

The goal was to extend the transformation without changing its existing behavior or introducing incorrect transformations.

### Understanding the Existing Transformation

Before modifying the code, I traced the existing implementation of:

```text
foldSelectToCopysign
```

in:

```text
llvm/lib/Transforms/InstCombine/InstCombineSelect.cpp
```

I also used LLVM IR test cases to understand how the original pattern was represented and how InstCombine transformed it.

This was useful because the transformation is easier to understand by looking at the IR pattern that InstCombine is trying to recognize rather than starting directly from the C++ implementation.

### The New Pattern

The additional pattern I investigated involved sign information represented using a bitwise XOR of the bit representations of two floating-point values.

Conceptually, the pattern can be represented as:

```text
signbit(bitcast(X) ^ bitcast(Y))
```

combined with a selection between `-Y` and `Y`.

This can be expressed more directly using:

```text
copysign(Y, X)
```

The change extended `foldSelectToCopysign` so that this equivalent form could also be recognized.

The implementation required carefully matching the IR pattern and ensuring that the transformation was only applied when the required sign-bit relationship was guaranteed.

### Regression Test

I added a dedicated LLVM IR regression test covering the new pattern:

```text
llvm/test/Transforms/InstCombine/fold-select-to-copysign-xor.ll
```

I also used a small C test case during development to generate and inspect the corresponding LLVM IR at different optimization levels.

This helped me move between the source-level expression and the actual IR pattern being handled by InstCombine.

### Result

After implementing the transformation and adding the regression test, I ran the relevant LLVM test using `llvm-lit` and verified that it passed.

This work gave me a deeper understanding of how InstCombine identifies canonicalization opportunities and how seemingly different IR expressions can represent the same underlying operation.

It also reinforced an important part of LLVM development: before adding a transformation, it is necessary to understand the existing canonical form and make the new pattern fit naturally into the existing optimization rather than creating a separate transformation unnecessarily.
## Loop Optimization and Vectorization

Another area I explored was LLVM's loop optimization and vectorization infrastructure, particularly a case involving a **predicated counting loop**.

### Predicated Counting Loop

I investigated an LLVM issue involving a loop where the number of iterations was determined through a conditional operation inside the loop.

The interesting part of the problem was that the loop contained enough structure for vectorization to potentially be beneficial, but the control flow and predication made the transformation less straightforward.

Rather than immediately attempting to modify the vectorizer, I first focused on understanding how LLVM represented the loop and how the vectorization analysis evaluated it.

### Understanding the Vectorization Opportunity

I studied the relevant LLVM IR and followed the loop through the optimization pipeline to understand which conditions were preventing or limiting vectorization.

This involved looking at the interaction between loop structure, conditional execution, and the requirements imposed by LLVM's vectorization analysis.

The investigation helped me understand that successful vectorization is not simply about identifying independent operations. LLVM also needs to establish that the transformed loop preserves the required control-flow and memory semantics.

### What I Learned

This investigation gave me a better understanding of how LLVM's loop vectorizer reasons about real-world loops that are more complicated than simple counted loops.

It also changed how I approach missed optimizations. Instead of immediately looking for a missing transformation, I now try to determine **which analysis or legality check prevents the optimization from happening**.

That distinction is useful when navigating a large compiler codebase because the visible missed optimization may be the result of a decision made much earlier in the optimization pipeline.
## Other LLVM Investigations

Alongside the contributions described above, I have spent time investigating several LLVM optimization and analysis issues. These investigations have helped me understand how different compiler analyses interact and where optimization opportunities can be missed.

### ValueTracking and KnownBits

I investigated cases where information available through `llvm.assume` could potentially be propagated further through LLVM's `ValueTracking` and `KnownBits` infrastructure.

One example involved reasoning about a masked value:

```llvm
%masked = and i8 %x, 127
%nz = icmp ne i8 %masked, 0
call void @llvm.assume(i1 %nz)
```

The assumption provides information about `%masked`, and because a non-zero masked value implies that `%x` is also non-zero, this creates an opportunity for stronger value reasoning.

Working through this case involved reducing the original reproducer, examining the existing `KnownBits` and value-tracking logic, and debugging the relevant InstCombine paths.

This investigation also gave me practical experience using **GDB to trace LLVM's optimization decisions** rather than treating the optimizer as a black box.

### Missed Optimizations

I have also investigated several missed-optimization cases where LLVM could potentially derive stronger information from existing conditions.

One example involved reasoning about conditions such as:

```text
a != b
```

and what additional facts can be inferred about `a` and `b`.

These cases are interesting because the challenge is often not implementing a new transformation, but determining whether the required fact can already be derived by an existing analysis such as `ValueTracking`, `KnownBits`, or another simplification pass.

Working on these issues has helped me become more comfortable with LLVM's approach to proving properties about values.

### SelectionDAG

I have also explored optimization opportunities in **SelectionDAG**, including a case involving constant folding of vector reduction operations such as `VECREDUCE_ADD`.

This required understanding where constant arithmetic is folded in SelectionDAG and how vector reduction operations are represented before instruction selection.

Although these investigations did not all result in upstream changes, they were valuable for understanding another part of LLVM's code-generation pipeline and how optimization opportunities can exist

## How I Work on LLVM Issues

One of the biggest changes in my approach to LLVM has been learning how to investigate a compiler problem systematically.

### Reproducing the Problem

I start by making sure I can reproduce the reported behavior locally. A small and reliable reproducer makes the rest of the investigation much easier.

I typically inspect the LLVM IR or MIR involved and compare the behavior before and after the relevant optimization pass.

### Reducing the Test Case

When the original reproducer is large, I try to reduce it to the smallest example that still demonstrates the problem.

LLVM's `llvm-reduce` tool has been particularly useful for this. A reduced test case makes it easier to understand the actual compiler behavior and significantly reduces the amount of code that needs to be investigated.

### Finding the Relevant Code

Once the behavior is reproducible, I trace the transformation through LLVM's source code.

This usually involves identifying the optimization or analysis responsible for the behavior and then locating the corresponding implementation.

For example, depending on the issue, this can lead to areas such as:

```text id="5djx7n"
InstCombine
ValueTracking
KnownBits
GlobalISel
SelectionDAG
Loop Vectorizer
```

Learning to find the relevant implementation independently has become an important part of my LLVM workflow.

### Debugging

For more complicated cases, I use LLVM's debugging facilities and GDB to follow the compiler's execution.

Setting breakpoints in the relevant transformation and inspecting the IR at different points can reveal why an expected optimization is not being performed.

This has been especially useful for understanding cases where the compiler's behavior is not obvious from the final generated IR.

### Testing

After identifying a possible change, I add or update a focused regression test.

I then run the relevant LLVM tests with `llvm-lit` and, when necessary, inspect the generated IR or MIR to verify that the transformation behaves as expected.

### Upstream Review

For upstream contributions, the process does not end when the code works locally.

Review feedback often leads to changes in implementation style, pattern matching, test structure, or code organization. Working through that feedback has helped me understand LLVM's development practices and the importance of keeping changes focused and maintainable.

This workflow — **reproduce, reduce, locate, debug, implement, test, and review** — has become the foundation of how I approach LLVM issues.

## What I've Learned

Working on LLVM has changed the way I approach compiler problems.

I have learned that understanding an optimization is often more important than simply knowing that it exists. Following a transformation from LLVM IR to the relevant C++ implementation, reducing a reproducer, and debugging the compiler has helped me build a much more concrete understanding of compiler internals.

The upstream contribution process has also taught me the importance of small, focused changes. A good compiler change is not only about making an optimization work; it also needs a clear reproducer, an appropriate regression test, maintainable implementation, and compatibility with the existing compiler infrastructure.

Most importantly, working on real LLVM issues has helped connect the concepts I studied with the behavior of an actual production compiler.

## What's Next

I want to continue contributing to LLVM while exploring deeper areas of compiler optimization and code generation.

Some of the areas I am particularly interested in are **Global Value Numbering, missed optimizations, loop transformations, vectorization, instruction selection, and backend performance**.

I also want to spend more time understanding how LLVM optimizations affect generated machine code and performance on real workloads.

My goal is to continue moving from investigating individual compiler behaviors toward taking on larger optimization problems and contributing more consistently upstream.

## Conclusion

My work with LLVM so far has been a combination of contributions, investigations, debugging, and continuous learning.

The two contributions described here — work in GlobalISel's `KnownBits` reasoning and an InstCombine transformation involving `copysign` — gave me the opportunity to work directly with LLVM's implementation and upstream development process.

The additional investigations into ValueTracking, KnownBits, SelectionDAG, loop optimization, and vectorization have helped broaden my understanding of how different parts of LLVM interact.

There is still a lot more to explore, but working on real compiler problems has given me a strong foundation for continuing in this direction.

I plan to keep contributing, investigating optimization opportunities, and documenting what I learn along the way.
