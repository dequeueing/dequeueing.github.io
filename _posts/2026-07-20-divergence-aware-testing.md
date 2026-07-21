---
title: "Paper Reading: Divergence-Aware Testing of Graphics Shader Compiler Back-Ends"
date: 2026-07-20
tags:
  - fuzzing
---

This paper is about testing graphics shader compiler backends. The main idea is that ordinary random shader programs do not sufficiently stress the backend. To find backend bugs, the generated programs should deliberately exercise GPU-specific behavior such as control-flow divergence, data-flow divergence, and long variable live ranges.

The proposed tool, **ShaDiv**, uses these ideas to generate better test programs for shader compilers.

## Outline {#outline}

- [The Problem](#the-problem)
- [Why Shader Compiler Backends Are Hard to Test](#why-shader-compiler-backends-are-hard-to-test)
- [Background: Shader Programs and SIMT](#background)
- [Divergence Analysis](#divergence-analysis)
- [Register Allocation and Liveness](#register-allocation-and-liveness)
- [Why Divergence and Liveness Matter](#why-divergence-and-liveness-matter)
- [ShaDiv Design](#shadiv-design)
- [Evaluation](#evaluation)
- [How I Understand This Paper](#how-i-understand-this-paper)
- [Suggested Background](#suggested-background)

## The Problem {#the-problem}

Graphics shaders are programs written in high-level shading languages such as GLSL. Shader compilers translate these programs into low-level binaries that run on GPUs.

Like other compilers, shader compilers are complex. They usually contain a front-end, a middle-end, and a back-end.

- The front-end and middle-end handle hardware-agnostic tasks such as parsing, type checking, dead code elimination, and constant folding.
- The back-end performs GPU-specific lowering and optimization.

The empirical observation of the paper is that existing shader compiler testing tools do not target backends well. They can generate shader programs, but those programs often fail to trigger the backend optimizations where GPU-specific bugs live.

ShaDiv is designed to solve this problem. It generates shader programs with richer divergence and liveness patterns, which are more likely to exercise backend logic.

## Why Shader Compiler Backends Are Hard to Test {#why-shader-compiler-backends-are-hard-to-test}

Backend bugs are difficult to trigger because the backend is the final stage of the compiler pipeline. A test input must first pass through the front-end and middle-end before it can reach backend optimizations.

This means a useful test program must satisfy two requirements:

1. It must be syntactically and semantically valid enough to survive early compilation stages.
2. It must contain GPU-specific patterns that stress backend decisions.

For shader compilers, the important GPU-specific patterns are often related to SIMT execution, divergence analysis, register allocation, and spilling.

## Background: Shader Programs and SIMT {#background}

Consider a simple shader program:

![Shader program](/images/blog/divergence-aware-testing/shader-program.png)

Each thread executes this program with a pixel coordinate `coord` as input and outputs the color for that pixel. Since `coord` differs across pixels, it is marked as `varying`.

Another input is `o`, the center of the ball. It is a `vec3` and is marked as `uniform` because its value is the same for all pixels at a given time.

The program checks whether the camera is inside the ball. If so, the color is black. Otherwise, it shoots a ray in the direction of the pixel and computes the distance to the ball.

The key intuition is:

> A pixel is not a point on the ball. It is a point on the screen. The shader conceptually casts a ray from the camera through that pixel and checks how the ray interacts with the object.

Here is the annotated version:

![Annotated shader program](/images/blog/divergence-aware-testing/shader_program_annoated.png)

On GPUs, threads are grouped into warps. A warp usually contains 32 threads. Threads in the same warp execute the same instruction at the same time, but each thread may hold different data.

This is the SIMT model: **single instruction, multiple threads**.

Because threads may receive different inputs, different threads in the same warp may need to take different control-flow paths. This is called **divergent control flow**.

![Uniform and divergent branching](/images/blog/divergence-aware-testing/uniform-and-divergent-branching.png)

The compiler must decide whether a branch is uniform or divergent.

- A uniform branch is taken the same way by all threads.
- A divergent branch may be taken differently by different threads in the same warp.

This decision is made through divergence analysis.

## Divergence Analysis {#divergence-analysis}

Divergence analysis identifies whether values and control-flow decisions are uniform or divergent across threads.

- **Divergent control flow**: threads in the same warp take different paths.
- **Divergent data flow**: threads hold different values for the same variable at the same program point.

![Divergence analysis](/images/blog/divergence-aware-testing/divergence-analysis.png)

In the example, constants and variables derived only from constants are uniform. The branch condition at line 7 is also uniform because all threads in the same warp see the same value.

By contrast, `coord` differs across pixels, so values derived from `coord` become divergent. Once a branch condition is divergent, the variables defined inside its branches may also become divergent.

This dependency structure can be represented as a graph. That is why divergence analysis is fundamentally a data-flow analysis problem.

## Register Allocation and Liveness {#register-allocation-and-liveness}

Shader compiler backends also need divergence-aware register allocation.

GPUs have two kinds of register files:

- **VGPRs**: vector general-purpose registers, local to each thread.
- **SGPRs**: scalar general-purpose registers, shared among threads in the same warp.

Divergent variables must be stored in VGPRs because each thread may hold a different value. Uniform variables may be stored in either VGPRs or SGPRs.

![Register allocation](/images/blog/divergence-aware-testing/register-allocation.png)

In the example, divergent variables must be placed in VGPRs. Variables such as `d` and `o` are uniform, so they can be stored in SGPRs. This matters because registers are limited. If register pressure becomes too high, the compiler may perform **register spilling**, moving values between registers and memory.

The backend must also reason about **live ranges**: the interval between the first and last use of a variable. Longer live ranges increase register pressure and make allocation harder.

## Why Divergence and Liveness Matter {#why-divergence-and-liveness-matter}

The paper's key insight is that simply generating valid shader programs is not enough.

To expose backend bugs, test programs need to contain difficult divergence and liveness patterns.

The paper gives two motivating examples.

First, a bug caused by uniform-divergent interleavings:

![Uniform-divergent interleaving](/images/blog/divergence-aware-testing/interleaving.png)

The compiler mistakenly deallocates a temporary mask at line 9, violating program semantics and causing a hang at runtime.

Second, a bug caused by extended liveness:

![Extended liveness](/images/blog/divergence-aware-testing/extended_use.png)

The lesson is that a program with one divergent `if` is not necessarily enough. To stress the backend, divergent variables should form long def-use chains across complex control flow. This forces the compiler to perform harder register allocation and spilling decisions, which can expose hidden backend bugs.

## ShaDiv Design {#shadiv-design}

ShaDiv focuses on testing the correctness of shader compiler backends.

It targets two types of bugs:

- **compilation failures**: crashes, memory violations, or obvious failures during compilation or runtime;
- **logic errors**: the program compiles and runs, but renders an incorrect image.

The pipeline is:

![ShaDiv pipeline](/images/blog/divergence-aware-testing/pipeline.png)

ShaDiv has three main stages.

First, **program generation** creates random shader programs. It generates a customized IR and then translates that IR into concrete shader languages. The IR abstracts away language-specific syntax and makes generation and mutation easier.

Second, **divergence analysis** guides the random generator so that it can create shader programs with complex control-flow divergence patterns.

Third, **liveness mutation** extends variable live ranges to create more difficult data-flow patterns.

In other words, ShaDiv is not just a random shader generator. It is a generation-based fuzzer with GPU-specific guidance.

## Evaluation {#evaluation}

The paper evaluates ShaDiv on four shader compilers.

The evaluation considers:

- testing time, including input generation time and compile/execute time;
- how errors are identified;
- how back-end errors are distinguished;
- how error-triggering inputs are uncovered;
- bug deduplication and analysis.

ShaDiv uncovers 12 back-end bugs, improves backend component coverage by 25%, and finds about 4 times as many bugs as prior approaches.

## How I Understand This Paper {#how-i-understand-this-paper}

At a high level, ShaDiv is a generation-based fuzzing system.

普通 generation-based fuzzing 会根据语法和类型规则生成合法程序。ShaDiv 的不同之处在于，它不满足于“生成合法 shader program”。它进一步问：

> 什么样的 shader program 更容易触发 GPU compiler backend 的困难逻辑？

它的答案是两层 GPU-specific guidance。

第一层是 **divergence-aware generation**。生成器持续记录每个变量是 uniform 还是 divergent，并主动生成 uniform 和 divergent 语句交错的控制流。它不是等程序生成完再检查，而是在生成每条语句时，根据期望的 divergence 类型筛选可用变量。

第二层是 **liveness mutation**。程序生成完成后，ShaDiv 选择已有变量，在后面安全的位置增加新的读取，从而延长变量 live range。它会分析 def-use 和 control flow，避免读取未初始化变量，也避免新增 use 破坏原本的 divergence 约束。

所以这篇论文的核心不是“用 fuzzing 测 shader compiler”，而是：

> 用 GPU backend 的关键分析逻辑来指导 fuzzing input generation。

## Suggested Background {#suggested-background}

理解这篇论文需要两块背景：compiler 和 fuzzing。

Compiler 方面，需要理解 front-end、middle-end、back-end 的分工，以及 AST、IR、CFG、SSA、dominance、def-use、data-flow analysis、liveness、register allocation、spilling 和 instruction selection。

第一份资源可以看 LLVM 的 Kaleidoscope Tutorial。它通过实现一个小语言，逐步介绍 lexer、parser、AST 和 LLVM IR，适合建立“源程序如何进入编译器”的整体模型。

之后可以看 Cornell CS 6120 的 self-guided course，重点关注 IR、CFG、SSA、data-flow analysis 和经典优化。

Fuzzing 方面，需要理解 seed/corpus、mutation-based fuzzing、generation-based fuzzing、coverage guidance、structured input generation、crash oracle、differential testing、metamorphic testing、undefined behavior、false positive、test-case reduction 和 bug deduplication。

可以先用 libFuzzer 做一个小实验，理解 harness、corpus 和 coverage feedback。然后快速阅读 AFL++ 文档，建立对 instrumentation、mutation、corpus minimization 和长期 fuzzing campaign 的认识。

进入 compiler fuzzing 时，最值得先读的是 Csmith。它能解释为什么随机字节 mutation 不适合 compiler testing，为什么需要生成语法和类型正确的程序，以及为什么 undefined behavior 会破坏 oracle。
