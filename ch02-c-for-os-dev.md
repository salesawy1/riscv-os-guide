# Chapter 2: C for OS Development

**[Difficulty: ★★☆☆☆]**

---

## Why This Chapter Exists

You know C. You've written functions, used pointers, allocated memory with `malloc`. But the C you wrote in school or at work is *hosted C* — C that runs on top of an operating system, with a standard library, with virtual memory protecting you from your mistakes, with an environment that initializes everything before `main()` is called.

The C you're about to write is different. It's C that runs on bare metal, where a stray pointer doesn't just segfault — it corrupts your kernel's data structures, overwrites your interrupt handler, or silently programs a hardware register to do something catastrophic. There's no segfault handler. There's no core dump. There's just a machine that suddenly does something inexplicable, and you staring at a black terminal wondering what happened.

This chapter is not a C tutorial. It's a focused review of the specific C features and techniques that matter in OS development — the ones that application programmers rarely use, and the ones that will bite you if you don't understand them at a hardware level. If you've never thought about what `volatile` actually means to the compiler, or why casting an integer to a pointer is sometimes exactly what you need to do, or how the compiler lays out a struct in memory down to the bit, this chapter is going to change how you think about C.

Think of this chapter as recalibrating your mental model. In application programming, C is a high-level language — you think in terms of variables, functions, and data structures, and the compiler figures out the rest. In OS programming, C is a thin veneer over assembly. You need to think about what the compiler *emits*, not just what your code *means*.

---

## Pointers as Addresses: The Real Story

In application programming, a pointer is "a variable that holds the address of another variable." That's a fine mental model when your OS manages memory for you. In OS programming, a pointer is a *machine word that the CPU feeds into the memory bus*. That address might point to RAM. It might point to a hardware device register. It might point to nothing at all. The CPU doesn't know the difference — it just issues a load or store to whatever address you give it, and the memory system figures out what to do with it.

This means that in OS development, you will regularly do things that application C considers insane:

**Casting integers to pointers.** The [UART](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver%E2%80%93transmitter) device on the QEMU `virt` machine lives at physical address `0x10000000`. To write a byte to the UART's transmit register, you need a pointer to that address. There's no variable there. There's no `malloc` that returned that address. You just *know* — from the hardware specification — that address `0x10000000` is the UART. So you cast:

The concept here is to take a known hardware address, cast it to a pointer of the appropriate type, and dereference it. When you write through that pointer, the CPU puts the address on the memory bus, the memory controller sees that it falls in the MMIO range, and routes the write to the UART hardware instead of to RAM.

This is completely normal in OS and embedded development. The C standard calls this "implementation-defined behavior" — it works on every real platform, but the standard won't guarantee it because the standard doesn't know your memory map. You'll do it constantly.

**Pointer arithmetic for navigating structures.** When you're managing page tables (Chapter 10), you'll compute addresses by combining a base address with offsets calculated from bitfields of a virtual address. You'll take a physical frame number, shift it left by 12 bits to get a physical address, cast it to a pointer, and index into a table. Every step is pointer arithmetic rooted in a deep understanding of the hardware's data structures.

**Pointers to specific memory regions.** Your linker script exports symbols like `_bss_start` and `_bss_end`. These aren't variables — they don't have storage. They're just addresses. To use them in C, you declare them as `extern char _bss_start[];` or `extern char _bss_end[];`. The "array of unknown size" trick is a C idiom for "this is an address, don't try to dereference it as a single value — just use it as a pointer." You then iterate from `_bss_start` to `_bss_end` and zero out every byte.

Wait — why `extern char _bss_start[]` and not `extern char *_bss_start`? This is subtle and important. If you declare it as a `char *`, the compiler thinks there's a *pointer variable* at the linker symbol's address — it will try to load the pointer's value from that memory location. But the linker symbol isn't a variable; it's just an address. There's no pointer stored in memory at `_bss_start`. The array declaration tells the compiler: "this symbol IS the address," not "this symbol is a variable that CONTAINS an address." The resulting machine code is different, and the pointer version will crash or give garbage.

> **Aside: Why C allows all this**
>
> C was designed at Bell Labs specifically to write Unix — an operating system. It was always meant to be a "portable assembly" that could express hardware-level operations. This is why C has raw pointers, allows arbitrary casts, and doesn't enforce memory safety. These aren't design flaws — they're features that exist because Dennis Ritchie was writing an OS and needed them. Every "unsafe" feature of C that modern languages try to eliminate is something that OS developers use on purpose, every day.
>
> Rust, which is increasingly popular for OS development, provides the same capabilities through `unsafe` blocks — but it forces you to be explicit about when you're doing something dangerous. It's a different trade-off, not a better or worse one for this level of work.

---

## Volatile: When the Compiler Is Too Smart

The C compiler is an aggressive optimizer. When it sees code like this conceptually — read a variable, check if it's zero, read it again, check if it's zero again — it says, "I already read this variable and it was zero. Nothing in this function changed it. The second read will return zero too. I'll just use the cached value."

For normal variables, this optimization is correct and desirable. For hardware registers, it's catastrophic.

Consider a UART status register. You want to wait until the UART is ready to transmit. Conceptually, you read the status register in a loop, checking a "ready" bit. The register is at a fixed memory address. From the compiler's perspective, you're reading the same memory location repeatedly in a loop, and nothing in your C code writes to that location. The optimizer concludes that the value will never change and replaces your loop with either an infinite loop (if the first read said "not ready") or a straight-through path (if the first read said "ready"). Either way, it's wrong — the hardware updates that register independently of your code.

**[`volatile`](https://en.cppreference.com/w/c/language/volatile) tells the compiler: "Every read and write to this variable must actually happen. Do not cache it. Do not reorder it. Do not optimize it away."**

More precisely, the C standard says that accesses to `volatile`-qualified objects are *side effects*, and the compiler must preserve all side effects in the order specified by the abstract machine (with some caveats about sequence points). This means:

1. Every read from a `volatile` variable generates an actual load instruction. The compiler cannot use a previously cached value.
2. Every write to a `volatile` variable generates an actual store instruction. The compiler cannot defer it, combine it with other writes, or eliminate it as "dead."
3. Volatile accesses are not reordered with respect to other volatile accesses. (Non-volatile accesses *can* still be reordered around volatile ones, though — `volatile` is weaker than a full memory barrier. More on this in the architecture chapter.)

### Where You'll Use Volatile

- **Every memory-mapped I/O register.** The UART data register, the UART status register, the CLINT timer registers, the PLIC configuration registers — all of these are `volatile`. If you forget `volatile` on an MMIO access, your code will work in debug mode (because `-O0` doesn't optimize away reads) and break in release mode (because `-O2` will). This is one of the most common and maddening bugs in OS development.

- **Variables shared between interrupt handlers and normal code.** When a timer interrupt fires, it might set a flag that your main loop checks. Without `volatile`, the compiler might hoist the flag check out of the loop, because it can "prove" that nothing in the loop body modifies the flag. It can't see the interrupt handler — that's an asynchronous event outside the compiler's analysis.

- **Spin-locks and synchronization primitives** (once you get to multicore, which is beyond our scope, but worth knowing).

### A Common Pattern

You'll see — and write — a lot of code that follows this pattern: define a pointer to a volatile unsigned integer (or byte, or whatever width the hardware register is) initialized to the hardware address, then read or write through that pointer.

The `volatile` is on the pointed-to type, not on the pointer itself. You want the *data at the address* to be treated as volatile, not the pointer variable itself. A "pointer-to-volatile" and a "volatile-pointer" are different things. You almost always want the former in OS development.

> **Aside: volatile is necessary but not sufficient**
>
> `volatile` prevents the *compiler* from reordering or eliding accesses. But modern CPUs also reorder memory accesses at the *hardware level*. On RISC-V, the memory model is relatively strict for regular memory (RVWMO — RISC-V Weak Memory Ordering), but for MMIO regions, the hardware's behavior depends on the PMA (Physical Memory Attributes). The QEMU `virt` machine marks device regions as "strongly ordered," so hardware reordering isn't an issue there.
>
> On more relaxed architectures (ARM, for instance), you'd need both `volatile` and explicit memory barriers (`fence` instructions on RISC-V) to ensure correct ordering. For our single-core, QEMU-based OS, `volatile` alone is sufficient for MMIO. But know that in production kernels, this is a much thornier topic.
>
> See CS:APP Chapter 12 (Concurrent Programming) for background on memory ordering, and the [RISC-V Unprivileged Specification](https://github.com/riscv/riscv-isa-manual/releases/download/Ratified-IMAFDQC/riscv-spec-20191213.pdf) Chapter 14 (RVWMO Memory Model) for the formal RISC-V model.

---

## Bitwise Operations: Talking to Hardware

Hardware registers are not integers. They're bags of bits, where each bit or group of bits means something different. A single 32-bit register might contain a "ready" flag in bit 0, an "interrupt enable" flag in bit 1, a 4-bit error code in bits 4–7, and a 16-bit data count in bits 16–31. You manipulate these fields with bitwise operations.

This is bread-and-butter OS programming. If you're rusty on bitwise ops, get comfortable now, because you'll use them in every single chapter from here on.

### The Operations

**AND (`&`)** — Mask off bits you don't care about. To extract bit 5 from a register value `x`: `x & (1 << 5)`. The result is either zero (bit was clear) or `(1 << 5)` (bit was set). If you want a 0/1 result, shift it back: `(x >> 5) & 1`.

**OR (`|`)** — Set bits. To set bit 3 in register `x`: `x | (1 << 3)`. This turns on bit 3 without disturbing any other bits. This is how you enable individual flags in a control register.

**XOR (`^`)** — Toggle bits. `x ^ (1 << 3)` flips bit 3. Less common in hardware manipulation, but you'll use it occasionally.

**NOT (`~`)** — Invert all bits. Used in combination with AND to clear specific bits: `x & ~(1 << 3)` clears bit 3. The pattern is "AND with the complement of the mask." This is the standard way to turn off a flag in a hardware register.

**Shift (`<<`, `>>`)** — Move bits left or right. `1 << n` creates a mask with only bit `n` set. `x >> n` moves bit `n` into the least-significant position so you can test or extract it. For multi-bit fields, you combine shift with mask: `(x >> 4) & 0xF` extracts bits 7–4 as a 4-bit value.

### Bit Field Extraction and Construction

When working with page table entries (Chapter 10), you'll need to build 64-bit values where specific bit ranges have specific meanings. The RISC-V Sv39 PTE format, for example, packs permission bits in the low 10 bits, and a physical page number in bits 10–53. To construct a PTE, you'll shift the physical page number left by 10 and OR in the permission bits. To read a PTE, you'll shift right by 10 and mask to extract the page number, or AND with individual bit masks to test permissions.

You'll do this *constantly*. Get to the point where building and tearing apart bit fields is automatic. A useful technique: define named constants for bit positions and masks. Instead of writing `(1 << 3)` everywhere and trying to remember what bit 3 means, define something like a constant for "PTE_READABLE" equal to `(1 << 1)`. Your code becomes self-documenting, and you stop making "off by one bit" errors.

> **Aside: C bit-fields — don't use them for hardware**
>
> C has a bit-field syntax in structs: `unsigned int ready : 1;` declares a 1-bit field. This is tempting for representing hardware registers, but don't use it. The C standard leaves the layout of bit-fields implementation-defined: the compiler chooses the order of fields, how they're packed, and which direction they fill. Different compilers — and different flags on the same compiler — can produce different layouts. When you're matching a hardware specification that says "bit 3 is the interrupt enable flag," you need precise control over bit positions. Manual shifting and masking gives you that control. Bit-fields don't.
>
> The Linux kernel avoids C bit-fields for hardware registers entirely, for exactly this reason.

---

## Structs and Memory Layout

In application C, you think of a struct as a bundle of related data. In OS C, you think of a struct as a *memory layout specification*. When you define a struct that maps to a hardware register block or an on-disk data structure, every field must be at a precise byte offset. The compiler's layout choices — padding, alignment, ordering — become your problem.

### Alignment and Padding

The compiler inserts padding between struct fields to satisfy alignment requirements. Consider conceptually a struct with a `char` field followed by a `uint32_t` field. On most architectures, `uint32_t` must be aligned to a 4-byte boundary. So the compiler inserts 3 bytes of padding after the `char` to push the `uint32_t` to offset 4. The struct's total size is 8 bytes, not 5.

For most kernel data structures, this padding is fine — the compiler knows what it's doing, and accessing misaligned data on RISC-V would cause a trap. But when you're matching an external specification — a device register block, an ELF header, a filesystem superblock — the layout must exactly match the spec. If the spec says a field is at offset 5, and the compiler puts it at offset 8 due to padding, your driver reads the wrong data.

### Controlling Layout

**`__attribute__((packed))`** tells GCC to eliminate all padding — fields are placed at the next available byte with no alignment consideration. This produces structs that exactly match their declared size, byte for byte. The trade-off is that accessing fields may generate slower code (because the compiler must handle potential misalignment) and on some architectures, misaligned access causes a trap. On RISC-V, misaligned access *may* be supported in hardware, *may* trap and be handled by firmware, or *may* just fault — it's implementation-defined. For hardware register structs, you typically don't need `packed` because hardware designers align their registers properly. For on-disk formats and network protocols, `packed` is sometimes necessary.

**`__attribute__((aligned(n)))`** forces a struct (or field) to be aligned to an `n`-byte boundary. You'll use this for structures that must be page-aligned, like page tables.

**Field ordering.** You can eliminate most padding by ordering fields from largest to smallest alignment. A struct with fields ordered as `uint64_t`, `uint32_t`, `uint16_t`, `uint8_t` has no internal padding. The reverse order would have padding everywhere.

### Using Structs as Hardware Overlays

A powerful pattern in OS development: define a struct whose fields correspond to the registers of a hardware device, then cast the device's base address to a pointer to that struct. Now you can access individual registers as struct fields. The UART, for instance, has registers at offsets 0, 1, 2, 3, etc., from its base address. Define a struct with `uint8_t` fields for each register, cast `0x10000000` to a pointer to that struct, and you can read/write registers by name.

The fields of this struct should be `volatile`, since they're MMIO registers. You can either make each field individually `volatile`, or cast to a pointer-to-volatile-struct.

> **Aside: `offsetof` — your friend for verification**
>
> The `offsetof` macro (from `<stddef.h>`, which is available in freestanding mode) tells you the byte offset of a field within a struct. When you're mapping a struct to a hardware register block or an on-disk format, use `offsetof` in a static assertion to verify at compile time that each field is at the expected offset. If you change the struct and a field moves, the static assertion will fire at compile time instead of producing a silent, maddening runtime bug.
>
> This is the kind of defensive practice that separates tolerable OS debugging from hair-pulling OS debugging.

---

## Function Pointers

In application C, function pointers are an advanced feature — used for callbacks, maybe for a plugin architecture. In OS development, they're fundamental infrastructure. You'll use them in at least three critical places:

**1. Interrupt dispatch tables.** When a trap occurs, your trap handler examines the cause and calls the appropriate handler function. Rather than a giant `switch` statement, you can build a table of function pointers indexed by cause number. This is faster (a single indirect call vs. a chain of comparisons) and more modular (you can register handlers at runtime).

**2. System call dispatch.** When user code executes `ecall`, your kernel receives a trap with a system call number. A table of function pointers maps system call numbers to handler functions.

**3. Virtual filesystem operations.** When your filesystem layer supports multiple filesystem types, each type provides its own `read`, `write`, `open`, and `close` functions. A struct of function pointers — essentially a vtable — lets you call the right implementation without knowing the concrete type. This is how Unix's VFS works, and it's how you'll build yours in Chapter 22.

The syntax for function pointers in C is admittedly ugly. A pointer to a function that takes an `int` and returns a `void*` is declared as: `void* (*func_ptr)(int)`. The key to reading these declarations is the "spiral rule" or the "cdecl" approach — start from the variable name, go right (seeing the parameter list), then go left (seeing the return type). Or use `typedef` to give the function pointer type a name and never think about the syntax again.

Make sure you're comfortable with:
- Declaring function pointer variables
- Declaring arrays of function pointers
- Declaring structs with function pointer fields
- Calling through a function pointer (the syntax is the same as calling a regular function — just use the variable name as if it were the function)
- The fact that `NULL` function pointers must be checked before calling, because calling through a null function pointer on bare metal will jump to address 0, which on RISC-V is... well, it's probably the debug ROM or unmapped memory. Either way, it won't end well.

---

## Inline Assembly

There are things C cannot express. C has no concept of the stack pointer. C cannot read or write CPU control registers. C cannot execute a trap-return instruction or a memory fence. For these operations, you need assembly — but you don't necessarily need to write a separate assembly file. GCC and Clang provide **inline assembly**, which lets you embed assembly instructions directly in C code.

### Why Inline Assembly Instead of Separate `.S` Files

Two reasons:

1. **The compiler can optimize around it.** When you put assembly in a separate file, the compiler treats every call to that function as a black box — it has to save all caller-saved registers, can't inline the function, and can't propagate constants through it. Inline assembly gives the compiler enough information to integrate the assembly into its optimization passes.

2. **You can use C variables directly.** Inline assembly can refer to C variables, and the compiler handles the register allocation for you. You tell the compiler "put this C variable into a register and give me the register name" instead of manually managing which registers hold what.

### The GCC Inline Assembly Syntax

[GCC inline assembly](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html) is notoriously opaque. Here's the anatomy:

```c
asm volatile (
    "assembly template"
    : output operands
    : input operands
    : clobber list
);
```

**The `volatile` keyword** on `asm` tells the compiler not to optimize away or reorder this assembly block. For most OS-level inline assembly (CSR reads/writes, fences, special instructions), you want `volatile`. Without it, the compiler might decide your assembly has no observable effect and eliminate it.

**The assembly template** is a string containing one or more assembly instructions, separated by `\n\t` (newline and tab, for readability in the compiler's assembly output). Operands are referred to by `%0`, `%1`, etc., matching their position in the operand lists.

**Output operands** tell the compiler where to put results. The syntax is `"constraint" (c_variable)`. The constraint letter tells the compiler what kind of location is acceptable:
- `"=r"` — any general-purpose register, write-only
- `"+r"` — any general-purpose register, read-write
- `"=m"` — a memory location

**Input operands** provide values to the assembly. Same constraint syntax but without `=` or `+`:
- `"r"` — put this value in a register
- `"i"` — an immediate (compile-time constant)

**The clobber list** tells the compiler which registers or resources the assembly modifies beyond the explicit outputs. `"memory"` tells the compiler that the assembly may read or write arbitrary memory, preventing the compiler from caching memory values across the asm block. `"cc"` says the condition codes are modified (less relevant on RISC-V, which doesn't have traditional flags).

### A Conceptual Example

Suppose you want to read the `mstatus` CSR (a RISC-V control register). You need to use the `csrr` instruction, which reads a CSR into a general-purpose register. In inline assembly, you'd:

1. Declare a C variable for the result
2. Write an `asm volatile` block with a `csrr` instruction
3. Use an output operand constraint of `"=r"` bound to your result variable
4. The `csrr` instruction refers to the output as `%0`

The compiler will choose a register, emit the `csrr` instruction using that register, and then treat your C variable as holding the result. You never had to manually choose a register or move values around.

Similarly, to write a CSR, you'd use a `csrw` instruction with an input operand. To read-modify-write (like setting specific bits in a CSR), you'd use `csrrs` or `csrrc`, or do a read, modify in C, and write back.

### What You'll Use Inline Assembly For

In your OS, inline assembly will appear in a few specific places:

- **[CSR](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf) access functions:** A small set of functions (or macros) that read/write specific CSRs: `mstatus`, `mepc`, `mcause`, `satp`, etc. You'll write these once and use them everywhere.
- **The `mret`/`sret` instruction:** Return from a trap. This can't be expressed in C.
- **The `ecall` instruction:** Trigger a system call from user mode. Can't be expressed in C.
- **The `fence` instruction:** [Memory barrier](https://en.wikipedia.org/wiki/Memory_barrier). C11 has `_Atomic` and `atomic_thread_fence`, but for OS development, you'll often use the fence directly.
- **The `wfi` instruction:** "Wait for interrupt" — puts the CPU in a low-power state until an interrupt arrives. Used in your idle loop.
- **The `sfence.vma` instruction:** Flush the [TLB (translation lookaside buffer)](https://en.wikipedia.org/wiki/Translation_lookaside_buffer) after modifying page tables. No C equivalent.

You will *not* need inline assembly for general computation. If you find yourself writing inline assembly for something that could be expressed in C, stop and use C. Inline assembly should be reserved for operations that genuinely require CPU instructions with no C equivalent.

> **Aside: Compiler-specific extensions vs. inline assembly**
>
> GCC and Clang provide built-in functions (intrinsics) for some CPU-specific operations. For example, `__builtin_ctz()` counts trailing zeros. However, RISC-V CSR access doesn't have standard compiler built-ins, so inline assembly is the way. Some projects use a wrapper library (like the `riscv` crate in Rust) that provides safe CSR access functions — in C, you'll write these wrappers yourself.
>
> The [xv6 source code](https://github.com/mit-pdos/xv6-riscv) has a header file (`riscv.h`) full of inline assembly wrappers for CSR access. You'll write something similar. Study theirs after you've written yours.

---

## The C Preprocessor in OS Development

The preprocessor plays a bigger role in OS code than in application code. You'll use it for:

**Constant definitions.** Hardware register addresses, bit positions, [page sizes](https://wiki.osdev.org/Paging), maximum number of processes — all defined as macros. The preprocessor works in terms of text substitution, which means you can use these constants in both C code and in assembly files (`.S` files, with capital S, are preprocessed).

**Conditional compilation.** `#ifdef DEBUG` to enable verbose logging. `#ifdef CONFIG_SMP` to include multicore code. This lets a single codebase support different configurations.

**Header guards.** Every header file needs an `#ifndef HEADER_H / #define HEADER_H / ... / #endif` guard (or `#pragma once` if you're okay with a non-standard but universally supported extension). In OS code with complex header dependencies, forgetting a guard leads to redefinition errors.

**Macro "functions."** For very small operations — like extracting a bit field or computing an address — macros can be appropriate. They're expanded inline by the preprocessor, so there's no function call overhead, and they work in contexts where the compiler's optimizer might not be able to inline a regular function (though modern compilers inline small functions reliably). The danger of macros is well-known: no type safety, no scoping, argument double-evaluation. Use `static inline` functions where you can; reserve macros for cases where you need them (like sharing constants with assembly).

**`static_assert` (C11).** Compile-time assertions. Use these to verify your struct layouts, your assumptions about type sizes, and your constant definitions. A `static_assert(sizeof(struct pte) == 8, "PTE must be 8 bytes")` in your [page table](https://wiki.osdev.org/Paging#The_Page_Table) code catches layout errors at compile time instead of producing inscrutable runtime bugs.

---

## Types, Sizes, and the Illusion of Portability

In application C, you might use `int` and assume it's 32 bits. In OS development, you never assume — you use fixed-width types exclusively: `uint8_t`, `uint16_t`, `uint32_t`, `uint64_t`, and their signed variants. Every register is a specific width. Every hardware field is a specific number of bits. An `int` is "at least 16 bits" by the C standard — that ambiguity is unacceptable when you're programming to a hardware specification.

On RV64, pointers are 64 bits. `size_t` and `uintptr_t` are 64 bits. When you cast between pointers and integers (which you will, constantly, because addresses are both), use `uintptr_t` — it's guaranteed to be wide enough to hold a pointer value.

Another type to know: `uintptr_t` vs `ptrdiff_t`. `uintptr_t` is an unsigned integer that can hold a [pointer](https://en.wikipedia.org/wiki/Pointer_%28computer_programming%29). `ptrdiff_t` is a signed integer representing the difference between two pointers. For OS work, `uintptr_t` is your workhorse — you'll use it whenever you need to treat an address as a number (for bit manipulation, arithmetic, or comparison).

The `NULL` pointer: in C, `NULL` is typically defined as `((void *)0)`. In a freestanding environment, you should provide your own definition in your headers, or use `<stddef.h>` (which is available in freestanding mode). On RISC-V, address 0 is typically the debug ROM or unmapped memory, so dereferencing NULL will cause a load/store access fault — which is actually helpful, because your trap handler will catch it. But don't rely on this; check for NULL explicitly.

---

## The Stack: What You Never Think About

In application programming, the stack just *exists*. The OS allocates it, sets up the stack pointer, and your functions push and pop frames without you thinking about it. In OS development, there is no stack until you create one.

When your CPU comes out of reset, the stack pointer (`sp`, register `x2` in RISC-V) contains... whatever it contains. Maybe zero. Maybe a random value from the previous power cycle. Your very first action in your boot assembly stub is to set `sp` to point to a valid region of memory.

What constitutes a "valid region"? You need a chunk of memory that:

1. Is actually RAM (not an MMIO region, not unmapped space)
2. Is large enough for your kernel's stack usage (4 KiB is a common starting point for a kernel stack; 8 KiB or 16 KiB is safer)
3. Won't be overwritten by other things (not in the middle of your kernel's `.text` or `.data` sections)

A common approach: reserve a region in your `.bss` section for the stack. Declare a global array of bytes in your assembly boot stub (or in C, but you need to reference it in assembly before C code runs), and set `sp` to the *top* of that array. Stacks grow downward on RISC-V — `sp` starts at a high address and decreases as functions are called. So you set `sp` to the end of the array, not the beginning.

Once processes exist (Chapter 13), each process will have its own kernel stack. The mechanism is the same: allocate a page of memory, set `sp` to the top of it.

**Stack overflow** is a silent killer in OS development. There's no [guard page](https://en.wikipedia.org/wiki/Buffer_overflow_protection#Guard_page) (unless you create one). If a deep call chain or a large local variable pushes the stack past its allocated region, it overwrites whatever's below — probably another process's kernel stack, or your kernel's data structures. The symptom is insane, seemingly random corruption. When you suspect a stack overflow, increase your stack size and see if the bug disappears. In Chapter 10, when you have virtual memory, you can put an unmapped guard page below each stack to turn silent corruption into a clean trap.

> **Aside: How function calls actually work on RISC-V**
>
> The RISC-V calling convention (defined in the RISC-V ELF psABI specification) dictates how functions pass arguments, return values, and manage registers:
>
> - **Arguments:** The first 8 integer arguments go in registers `a0`–`a7` (that's `x10`–`x17`). Additional arguments go on the stack. Floating-point arguments go in `fa0`–`fa7`.
> - **Return values:** Integer return values in `a0` (and `a1` for 128-bit returns). Floating-point in `fa0`.
> - **Caller-saved registers:** `t0`–`t6` (temporaries) and `a0`–`a7` (arguments) — the callee may overwrite these, so the caller must save them if it needs them after the call.
> - **Callee-saved registers:** `s0`–`s11` (saved registers) — the callee must preserve these. If it uses them, it saves them on the stack in its prologue and restores them in its epilogue. ([RISC-V ABI](https://github.com/riscv-non-isa/riscv-elf-psabi-doc) calling convention)
> - **Return address:** `ra` (register `x1`) holds the return address. `jal` (jump and link) stores the return address in `ra` before jumping.
> - **Stack pointer:** `sp` must be 16-byte aligned at function entry.
> - **Frame pointer:** `s0` (also called `fp`) is conventionally used as the frame pointer, though optimized code may omit it. For debugging, compile with `-fno-omit-frame-pointer` to keep it.
>
> You need to understand this convention intimately for Chapter 14 (Context Switching), where you'll manually save and restore every register. But even before that, understanding the calling convention helps you read disassembly output and debug crashes.
>
> See the RISC-V ELF psABI Specification for the complete details, and CS:APP Chapter 3 (Machine-Level Representation of Programs) for general background on calling conventions and stack management.

---

## Memory You Must Provide

Remember `-ffreestanding`? The compiler *promises* not to assume the standard library exists, but it *reserves the right* to generate calls to a few fundamental functions. These are:

- **`memcpy(dest, src, n)`** — Copy `n` bytes from `src` to `dest`. The compiler generates this for struct assignment, large constant initializers, and sometimes for passing structs by value.
- **`memset(dest, c, n)`** — Set `n` bytes at `dest` to the value `c`. Generated for zero-initialization of local variables, arrays, and structs.
- **`memmove(dest, src, n)`** — Like `memcpy`, but handles overlapping source and destination. The compiler may generate this for some struct operations.
- **`memcmp(a, b, n)`** — Compare `n` bytes. Less commonly generated by the compiler, but you'll want it anyway.

You must implement these yourself. If you don't, you'll get linker errors like "undefined reference to `memcpy`" — not from code you wrote, but from code the *compiler generated* behind your back.

The implementations are straightforward: `memset` is a loop that writes a byte value. `memcpy` is a loop that copies bytes. `memmove` is `memcpy` but handles the case where source and destination overlap (copy backward if destination is above source). These are your first C functions, and they'll be among the most-called functions in your entire kernel.

A subtlety: for performance, real implementations of `memcpy` copy word-sized (8-byte on RV64) chunks when alignment allows, falling back to byte-by-byte for unaligned tails. For your initial implementation, byte-by-byte is fine. Optimize later if you care — but frankly, for a teaching OS, you won't notice the difference.

---

## Strings Without a Standard Library

You'll also need basic string functions: `strlen`, `strcmp`, `strncpy`, `strncmp`. These are not generated by the compiler, but you'll use them constantly once you have a console, a filesystem, and a shell. Implement them as you need them. Don't implement the entire `<string.h>` up front — that's premature. Write `strlen` when you first need it, `strcmp` when you first compare strings, and so on.

One important note: never use `strcpy` — use `strncpy` or, better yet, your own bounded copy function. Buffer overflows in kernel code are not "security vulnerabilities" in the usual sense (there's no attacker at this point), but they're *bugs*, and they'll corrupt your kernel's memory in ways that are nearly impossible to debug. Get in the habit of always passing buffer lengths.

---

## A Word on `static` and Linkage

In C, `static` has two meanings depending on context:

**At file scope** (outside any function): `static` gives the variable or function *internal linkage* — it's only visible within that translation unit (`.c` file). This is the primary encapsulation mechanism in C. Use it for every function and variable that doesn't need to be accessed from other files. This prevents name collisions (you can have a `static` helper function named `init()` in every file without conflict) and enables the compiler to inline or eliminate functions it can prove are only called locally.

**At block scope** (inside a function): `static` gives the variable *static storage duration* — it persists across function calls, initialized once. Less common in kernel code, but you might use it for "initialized on first call" patterns.

In OS development, where your entire codebase links into a single binary, name collisions are a real hazard. Two files that both define a global function `init()` will cause a linker error (or worse, one silently shadows the other). Make everything `static` by default; only remove `static` when you need cross-file access. This is the C equivalent of "private by default."

---

## Conceptual Exercises

1. **Why does `volatile` matter for the UART status register but not for a local variable?** What specifically does the compiler do differently for volatile vs. non-volatile accesses? How would you verify the difference? (Hint: `objdump -d`)

2. **You declare `extern char _bss_start;` (not `_bss_start[]`). What goes wrong?** Trace through what the compiler generates for `&_bss_start` vs. `_bss_start` in each case. Which one gives you the address of the linker symbol?

3. **A struct has these fields in order: `uint8_t a; uint64_t b; uint8_t c; uint32_t d;`** What is the struct's size with default alignment? What is it with `packed`? What would you reorder the fields to in order to minimize padding without packing?

4. **Why is `memcpy` needed even though you never call it in your code?** Under what circumstances does the compiler generate a `memcpy` call? What happens if you don't provide one?

5. **You write an inline assembly block that reads a CSR but forget the `volatile` keyword on the `asm` statement.** What might the compiler do if it determines the result is unused? What about if you read the same CSR twice in a row — could the compiler eliminate the second read?

6. **Your kernel stack is 4096 bytes. A function declares a local array of 4000 bytes.** What happens? How would you detect this problem? What's the fix?

7. **Explain the difference between `volatile uint32_t *reg` and `uint32_t * volatile reg`.** Which one do you want for MMIO, and why?

---

## Build This

No new code to run this chapter — this is about making sure your mental model is calibrated before you start writing real kernel code.

**Implement the following in your project (they'll be needed starting in Chapter 5):**

1. **Write `memset`, `memcpy`, `memmove`, and `memcmp`.** Put them in a file like `string.c` or `mem.c`. Keep the implementations simple (byte-by-byte is fine). Make sure they match the standard signatures with the correct types.

2. **Write a header with CSR access macros or inline functions.** Create a header (something like `riscv.h`) with inline assembly wrappers for:
   - Reading a CSR: a function that takes no arguments and returns `uint64_t`, using `csrr`
   - Writing a CSR: a function that takes a `uint64_t` and uses `csrw`
   - Setting bits in a CSR: a function that uses `csrs` (CSR set bits)
   - Clearing bits in a CSR: a function that uses `csrc` (CSR clear bits)

   You'll need these for specific CSRs: `mstatus`, `mepc`, `mcause`, `mtvec`, `mie`, `mip`, `satp`, `stvec`, `sepc`, `scause`, `sie`, `sip`. Don't write them all now — just set up the pattern for one or two. You'll add more as you need them.

3. **Write a `types.h`** (or use `<stdint.h>` — your choice) and a `defs.h` with common type definitions and macros you anticipate needing. Don't over-engineer this — start minimal and add to it.

4. **Verify your struct layouts.** Write a test struct with mixed-size fields. Use `offsetof` and `sizeof` in `static_assert` declarations to verify that fields are where you expect them. This is practice for when you map structs to hardware register blocks.

**Checkpoint:** Your project should compile cleanly with all the new files. Run `riscv64-unknown-elf-objdump -t build/kernel.elf | grep memcpy` and verify that `memcpy` appears as a defined symbol. Your CSR access wrappers should compile without errors (even though nothing calls them yet).

---

## When Things Go Wrong

**"undefined reference to `memcpy`" or `memset`**
You haven't implemented these yet, but the compiler generated a call. Either implement them now (you'll need them anyway) or figure out what code is triggering the compiler to generate the call and restructure it. Usually it's struct assignment or local array initialization.

**Inline assembly errors: "impossible constraint"**
Your operand constraints don't match the instruction's requirements. For CSR instructions, the destination/source must be a general-purpose register — use `"r"` constraints. If you're using an immediate operand where a register is required (or vice versa), you'll get this error.

**Inline assembly errors: "invalid CSR name"**
CSR names in assembly are case-sensitive and must match the RISC-V specification exactly. It's `mstatus`, not `MSTATUS`. Some assembler versions require the numeric CSR address (e.g., `0x300` for `mstatus`) instead of the name — check your assembler's version and documentation.

**"warning: 'packed' attribute ignored"**
This can happen if you apply `packed` to a typedef instead of a struct definition. Apply it to the struct itself.

**Strange behavior only at `-O2` but not `-O0`**
Almost certainly a missing `volatile`. At `-O0`, the compiler doesn't optimize away reads or writes. At `-O2`, it aggressively caches values in registers, reorders operations, and eliminates "redundant" loads. If your code works at `-O0` and breaks at `-O2`, find the MMIO access that's missing `volatile`.

---

## Further Reading

- **CS:APP Chapter 3: Machine-Level Representation of Programs** — How C maps to assembly, calling conventions, stack frames, data representations. Essential background.
- **RISC-V ELF psABI Specification** — The calling convention, register usage, and data type layout for RISC-V. Available on GitHub from the `riscv-non-isa` organization.
- **GCC Manual, Section 6.47: Extended Asm** — The authoritative reference for GCC inline assembly syntax. Dense but complete.
- **The xv6 `riscv.h` header** — A clean, readable set of inline assembly wrappers for RISC-V CSR access. Study it after writing your own.
- **OSTEP Chapter 6: Mechanism: Limited Direct Execution** — Arpaci-Dusseau's introduction to system calls, traps, and why the OS needs hardware support. Good warmup for Chapter 7.

---

*Next: [Chapter 3 — How Computers Boot](ch03-how-computers-boot.md), where we trace the journey from electrons to your `kernel_main()` and learn why your kernel is never the first thing to run.*
