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

<details>
<summary>How does the allocator find the header when `kfree(ptr)` is called?</summary>
<div>

The header is stored immediately before the usable memory. `kfree` subtracts a fixed offset (the header size) from the returned pointer to find it. This requires the caller to pass the exact pointer that `kmalloc` returned; any modification breaks the lookup.

</div>
</details>

When `kmalloc(48)` is called:
1. Walk the free list looking for a block of at least 48 bytes
2. If the block is much larger than needed, **split** it: carve out 48 bytes (plus header) and leave the rest as a smaller free block
3. Mark the block as allocated and return a pointer to the usable area (past the header)

When `kfree(ptr)` is called:
1. Find the header (it's just before `ptr`)
2. Mark the block as free
3. Add it to the free list
4. **Coalesce** with adjacent free blocks if possible (merge two neighboring free blocks into one larger block)

<details>
<summary>Why split blocks instead of returning the entire block, and what's the minimum block size?</summary>
<div>

Without splitting, a 48-byte request from a 4096-byte block wastes 4048 bytes (internal fragmentation). Splitting carves the block into two pieces—one for the request, one for the remainder. The minimum block size is the header size (so the block can hold its header when free). Most implementations use 16-32 bytes as the minimum.

</div>
</details>

<details>
<summary>What is coalescing, and how do you efficiently find adjacent free blocks to merge?</summary>
<div>

Coalescing merges physically adjacent free blocks to combat fragmentation. To find neighbors, you can use boundary tags (copy the size at both ends—constant time), sequential scan (walk the free list—O(n)), or implicit free lists (walk all blocks sequentially—simpler but O(n) for allocation). For a teaching OS, the implicit free list with splitting and coalescing balances simplicity and correctness.

</div>
</details>

<details>
<summary>Why do production kernels use slab allocators instead of simple free lists?</summary>
<div>

Most kernel allocations are fixed-size (task structs, inodes, buffer heads, etc.). Slab allocators pre-allocate pools for each type—no splitting, no coalescing, no fragmentation. They layer on top of the page allocator just like your heap layers on the frame allocator. For a teaching OS, a simple free list works fine.

</div>
</details>

---

<details>
<summary>When should the heap request new pages from the frame allocator, and can the heap shrink?</summary>
<div>

The heap requests pages on-demand when no free block is large enough—it starts small and grows as needed. It can potentially shrink by returning pages to the frame allocator when they're entirely free, but most implementations don't bother. If you have a direct map, you can allocate pages and access them without additional page table entries.

</div>
</details>

---

<details>
<summary>Why must `kmalloc` return aligned addresses, and how do you ensure alignment?</summary>
<div>

Alignment ensures any data type can be stored at the returned address without issues. A 16-byte header naturally aligns the usable area to 16 bytes (on 64-bit systems) if the block starts 16-byte-aligned. This is why many implementations use 16-byte headers instead of smaller ones.

</div>
</details>

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
