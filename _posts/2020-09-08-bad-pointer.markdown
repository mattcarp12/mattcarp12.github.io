---
layout: post
title: Bad Pointer! No Biscut!
categories: programming c
tags: C pointers 
---

The inspiration for this post comes from exercise 2.4 of David Hanson's [C Interfaces and Implementations](https://amzn.to/3385vTI).

## Uninitialized Pointers

Say you want to want to create an object using dynamic memory allocation. Here is one way to do this:
```c
Object *obj;
...do some stuff...
obj = (Object*) malloc(sizeof Object);
```
During the "do some stuff" period of time, the pointer `obj` is uninitialized. An uninitialized pointer will contain a garbage memory address, which is why you may get a segmentation fault or other error if you try to dereference it.

Preventing errors resulting from uninitialized pointers is simple: always initialize pointers to `NULL`:
```c
Object *obj = NULL;
```

## Dangling Pointers

After you free dynamically allocated memory, your pointer to that memory location is known as a *dangling pointer*. Dereferencing a dangling pointer is undefined behavior. Dangling pointers can also arise when referencing a pointer to automatically allocated memory on the stack that has gone out of scope.

The thing with dangling pointers is the pointer still exists after the memory has been freed, so your program may appear to work normally. Consider the following example:

```c
#include <stdlib.h>
#include <stdio.h>
int main() {
    int *p = malloc(sizeof(int));
    *p = 45;
    printf("p = %d\n", *p);
    free(p);
    *p = 46;
    printf("p = %d\n", *p);
}
```

This program runs fine because the memory pointed to by `p` has not been overwritten by other data. However trouble arises when your dangling pointer points to newly allocated memory:

```c
#include <stdlib.h>
#include <stdio.h>
int main() {
    int *p = malloc(sizeof(int));
    *p = 45;
    printf("p = %d\n", *p);
    free(p);
    int *q = malloc(sizeof(int));
    *p = 46;
    printf("q = %d\n", *q);
}
```
Here we dereferenced `p` but ir actually modified the value pointed to by `q`. In larger programs this can lead to seriously nasty bugs.

The solution is simple. Always set your pointers to `NULL` after freeing them!

```c
#define FREE(x)\
        do {\
            free(x);\
            x = NULL;\
        } while (0)
```

If you add this macro to the second example above, you will get a segfault from dereferencing a `NULL` pointer, which is preferable to unknowingly overwriting other data.


## Unaligned Pointers

An object in memory may have a alignment requirement, e.g. a 4-byte integer is required to be stored at a memory location that is divisible by 4. Another example may be an 8-byte float being stored at a memory location that is divisible by 8. In this example, the float is more *strictly aligned* than the integer.

For structs in C, the object must be aligned to the **most strictly-aligned** object that is stored in the struct. An issue may arise if you have a function that accepts `void` pointer as an argument. To prevent data loss, never cast a pointer to a type that is more structly aligned.

