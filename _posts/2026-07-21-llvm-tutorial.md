---
title: "Notes for *My First Language Frontend with LLVM Tutorial*"
date: 2026-07-21
tags:
  - compiler
---

This post is my note for the LLVM Kaleidoscope tutorial. The tutorial builds a small language frontend step by step and uses LLVM to generate code.

The first chapter is about the lexer. It is the smallest component in the compiler pipeline, but it is also the first place where raw source code becomes structured information.

## Outline {#outline}

- [Why This Tutorial Matters](#why-this-tutorial-matters)
- [What Kaleidoscope Is](#what-kaleidoscope-is)
- [Where the Lexer Fits](#where-the-lexer-fits)
- [Token Definitions](#token-definitions)
- [How `gettok` Works](#how-gettok-works)
- [Takeaways](#takeaways)

## Why This Tutorial Matters {#why-this-tutorial-matters}

LLVM is a compiler infrastructure. It provides reusable building blocks for compiler construction, including IR, optimization passes, code generation, and JIT support.

LLVM itself does not know how to parse C, C++, or any other source language. For C/C++, that job is done by Clang. Clang is the frontend: it understands C/C++ syntax and translates source code into LLVM IR. LLVM then optimizes the IR and generates machine code.

The Kaleidoscope tutorial is useful because it shows this pipeline on a small language that we can fully understand.

The full tutorial covers:

1. lexer, building a parser for a language.
2. parser and AST.
3. code generation to LLVM IR.
4. adding JIT and optimizer support.
5. extending the language: control flow
6. extending the language: user-defined operators.
7. compiling to object files.
8. debug information
9. conclusion

By the end of the tutorial, we’ll have written a bit less than 1000 lines of non-comment, non-blank code. With this small amount of code, we’ll have built a nice little compiler for a non-trivial language, including a hand-written lexer, parser, AST, and code generation support, both static and JIT.

## What Kaleidoscope Is {#what-kaleidoscope-is}

Kaleidoscope is a small *procedural* language. It supports function definitions, conditionals, math expressions, and later extensions such as loops, user-defined operators, JIT compilation, and debug information.

The language is intentionally simple. Its only datatype is a 64-bit floating point number, equivalent to `double` in C.

This simplicity is useful. It lets the tutorial focus on the compiler pipeline instead of language complexity.

## Where the Lexer Fits {#where-the-lexer-fits}

When implementing a language, the first task is to process source text and recognize what it says.

The lexer, also called a scanner, breaks raw input text into **tokens**. A token is a small structured unit such as:

- keyword: `def`, `extern`
- identifier: `foo`, `bar`
- number: `1.0`, `42`
- operator or punctuation: `+`, `(`, `)`

The parser will later consume these tokens and build an AST. So the lexer is the bridge between plain text and structured syntax.

## Token Definitions {#token-definitions}

The tutorial defines a small set of token codes:

```c
// The lexer returns tokens [0-255] if it is an unknown character, otherwise one
// of these for known things.
enum Token {
  tok_eof = -1,

  // commands
  tok_def = -2,
  tok_extern = -3,

  // primary
  tok_identifier = -4,
  tok_number = -5,
};

static std::string IdentifierStr; // Filled in if tok_identifier
static double NumVal;             // Filled in if tok_number

```

There are two important details.

First, known token types use negative enum values such as `tok_def`, `tok_identifier`, and `tok_number`.

Second, unknown single-character tokens such as `+` are returned as their ASCII values. This keeps the lexer simple.

When the current token is an identifier, `IdentifierStr` stores its name. When the current token is a numeric literal, `NumVal` stores its value.

![Token definitions](/images/blog/llvm-tutorial/token-def.png)

## How `gettok` Works {#how-gettok-works}

The actual lexer is a single function named `gettok`. Each call returns the next token from standard input.

The full lexer code:

```c
#include <cctype>
#include <cstdio>
#include <cstdlib>
#include <map>
#include <memory>
#include <string>
#include <utility>
#include <vector>

//===----------------------------------------------------------------------===//
// Lexer
//===----------------------------------------------------------------------===//

// The lexer returns tokens [0-255] if it is an unknown character, otherwise one
// of these for known things.
enum Token {
  tok_eof = -1,

  // commands
  tok_def = -2,
  tok_extern = -3,

  // primary
  tok_identifier = -4,
  tok_number = -5
};

static std::string IdentifierStr; // Filled in if tok_identifier
static double NumVal;             // Filled in if tok_number

/// gettok - Return the next token from standard input.
static int gettok() {
  static int LastChar = ' ';

  // Skip any whitespace.
  while (isspace(LastChar))
    LastChar = getchar();

  if (isalpha(LastChar)) { // identifier: [a-zA-Z][a-zA-Z0-9]*
    IdentifierStr = LastChar;
    while (isalnum((LastChar = getchar())))
      IdentifierStr += LastChar;

    if (IdentifierStr == "def")
      return tok_def;
    if (IdentifierStr == "extern")
      return tok_extern;
    return tok_identifier;
  }

  if (isdigit(LastChar) || LastChar == '.') { // Number: [0-9.]+
    std::string NumStr;
    do {
      NumStr += LastChar;
      LastChar = getchar();
    } while (isdigit(LastChar) || LastChar == '.');

    NumVal = strtod(NumStr.c_str(), nullptr);
    return tok_number;
  }

  if (LastChar == '#') {
    // Comment until end of line.
    do
      LastChar = getchar();
    while (LastChar != EOF && LastChar != '\n' && LastChar != '\r');

    if (LastChar != EOF)
      return gettok();
  }

  // Check for end of file.  Don't eat the EOF.
  if (LastChar == EOF)
    return tok_eof;

  // Otherwise, just return the character as its ascii value.
  int ThisChar = LastChar;
  LastChar = getchar();
  return ThisChar;
}
```

The function handles four main cases.

First, it skips whitespace.

![gettok implementation](/images/blog/llvm-tutorial/gettok.png)

Second, it recognizes identifiers and keywords. If the identifier is `def`, it returns `tok_def`. If it is `extern`, it returns `tok_extern`. Otherwise, it returns `tok_identifier`.

Third, it recognizes numbers and stores the parsed value in `NumVal`.

![Number token handling](/images/blog/llvm-tutorial/number_token.png)

Fourth, it skips comments that begin with `#`.

![Comment handling](/images/blog/llvm-tutorial/comment.png)

If the character is none of the above, the lexer returns the character's ASCII value. This is how simple operators and punctuation are handled.

## Takeaways {#takeaways}

The lexer is small, but it shows the first key compiler idea:

```text
raw source text
   ->
tokens
   ->
parser
   ->
AST
```

For Kaleidoscope, the lexer only needs to understand identifiers, numbers, comments, keywords, and single-character operators. This is enough to support the next stage: building a parser and AST.
