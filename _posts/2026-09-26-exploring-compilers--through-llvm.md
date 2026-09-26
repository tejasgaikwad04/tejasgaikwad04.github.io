---


# Exploring Compilers Through LLVM

Over the past several months, I have been exploring compiler engineering through LLVM, with a particular focus on optimization and code generation.

I started by studying LLVM's architecture and understanding how its different components interact. This gradually developed into hands-on investigation of LLVM issues, reduced test cases, compiler transformations, and the implementation behind them.

My work so far has included GlobalISel, SelectionDAG, InstCombine, ValueTracking, KnownBits, loop optimization, and vectorization. Along the way, I have investigated missed optimizations and contributed changes upstream.

One of my first contributions involved GlobalISel and KnownBits, where I worked on reasoning about the parity of `ctpop` results. I later explored an InstCombine transformation involving `copysign`, investigated ValueTracking and KnownBits behavior, and worked through several other LLVM optimization and code-generation issues.

These contributions have given me an opportunity to work beyond the surface level of compiler behavior — reading LLVM's C++ implementation, reducing test cases, tracing transformations, writing regression tests, debugging with GDB, and working through upstream review.

This post is a short overview of that work so far, the problems I have investigated, and the areas of LLVM I am continuing to explore.
