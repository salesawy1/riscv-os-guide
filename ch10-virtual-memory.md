# Chapter 10: Virtual Memory and Paging

**[Difficulty: ★★★★★]**

---

## Why This Chapter Exists

This is the big one. If you understand virtual memory — truly understand it, down to the bit layout of a page table entry and the exact sequence of steps the hardware performs during a page walk — you understand the most powerful abstraction in all of computing.

Virtual memory does three things simultaneously:

1. **Isolation.** Each process gets its own address space. Process A's pointer `0x1000` points to different physical memory than process B's pointer `0x1000`. They can't see each other's data, can't corrupt each other's memory, can't even tell the other exists. This is the foundation of all OS security and stability.

2. **Abstraction.** Every process thinks it has a contiguous, private address space starting from 0 (or some other fixed address). In reality, its pages are scattered across physical memory — page 0 of the process might be at physical frame 57,312, page 1 at frame 12, page 2 at frame 41,000. The process doesn't know or care. The hardware translates every address transparently.

3. **Overcommitment.** The virtual address space can be larger than physical memory. Each process can have a 256 GiB virtual address space on Sv39, even though you only have 128 MiB of physical RAM. Only the pages the process actually uses need physical frames. This is how modern systems run hundreds of programs with modest RAM — most of each program's virtual space is empty and costs nothing.

The mechanism that provides all three is the **page table**: a tree-structured data structure, stored in physical memory, that the hardware (the MMU — Memory Management Unit) walks on every memory access to translate virtual addresses to physical addresses. Your OS builds and maintains the page tables. The hardware reads them. Between the two, the illusion of private address spaces emerges.

This chapter is long, dense, and demands careful attention. Don't rush it. Read the [RISC-V privileged spec](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf) alongside this chapter (Chapter 4, "Supervisor-Level ISA," Section 4.4, "Sv39"). When you finish this chapter, you should be able to draw the Sv39 page table structure from memory, trace a [page walk](https://en.wikipedia.org/wiki/Page_table#Page_table_lookup) by hand, explain every bit of a PTE, and articulate exactly what happens when the CPU tries to access a page that isn't mapped.

---

## The Translation: Virtual → Physical

Without paging, the CPU puts the address from the instruction directly on the memory bus. Address `0x80001234` means physical address `0x80001234`. Simple. Direct.

With paging enabled, every address the CPU generates (for instruction fetches, loads, and stores) is a **virtual address**. The MMU intercepts this address and translates it to a **physical address** before it reaches the memory bus. The translation uses a page table, which is a data structure the OS built in physical memory and told the MMU about (via the `satp` register).

The key insight: **the page table maps virtual page numbers to physical frame numbers.** The low 12 bits of an address (the **page offset**) pass through unchanged — they select a byte within the page/frame. Only the upper bits (the page number) are translated.

```
Virtual Address (39 bits for Sv39):
┌────────────────────────┬──────────────┐
│  Virtual Page Number   │ Page Offset  │
│     (27 bits)          │  (12 bits)   │
└────────────────────────┴──────────────┘
         │                      │
    Page Table                  │
    Translation                 │
         │                      │
         ▼                      ▼
┌────────────────────────┬──────────────┐
│ Physical Frame Number  │ Page Offset  │
│     (44 bits)          │  (12 bits)   │
└────────────────────────┴──────────────┘
Physical Address (56 bits)
```

The page offset is 12 bits because `log2(4096) = 12`. A 4 KiB page needs 12 bits to address each byte within it.

---

## Sv39: Three-Level Page Tables

RISC-V supports multiple paging modes: Sv32 (32-bit virtual addresses, 2-level page tables), Sv39 (39-bit, 3-level), Sv48 (48-bit, 4-level), and Sv57 (57-bit, 5-level). We'll use **Sv39**, which is the most common mode for RV64 and what QEMU's `virt` machine supports.

Sv39 gives you a 39-bit virtual address space: 2^39 = 512 GiB. Half of this (the upper half, where bit 38 is set) is conventionally used for the kernel, and the other half for user programs. But we're getting ahead of ourselves.

### Why Three Levels?

A 27-bit virtual page number could index a flat page table with 2^27 = 134 million entries. At 8 bytes per entry, that's 1 GiB of page table per process — absurdly wasteful, especially when most of the address space is empty.

The three-level structure is a **[radix tree](https://en.wikipedia.org/wiki/Radix_tree)** that only allocates page table pages for the parts of the address space that are actually used. Each level covers 9 bits of the virtual page number:

```
39-bit Virtual Address:
┌─────────┬─────────┬─────────┬──────────────┐
│ VPN[2]  │ VPN[1]  │ VPN[0]  │ Page Offset  │
│ (9 bits)│ (9 bits)│ (9 bits)│  (12 bits)   │
│ bits    │ bits    │ bits    │ bits         │
│ 38..30  │ 29..21  │ 20..12  │ 11..0        │
└─────────┴─────────┴─────────┴──────────────┘
```

Each 9-bit index selects one of 512 entries in a page table page. A page table page is exactly 4 KiB (512 entries × 8 bytes per entry = 4096 bytes), which fits perfectly in one physical frame.

### The Page Walk

When the CPU needs to translate a virtual address, the MMU performs a **[page walk](https://en.wikipedia.org/wiki/Page_table#Page_table_lookup)** — a series of memory reads through the three levels of the page table:

```
Step 1: Read satp to get the root page table's physical address
        root_ppn = satp.PPN
        root_table = root_ppn << 12  (convert PPN to physical address)

Step 2: Index into Level 2 (the root table)
        level2_entry = root_table[VPN[2]]
        (This reads 8 bytes from physical address: root_table + VPN[2] * 8)

Step 3: If level2_entry is a valid, non-leaf entry:
        level1_table = level2_entry.PPN << 12
        level1_entry = level1_table[VPN[1]]

Step 4: If level1_entry is a valid, non-leaf entry:
        level0_table = level1_entry.PPN << 12
        level0_entry = level0_table[VPN[0]]

Step 5: level0_entry is the leaf PTE
        physical_address = (level0_entry.PPN << 12) | page_offset
```

Three memory reads to translate one address. This sounds expensive, and it is — which is why the [TLB (Translation Lookaside Buffer)](https://en.wikipedia.org/wiki/Translation_lookaside_buffer) exists to cache recent translations. We'll discuss the TLB shortly.

### Superpages

If at level 2 or level 1, the PTE is a **leaf** (has read, write, or execute permission bits set), it's a superpage mapping:

- A leaf at level 2 maps a **1 GiB gigapage** (VPN[1] and VPN[0] and page offset are all part of the offset within the gigapage)
- A leaf at level 1 maps a **2 MiB megapage** (VPN[0] and page offset are the offset)

Superpages are useful for mapping large contiguous regions efficiently (e.g., the kernel's mapping of all physical memory). For now, we'll use only 4 KiB pages.

---

## The Page Table Entry (PTE)

Each entry in a page table is a 64-bit **Page Table Entry (PTE)**:

```
Bit(s)  Field   Meaning
──────  ─────   ───────
0       V       Valid. If 0, the rest of the PTE is ignored and the translation
                fails (page fault).

1       R       Readable. If set, loads from this page are permitted.

2       W       Writable. If set, stores to this page are permitted.

3       X       Executable. If set, instruction fetches from this page are
                permitted.

4       U       User. If set, U-mode code can access this page. If clear, only
                S-mode (or M-mode) can access it.

5       G       Global. If set, this mapping exists in all address spaces (not
                flushed from TLB on address space switch).

6       A       Accessed. Set by hardware when the page is read, written, or
                executed. Used by the OS for page replacement algorithms.

7       D       Dirty. Set by hardware when the page is written. Used by the OS
                to know which pages need to be written back to disk.

9:8     RSW     Reserved for Software. The hardware ignores these bits. The OS
                can use them for bookkeeping.

53:10   PPN     Physical Page Number. For leaf PTEs, this is the physical frame
                number that the virtual page maps to. For non-leaf PTEs, this is
                the physical frame number of the next-level page table.

63:54   Reserved (must be zero)
```

### Permission Bits: R, W, X

The combination of R, W, X determines whether the PTE is a leaf (an actual mapping) or a non-leaf (a pointer to the next level):

| R | W | X | Meaning |
|---|---|---|---------|
| 0 | 0 | 0 | Non-leaf: PPN points to next-level page table |
| 1 | 0 | 0 | Read-only page |
| 1 | 1 | 0 | Read-write page |
| 0 | 0 | 1 | Execute-only page |
| 1 | 0 | 1 | Read-execute page (typical for code) |
| 1 | 1 | 1 | Read-write-execute page (use sparingly) |
| 0 | 1 | 0 | *Reserved* — illegal combination |
| 0 | 1 | 1 | *Reserved* — illegal combination |

A PTE with V=1 and R=W=X=0 is a non-leaf entry: its PPN points to the next level of the page table. A PTE with V=1 and any of R, W, X set is a leaf entry: its PPN is the physical frame number of the mapped page.

This is elegant: the same structure serves as both "pointer to subtree" and "leaf mapping," distinguished solely by the permission bits.

### The U Bit

The U (User) bit is how the OS enforces privilege separation at the memory level:
- Pages with U=1 are accessible to U-mode code (and S-mode, if the SUM bit in `sstatus` is set)
- Pages with U=0 are accessible only to S-mode (kernel pages)

This means the kernel and user processes can coexist in the same page table. The kernel maps its pages with U=0, and user code can't touch them — any access from U-mode to a U=0 page causes a page fault.

### The A and D Bits

The Accessed (A) and Dirty (D) bits are set by hardware as the page is used:
- A is set whenever the page is accessed (read, written, or executed)
- D is set whenever the page is written

The OS periodically checks these bits for page replacement: if physical memory is full and a new frame is needed, the OS evicts a page that hasn't been accessed recently (A=0). If the evicted page is dirty (D=1), it must be written to disk before the frame can be reused. If it's clean (D=0), the frame can be reclaimed directly.

For your teaching OS, you probably won't implement page replacement (demand paging / swapping), but understanding A and D bits matters for interviews and for understanding how real OSes manage memory.

> **Aside: Hardware vs. software page table walking**
>
> On RISC-V, the page walk is done by **hardware** — the MMU automatically reads the page table from memory. The OS never sees individual translations (except on a TLB miss, which the hardware handles by walking the page table). This is also how x86 works.
>
> On MIPS and some ARM configurations, the page walk is done by **software** — on a TLB miss, the hardware raises an exception, and the OS's TLB miss handler does the page table walk and loads the TLB entry manually. This gives the OS more flexibility (it can use any data structure for page tables) but adds overhead.
>
> RISC-V's hardware page walk means you MUST use the Sv39 page table format. You can't invent your own. The hardware dictates the format.
>
> CS:APP Chapter 9 (Virtual Memory) covers both hardware-walked and software-walked TLBs, with excellent diagrams.

---

## The satp Register

The `satp` (Supervisor Address Translation and Protection) register controls the MMU:

```
Bits    Field   Meaning
──────  ─────   ───────
63:60   MODE    Translation mode:
                0 = Bare (no translation — physical addressing)
                8 = Sv39 (39-bit virtual, 3-level page table)
                9 = Sv48 (48-bit virtual, 4-level page table)

59:44   ASID    Address Space Identifier. Tags TLB entries so the TLB
                doesn't need to be flushed on every address space switch.
                (Optional optimization — you can ignore it initially.)

43:0    PPN     Physical Page Number of the root page table. The
                physical address of the root table is PPN << 12.
```

To enable paging:
1. Build a page table in physical memory
2. Write `satp` with MODE=8 (Sv39), ASID=0, and PPN = root_table_address >> 12
3. Execute `sfence.vma` to flush the TLB

After writing `satp`, the **very next instruction fetch** goes through the page table. If the page table doesn't map the address of the next instruction, the CPU will immediately page fault. This is why enabling the MMU is so delicate — you must ensure that the code you're currently executing is mapped in the new page table. We'll discuss this in Chapter 11.

**Important:** `satp` is an S-mode register. In M-mode, all addresses are physical — the MMU is not used. This means you must be in S-mode (or transitioning to S-mode) to use page tables. We'll handle the M-to-S transition as part of enabling the MMU.

---

## The TLB: Translation Lookaside Buffer

Every memory access requires a virtual-to-physical translation. A page walk requires three memory reads (for Sv39). If every instruction fetch and every load/store required three extra memory reads, performance would be catastrophic — roughly 4x slowdown for every memory access.

The **[TLB (Translation Lookaside Buffer)](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-tlbs.pdf)** is a hardware cache of recent page table translations. It stores recent VPN→PPN mappings, and the MMU checks the TLB before doing a page walk. TLB hits are fast (single cycle). TLB misses trigger the full page walk.

In practice, TLBs have 32–128 entries and hit rates above 99%, because programs exhibit [locality](https://en.wikipedia.org/wiki/Locality_of_reference) — they access the same pages repeatedly. The few pages that a program's hot loop touches all fit in the TLB, so the page walk overhead is amortized across many accesses.

### TLB Flushing: sfence.vma

When you modify a page table — change a PTE, add a mapping, remove a mapping — the TLB might have a stale entry. You must flush the stale entry (or the entire TLB) with the `sfence.vma` instruction:

- `sfence.vma` — Flush the entire TLB
- `sfence.vma rs1, x0` — Flush the TLB entry for virtual address in `rs1`
- `sfence.vma x0, rs2` — Flush all TLB entries with ASID in `rs2`
- `sfence.vma rs1, rs2` — Flush the entry for a specific address and ASID

For your initial implementation, use `sfence.vma` (flush everything) after any page table modification. It's less efficient than targeted flushes, but it's safe and simple.

Forgetting to flush the TLB after modifying a page table is one of the most common and maddening bugs in OS development. The symptom: your page table modification seems to have no effect, because the CPU is still using the old, cached translation. Then, seemingly at random (when the TLB entry gets evicted), the new mapping takes effect. The bug is intermittent and appears timing-dependent. Always flush after modifying page tables.

---

## Building a Page Table

To create a mapping from virtual address V to physical address P with permissions PERM:

1. **Extract the VPN components:** VPN[2] from bits 38:30, VPN[1] from bits 29:21, VPN[0] from bits 20:12.

2. **Walk the existing page table to level 0:**
   - Read the Level 2 entry at `root_table[VPN[2]]`
   - If invalid (V=0), allocate a new frame for a Level 1 table, zero it, and write its address into this entry (V=1, PPN=new_frame >> 12, R=W=X=0 for non-leaf)
   - Read the Level 1 entry at `level1_table[VPN[1]]`
   - If invalid, allocate a new frame for a Level 0 table, zero it, write its address

3. **Write the leaf PTE at level 0:**
   - `level0_table[VPN[0]] = (P >> 12) << 10 | PERM | PTE_V`

The PPN is shifted left by 10 because the PPN field starts at bit 10 of the PTE (bits 0-9 are the flags).

### Unmapping

To remove a mapping: set the PTE's V bit to 0 (or set the entire PTE to 0). Don't forget to free the physical frame if it's no longer needed. Also don't forget `sfence.vma`.

If all entries in a page table page become invalid, you can free that page table page too. This is optional and adds complexity — most OS implementations don't bother for the kernel page table but do it for process page tables when processes exit.

---

## Addressing Modes and Canonical Addresses

Sv39 uses 39-bit virtual addresses, but RISC-V registers are 64 bits. The upper 25 bits (bits 63:39) must be copies of bit 38 — a [sign extension](https://en.wikipedia.org/wiki/Sign_extension). This means there are two valid address ranges:

```
0x0000_0000_0000_0000 to 0x0000_003F_FFFF_FFFF  (lower half, bit 38 = 0)
0xFFFF_FFC0_0000_0000 to 0xFFFF_FFFF_FFFF_FFFF  (upper half, bit 38 = 1)
```

There's a massive hole in the middle (from `0x0000_0040_0000_0000` to `0xFFFF_FFBF_FFFF_FFFF`) that cannot be addressed. Any address in this range is non-canonical and will cause a page fault.

By convention:
- **Lower half** (0x0000...) → User space
- **Upper half** (0xFFFF...) → Kernel space

This is the **higher-half kernel** design, which we'll discuss in Chapter 11.

---

## Page Faults

When the MMU can't complete a translation — because a PTE has V=0, or the permissions don't match the access type, or the address is non-canonical — it generates a **[page fault](https://en.wikipedia.org/wiki/Page_fault)**. There are three types:

- **Instruction page fault** (cause 12): Tried to fetch an instruction from an unmapped or non-executable page
- **Load page fault** (cause 13): Tried to read from an unmapped or non-readable page
- **Store page fault** (cause 15): Tried to write to an unmapped, non-writable, or read-only page

On a page fault:
- `scause` contains the fault type (12, 13, or 15)
- `sepc` contains the address of the faulting instruction
- `stval` contains the **faulting virtual address** — the address that couldn't be translated

Page faults are not always errors. In a fully-featured OS, page faults are used for:
- **[Demand paging](https://en.wikipedia.org/wiki/Demand_paging):** Don't allocate physical frames until the process actually accesses the page. Map the page as invalid initially. When the process touches it, the page fault handler allocates a frame, maps it, and retries the instruction. This saves memory for pages that are allocated but never used.
- **[Copy-on-write (COW)](https://en.wikipedia.org/wiki/Copy-on-write):** After `fork()`, parent and child share physical frames with read-only permissions. When either writes to a shared page, a page fault triggers, the handler copies the frame, gives each process its own copy, makes both writable, and retries. This makes `fork()` fast for programs that don't modify much memory.
- **[Swapping](https://en.wikipedia.org/wiki/Memory_paging):** When physical memory is full, evict a page to disk and mark it invalid. When the process accesses it, the page fault handler reads it back from disk.

For your initial OS, page faults are errors — they mean you have a bug in your page table setup. Print the fault type, `sepc`, and `stval`, then panic. Later, you can add demand paging or COW if you want.

---

## Conceptual Exercises

1. **A virtual address is `0x00000000_80001234`. Extract VPN[2], VPN[1], VPN[0], and the page offset.** Then trace the page walk: describe the three memory reads the MMU performs, including the physical addresses it reads from (assuming you know the root table address and the intermediate table addresses).

2. **How many physical frames does a page table for a process that has mapped exactly one 4 KiB page require?** (Count the root table, the level 1 table, and the level 0 table.) What about a process that has mapped 512 contiguous pages starting at address 0?

3. **Why is the PTE format designed so that R=W=X=0 means "non-leaf"?** What would go wrong if non-leaf PTEs were indicated by a separate flag bit instead?

4. **You modify a PTE to change a page from read-only to read-write, but forget to execute `sfence.vma`.** What happens on the next write to that page? Does it always fail? Could it sometimes succeed? Why?

5. **Two processes each have a virtual page at address `0x1000`. Process A's page maps to physical frame 100, and Process B's page maps to physical frame 200.** What changes in the hardware when the OS switches from Process A to Process B? (Think about `satp` and the TLB.)

6. **The Sv39 PPN field is 44 bits. What is the maximum amount of physical memory that Sv39 can address?** Is this more or less than the 39-bit virtual address space?

7. **Why does the hardware check the U bit on every memory access, rather than checking the privilege mode only on trap entry?** What would go wrong if the hardware trusted the privilege mode and let S-mode access U-mode pages freely?

8. **Explain why writing to `satp` is one of the most dangerous operations in your kernel.** What specifically happens on the instruction immediately after the `csrw satp` executes? What must be true about the new page table for execution to continue?

---

## Build This

### Page Table Management

Create `kernel/vm.c` and `kernel/include/vm.h`:

1. **Define PTE flag constants:**
   - `PTE_V`, `PTE_R`, `PTE_W`, `PTE_X`, `PTE_U`, `PTE_G`, `PTE_A`, `PTE_D`

2. **Define helper macros/functions:**
   - Extract VPN[2], VPN[1], VPN[0] from a virtual address
   - Extract PPN from a PTE
   - Construct a PTE from a PPN and flags
   - Convert PPN to physical address and back

3. **Implement `pagetable_t vm_create(void)`:**
   - Allocate a page for the root table using `kalloc()`
   - Zero it
   - Return the address

4. **Implement `int vm_map(pagetable_t pt, uint64_t va, uint64_t pa, uint64_t flags)`:**
   - Walk the three levels, allocating intermediate tables as needed
   - Write the leaf PTE at level 0
   - Return 0 on success, -1 on failure (e.g., out of memory)

5. **Implement `uint64_t vm_lookup(pagetable_t pt, uint64_t va)`:**
   - Walk the page table and return the physical address for a virtual address
   - Return 0 (or -1) if the mapping doesn't exist

6. **Implement `void vm_unmap(pagetable_t pt, uint64_t va)`:**
   - Find the leaf PTE and clear it
   - Free the physical frame if appropriate

### DO NOT enable the MMU yet. That's Chapter 11.

For now, test your page table functions by building a page table, adding mappings, and verifying them with `vm_lookup`:

```c
pagetable_t pt = vm_create();
vm_map(pt, 0x1000, 0x80010000, PTE_R | PTE_W);
uint64_t pa = vm_lookup(pt, 0x1000);
kprintf("VA 0x1000 -> PA %p\n", (void *)pa);
// Should print: VA 0x1000 -> PA 0x80010000
```

### Checkpoint

The page table management functions work correctly in isolation: you can create tables, add mappings, look up mappings, and remove them. You can verify by printing the results. The MMU is still off — all addresses are physical. Enabling it is the next chapter's challenge.

---

## When Things Go Wrong

**`vm_map` crashes or returns garbage**
Check your VPN extraction. The most common error is getting the bit positions wrong. VPN[2] is bits 38:30 (not 38:31 or 39:31). Use a test address with known VPN values and verify each extraction.

**`vm_lookup` returns the wrong physical address**
Check your PPN extraction from the PTE. Remember, PPN is at bits 53:10 (44 bits), and you shift left by 12 to get the physical address. Also check that you're adding the page offset correctly.

**Intermediate page tables contain garbage**
You forgot to zero the page after `kalloc()`. A non-zeroed page table page has garbage PTEs, some of which might have V=1, leading to phantom mappings.

**Running out of memory during page table creation**
Each mapping can require up to 3 additional frames (for intermediate tables). If your frame allocator is nearly empty, `kalloc()` returns NULL and your `vm_map` needs to handle this gracefully.

---

## Further Reading

- **[RISC-V Privileged Specification, Section 4.4 (Sv39)](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf)** — The definitive reference for the page table format, the page walk algorithm, and PTE bit definitions. Read this carefully.
- **[CS:APP Chapter 9: Virtual Memory](https://en.wikipedia.org/wiki/Virtual_memory)** — The best textbook treatment of virtual memory. Covers address translation, TLBs, page faults, and memory-mapped files. Sections 9.3 through 9.7 are directly relevant.
- **[OSTEP Chapters 18-23](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-mechanism.pdf)** — Six chapters on virtual memory: address translation, paging, TLBs, smaller tables, swapping mechanisms, and swapping policies. Thorough and accessible.
- **[xv6 Book, Chapter 3: Page Tables](https://pdos.csail.mit.edu/6.828/2024/xv6/book-riscv-rev4.pdf)** — xv6's implementation of Sv39 page tables. Clear, with ASCII diagrams.
- **[xv6 source: `vm.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c)** — The page table manipulation functions. Read `walk()`, `mappages()`, and `uvmcreate()`.

---

*Next: [Chapter 11 — Kernel Address Space Design](ch11-kernel-address-space.md), where we design the memory layout, build the kernel page table, enable the MMU, and survive the most dangerous transition in OS development.*
