---
layout: post
title: Introduction to Paging
date: 2023-10-03 17:20 -0500
category: [Operating Systems, Paging]
tags: [osdev, paging, low level, c, assembly]
---
# Introduction

# Disclaimer

I for one am also attempting to understand how paging works, so the subsequent documents are all **my understanding of paging.** If they are fundamentally incorrect, please inform me.

All of the code samples will be in C with some Assembly here and there. so prior knowledge of at least C is **required.**

This page will also assume you are in long mode, not protected mode.

# What is paging?

Paging is a method for each process to see the entire virtual address space, even if said memory isn’t physically present. This allows for processes to have memory protection, preventing processes from writing to each other’s memory without permission from the kernel.

# Implementation

Paging is implemented by the Memory Management Unit (MMU), which does all of the translation of the virtual addresses into physical addresses. As in the name, paging maps **pages**, which can be anywhere from 4KB to 2GB, but for the sake of simplicity, they will be 4KB. A page is just a 4KB block in memory, that’s really all there is to it. The reason why pages are allocated is information allocated in said blocks of memory are easier to find. So, how does one allocate a “page”?

Here is where the page frame allocator, which I’ll be refering to as the PFA for the rest of this post. 

## Page Frame Allocator

The page frame allocator’s job is to allocate a fixed size number of “page frames” to the user. How this is achieved is a lot simpler than you may think. I always thought dividing the memory into pages was done by setting some value in a control register, but it’s much more fundamental than that.

The first step is to obtain a memory map from the bootloader (or if you are writing the bootloader yourself, obtain the memory map via the EFI API you’re using).

It’s safe to assume that the memory map will return the **physical address**of the memory segments.

Your next job is to find a method to keep track of the pages, a bitmap is a very beginner friendly option. 

Next, you should iterate through the usable entries of the memory map and get the **highest usable address** and store it in a variable. The reason why you want to use the highest **usable** address and not the highest address of the memory map is due to the fact that if the memory map is organized as follows:

**Note:** **Green** is a free block of memory that the memory map entails (NOT A PAGE!), and **red** is the reserved.

![Untitled](/v1696377720/Untitled_f3bhcs.png)

Where the arrow is pointing is where the **highest usable address** is, past the arrow are **no usable entries**

So why not map the entire memory map? It’s just a waste, and a big waste. If you sum up the number of reserved and unusable entries on your bootloader’s memory map, it can add up to a significant amount. So minimizing the number of reserved entries is a good idea, since it will **not** require us to mess around and create overhead, while also reducing the number of space the method of tracking each page will take.

Next, you want to iterate through the usable entries of the memory map and find a place to store your method of tracking each page. Mark the entire data structure you’re using as **used** and then iterate through it again, marking each page as free.

Wait, how do I mark a **page** as free? Simple, when you iterate through the usable entries of the bitmap, make sure to offset the address by the page size, here is a pseudo C example:

```c
for(usize i = 0; i < entry_count; i++) {
    if(entry[i].type == USABLE) {
        for(u64 start = entry[i].address; start < entry[i].address + entry[i].length; start += PAGE_SIZE) {
            u64 index = start / PAGE_SIZE;
            pfa_mark_free(index);
        }
    }
}
```

Now, when writing an allocation function, iterate through the method you used to track the free pages, and return the index of a free page multiplied by the page size for the **physical address** of the page. Make sure to mark the page as used.

**Note:** If your bootloader maps the kernel to the higher half, you have to offset the address returned by the higher half offset. 

Writing the freeing function is practically the same, just divide the address given by the page size and set the index in whatever method you’re using to keep track of the free pages to free.

### Concerns

Yes, theoretically, nothing is stopping you from writing across the arbitrary page boundary. In some cases when paging is enabled, this will cause a page fault, however, in most cases, if the address you’re accessing is valid, you **will** be able to write to it. If you don’t believe me, try it out!  

Here is me attempting it in my kernel:

![Untitled](/v1696377720/Untitled_1_vmxqyl.png)

## Paging

The following diagram shows 4 level paging with 4KB pages:

![Untitled](/v1696377720/Untitled_2_rkjwys.png)

However, you might’ve already seen this diagram a million times and have been expected to follow it, but you have no clue how the diagram even works or represents.

Think of it of a long linked list of arrays, and each index in the array is a memory address to the next table.

Here, I can write some pseudo C code which shows a rough example of the concept behind paging.

**Note:** **THIS IS NOT PAGING CODE! All this code does it show you rough understanding of paging.**

### END, to be continued.
