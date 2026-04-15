# Chapter 12: The Kernel Heap

**[Difficulty: ★★★☆☆]**

---

## Why This Chapter Exists

Your frame allocator hands out 4 KiB chunks. That's perfect for page tables, process stacks, and page-sized buffers. But what about a 48-byte process control block? A 256-byte buffer for a filename? A 16-byte linked list node? Allocating an entire 4 KiB page for a 48-byte struct wastes 4,048 bytes — a 99% overhead.

The **kernel heap** sits on top of the frame allocator and provides variable-sized allocation: `kmalloc(size)` returns a pointer to `size` contiguous bytes, and `kfree(ptr)` returns them. This is your kernel's equivalent of `malloc`/`free` from the C standard library, and it follows the same interface.

The heap allocator requests large chunks (pages) from the frame allocator and carves them into smaller pieces for callers. When all pieces in a page are freed, the page can be returned to the frame allocator. This layering — frame allocator for page-granularity, heap allocator for byte-granularity — is how every real OS structures its memory management.

---

## Free List Allocator for the Heap

The simplest heap allocator is, again, a [free list](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-freespace.pdf) — but this time, the list manages variable-sized blocks rather than fixed-size pages.

### Block Structure

Each block (allocated or free) has a header that records its size. Free blocks additionally store a "next" pointer linking to the next free block:

```
┌──────────────────┬───────────────────────────┐
│   Header         │   Usable Memory           │
│ (size, is_free)  │   (returned to caller)    │
└──────────────────┴───────────────────────────┘
```

The header is typically 16 bytes on a 64-bit system: 8 bytes for the size, 8 bytes for the next pointer (or some combined structure).

When `kmalloc(48)` is called:
1. Walk the free list looking for a block of at least 48 bytes
2. If the block is much larger than needed, **split** it: carve out 48 bytes (plus header) and leave the rest as a smaller free block
3. Mark the block as allocated and return a pointer to the usable area (past the header)

When `kfree(ptr)` is called:
1. Find the header (it's just before `ptr`)
2. Mark the block as free
3. Add it to the free list
4. **Coalesce** with adjacent free blocks if possible (merge two neighboring free blocks into one larger block)

### Splitting

Without splitting, every allocation returns the first block that's big enough, regardless of how much space is wasted. A 48-byte allocation from a 4096-byte block wastes 4048 bytes (internal [fragmentation](https://en.wikipedia.org/wiki/Fragmentation_(computing))).

With splitting, the allocator carves the block into two pieces: one of the requested size (plus header) and one containing the remainder. Both have headers. The remainder goes back on the free list.

Splitting requires a minimum block size — there's no point creating a free block of 0 usable bytes. The minimum block size is typically the header size (so the block can at least hold the header when free). On most implementations, the minimum allocation unit is 16 or 32 bytes.

### Coalescing

Without coalescing, the free list accumulates many small free blocks that are physically adjacent but logically separate. A program that allocates 100 blocks of 32 bytes and frees them all ends up with 100 free blocks of 32 bytes, not one free block of 3200 bytes. A subsequent request for 256 bytes fails because no single free block is large enough.

**[Coalescing](https://en.wikipedia.org/wiki/Memory_management#Fragmentation)** merges adjacent free blocks. When freeing a block, check if the block immediately before or after it is also free. If so, merge them into one larger block. This combats **[fragmentation](https://en.wikipedia.org/wiki/Fragmentation_(computing))** — the tendency for free memory to be broken into small, unusable pieces.

Coalescing adjacent blocks requires knowing where the neighboring blocks are. There are several approaches:

**Boundary tags:** Each block has a header at the start AND a footer at the end (a copy of the size). To find the previous block, read the footer immediately before your header. To find the next block, use your size to compute the address of the next header. This enables constant-time coalescing but adds overhead (the footer).

**Sequential scan:** Walk the free list to find neighbors. Simple but O(n) in the number of free blocks.

**Implicit free list:** Don't maintain an explicit linked list of free blocks. Instead, walk through all blocks (free and allocated) sequentially using the size field in each header. This is simpler but makes allocation O(n) in the total number of blocks.

For a teaching OS, the implicit free list with splitting and coalescing is the best balance of simplicity and correctness. It's the algorithm taught in CS:APP Chapter 9.9, and it works well for kernels that don't allocate millions of small objects.

> **Aside: Slab allocation**
>
> The Linux kernel uses a more sophisticated allocator called the **[slab allocator](https://en.wikipedia.org/wiki/Memory_pool#Slab_allocation)** (and its successors SLUB and SLOB). The idea: most kernel allocations are for a small number of fixed-size object types (task structs, inode structs, buffer heads, etc.). The slab allocator pre-allocates pools of objects of each type. Allocating a task struct just pops one from the task struct pool. Freeing it pushes it back. No splitting, no coalescing, no fragmentation.
>
> The slab allocator is layered on top of the page allocator (buddy system), just as your heap is layered on top of the frame allocator. The difference is in the fragmentation characteristics and performance for repeated same-size allocations.
>
> For a teaching OS, a simple free list is sufficient. If you notice fragmentation becoming a problem (it probably won't at this scale), you could implement a simple slab allocator for your most-allocated struct type.
>
> [OSTEP Chapter 17](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-freespace.pdf) discusses slab allocation. Bonwick's original paper, "[The Slab Allocator: An Object-Caching Kernel Memory Allocator](https://www.usenix.org/proceedings/summer-1994/bonwick)" (1994), is a classic and very readable.

---

## Growing the Heap

When the free list has no block large enough for a request, the allocator needs more memory. It requests a page (or multiple pages) from the frame allocator using `kalloc()`, adds the page to the heap's managed region, and creates a free block covering the new page. Then it retries the allocation.

This means the heap grows on demand: it starts empty and requests pages as needed. It can also potentially shrink (return pages to the frame allocator when they're entirely free), though most simple implementations don't bother.

The heap occupies a region of the kernel's virtual address space. You should designate a range — say, `KERNEL_HEAP_START` to `KERNEL_HEAP_END` — and map new pages into this range as the heap grows. Alternatively, since you have a direct map of all physical memory, you can allocate physical pages and access them through the direct map without additional page table entries.

---

## Alignment

`kmalloc` should return addresses aligned to at least 8 bytes (on a 64-bit system) so that any data type can be stored at the returned address without alignment issues. Many implementations align to 16 bytes for compatibility with SIMD and atomic operations.

Your header size naturally provides alignment if it's a multiple of 8 or 16 bytes. If you design the header to be 16 bytes, and place the usable area immediately after the header, the usable area will be 16-byte aligned as long as the block itself starts at a 16-byte-aligned address.

---

## Conceptual Exercises

1. **Why can't you just use the frame allocator for all kernel memory allocation?** Give three examples of kernel data structures that are significantly smaller than 4 KiB.

2. **You allocate blocks A (32 bytes), B (64 bytes), C (32 bytes) from the heap, then free B. Now you try to allocate D (96 bytes). Can the free space from B satisfy this request?** Why or why not? How would coalescing help (or not help) in this case?

3. **External fragmentation vs. internal fragmentation.** Define each. Give an example of each in the context of your heap allocator. Which one does splitting address? Which one does coalescing address?

4. **Your heap allocator has a bug: it never coalesces free blocks. Over time, what happens to the heap?** Describe the degradation pattern. How would you detect this problem?

5. **Why is the minimum block size equal to the header size (or larger)?** What would happen if you tried to create a free block smaller than the header?

---

## Build This

### Kernel Heap Allocator

Create `kernel/heap.c` and `kernel/include/heap.h`:

1. **Implement `void heap_init(void)`:**
   - Request an initial page from `kalloc()`
   - Set up the first free block spanning the entire page (minus header overhead)

2. **Implement `void *kmalloc(size_t size)`:**
   - Walk the free list for a block of sufficient size
   - Split if the block is much larger than needed
   - If no block is found, request a new page from `kalloc()`, add it to the managed region, and retry
   - Return a pointer to the usable area (past the header)
   - Round up size to maintain alignment

3. **Implement `void kfree(void *ptr)`:**
   - Find the header (immediately before `ptr`)
   - Mark the block as free
   - Coalesce with adjacent free blocks if possible
   - Add to the free list

### Checkpoint

```c
void *a = kmalloc(48);
void *b = kmalloc(256);
void *c = kmalloc(16);
kprintf("a=%p b=%p c=%p\n", a, b, c);
kfree(b);
void *d = kmalloc(128);
kprintf("d=%p (reusing freed space from b)\n", d);
kfree(a);
kfree(c);
kfree(d);
```

Verify that allocations return valid, aligned pointers within the kernel's address range, that `kfree` doesn't crash, and that freed space is reused by subsequent allocations.

---

## When Things Go Wrong

**kmalloc returns NULL unexpectedly**
The heap can't find a free block and `kalloc()` failed (out of physical frames). Or your free list traversal has a bug — check for off-by-one errors in size comparisons.

**Mysterious corruption after kfree**
You're writing to freed memory (use-after-free), or your coalescing logic has a bug that corrupts block headers. Fill freed blocks with a junk pattern to catch use-after-free.

**Heap never reuses freed space**
Your free block isn't being added to the free list, or the size recorded in the header is wrong so no subsequent request matches.

---

## Further Reading

- **CS:APP Chapter 9.9: Dynamic Memory Allocation** — The best textbook treatment of `malloc`/`free` implementation. Covers implicit free lists, explicit free lists, segregated lists, splitting, coalescing, and boundary tags.
- **[OSTEP Chapter 17: Free-Space Management](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-freespace.pdf)** — Covers the same algorithms from an OS perspective.
- **[Bonwick, "The Slab Allocator" (1994)](https://www.usenix.org/proceedings/summer-1994/bonwick)** — The original paper on slab allocation in SunOS/Solaris.

---

*Next: [Chapter 13 — What Is a Process?](ch13-processes.md), where we define the abstraction that turns your single-program kernel into a multi-program operating system.*
