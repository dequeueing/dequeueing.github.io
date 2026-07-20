---
title: "Paper Reading: Divergence-Aware Testing of Graphics Shader Compiler Back-Ends"
date: 2026-07-20
tags:
  - fuzzing
---

## Outline {#outline}

- [Abstract](#abstract)
- [Introduction](#introduction)
- [Background](#background)
- [Motivation](#motivation)
- [Design of ShaDiv](#design-of-shadiv)
- [Experiment Results](#experiment-results)
- [Discussion](#discussion)
- [My Questions](#my-questions)

## Abstract {#abstract}

Graphics shaders are programs written in high-level shading languages like GLSL, and graphics shader compilers translate these high-level shader programs into low-level binaries that run on GPUs.

Compilers are complex programs with front-end, middle-end, and back-end stages.

Fact: compilers have bugs.

Since compilers are complex, it is difficult to test the back-end part of the compiler. Empirical finding: state-of-the-art testing tools for shader compilers do not target backends and are ineffective in uncovering back-end bugs.

Contribution: ShaDiv, a testing tool designed to uncover bugs in the back-ends of graphics shader compilers. It generates test inputs with two strategies:

- control flow divergence
- data flow divergence

Evaluation: ShaDiv uncovers 12 back-end bugs, achieves a 25% increase in the coverage of backend components, and finds 4 times as many bugs.

总结一下就是，如何测试一个 GPU compiler，从而可以在里面找到 bug。

## Introduction {#introduction}

Graphic rendering: generating images from 2D or 3D models.

Shader program: describes the expected visual effects. The program is executed on GPU. To run on hardware platforms, the program must be translated and optimized into low-level machine code specific to the target hardware.

The translation process is done by graphics shader compilers. These compilers still contain bugs that lead to incorrect rendering results.

Compilers use a three-phase pipeline: front-end, middle-end, and back-end.

- front-end and middle-end: handle hardware-agnostic optimizations like dead code elimination and constant folding.
- back-end: specific to GPU. SIMT is used to exploit the parallelism of GPUs, which gives compiler backends more optimization opportunities.

Shader compilers are very complex.

Bugs in backends are very hard to detect. The backend is the last stage in the compilation pipeline, so the input must carefully pass the front-end and middle-end optimizations before triggering back-end optimization. Therefore, it is challenging to explore the optimization space of the back-end.

Fact: existing tools for testing shader compilers can hardly test back-end optimizations.

ShaDiv: a testing framework to uncover bugs in shader compiler back-ends. It generates test programs using strategies inspired by the unique SIMT execution model: control flow divergence and data flow divergence.

Contribution summary:

1. Conceptually: test the shader compiler backend. These backends have challenges compared to traditional CPU-based compiler testing.
2. Technically: use divergence and liveness analysis to guide the generation of effective test programs.
3. Empirically: advance the state of the art in compiler testing.

## Background {#background}

### Graphics Shader Programs

![Shader program](/images/blog/divergence-aware-testing/shader-program.png)

Explain this program:

Each thread executes this program by taking a pixel's coordinates `coord` as input and outputs the pixel's color. Since the pixel coordinates vary across pixels, `coord` is qualified with the `varying` keyword.

Another input is the center of the ball, which is of the built-in type `vec3`. `o` is qualified as `uniform` since the value of `o` is the same for all pixels at any given time.

Line 7-9: if the camera is inside the ball, the color is set to black; otherwise, the camera shoots a ray in the direction of the pixel.

Line 12-17: shoot a ray in the direction of the pixel, and then calculate the distance the ray travels to hit the ball or return `-1` if it misses.

> 什么是 camera？ - 观察点，也就是相机的位置。pixel 并不是一个球上的一点，而是屏幕上的一点。因此，从 camera 出发，跟 pixel 的 coordinate 连线，再与球产生相互作用。这个是整个的工作原理。

注释版：

![Annotated shader program](/images/blog/divergence-aware-testing/shader_program_annoated.png)

### Shader Execution Pattern: SIMT Model

Threads are grouped into warps. Each warp has 32 threads. Threads in the same warp execute the same program instruction at the same time but with different data.

Since different threads take different input data, threads in the same warp may take different control flow paths. This is divergent control flow, implemented with execution masks.

GPU distinguishes two types of branching instructions: uniform branching and divergent branching.

- uniform: all threads in the kernel uniformly take either the true or false branch at any time.
- divergent branches: threads in the same warp potentially take different branches due to input data.

The determination of uniform/divergent branches is decided by the compiler based on divergence analysis.

![Uniform and divergent branching](/images/blog/divergence-aware-testing/uniform-and-divergent-branching.png)

### Graphics Shader Compiler

What is the challenge in the back-end of shader compilers?

**control/data flow divergence analysis**: a process that identifies and manages differences in execution paths and data usage among shader threads. 

- divergence control flow: threads within a warp take different execution paths
- divergent data flow: threads hold different values for the same variable at the same program point. 

![Divergence analysis](/images/blog/divergence-aware-testing/divergence-analysis.png)

以这个图片为例子，进行 divergence analysis。

4-5 行，这些变量全部都是常量或者依赖于常量，因此他们是 UNIFORM。第七行的这个 condition，因为所有在同一个 warp 中的 Threads 都是有相同的值的，因此这个也是 uniform。11-12 行，因为 coord 是不同的，因此都是 divergent，这也导致第 13 行是 divergent。一个 divergent 的 condition 下面的 branch 都是 divergent 的。由此完成分析。可见这里的依赖关系其实是可以用一张图来表示的。

**divergence-aware** register allocation: handle the mapping of variables in the shader program to the physical registers in the GPU. 

GPU has two sets of register files: vector general purpose registers and scalar general purpose registers.

VGPRs are local to each thread, and SGPRs are shared among threads in the same warp.

Variables with different values are stored in VGPRs, and uniform variables can be mapped to both VGPRs and SGPRs.

The number of registers is limited. If the register pressure exceeds the limit, *register spilling* occurs. Therefore, the compiler must carefully allocate registers to minimize the pressure and avoid violations of program correctness.

It also needs to consider the live ranges of variables, which are the intervals between the first and the last use of a variable.

![Register allocation](/images/blog/divergence-aware-testing/register-allocation.png)

这里有一个例子，依旧是从之前的 program 来的。由于一些变量是 divergent 的，因此它们必须被存储到 VGPR 当中。d 和o 是 uniform 的，既可以被存储到 vgpr 也可以被存储到 sgpr 当中。但是如果存储到 vgpr，会发生 register spilling，因此 compiler 只能将它们存储到 sgpr 当中。

## Motivation {#motivation}

The output of ShaDiv: shader programs with diverse **divergence and liveness characteristics** to test shader compiler backends.

Open-source shader compiler says: divergence info is embedded within the intermediate representation and used extensively throughout the backend for optimization.

Why are divergence and liveness checks so important? Here are two examples:

1. bug discovered by uniform-divergent interleavings. 

![Uniform-divergent interleaving](/images/blog/divergence-aware-testing/interleaving.png)

The bug is from the compiler **mistakenly deallocating the temporary mask** at line 9, leading to a violation of semantics and a hang during runtime execution.

2. bug from extended liveness. 

![Extended liveness](/images/blog/divergence-aware-testing/extended_use.png)

想表达的意思：单纯生成一个带 divergent if 的程序还不够。必须让 divergent 变量形成较长、跨越复杂控制流的 def-use chains，才能迫使 GPU 编译器后端执行复杂的寄存器分配和 spilling，从而暴露隐藏 bug。

## Design of ShaDiv {#design-of-shadiv}

Focus: testing the correctness of shader compiler backends.

2 types of bugs:

- compilation failures: cause obvious misbehaviors, such as crashes or memory violations. They can happen both at compilation time and at runtime.
- logic errors: no errors, but the binary renders incorrect images during execution. These are more subtle and stealthy.

--- 
The pipeline of ShaDiv:

![ShaDiv pipeline](/images/blog/divergence-aware-testing/pipeline.png)

1. program generation: generate random shader programs as testing inputs. It first generates a customized IR and then translates the IR to concrete shader languages. IR allows us to abstract the syntax and semantics of shader languages to simplify the generation and mutation of shader programs.

2. divergence analysis: augments the random program generator by **generating shader programs with complex control flow divergence patterns.**

3. liveness mutation: complicates the data flow patterns of shader programs by mutating the liveness range of variables in the generated program.


> 我们暂时跳过了具体将每一个部分 design 的 subsection，然后跳到 evaluation。

## Experiment Results {#experiment-results}

Four shader compilers.

- testing time: time of generating the testing inputs and the time of compiling and executing the generated programs.
- identifying errors.
- differentiating back-end errors
- uncovered error-triggering inputs.
- error deduplication and bug analysis.

## Discussion {#discussion}

- Testing versus verification: validate the correctness of the shader compiler back-ends. 
- accommodating variances of different GPUs
- white-box testing versus black-box testing
- challenges to coverage-guided fuzzing
- alternatives to our testing approach
- extensions to other GPU compilers
- limitation of language feature support

## My Questions {#my-questions}

fuzzing的基本流程：产生测试输入，交给目标执行，观察是否出现异常，根据结果继续产生更多的输出。不同fuzzer 主要的区别就在于，怎么产生输入以及是否根据执行反馈调整输入，怎么判断发现了 bug。

- mutation based fuzzing：准备正常输入作为seed corpus，然后进行随机修改。
- generation-based fuzzing：根据语法和类型规则，从零生成合法的程序。

ShaDiv属于generation-based fuzzing。首先设计一套简化的中间语言，随机选择语句类型，递归生成表达式，得到一颗结构合法的程序树，然后将IR 转换成真正的shader program。

但仅仅生成合法程序还不是 ShaDiv 的主要创新。一个普通随机程序生成器可能生成大量简单程序，却很少触发 GPU back-end 中困难的逻辑。因此 ShaDiv 又加入了两层 GPU-specific guidance。

第一层是 divergence-aware generation。生成器持续记录每个变量是 uniform 还是 divergent，并主动生成 uniform 和 divergent 语句交错的控制流。它不是等程序生成完再检查，而是在生成每条语句时，根据期望的 divergence 类型筛选可以使用的变量。

第二层是 liveness mutation。程序生成完成后，ShaDiv 会选择已有变量，在后面安全的位置增加新的读取，从而延长变量的 live range。它会分析 def-use 和 control flow，避免读取未初始化变量，也避免新增 use 破坏原本的 divergence 约束。

---
知识补充：

1. compiler 

Compiler 基础需要掌握前端、中端、后端的分工，以及 AST、IR、CFG、SSA、dominance、def-use、data-flow analysis、liveness、register allocation、spilling 和 instruction selection。

第一份资源建议使用 LLVM 的 Kaleidoscope Tutorial。它通过实现一个小语言，逐步介绍 lexer、parser、AST 和 LLVM IR，适合建立“源程序如何进入编译器”的整体模型。 随后学习 Cornell CS 6120 的 self-guided course，重点看 IR、CFG、SSA、data-flow analysis 和经典优化，不需要完整学完课程。

2. Fuzzing

具体概念包括 seed/corpus、mutation-based 与 generation-based fuzzing、coverage guidance、structured input generation、crash oracle、differential testing、metamorphic testing、undefined behavior、false positive、test-case reduction 和 bug deduplication。

可以先用 libFuzzer 做第一个实验。LLVM 官方文档将其定义为 in-process、coverage-guided、evolutionary fuzzing engine，适合用一个很小的 C/C++ parser 理解 harness、corpus 和 coverage feedback。 然后快速阅读 AFL++ 的文档，建立对 instrumentation、mutation、corpus minimization 和长期 fuzzing campaign 的认识，不需要深入每一种高级模式。

进入 compiler fuzzing 时，最值得先读的是 Csmith 论文。它能让你理解为什么随机字节 mutation 不适合 compiler、为什么需要生成语法和类型正确的程序，以及为什么 undefined behavior 会破坏 oracle。