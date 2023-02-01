---
layout: post
title: "Write a Compiler in Ocaml - Part 1: Introduction"
date: 2023-01-29
categories: compiler
tags: compiler
usemathjax: false
---

This article is the fist in the series "Write a Compiler in Ocaml". This article will be an introduction, including discussion of initial decisions that need to be made before undertaking a new comiler project.

## Which host language?
The host language is the language the compiler is written in. I have chosen to write my compiler on Ocaml for several reasons:
- I have never used Ocaml before and I want to learn more about functional programming
- Tail recursion optimization, ADTs, and pattern matching go well with writing a compiler
- Ocamlex and Menhir are lexer and parser packages 

## Which target language?
A compiler is a program that takes a string and returns another string. The first string is the program written in our programming language. The second string is the __target language__. Generally when we think of a compiler like gcc, we think of compiling to assembly language, then running that through a separate assembler and linker to produce the final executable file.

![compiler_pipeline](/assets/compiler_pipeline_simple.jpg){: width="700" }

However, the decision of target language is not that simple. Remember that a compiler is a pipeline where our program is transformed at each step. It is common for a compiler to first transform the source language into an intermediate representation and then finally generate assembly at the end in the Code Gen phase. Examples of the intermediate representation is LLVM, JVM Bytecode, and CLR Bytecode.

For this project, I've decided to write all the pieces of the compiler, so I won't be targeting any intermediate representation, other than my own.

So my target language will be assembly. But which assembly? My two main options are X86 and ARM. Both are widely used and would be great for my compiler, but I can only choose one so I've chosen x86.

Now things get interesting. Which version of x86? There is 32 bit and 64 bit, and [they are not the same](https://stackoverflow.com/questions/29538742/difference-between-32bit-and-64bit-assembler-programs). However, since 64 bit processors started becoming dominant when I was a teenager, I think that is the clear choice.

## Which development methodology?
We can think of two ways to develop a new compiler: write the entirety of each phase of the compiler in sequence, starting with the frontend, and finishing with the code generation. This is the methodology used in my very first compiler that I wrote, which was following the excellent Stanford course [CS143](https://www.edx.org/course/compilers). 

The other methodology is to construct the compiler incrementally, i.e. start out with a very basic but complete compiler, then add new features incrementally. There are many resources on the internet that use this methodology: [1](https://github.com/namin/inc), [2](https://www.wilfred.me.uk/blog/2014/08/27/baby-steps-to-a-c-compiler/), [3](https://norasandler.com/2017/11/29/Write-a-Compiler.html). This is the methodology I will use in this project.

Here is the link to the code: [https://github.com/mattcarp12/maml](https://github.com/mattcarp12/maml).