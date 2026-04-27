---
layout: post
title:  "The GOTO That Refused to Die"
date:   2026-4-27 10:46:39 -0700
# categories: jekyll update
---

Goto has been declared dead many times. Dijkstra wrote his famous letter in 1968, structured programming took over, and most people today assume the debate ended decades ago. The real story is more interesting. Removing goto, or trying to, has produced some of the more illuminating successes and failures in language design.

## Dijkstra's Letter

In 1968, Edsger Dijkstra wrote a short note to *Communications of the ACM* titled "A Case Against the GO TO Statement." The editor, Niklaus Wirth, retitled it to "Go To Statement Considered Harmful," and the new title outlived most of the languages it critiqued.

The actual argument is more subtle than the title suggests. Dijkstra was not claiming goto is morally evil. He was making a claim about human cognition: our ability to reason about a program's state depends on a small, understandable relationship between the text of the program and its execution. With `if`, `while`, and `for`, you can look at any point in the code and roughly know where you are. With `goto`, you cannot. A label is a portal that can be entered from anywhere.

The reaction was immediate. Frank Rubin wrote "'GOTO Considered Harmful' Considered Harmful" in 1987. Someone wrote a third-level rebuttal after that. The argument never quite ended.

## Pascal: Removal by Replacement

Pascal, designed by Niklaus Wirth, was the first mainstream language to take the position that goto was unnecessary. Pascal had `goto`, technically, but the culture around the language treated using it as a confession of failure. Programmers taught with Pascal in the 70s and 80s learned not to reach for it.

This worked. Pascal programs were on average more readable than the Fortran and assembly-flavored C that came before. The structured programming vocabulary it championed (blocks, scopes, nested conditionals) is what every modern language now takes for granted.

You can sometimes remove a feature not by deleting it but by building such good alternatives that nobody wants it.

## Java: The Reserved Word That Does Nothing

Java does not have goto. It does have `goto` listed as a reserved word, which means you cannot use it as an identifier. Write `int goto = 5;` and the compiler rejects it.

James Gosling reserved the keyword so future Java versions could potentially add goto, and so programmers migrating from C could not accidentally use `goto` as a variable name. Java never added it. The reservation is a small monument to a feature that exists only as an absence.

Java programmers eventually wanted goto anyway and invented workarounds: labeled breaks, labeled continues, and the "break out of nested loops" pattern.

```java
outer:
for (int i = 0; i < n; i++) {
    for (int j = 0; j < m; j++) {
        if (found) break outer;
    }
}
```

This is structured goto with a costume on.

## PHP: Late Addition

PHP did not have goto for its first dozen years. Then in 2009, PHP 5.3 added it because people kept asking for it. PHP's goto is restricted, you cannot jump into a loop or switch statement, only out of or within them. Almost no well-written PHP code uses it.

## Go: Tasteful Inclusion

Go, released in 2009, includes `goto`. This surprised many people, because Go was designed by Ken Thompson, Rob Pike, and Robert Griesemer, who all knew the literature.

Their reasoning was pragmatic. In hand-written state machines, in generated code, and in certain error-handling patterns, goto is the cleanest tool. Go's goto has constraints (you cannot jump over variable declarations, you cannot jump into a block) that eliminate most of the worst scenarios. In practice, Go programmers almost never write goto, but when they need it, it is there.

## The Apple goto fail Bug

In 2014, a bug was found in Apple's SSL/TLS implementation. The relevant code is roughly:

```c
if ((err = SSLHashSHA1.update(&hashCtx, &signedParams)) != 0)
    goto fail;
    goto fail;
if ((err = SSLHashSHA1.final(&hashCtx, &hashOut)) != 0)
    goto fail;
```

There are two `goto fail;` lines after the first `if`. The second one is unconditional. It always jumps to the cleanup code, skipping the actual signature verification. For years, Apple devices accepted invalid SSL certificates without complaint.

This bug became the poster child for "goto considered harmful" all over again. The honest reading is more nuanced. The bug was not really about goto. It was about the lack of mandatory braces after `if`. You could write the same bug with early `return` followed by another unconditional statement. The actual cause is C's syntax letting an unbraced block look like a braced one. Goto was just the witness.

## Everything Is Goto in Assembly

At the machine level, everything is goto. Every `if`, every `while`, every function call, every exception, every `await` compiles down to conditional and unconditional jumps. The CPU does not know what a block is. It does not know what a function is. It has a program counter and a set of instructions that either increment it or overwrite it.

Here is a simple if/else in x86-64:

```nasm
    cmp  eax, 10        ; compare eax to 10
    jle  .else_branch   ; jump if less-or-equal
    mov  ebx, 1         ; then-branch body
    jmp  .end
.else_branch:
    mov  ebx, 2         ; else-branch body
.end:
```

Those `jle` and `jmp` instructions are pure unrestricted goto. The labels `.else_branch` and `.end` are exactly the kind of portals Dijkstra warned about. There is no scope, no block, no structure, just addresses and jumps.

A while loop is the same thing with the jumps arranged slightly differently.

```nasm
.loop_top:
    cmp  eax, 100
    jge  .loop_end
    ; loop body here
    inc  eax
    jmp  .loop_top
.loop_end:
```

Two gotos: one conditional forward jump to exit, one unconditional backward jump to continue. That is all a while is. C, Haskell, and Python all compile down to this same shape, just via more or less scenic routes.

x86-64 alone has `jmp`, `je`, `jne`, `jg`, `jge`, `jl`, `jle`, `ja`, `jae`, `jb`, `jbe`, `jo`, `jno`, `js`, `jns`, `jp`, `jnp`, `jcxz`, and more. Each is a specialized goto. The structured programming we enjoy in high-level languages is an illusion running on top of a machine that only speaks in jumps.

Once you see assembly, Dijkstra's letter reads differently. He was not saying jumps are bad. He knew the CPU runs on jumps. He was saying that the source code you write should not look like assembly. The compiler's job is to translate structured intent into unstructured jumps. When you write `goto` in a high-level language, you skip that translation and hand the CPU's view of the world directly back to the human reader.

## Goto in the Linux Kernel

The Linux kernel still has thousands of `goto` statements in its C code. Kernel code sits close enough to the metal that pretending goto does not exist would be a kind of lie. The classic pattern, `goto cleanup` for error handling, is universal in systems C.

```c
int do_stuff(void) {
    if (!acquire_lock())    goto err_lock;
    if (!allocate_buffer()) goto err_buffer;
    if (!start_device())    goto err_device;
    return 0;

err_device:
    free_buffer();
err_buffer:
    release_lock();
err_lock:
    return -1;
}
```

This is goto used well. Each label corresponds to a specific recovery point. The jumps are all forward, all into a single cleanup cascade. It is a hand-rolled `try`/`finally`, written this way because C does not have one. It also maps cleanly onto the assembly that will execute, so little is lost in translation.

## Renamed, Not Removed

We did not really get rid of goto. We renamed it.

Exception handling is a goto that unwinds the stack. Early `return` is a limited goto. `break` and `continue` are gotos. Coroutines, generators, and async/await are structured non-local control flow.

Rust's `?` operator is goto for error propagation. Python's `yield` is goto for iteration. JavaScript's `async` is goto for I/O waiting. None of these are called goto, but the underlying mechanism (jumping somewhere that is not the next instruction) is the same. What changed is that each of these jumps has a defined source, a defined destination, and a type system that keeps the whole thing sane.

Dijkstra was not wrong. Unrestricted goto does make programs harder to reason about. But the story is not really "goto is evil." It is "non-local control flow is powerful, and we need to categorize and constrain it to use it safely."

Every language that tried to remove goto entirely either failed (Java, with its labeled breaks), added it back (PHP), or succeeded by inventing replacements so good nobody missed it (Pascal). Every language that kept it in some form (C, Go, Perl) relies on cultural taboo to keep it sequestered. Every modern feature that programmers love (exceptions, async, generators, early returns) is a constrained descendant of the same mechanism Dijkstra warned us about.

The goto did not die. It got several new jobs under assumed names.

**References**
- Dijkstra, E. W. (March 1968). "Go To Statement Considered Harmful." *Communications of the ACM*.
- Rubin, F. (March 1987). "'GOTO Considered Harmful' Considered Harmful." *Communications of the ACM*.
- [Apple goto fail; bug analysis (ImperialViolet)](https://www.imperialviolet.org/2014/02/22/applebug.html)
- [Linux kernel goto usage discussion (LWN)](https://lwn.net/Articles/16721/)
