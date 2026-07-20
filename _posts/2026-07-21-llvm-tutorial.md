---
title: "Notes for *My First Language Frontend with LLVM Tutorial*"
date: 2026-07-21
tags:
  - compiler
---

## Outline {#outline}

- [Preface](#preface)
- [Chapter 1: Kaleidoscope Introduction and the Lexer](#chapter-1-kaleidoscope-introduction-and-the-lexer)
- [The Lexer](#the-lexer)

## Preface {#preface}

This tutorial will get you up and running fast and show a concrete example of something that **uses LLVM to generate code.**

This tutorial introduces the simple “Kaleidoscope” language, building it iteratively over the course of several chapters, showing how it is built over time.

The tutorial outline:

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


> llvm 是一套用于构建 compiler 的基础设施和工具链，官方称其为 compiler infrastructure。llvm 本身不知道如何解析 c 或者其他语言。负责理解 C/C++ 语法的是 Clang。Clang 是 frontend，它把 C/C++ 代码转换为 LLVM IR；LLVM 再进行优化和机器代码生成。


## Chapter 1: Kaleidoscope Introduction and the Lexer {#chapter-1-kaleidoscope-introduction-and-the-lexer}

Kaleidoscope is a *procedural* language that allows you to define functions, use conditionals, math, etc.

Over the course of the tutorial, we’ll extend Kaleidoscope to support the if/then/else construct, a for loop, user defined operators, JIT compilation with a simple command line interface, debug info, etc.

The only datatype in Kaleidoscope is a 64-bit floating point type, also known as `double` in C.

---

### The Lexer {#the-lexer}


When it comes to implementing a language, the first thing needed is the ability to process a text file and recognize what it says. 

The traditional way to do this is to use a “lexer” (aka ‘scanner’) to break the input up into *“tokens”*. Each token returned by the lexer includes a *token code* and potentially some *metadata* (e.g. the numeric value of a number).

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

- Each token returned by our lexer will either be one of the **Token enum values** or it will be an ‘unknown’ character like ‘+’, which is returned as its ASCII value.
- If the current token is an identifier, the **IdentifierStr** global variable holds the name of the identifier. If the current token is a numeric literal (like 1.0), **NumVal** holds its value. 

---


The actual implementation of the lexer is a single function named *gettok*. The gettok function is called to return the next token from standard input.

完整的 lexer 代码：

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

注释版：

![Token definitions](/images/blog/llvm-tutorial/token-def.png)

![gettok implementation](/images/blog/llvm-tutorial/gettok.png)

![Number token handling](/images/blog/llvm-tutorial/number_token.png)

![Comment handling](/images/blog/llvm-tutorial/comment.png)
