---
title: "Paper Reading: Automated Testing of Graphics Shader Compilers"
date: 2026-07-21
tags:
  - fuzzing
---

This note is about a central problem in compiler testing: how do we know whether a compiler's output is correct?

For ordinary software, a test often has a clear expected output. Compiler testing is harder. A compiler takes a program as input and produces another program or binary as output. If the generated output behaves incorrectly, we need a way to detect that mismatch. This is known as the **test oracle problem**.

## Outline {#outline}

- [The Oracle Problem](#the-oracle-problem)
- [Differential Testing](#differential-testing)
- [Metamorphic Testing](#metamorphic-testing)
- [Why This Matters for Shader Compilers](#why-this-matters-for-shader-compilers)
- [Takeaways](#takeaways)

## The Oracle Problem {#the-oracle-problem}

The main challenge is the lack of an oracle that can classify an output as correct or incorrect.

In shader compiler testing, this is especially difficult because the output is not just a number or a simple text string. The output may be rendered graphics, GPU behavior, or low-level code whose correctness depends on the semantics of the original shader.

So the core question becomes:

> If we generate a shader program and compile it, how do we know whether the compiler handled it correctly?

The paper discusses two broad ways to address this problem.

## Differential Testing {#differential-testing}

Differential testing compares multiple systems that should behave equivalently.

For compiler testing, the idea is:

```text
same input program
   ->
compiler A / compiler B / compiler C
   ->
compare outputs
```

If the outputs disagree, at least one compiler may be wrong.

The challenge is that disagreement does not immediately tell us which compiler is wrong. Another challenge is that we need a rich set of benchmark programs that can run across all compilers being compared.

## Metamorphic Testing {#metamorphic-testing}

Metamorphic testing generates families of programs that should produce equivalent results.

Instead of asking "what is the correct output for this one program?", it asks:

> If these two programs should be semantically equivalent, do they still produce the same result after compilation?

This avoids the need for a traditional oracle. If two equivalent shaders produce different rendered results, that mismatch may indicate a compiler bug.

## Why This Matters for Shader Compilers {#why-this-matters-for-shader-compilers}

Graphics shader compilers are hard to test because shader programs run on GPUs and interact with compiler optimizations, graphics APIs, and hardware-specific behavior.

Metamorphic testing is useful here because it gives the tester a way to generate related shader programs whose behavior should remain consistent. This makes it possible to detect wrong-code bugs even when we do not know the exact expected output in advance.

## Takeaways {#takeaways}

The key idea is that compiler testing often depends on constructing an oracle indirectly.

- Differential testing compares multiple compilers or configurations.
- Metamorphic testing compares related programs that should preserve behavior.
- For shader compilers, these techniques help expose bugs that would be hard to detect with ordinary example-based tests.
