---
layout: post
title: Optimizing the mem* functions in C
date: 2023-10-06 21:34 -0500
category: [C, Assembly, NASM, Low Level, Optimization]
tags: [low level, c, assembly, optimization]
---
# Optimizing the mem* functions in C

# Prelude

I was in my English III class which had a substitute teacher since my teacher had gone to a nearby university to give some sort of presentation. I decided to slack off and pulled out my laptop with one thing in mind: optimizing the mem* family of functions in C.

What do I mean by the mem* functions in C?

I'm referring to these 4 functions:

- `memcpy`
- `memset`
- `memmove`
- `memcmp`

I opened my laptop and started typing away, comparing my version of the `memset` function with glibc version, and boy was I humbled.

# Initial Findings

I created a test case comparing the 2 versions of the functions, the rule was:

- no optimization

That's pretty much it, so with this in mind, I set off and wrote my `memset` function and compared the output to glibc's output.

Here is the test for both of the methods:

```c
// my version of memset
typedef unsigned long size_t;
void* my_memset(void* dest, int c, size_t n) {
    unsigned char* d = (unsigned char*)dest;
    for (; n; n--, d++) {
        *d = (unsigned char)c;
    }
    return dest;
}
int main() {
    char val[4096];
    for (size_t i = 0; i < 10000000; i++) {
        my_memset(val, 0, 4096);
    }
}
```

glibc's test:

```c
// glibc version
#include <string.h>
int main() {
    char val[4096];
    for (size_t i = 0; i < 10000000; i++) {
        memset(val, 0, 4096);
    }
}
```

It won't be that slow, right? WRONG.

![**GLIBC**](/v1696645376/Untitled_ebztwb.png)

**GLIBC**

![**My `memset`**](/v1696645376/Untitled_1_xlryer.png)

**My `memset`**

You see how much slower mine is? But why? 

### Note

I understand that it says `*_memcpy`, I'm stupid sometimes.

# Brainstorming

Without spoiling the fun, I try to think of a solution. The first thing that comes to mind is, what if I copy 8 bytes at a time whenever I can, then let the remaining be copied byte by byte?

I initially thought this wasn't gonna do much, but I was wrong.

Changing my code to this:

```c
typedef unsigned long size_t;
void* my_memset(void* dest, int c, size_t n) {

    unsigned long* long_d = dest;
    if (n >= 8) {
        for (; n; n -= 8, long_d++) {
            *long_d = (unsigned long)c;
        }
    }

    unsigned char* d = (unsigned char*)long_d;
    for (; n; n--, d++) {
        *d = (unsigned char)c;
    }
    return dest;
}
int main() {
    char val[4096];
    for (size_t i = 0; i < 10000000; i++) {
        my_memset(val, 0, 4096);
    }
}
```

And once I compiled and ran it, I was pleasantly surprised:

![*Second test with my `memset`**](/v1696645376/Untitled_2_rjibjl.png)

*Second test with my `memset`**

Around a 10 times improvement, I was expecting around maybe a 10 second improvement, not a 10x improvement!

However, my brain now stopped working and I decided to look for more simple optimized mem* functions.

I looked over to Limine, since I had seen their insanely optimized mem* functions, which were written in Intel Assembly.

The link to the source code is [here](https://github.com/limine-bootloader/limine/blob/v5.x-branch/common/lib/mem.asm_x86_64).

### Note

All source code belongs to their respective copyright holders.

This blog may include parts of source code from the aforementioned link, however, I have no purpose of defamation, scrutiny, or any ill-intent. I chose Limine as a reference since I felt it was the best source.

# Writing the optimized version

I knew about Limine's optimized `mem*` functions, so I looked there for something simple and fast. Using Limine as a reference and reading the Intel SDM, I managed to write a fast `memset` function. Take a look for yourself:

![Untitled](/v1696645376/Untitled_3_yl7pmz.png)

It's as fast as GLIBC in my tests, even being a little fast occasionally.

The source code is based on Limine's, however I'll link the [repository](https://github.com/xyve7/fast_mem).

I'll give a sneak peek on the `memset` function and what it does.

```nasm
; rax memset(rdi, rsi, rdx);
; void* memset(void* dest, int val, size_t count);
global memset
memset:
    ; move rsi into rax (the second parameter into rax so stosb can copy it)
    mov rax, rsi
    ; move rdx into rcx (the count so that rep can use it as a count)
    mov rcx, rdx
    ; save the rdi register which has the destination address, which we will mov into rax later
    mov r8, rdi
    ; repeat the fill
    rep stosb
    ; restore the original address of rdi and return it
    mov rax, r8
    ret
```

I'm not the greatest with assembly, so I tend to comment a lot of my code and have a representation of the C method signature above which helps me correlate them with the System V calling convention.

The major line here to look at is the instruction `rep stosb`.

The Intel SDM explains `rep` to repeat the instruction given, which is `stosb` in this case.

![Untitled](/v1696645376/Untitled_4_trzw75.png)

This is why I copied `count` which was in `rdx` into `rcx`, however this doesn't explain what `stosb`, all it explains is that it continues until `count` or `rcx` in this case is `0`.

`stosb` does this:

![Untitled](/v1696645376/Untitled_5_miq52g.png)

It uses `rax` as the value to fill:

![Untitled](/v1696645376/Untitled_6_lfherm.png)

So, you might be asking, why use `stosb` and not a loop and copy the data like this:

```nasm
.loop:
    cmp rcx, 0
    je .done

    mov BYTE [rdi], rax

    inc rdi
    dec rcx
    jmp .loop
.done:
    ret
```

Well, it's because it's faster, but a better explanation is as [follows](https://stackoverflow.com/questions/33480999/how-can-the-rep-stosb-instruction-execute-faster-than-the-equivalent-loop).

That is a much better explanation than what I could provide. Since it's done by someone far more knowledgeable than I am

# Finish

Thanks for reading as always. I learned a lot with this experience, and I hope this helps other people that read this.

I haven't gone over the rest of the `mem*` functions, however the principle is very similar. I didn't really go over the purposes of the functions, only the implementations in assembly.