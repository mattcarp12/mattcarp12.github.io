---
layout: post
title: "strcpy considered harmful"
date: 2020-09-04
categories: programming c
tags: C
---

The inspiration for this post comes from exercise 1.3 of David Hanson's [C Interfaces and Implementations](https://amzn.to/3385vTI).


Every budding C hacker has seen the following snippet:
```c
void strcpy(char *s, char *t) {
    while (*s++ = *t++)
        ;
}
```

There's a lot going on in these 4 lines. 

The first thing to notice is the expression between the parentheses of the while loop, known as the *control expression*, is an assignment and not a logical expression. 

This is fine because the assignment expression actually produces a return value, which is the value of the right operand (which the left operand takes on as a result of assignment).

In our case, at the end of a string a `\0` will be returned from the assignment expression. This evaluates to false (zero) and the while loop ends. Neat huh?

Okay great! Lets compile this baby!

```bash
$ gcc -c strcpy.c -Wall
```

Uh oh. The compiler isn't happy:

```bash
strcpy.c: In function ‘strcpy’:
strcpy.c:2:12: warning: suggest parentheses around assignment used as truth value [-Wparentheses]
    2 |     while (*s++ = *t++)
      |            ^
```

Because logical comparison operators are most often used in control expressions, the compiler gives a 'safety' warning to make sure you didn't really mean `==`.

A nicer way to write this would make the logical comparison **explicit**:

```c
void strcpy(char *s, char *t) {
    while ((*s++ = *t++) != '\0')
        ;
}
```

Boom! No more compiler warnings and no more head scratching by noobs reading your code.



