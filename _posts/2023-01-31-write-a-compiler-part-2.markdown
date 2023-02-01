---
layout: post
title: "Write a Compiler in Ocaml - Part 2: First Program"
date: 2023-01-31
categories: compiler
tags: compiler
usemathjax: false
---

In this second post in the series, we will create the first implementation of our compiler, which will produce an actual binary program. The language will be trivial, but will serve as the foundation for later iterations (remember we're using the incremental approach to compiler construction).

Here is the branch for this post: [https://github.com/mattcarp12/maml/tree/constants](https://github.com/mattcarp12/maml/tree/constants).

<!-- // Make a compiler diagram here -->

## Initial Language

The first program we're going to compile in our new language looks like this:

```js
let main = () { 
    3
}
```

Our language consists of a single function, `main`, which has no arguments, and who's body is a single integer.

When we run our compiler, it will read the file that has our program text, and will create a new file with the generated x86 assembly.

## Lexing and Parsing

The first step in the compiler pipeline is lexing. The job of the lexer is to return a token stream which is then read in by the parser. We're using the (ocamlex)[https://dev.realworldocaml.org/parsing-with-ocamllex-and-menhir.html] module to generate a lexer for us. All we need to do is write a `.mll` file describing the lexemes in our language.

```ocaml
(* lexer.mll *)

let digit = ['0'-'9']
let alpha = ['a'-'z' 'A'-'Z']
let int = '-'? digit+  
let whitespace = [' ' '\t']+
let newline = '\r' | '\n' | "\r\n"

rule token = parse
  | whitespace {token lexbuf}
  | newline {next_line lexbuf; token lexbuf}
  | int { INT (int_of_string(lexeme lexbuf)) }
  | "=" {EQUALS}
  | "{" {LEFT_BRACE}
  | "}" {RIGHT_BRACE}
  | "(" {LEFT_PAREN}
  | ")" {RIGHT_PAREN}
  | "let" {LET}
  | (alpha) (alpha|digit|'_')* { ID (lexeme lexbuf) }
  | _ { raise (SyntaxError ("Unexpected char: " ^ Lexing.lexeme lexbuf)) }
  | eof {EOF}
```

Pretty cool, right?

For the parsing, we will also use an ocaml module to generate the parser code. (Menhir)[https://dev.realworldocaml.org/parsing-with-ocamllex-and-menhir.html] is the defacto parser generator for ocaml.
Remember, the parser reads in the lexemes, and returns an Abstract Syntax Tree (AST). This is the internal representation of the source code.
So before showing you the parser, I'll show you the AST first:

```ocaml
(* syntax.ml *)
type expr = Int of int
type func = Function of (string * expr)
type program = Program of func
```

This is enough to encode our simple language.

Now, here is the `.mly` file that is used to generate our parser:

```ocaml
(* parser.mly *)

%{
open Syntax
%}
%token <int> INT
%token <string> ID
%token EQUALS
%token LEFT_BRACE
%token RIGHT_BRACE
%token LEFT_PAREN
%token RIGHT_PAREN
%token LET
%token EOF

%start <Syntax.program> program
%%

(* Menhir expresses grammars as context-free grammars *)

program :
    | f = func; EOF { Program f }
    ;

func :
    | LET id=ID EQUALS LEFT_PAREN RIGHT_PAREN LEFT_BRACE e=expr RIGHT_BRACE { Function(id,e) }

expr :
    | i = INT { Int i }
    ;
```

## Code generation

Here is all the code generation for this part:

```ocaml
let print_header out =
  Printf.fprintf out "global _start\n";
  Printf.fprintf out "section .text\n";
  ()

let print_start out =
  Printf.fprintf out "\n";
  Printf.fprintf out "_start:\n";
  Printf.fprintf out "\t mov rax, 1  ; system call for write\n";
  Printf.fprintf out "\t mov rdi, 1  ; file handle 1 is stdout\n";
  Printf.fprintf out "\t mov rsi, message  ; address of string to output\n";
  Printf.fprintf out "\t mov rdx, message_len  ; number of bytes\n";
  Printf.fprintf out "\t syscall ; invoke operating system \n";
  Printf.fprintf out "\t mov rax, 60 ; system call for exit \n";
  Printf.fprintf out "\t xor rdi,rdi ; exit code 0 \n";
  Printf.fprintf out "\t syscall ; invoke the operating system\n";
  ()

let print_data_section out =
  Printf.fprintf out "\n";
  Printf.fprintf out "section .data\n"

let print_data out i =
  Printf.fprintf out "message: db \"%d\",10\n" i;
  Printf.fprintf out "message_len: equ $ - message"

let codegen program =
  let out = open_out "test/test_out.asm" in

  print_header out;
  print_start out;
  print_data_section out;

  match program with
  | Syntax.Program (Function (_, Int i)) ->
      print_data out i;
      ()
```

When the compiler is run with the above program, it will produce the following assembly file:

```asm
global _start
section .text

_start:
  mov rax, 1  ; system call for write
  mov rdi, 1  ; file handle 1 is stdout
  mov rsi, message  ; address of string to output
  mov rdx, message_len  ; number of bytes
  syscall ; invoke operating system 
  mov rax, 60 ; system call for exit 
  xor rdi,rdi ; exit code 0 
  syscall ; invoke the operating system

section .data
message: db "3",10
message_len: equ $ - message
```

## Building and linking

Now that we have all the code written, let's run it!

Here are the commands to compile our first program.

```bash
dune exec maml # makes test_out.asm file
nasm -felf64 test_out.asm # makes test_out.o object file
ld test_out.o # makes a.out
./a.out # run our program
```
