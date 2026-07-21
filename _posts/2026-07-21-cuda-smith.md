---
title: "Paper Reading: CUDAsmith: A Fuzzer for CUDA Compilers"
date: 2026-07-21
tags:
  - fuzzing
---

This paper studies how to fuzz CUDA compilers. The main difficulty is not only generating CUDA programs, but generating CUDA programs whose behavior is deterministic, valid, and useful for finding compiler bugs.

CUDAsmith adapts ideas from Csmith and CLSmith to CUDA. It combines random program generation, differential testing, and EMI-style transformations to test CUDA compilers.

## Outline {#outline}

- [The Problem](#the-problem)
- [Why CUDA Compiler Fuzzing Is Hard](#why-cuda-compiler-fuzzing-is-hard)
- [Background: Csmith, CLSmith, Differential Testing, and EMI](#background)
- [CUDAsmith Design](#cudasmith-design)
- [How to Read This Paper](#how-to-read-this-paper)
- [Takeaways](#takeaways)

## The Problem {#the-problem}

CUDAsmith is a fuzzing framework for CUDA compilers. It randomly generates deterministic and valid CUDA kernels with different strategies.

The main techniques are:

- random differential testing
- EMI testing

Both are ways to address the **test oracle problem**: how do we know whether the compiler output is correct?

## Why CUDA Compiler Fuzzing Is Hard {#why-cuda-compiler-fuzzing-is-hard}

There are two central challenges in fuzzing CUDA compilers.

First, the fuzzer must generate effective and deterministic kernel code as compiler inputs. These kernels should contain enough GPU features to exercise the compiler under test.

Second, the fuzzer must determine the test oracle. Unlike many ordinary programs, CUDA compiler outputs do not always have a simple expected answer. The paper adapts existing testing techniques into the CUDA compiler testing context.

## Background: Csmith, CLSmith, Differential Testing, and EMI {#background}

The original paper places related work later, but it is easier to understand CUDAsmith if we introduce the background first.

**Csmith** is a testing tool for C compilers. It randomly generates C programs while avoiding undefined behavior.

**CLSmith** extends similar ideas to OpenCL compilers and uses **differential testing** and **EMI testing** to handle the oracle problem.

**Differential testing** compares outputs across systems that should be equivalent. For example, the same CUDA program may be compiled with different compiler configurations. If the outputs differ, at least one configuration may have a bug.

**Equivalence Modulo Inputs (EMI)** is a compiler validation technique. The idea is to transform code regions that are not executed under a given input. Since those regions should not affect the program's behavior for that input, the original program and the transformed program should produce the same output.

**Metamorphic testing** is the broader idea: check relations among multiple executions of a program under test. EMI is one concrete form of metamorphic testing in compiler validation.

Test case reduction is also important. Once a failure-triggering program is found, reduction techniques shrink it so debugging becomes easier.

## CUDAsmith Design {#cudasmith-design}

![CUDAsmith pipeline](/images/blog/cuda-smith/cudasmith_pipeline.png)

The CUDAsmith pipeline can be understood as:

1. The generator creates a pool of CUDA kernel functions.
2. Some kernels are fed into the EMI variant generator to produce equivalent variants.
3. The kernels are merged with host code to produce mixed host/device CUDA programs.
4. The generated CUDA code is compiled and executed under different compiler configurations.
5. The fuzzing logs are analyzed to report failures.

The central idea is:

```text
generate valid CUDA kernels
   ->
construct equivalent variants
   ->
compile/run under different settings
   ->
compare behavior
   ->
report inconsistencies
```

## How to Read This Paper {#how-to-read-this-paper}

The most important concept is the testing pipeline:

```text
input generation
   ->
execute the system under test
   ->
decide whether the result is correct
   ->
reduce failing test cases
   ->
confirm real bugs
```

The hardest step is usually deciding whether the result is correct. Differential testing and EMI are both ways to construct an oracle when there is no simple ground truth.

A useful reading path is:

1. First understand basic fuzzing concepts: input generation, mutation, coverage, oracle, crash, and wrong-code bug.
2. Then study compiler testing: compiler crash, miscompilation, performance bug, and why random generators must avoid undefined behavior.
3. Then learn the oracle techniques: differential testing, metamorphic testing, and EMI.
4. Finally return to CUDAsmith and ask four questions:
   - What programs does it generate?
   - How does it keep them valid?
   - Which outputs are expected to be equivalent?
   - How does it reduce false positives?

## Takeaways {#takeaways}

CUDAsmith is not just "random CUDA program generation." It is a complete compiler testing workflow.

The contribution is the adaptation of compiler fuzzing ideas to CUDA:

- generate CUDA kernels with GPU-specific features;
- keep generated programs deterministic and valid;
- use differential testing and EMI to solve the oracle problem;
- report mismatches as potential compiler bugs.
