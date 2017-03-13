---
layout: default
---

# Calling a C Function without Prototype

## Implicit Function Declaration in C

It should be considered an error. But C is an ancient language, so it's only a warning.
Compiling with `-Werror` (gcc) fixes this problem.

When C doesn't find a declaration, it assumes this implicit declaration: int f();, which means the function can receive whatever you give it, and returns an integer. If this happens to be close enough (and in case of printf, it is), then things can work. In some cases (e.g. the function actually returns a pointer, and pointers are larger than ints), it may cause real trouble.

Note that this was fixed in newer C standards (C99, C11). In these standards, this is an error. However, gcc doesn't implement these standards by default, so you still get the warning.

## Are prototypes required for all functions in C89, C90 or C99?

It depends on what you mean by 'truly standards compliant'. However, the short answer is "it is a good idea to ensure that all functions have a prototype in scope before being used".

A more qualified answer notes that if the function accepts variable arguments (notably the printf() family of functions), then a prototype must be in scope to be strictly standards compliant. This is true of C89 (from ANSI) and C90 (from ISO; the same as C89 except for the section numbering). Other than 'varargs' functions, though, functions which return an int do not have to be declared, and functions that return something other than an int do need a declaration that shows the return type but do not need the prototype for the argument list.

Note, however, that if the function takes arguments that are subject to 'normal promotions' in the absence of prototypes (for example, a function that takes a char or short - both of which are converted to int; more seriously, perhaps, a function that takes a float instead of a double), then a prototype is needed. The standard was lax about this to allow old C code to compile under standard conformant compilers; older code was not written to worry about ensuring that functions were declared before use - and by definition, older code did not use prototypes since they did not become available in C until there was a standard.

C99 disallows 'implicit int'...that means both oddball cases like 'static a;' (an int by default) and also implicit function declarations. These are mentioned (along with about 50 other major changes) in the forward to ISO/IEC 9899:1999, which compares that standard to the previous versions:

* remove implicit int
* remove implicit function declaration

In ISO/IEC 9899:1990, §6.3.2.2 Function calls stated:

If the expression that precedes the parenthesized argument list in a function
call consists solely of an identifier. and if no declaration is visible for
this identifier, the identifier is implicitly declared exactly as if, in the
innermost block containing the function call, the declaration:
`extern int identifier();` appeared (^38).

(^38) That is, an identifier with block scope declared to have external linkage with type function without parameter information and returning an int. If in fact it is not defined as having type “function returning int,” the behavior is undefined.

This paragraph is missing in the 1999 standard. I've not (yet) tracked the change in verbiage that allows static a; in C90 and disallows it (requiring static int a;) in C99.

Note that if a function is static, it may be defined before it is used, and need not be preceded by a declaration. GCC can be persuaded to witter if a non-static function is defined without a declaration preceding it (-Wmissing-prototypes).

## GCC Warning Option: `-Wall` and `-Wimplicit-function-declaration`

The option `-Wno-implicit-function-declaration` is harmful and implicit function desclaration 
should be considered as an error. But C is an ancient language, so it's only a warning. 
Compiling with `-Werror` fixes this problem.

The option `-Wimplicit-function-declaration` is enabled by `-Wall`.
It can catch potential bugs which results from "Implicit function declaration" such as following demo.

## Demo Code

```
@file: utils.c
// gcc -shared -fPIC -o libutil.so
char util_one(int input)
{
	return input == 0;
}
```

```
@file: sys.c
// gcc -shared -fPIC -o libsys.so
// without the declaration of function `util_one()`
char sys_one(int input)
{
	return util_one()
}
```

We can see the problem from the disassembly of `sys_one()` by `objdump -d libsys.so`.

The bug will be looks like:

```
callq <util_one>
test %eax, %eax
```

It should be `test %al, %al`, but it's not due to the implicit function declaration.

## If a function is defined as `uint8_t f()`, why it runs well under the older GCC, such as gcc-4.1.2, gcc-3.4.4?

Well, the compiler behavior on implicit function declaration has changed a bit.
In this case, the return value `uint8_t` or `BOOLEAN` will be extened to `unsigned int` just before the return.


[back](../)
