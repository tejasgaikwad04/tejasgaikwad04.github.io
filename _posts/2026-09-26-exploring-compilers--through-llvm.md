---
layout: post
title: "Exploring Compilers Through LLVM"
date: 2026-09-26
author: "Tejas Gaikwad"
categories: [LLVM, Compilers]
tags: [LLVM, Compilers, Optimization, GlobalISel, InstCombine]
---

# Introduction

Compilers brings together several areas of computer science and systems engineering: programming languages, program analysis, optimization, instruction selection, and computer architecture. My interest in the field grew from wanting to understand what happens between source code and the machine instructions ultimately executed by a processor, and more importantly, how compilers reason about programs to produce better code.

LLVM provided a practical way to explore these concepts in depth. Rather than treating the compiler as a black box, I began studying its intermediate representations, analyses, transformations, and code-generation infrastructure. This led me through topics such as LLVM IR and SSA, basic blocks and control flow, optimization passes, SelectionDAG, GlobalISel, Machine IR, TableGen, InstCombine, ValueTracking, KnownBits, loop optimization, and vectorization.

As my understanding developed, I started moving from studying concepts to investigating real compiler behavior. I worked through LLVM issues involving missed optimizations, analyzed reduced test cases, traced transformations through the LLVM source, and used tools such as `opt`, `llc`, `llvm-reduce`, `llvm-lit`, FileCheck, and GDB to understand why a particular transformation did or did not occur.

This eventually led to upstream contributions in LLVM, including work in GlobalISel and KnownBits, along with further investigations into ValueTracking and InstCombine. Working on these problems provided a different perspective on compiler development: a seemingly small optimization can involve understanding the semantics of an IR operation, existing analysis infrastructure, legality and correctness constraints, and the interaction between multiple compiler components.

This article is a technical retrospective of that learning process. It covers the compiler concepts and LLVM infrastructure I explored, the problems I investigated, the contributions I made, and the development and debugging workflow I have been building along the way.

