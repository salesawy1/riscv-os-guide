# Chapter 9: Physical Memory Management

**[Difficulty: ★★★☆☆]**

---

## Why This Chapter Exists

Up to now, your kernel has used memory in the simplest possible way: the stack is statically allocated in BSS, and everything else is either a global variable or a local variable on the stack. This works for a tiny kernel, but it doesn't scale. You can't create new processes without allocating memory for their stacks and page tables. You can't map new pages without allocating physical frames. You can't grow data structures without dynamic allocation.

Physical memory management is the foundation of everything else. The frame allocator you build in this chapter answers one question: "Give me a page of physical RAM that nobody else is using." And its complement: "I'm done with this page; take it back." Every higher-level memory system in your OS — the page table manager (Chapter 10), the kernel heap (Chapter 12), process memory (Chapter 13) — depends on this allocator.

The concept is simple. The implementation is simple. But the thinking behind it is important: you need to know how much RAM exists, which parts are already occupied by your kernel, and how to efficiently track which pages are free and which are in use.

---

## How Much Memory Do You Have?

On the QEMU `virt` machine, you specified `-m 128M`, so you have 128 MiB of RAM starting at physical address `0x80000000` and ending at `0x88000000`. But you can't use all of it — your kernel is loaded at the bottom of that range. The free memory starts after the kernel.

### The Kernel's Footprint

Your linker script exports a `_kernel_end` symbol at the end of the BSS section (the last kernel section). Everything from `0x80000000` to `_kernel_end` is occupied by the kernel's code, read-only data, initialized data, uninitialized data (BSS), and the boot stack. This memory is off-limits to the allocator.

```
Physical Memory Layout:
┌──────────────────────┐ 0x8000_0000
│   .text (code)       │
├──────────────────────┤
│   .rodata            │
├──────────────────────┤
│   .data              │
├──────────────────────┤
│   .bss (+ stack)     │
├──────────────────────┤ _kernel_end (page-aligned)
│                      │
│   FREE MEMORY        │ ← This is what your allocator manages
│                      │
│                      │
└──────────────────────┘ 0x8800_0000 (128 MiB end)
```

The free memory pool starts at `_kernel_end` and ends at `0x80000000 + memory_size`. Since we hardcode 128 MiB for QEMU, the end is `0x88000000`.

On real hardware, you'd discover the memory size from the device tree blob (DTB) or from firmware-provided tables. The DTB contains a `memory` node that specifies the base address and size of each physical memory region. For QEMU, hardcoding is fine and much simpler.

> **Aside: Memory detection on real hardware**
>
> On x86, memory detection has a long and painful history. BIOS systems use interrupt `INT 15h, E820h` to get a memory map — a list of address ranges and their types (usable, reserved, ACPI, etc.). UEFI systems provide a memory map through the EFI boot services. The x86 memory map is notoriously messy, with holes for legacy devices (VGA memory at 0xA0000-0xBFFFF), reserved regions for firmware, and non-contiguous usable ranges.
>
> RISC-V and ARM systems use the device tree, which is cleaner but still requires parsing. The device tree's `/memory` node is straightforward:
> ```
> memory@80000000 {
>     device_type = "memory";
>     reg = <0x00 0x80000000 0x00 0x08000000>;  // base: 0x80000000, size: 128 MiB
> };
> ```
>
> The xv6 kernel hardcodes its memory size (`PHYSTOP = KERNBASE + 128*1024*1024`). We're doing the same.

---

## Pages and Frames

Before we build the allocator, let's nail down terminology.

A **[page](https://en.wikipedia.org/wiki/Virtual_memory#Paging)** is a fixed-size block of virtual memory. On RISC-V with [Sv39 paging](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf), a page is 4096 bytes (4 KiB). The virtual address space is divided into pages, and the [MMU](https://en.wikipedia.org/wiki/Memory_management_unit) translates virtual pages to physical frames via the page table.

A **frame** (or **physical page**) is a fixed-size block of physical memory. Same size as a page — 4096 bytes. The physical address space is divided into frames.

The **page frame number (PFN)** is the frame's physical address divided by the page size: `PFN = physical_address / 4096`. Equivalently, it's the physical address with the bottom 12 bits stripped off: `PFN = physical_address >> 12`.

The allocator manages **frames**, not pages. It gives you a physical address (or PFN) of an available frame. The page table manager (Chapter 10) maps virtual pages to physical frames. This separation is clean and important: the frame allocator knows nothing about virtual memory, and the page table manager knows nothing about which frames are free.

<details>
<summary>Why is 4 KiB the standard page size, not something smaller or larger?</summary>
<div>

4 KiB balances several competing concerns. Smaller pages reduce internal fragmentation and waste, but require larger page tables and more TLB entries. Larger pages reduce page table overhead and TLB pressure, but waste memory for small allocations. 4 KiB has been standard since the VAX in 1977 and is deeply embedded in hardware and software.

</div>
</details>

For your OS, you'll allocate and manage 4 KiB frames exclusively. Superpages (2 MiB and 1 GiB) are optimizations for later (or never — they're complex and not needed for a teaching OS).

---

## The Free List Allocator

The simplest frame allocator is a **[free list](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-freespace.pdf)**: a linked list of free frames. Each free frame contains a pointer to the next free frame. Allocation removes a frame from the head of the list. Deallocation adds a frame to the head. Both operations are O(1).

### How It Works

```
Initially:
free_list → [frame A] → [frame B] → [frame C] → ... → NULL

After allocating one frame:
  Returns frame A
  free_list → [frame B] → [frame C] → ... → NULL

After freeing frame A:
  free_list → [frame A] → [frame B] → [frame C] → ... → NULL
```

<details>
<summary>Why can the "next" pointer be stored inside the free frame itself?</summary>
<div>

A free frame isn't being used for anything — all 4096 bytes are available. So you store the 8-byte "next" pointer in the first 8 bytes of the frame. When the frame is allocated, the caller overwrites it with whatever they want. The metadata only exists while the frame is free, costing zero additional memory.

</div>
</details>

This means a free frame, viewed as a struct, looks like:

```
struct free_frame {
    struct free_frame *next;  // 8 bytes, stored at the start of the 4096-byte frame
    // remaining 4088 bytes are unused
};
```

When you allocate a frame, you remove it from the list and return its address. The caller can now use all 4096 bytes however they want — the "next" pointer was only meaningful while the frame was free.

When you free a frame, you write a "next" pointer at its start (pointing to the current head of the free list) and update the head to point to this frame. The frame's contents are now "free list metadata" again.

### Initialization

<details>
<summary>Does the order matter when you add frames to the free list during initialization?</summary>
<div>

The order doesn't affect correctness, but adding from high addresses to low addresses means the free list head points to the lowest free frame. This makes the first allocation return the lowest address, which is helpful for debugging (you can predict which addresses will be allocated).

</div>
</details>

At boot, you need to build the initial free list. Walk through all frames from `_kernel_end` to the end of physical memory, and add each one to the free list:

```
for each frame address from kernel_end to memory_end, step 4096:
    add this frame to the free list
```

Adding a frame means: write the current free list head into the first 8 bytes of the frame, then update the head to point to this frame.

<details>
<summary>Why must `_kernel_end` be page-aligned for the frame allocator to work?</summary>
<div>

The allocator walks frames in PAGE_SIZE steps. If `kernel_end` is at address `0x80005A00` (not page-aligned), you'd skip valid frames or include kernel memory in the free list. The alignment formula `(kernel_end + PAGE_SIZE - 1) & ~(PAGE_SIZE - 1)` rounds up to the next page boundary.

</div>
</details>

---

## The Allocator Interface

Your frame allocator should expose two functions:

**`void *kalloc(void)`** — Allocate one 4 KiB frame. Returns the physical address of the frame, or `NULL` (0) if no frames are available (out of memory). The returned frame is filled with garbage — don't assume it's zeroed (though you might want to zero it as a safety measure).

**`void kfree(void *pa)`** — Free a previously allocated frame. `pa` must be page-aligned and must be an address that was previously returned by `kalloc`. Freeing an address that wasn't allocated, or freeing the same address twice, is a bug — and on bare metal, it will corrupt the free list silently.

That's the complete interface. One function to get a page, one function to give it back.

<details>
<summary>Should `kalloc` return zeroed or unzeroed memory, and when?</summary>
<div>

There are tradeoffs. Zeroing on allocation is safer (prevents information leaks) but slower. Zeroing on free reuses the write you must do anyway (for the next pointer), but makes free slower. Some implementations provide both `kalloc()` and `kalloc_zeroed()`. For a teaching OS, zeroing on allocate is the safest default—the performance cost is negligible on QEMU.

</div>
</details>

> **Aside: The buddy allocator and beyond**
>
> The free list allocator is O(1) for allocate and free, which is great. But it has a weakness: it can only allocate one page at a time. What if you need 4 contiguous pages for a large buffer? The free list can give you 4 pages, but they might not be contiguous in physical memory.
>
> The **[buddy allocator](https://en.wikipedia.org/wiki/Buddy_memory_allocation)**, used by the Linux kernel, solves this. It maintains separate free lists for blocks of 1 page, 2 pages, 4 pages, 8 pages, and so on (powers of 2). When a 4-page block is requested and none is available, an 8-page block is split into two 4-page "buddies." When both buddies are freed, they're merged back into an 8-page block. This gives O(log n) allocation of contiguous blocks of any power-of-2 size.
>
> For our OS, the simple free list is sufficient. We only need single pages — page tables are one page each, and process stacks are one page each. If you later need contiguous multi-page allocations, you can upgrade to a buddy allocator.
>
> OSTEP [Chapter 17 (Free-Space Management)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-freespace.pdf) covers free lists, buddy allocation, and slab allocators in detail.

---

## Protecting the Allocator

A few safety measures worth implementing:

**Bounds checking on free.** When `kfree(pa)` is called, verify that `pa` is within the valid physical memory range and is page-aligned. If it isn't, you have a bug somewhere — panic rather than silently corrupting the free list.

<details>
<summary>Why is double-free such a serious bug, and how do you detect it?</summary>
<div>

Double-freeing creates a cycle in the free list, so the same frame is returned twice. Two kernel components then overwrite each other's data. Detection: fill freed frames with a known pattern (e.g., `0xDEAD`), then check if the pattern is already there before freeing again. This catches the most common cases.

</div>
</details>

<details>
<summary>How do you detect use-after-free bugs, which don't create an immediate crash?</summary>
<div>

Fill freed frames with a junk pattern (e.g., all bytes `0xAA`). If code later accesses the freed frame and sees this pattern, it's using freed memory. This doesn't prevent the bug, but makes symptoms more recognizable than random data.

</div>
</details>

These are debugging aids, not security measures. On a production OS, you'd use more robust techniques. For a teaching OS, they'll save you hours of debugging when you inevitably free something you shouldn't, or use something after you've freed it.

---

## How Many Frames Do You Have?

Quick arithmetic:

- Total RAM: 128 MiB = 134,217,728 bytes
- Total frames: 134,217,728 / 4,096 = 32,768 frames
- Kernel footprint: depends on your code size, but probably 10-50 pages (~40-200 KiB)
- Available frames: ~32,700 frames, or ~127.7 MiB

That's a lot of pages. Your free list will have ~32,000 entries. Each entry is 8 bytes (a pointer), stored in the frame itself. The free list is spread across 32,000 pages, not in a contiguous array. The only per-frame overhead is the 8-byte next pointer, and that disappears when the frame is allocated.

For perspective: Linux manages millions of frames on modern systems (a 64 GB machine has 16,777,216 frames). The buddy allocator handles this efficiently, but the simple free list works fine for 32,000 frames.

---

## Conceptual Exercises

1. **Why can the "next" pointer be stored inside the free frame itself?** What would go wrong if you tried to store the free list in a separate array? How much memory would that array need for 32,768 frames?

2. **Your free list allocator is O(1) for both allocate and free. Why?** What operations does each function perform? Why doesn't the list length affect the time?

3. **Two different parts of the kernel call `kalloc()` at the same time. On a single-core system with interrupts disabled, is this a problem?** What about on a single-core system with interrupts enabled? What about on a multicore system? (Preview of concurrency issues.)

4. **You allocate a frame, use it as a page table, then free it. Later, the same frame is allocated for a user process's data.** If you didn't zero the frame on allocation, what data does the user process see? Why is this a security concern?

5. **Your kernel has allocated all 32,768 frames. A new `kalloc()` call returns NULL. What should the caller do?** Give examples from different callers (page table creation, process creation, kernel heap).

6. **Explain why `_kernel_end` must be page-aligned for the frame allocator to work correctly.** What would happen if `_kernel_end` were at address `0x80005A00` (not page-aligned) and you started the free list there?

---

## Build This

### Frame Allocator

Create `kernel/pmm.c` (physical memory manager) and `kernel/include/pmm.h`:

1. **Define constants:**
   - `PAGE_SIZE = 4096`
   - `MEMORY_END = 0x88000000` (for 128 MiB starting at `0x80000000`)

2. **Define the free frame structure:**
   - A struct with a single `next` pointer

3. **Implement `pmm_init()`:**
   - Calculate the start of free memory (page-aligned `_kernel_end`)
   - Walk from start to `MEMORY_END` in `PAGE_SIZE` steps
   - Add each frame to the free list
   - Print the number of free frames (for verification)

4. **Implement `void *kalloc(void)`:**
   - Remove the head of the free list
   - Return its address (or NULL if empty)
   - Optionally zero the page with `memset`

5. **Implement `void kfree(void *pa)`:**
   - Validate that `pa` is page-aligned and in range
   - Optionally fill with junk pattern
   - Add to the head of the free list

### Update kernel_main

```c
void kernel_main(void) {
    uart_init();
    kprintf("Hello from the kernel!\n");
    trap_init();
    timer_init();

    pmm_init();

    // Test the allocator
    void *p1 = kalloc();
    void *p2 = kalloc();
    void *p3 = kalloc();
    kprintf("Allocated: %p, %p, %p\n", p1, p2, p3);

    kfree(p2);
    void *p4 = kalloc();
    kprintf("After free+alloc: %p (should be %p)\n", p4, p2);

    while (1) {}
}
```

### Checkpoint

Run `make run`. Expected output (addresses will vary):

```
Hello from the kernel!
Timer started. You should see ticks...
Physical memory: XXXXX free frames (XXX MiB)
Allocated: 0x80XXX000, 0x80XXX000, 0x80XXX000
After free+alloc: 0x80XXX000 (should be 0x80XXX000)
```

Key verifications:
- The number of free frames should be approximately 32,700 (give or take, depending on kernel size)
- Each allocated address should be page-aligned (ends in `000`)
- After freeing `p2` and allocating `p4`, `p4` should equal `p2` (the free list returns the most recently freed page)
- All addresses should be above `_kernel_end` and below `0x88000000`

---

## When Things Go Wrong

**Allocator returns addresses below `_kernel_end`**
Your start address calculation is wrong. Print `_kernel_end` and the calculated start address to verify.

**Allocator returns NULL immediately**
The free list was never built. Check that `pmm_init()` is actually adding frames to the list. Print the count of frames added.

**System crashes after allocating a few pages**
The free list is corrupted. This usually means a frame that's in the free list was also written to by something else (overwriting the "next" pointer). Check that your kernel code isn't writing to addresses in the free memory region.

**Allocated pages contain data when they should be zero**
If you're not zeroing on allocation, this is expected. If you are zeroing, check that your `memset` is using the correct size (4096 bytes, not some other value).

---

## Further Reading

- **[OSTEP Chapter 17: Free-Space Management](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-freespace.pdf)** — Covers free lists, splitting, coalescing, buddy allocation, and slab allocation. Excellent conceptual treatment.
- **[xv6 Book, Chapter 3: Page Tables](https://pdos.csail.mit.edu/6.828/2024/xv6/book-riscv-rev4.pdf)** — The xv6 frame allocator (`kalloc.c`) is described alongside the page table implementation. Read the allocator section.
- **[xv6 source: `kalloc.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/kalloc.c)** — About 60 lines. A free list allocator almost identical to what you're building. The clearest reference implementation.
- **CS:APP Chapter 9: Virtual Memory** — Section 9.9 covers dynamic memory allocation. While focused on `malloc` (heap allocation), the concepts of free lists and fragmentation apply directly.

---

*Next: [Chapter 10 — Virtual Memory and Paging](ch10-virtual-memory.md), the most important and most difficult chapter in this guide. We'll build Sv39 page tables, understand the MMU's page walk, and give every process the illusion that it owns all of memory.*
