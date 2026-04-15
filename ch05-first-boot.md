# Chapter 5: First Boot

**[Difficulty: ★★☆☆☆]**

---

## Why This Chapter Exists

Four chapters of theory. Four chapters of "here's what the hardware does," "here's what the tools do," "here's how C works at the metal level." You've been patient. Now we put it together.

This chapter is where your operating system is born. Not as a concept, not as a diagram, but as a binary that runs on a (virtual) machine and does something observable. By the end of this chapter, you'll have a kernel that boots on QEMU, reaches C code, and proves it got there. It won't do anything useful yet — that's Chapter 6 — but it will *exist*, and you will have verified, with your own eyes and your own debugger, that it works.

The work here is small in terms of lines of code. Your boot stub is maybe 30 lines of assembly. Your `kernel_main` is essentially empty. But every one of those lines carries weight, because there's nothing underneath you. No runtime. No OS. No safety net. If the stack pointer is wrong by one byte, if the [BSS](https://en.wikipedia.org/wiki/.bss) loop has an off-by-one, if the linker script puts the wrong section first — the machine silently does the wrong thing and you stare at a blank terminal wondering why.

This is the chapter where you learn the most important skill in OS development: methodical debugging from first principles. When you have no output, no error messages, and no stack traces, you have GDB, you have `objdump`, and you have your understanding of what the hardware should be doing at each step. That's enough. It's always enough, if your understanding is right.

---

## The Big Picture

Here is the complete chain of events we need to make happen:

```
QEMU loads kernel.elf into RAM at 0x8000_0000
    │
    ▼
CPU starts at reset vector (0x1000)
    │
    ▼
Trampoline at 0x1000 jumps to 0x8000_0000
    │
    ▼
_start (boot.S):
    ├── 1. Park non-boot harts
    ├── 2. Set the stack pointer
    ├── 3. Zero the .bss section
    └── 4. Call kernel_main
            │
            ▼
        kernel_main (main.c):
            └── We're in C. We made it.
```

That's five transitions, and you need every single one of them to work. Let's go through each piece in detail.

---

## Step 0: The Linker Script (Revisited)

You wrote a linker script in Chapter 1. Now let's make sure it's actually correct for what we need. The linker script is the foundation — if it's wrong, everything else breaks in confusing ways.

### What Your Linker Script Must Guarantee

**1. The entry point is `_start`.** The `ENTRY(_start)` directive tells the linker to record `_start` as the ELF entry point. QEMU reads this and sets the program counter to the address of `_start` (via the trampoline at `0x1000`). If you forget `ENTRY(_start)`, the linker uses a default entry point, which is almost certainly wrong.

**2. The `.text` section starts at `0x80000000`.** This is where QEMU's `virt` machine places DRAM. The location counter (`. = 0x80000000;`) must be set before the `.text` output section.

**3. `_start` is the very first thing in `.text`.** This is subtle but critical. Your `.text` section will contain code from multiple object files: `boot.o`, `main.o`, `uart.o`, etc. The linker places them in the order they appear in the linker script's input section list, or in the order they're passed on the command line. You need `boot.o`'s `.text` to come first, because the CPU starts executing at `0x80000000`, and that address must contain the first instruction of `_start`.

There are two ways to ensure this:

- **Order your object files carefully** in the linker command: put `boot.o` first. The linker processes input sections in the order they appear, so `boot.o`'s `.text` goes first.
- **Use a separate input section name.** In `boot.S`, place `_start` in a section like `.text.boot` (via `.section .text.boot`), and in your linker script, list `.text.boot` before `.text`:

```
.text : {
    *(.text.boot)
    *(.text .text.*)
}
```

This guarantees that `_start` comes first regardless of link order. This is the more robust approach, and it's what we'll use.

**4. Sections are page-aligned.** Each section should start on a 4096-byte boundary (`ALIGN(4096)` or `ALIGN(0x1000)`). This isn't strictly necessary yet, but when we enable paging in Chapter 10, we'll need `.text` and `.rodata` (read-only, executable) to be on different pages from `.data` (read-write, not executable) so we can set different permissions. Setting up alignment now means we don't have to restructure the linker script later.

**5. BSS symbols are exported.** You need `_bss_start` and `_bss_end` symbols that your boot assembly can reference. These mark the [BSS](https://en.wikipedia.org/wiki/.bss) region that must be zeroed.

**6. A kernel end symbol is exported.** `_kernel_end` (or similar) marks the end of all kernel sections. Everything above this address is free memory, available for your physical memory allocator (Chapter 9). Align this to a page boundary too.

### What the Linker Script Produces

When the linker runs with your script, it produces an ELF with:

- Program headers that tell QEMU to load the kernel at `0x80000000`
- An entry point of `0x80000000` (the address of `_start`)
- Sections laid out sequentially: `.text` → `.rodata` → `.data` → `.bss`
- Symbols `_bss_start`, `_bss_end`, and `_kernel_end` with correct addresses

Verify all of this with `readelf -l` (program headers), `readelf -h` (entry point), and `objdump -t | grep _bss` (symbols). If any of these are wrong, fix the linker script before proceeding. Debugging a broken linker script by observing runtime behavior is like trying to diagnose engine problems by listening to the exhaust — possible, but agonizing.

---

## Step 1: Parking Non-Boot Harts

Even with `-smp 1`, it's good practice to handle the case where multiple [harts](https://en.wikipedia.org/wiki/Hardware_thread) (hardware threads) try to boot. On real hardware and on some QEMU configurations, all harts start executing simultaneously at the reset vector. You only want one hart (hart 0, by convention) to proceed with boot. All others should park themselves.

At the very start of `_start`, before doing anything else:

1. Read the hart ID. You can get it from register `a0` (QEMU passes the hart ID there) or from the `mhartid` CSR.
2. If the hart ID is not 0, jump to an infinite loop (`wfi` in a loop is even better — it puts the hart in a low-power sleep until an interrupt wakes it).
3. If the hart ID is 0, continue with the boot sequence.

This is three or four instructions. It costs nothing and prevents mysterious double-initialization bugs if you ever change your `-smp` setting.

The [`wfi`](https://en.wikipedia.org/wiki/Instruction_set_architecture) (Wait for Interrupt) instruction is preferable to a bare spin loop (`j .`) because it lets QEMU skip emulating the spinning hart, which saves host CPU time. On real hardware, `wfi` puts the core in a low-power state. On QEMU, it makes the emulator more efficient. Use it.

> **Aside: Waking parked harts**
>
> When you eventually implement multicore (beyond this guide's scope), you'll wake parked harts by sending them an Inter-Processor Interrupt (IPI) via the [CLINT](https://sifive.cdn.prismic.io/sifive/0d163928-2128-42be-a75a-464df65e04e0_sifive-interrupt-cookbook.pdf) (Core Local Interruptor)'s software interrupt registers. The parked hart's `wfi` will return when the interrupt arrives, and it can then proceed to its own initialization. This is the standard pattern in RISC-V multicore boot: one hart boots, sets everything up, then wakes the others when it's ready.
>
> Linux's RISC-V port does exactly this. The boot hart runs the full kernel initialization, then uses SBI calls to start the secondary harts.

---

## Step 2: Setting Up the Stack

The stack pointer must point to valid, writable memory before you can call any function (including `kernel_main`). On RISC-V, the stack grows downward — `sp` starts at a high address and decreases as data is pushed.

### Allocating Stack Space

You need to reserve memory for the stack. The cleanest approach: declare a block of memory in the `.bss` section of your boot assembly file. The `.bss` section is for uninitialized data — it takes up no space in the ELF file but is allocated in memory. A stack doesn't need to be initialized to any particular value, so `.bss` is perfect.

In your `boot.S`, declare a region something like:

```
.section .bss
.align 16
stack_bottom:
    .space 4096 * 4    # 16 KiB stack
stack_top:
```

Then in your `_start` code:

```
la sp, stack_top
```

The [`la`](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html) (load address) [pseudoinstruction](https://github.com/riscv/riscv-isa-manual) loads the address of `stack_top` into `sp`. Since the stack grows downward, `sp` starts at the *top* of the allocated region. As functions are called, `sp` decreases toward `stack_bottom`.

<details>
<summary>Why is 16 KiB better than 4 KiB for the kernel stack?</summary>
<div>

4096 bytes (one page) is the absolute minimum — enough for simple call chains but risky for deep recursion or large local variables. Linux uses 16 KiB for its kernel stacks. A larger stack prevents overflows that are hard to debug. You could start with 4 KiB and grow it, but 16 KiB is a reasonable upfront choice for a teaching OS.

</div>
</details>

<details>
<summary>What happens if the stack pointer isn't 16-byte aligned?</summary>
<div>

Many instructions tolerate misalignment, so you won't always see immediate crashes. But the calling convention requires 16-byte alignment, and some operations (floating-point, atomics) fault on misaligned `sp`. The `.align 16` directive before the stack region guarantees both `stack_bottom` and `stack_top` are aligned, as long as the stack size is a multiple of 16 (which 16 KiB is).

</div>
</details>

> **Aside: Why the stack grows downward**
>
> This is a convention, not a hardware requirement. RISC-V hardware doesn't enforce a stack direction — `sp` is just a register. The downward growth convention comes from historical reasons: early systems placed the stack at the top of memory and the heap at the bottom, growing toward each other. If either grew too large, they'd collide, which was at least detectable. The convention stuck.
>
> Some architectures (notably PA-RISC and the original System/370) used upward-growing stacks. There's no technical superiority to either direction — it's purely conventional. But on RISC-V, the ABI mandates downward growth, and all toolchain assumptions depend on it. Don't fight it.

---

## Step 3: Zeroing the BSS

The C standard guarantees that uninitialized global and static variables are zero-initialized. In a hosted environment, the OS loader handles this. On bare metal, you handle it. The [`.bss`](https://en.wikipedia.org/wiki/.bss) section in your [ELF](https://refspecs.linuxfoundation.org/elf/elf.pdf) file specifies the size of this region but contains no data — the ELF loader (QEMU) allocates space for it in RAM but doesn't write zeros. The RAM might already be zero (QEMU typically zeros memory), but you must not rely on this — real hardware won't, and even QEMU's behavior might change.

Your boot stub needs to:

1. Load the address of `_bss_start` into a register
2. Load the address of `_bss_end` into another register
3. Loop, storing zero to each doubleword (8-byte) location from `_bss_start` to `_bss_end`

<details>
<summary>Why use 8-byte stores instead of 1-byte stores for BSS zeroing?</summary>
<div>

Storing 8 bytes at a time is 8x faster than 1-byte stores. The `.bss` section is page-aligned at both ends (from your linker script), so both `_bss_start` and `_bss_end` are 8-byte aligned. This makes `sd` (store doubleword) safe and efficient.

</div>
</details>

### The Loop

Conceptually, the loop is:

```
    load address of _bss_start into register A
    load address of _bss_end into register B
loop:
    if A >= B, jump to done
    store zero (x0) to the address in A
    add 8 to A
    jump to loop
done:
```

This is straightforward, but there are two pitfalls:

<details>
<summary>What's the difference between `la` and `ld` when loading BSS addresses?</summary>
<div>

`la` (load address) puts the symbol's address into a register — what you want for a pointer to BSS. `ld` (load from memory) tries to fetch a value FROM that address, giving you garbage. For linker symbols, always use `la`.

</div>
</details>

<details>
<summary>What happens if the `.bss` section is empty?</summary>
<div>

Then `_bss_start == _bss_end`. Your loop condition must check this: a `bge` (branch if >=) at the start handles it by jumping to `done` immediately if start >= end. Empty BSS is valid; your loop just does nothing.

</div>
</details>

<details>
<summary>What bugs occur if you skip BSS zeroing?</summary>
<div>

Global variables won't be zero-initialized. A counter expected to be 0 might be random. A pointer might point to garbage, causing silent memory corruption. A flag might appear "true" when it should be "false," causing wrong code paths. These bugs are intermittent — they depend on whatever was in RAM. QEMU often zeros RAM at startup, masking the problem, but real hardware won't. BSS zeroing costs microseconds and prevents an entire class of bugs.

</div>
</details>

---

## Step 4: Calling kernel_main

Once the stack is set and BSS is zeroed, you can call your C entry point. In assembly:

```
call kernel_main
```

<details>
<summary>Why does the boot stub need a spin loop after `call kernel_main`?</summary>
<div>

The `call` instruction stores the return address in `ra` and jumps to `kernel_main`. If `kernel_main` returns (which it shouldn't, but if it does), the CPU would execute random memory. A `wfi` + `j` loop at the return point prevents this. `wfi` puts the core into low-power sleep, saving CPU time. On real hardware, if somehow execution reaches the spin loop, at least the core idles cleanly instead of corrupting memory.

</div>
</details>

<details>
<summary>Why not just name it `main` like normal C programs?</summary>
<div>

You could. But `main` in hosted C programs can have special compiler-assigned semantics (return type assumptions, implicit handling). Using `kernel_main` clarifies that this is a freestanding entry point, not a hosted main. It's a naming convention that prevents confusion. xv6 uses `main()`, but `kernel_main` is common in OS tutorials. Either works — just be consistent.

</div>
</details>

---

## The Complete Boot Sequence: What to Verify

At this point, let me be very explicit about what should happen when you run your kernel, because this is where subtle bugs hide and you need a systematic way to find them.

### The Assembly You Should See

Run `riscv64-unknown-elf-objdump -d kernel.elf` and examine the disassembly at `0x80000000`. You should see:

1. **First few instructions:** Reading `mhartid` (or using `a0`), comparing with zero, branching to a spin loop if not hart 0.
2. **Stack setup:** An `la sp, stack_top` sequence (which expands to `auipc` + `addi` or `lui` + `addi`).
3. **BSS zeroing:** An `la` for `_bss_start`, an `la` for `_bss_end`, a loop with `sd x0` and an increment.
4. **Function call:** A `call kernel_main` (expanding to `auipc` + `jalr` or `jal`).
5. **Spin loop:** `wfi` + `j` after the call.

If the disassembly doesn't show this sequence at `0x80000000`, something is wrong — either your linker script isn't placing `.text.boot` first, or your Makefile isn't compiling `boot.S` correctly.

### GDB Verification

This is the most reliable way to verify your boot sequence works:

**Terminal 1:**
```
make debug
# (QEMU starts, paused, waiting for GDB)
```

**Terminal 2:**
```
riscv64-unknown-elf-gdb kernel.elf
(gdb) target remote :1234
```

Now you're connected to the paused CPU. The PC is at the reset vector (`0x1000`), not at your code yet. Let's step through:

```
(gdb) info registers pc
# Should show 0x1000

(gdb) break _start
(gdb) continue
# Should hit the breakpoint at 0x80000000

(gdb) info registers sp
# sp is probably 0 or garbage — we haven't set it yet

(gdb) stepi 20
# Step through 20 instructions (adjust as needed)
# Watch sp change to a valid address
# Watch the BSS zeroing loop execute

(gdb) break kernel_main
(gdb) continue
# Should hit the breakpoint at kernel_main

(gdb) info registers sp
# sp should now point to your stack_top address

(gdb) info registers pc
# Should be at kernel_main's address
```

If GDB hits the `kernel_main` breakpoint with a valid `sp`, your boot sequence is correct. Every step from reset vector to C code has worked.

### Verifying Section Placement

```
riscv64-unknown-elf-objdump -h kernel.elf
```

This shows all sections with their addresses, sizes, and flags. Check:

- `.text` starts at `0x80000000`
- `.rodata` follows `.text`, page-aligned
- `.data` follows `.rodata`, page-aligned
- `.bss` follows `.data`, page-aligned
- All sections are within the DRAM range (`0x80000000` to `0x88000000` for 128 MiB)

```
riscv64-unknown-elf-readelf -h kernel.elf
```

Check:
- Entry point address: `0x80000000`
- Machine: RISC-V
- Class: ELF64

```
riscv64-unknown-elf-objdump -t kernel.elf | grep -E "_bss_start|_bss_end|_kernel_end|_start|kernel_main"
```

Check that all these symbols exist and have reasonable addresses.

---

## What kernel_main Should Do (For Now)

Right now, `kernel_main` has nothing to do. You have no UART driver yet (that's Chapter 6), so you can't print anything. You have no trap handler (Chapter 7), so you shouldn't enable interrupts. Your `kernel_main` should just loop:

```c
void kernel_main(void) {
    // We'll fill this in over the next chapters.
    // For now, just prove we got here.
    while (1) {
        // wfi assembly instruction here, or just spin
    }
}
```

The proof that you got here is the GDB breakpoint. In the next chapter, you'll replace this empty loop with UART initialization and a `kprintf("Hello from the kernel!\n")` that gives you visible output.

> **Aside: The emotional milestone**
>
> I want to acknowledge something: this checkpoint feels anticlimactic. You've read four chapters, set up a cross-compiler, written a linker script, written assembly, and what do you have? A blank terminal. No output. Nothing visible.
>
> But if you did the GDB verification and hit the `kernel_main` breakpoint — you've accomplished something genuinely non-trivial. You've taken a raw CPU with no software, no runtime, no OS, and made it execute your C function. Every operating system in history started with this exact moment: bare metal to C, nothing above you, nothing below you but silicon. The UART comes next chapter, and the moment you see characters appear on the terminal, it'll feel like magic. Because in a very real sense, you created the universe in which that output is possible.
>
> xv6's first sign of life is also a UART write. Linux's is a serial console message. Every OS developer remembers the first time they saw output from their kernel.

---

<details>
<summary>How does `la` expand when RISC-V instructions only have 20-bit immediates?</summary>
<div>

`la rd, symbol` loads a 64-bit address into `rd`, but RISC-V immediates are only 20 bits. The assembler expands it to two instructions: `lui` (load upper immediate, 20 bits) + `addi` (add immediate, 12 bits). Or, for PC-relative addresses: `auipc` (add upper immediate to PC, 20 bits) + `addi` (12 bits). With `-mno-relax`, you get the full two-instruction sequence; with linker relaxation enabled, the linker might collapse it to one instruction if the symbol is close.

</div>
</details>

<details>
<summary>Why does it matter to understand how `la` expands to real instructions?</summary>
<div>

When debugging with `objdump`, you'll see `auipc` + `addi` sequences instead of `la`. If the offsets are wrong, your address is wrong. Verify critical addresses with GDB: after `la sp, stack_top` executes, check `info registers sp` against `objdump -t` output. Mismatched stack pointers or BSS addresses cause catastrophic bugs that are hard to trace.

</div>
</details>

---

## Multi-Hart Considerations

We mentioned parking non-boot harts in Step 1. Let's think about this more carefully, because even though we're running with `-smp 1`, the logic reveals important concepts.

<details>
<summary>Why use the `mhartid` CSR instead of `a0` to get the hart ID?</summary>
<div>

`a0` is a convention from QEMU/SBI — they pass the hart ID in `a0` at boot. But it's not hardware-guaranteed and could be clobbered. The `mhartid` CSR is read-only and always reflects the current hart's ID. Reading it via `csrr` is more robust — it's the hardware-guaranteed way.

</div>
</details>

### The Parking Loop

Non-boot harts should execute:

```
    csrr a0, mhartid     # Read hart ID
    bnez a0, park         # If not hart 0, park
    # ... boot sequence for hart 0 ...
park:
    wfi                   # Wait for interrupt
    j park                # If wfi returns spuriously, wait again
```

`wfi` can return spuriously (without an interrupt actually being pending). The `j park` after `wfi` handles this by going back to `wfi`. This is the standard `wfi` usage pattern — always put it in a loop.

---

## The Kernel Stack vs. Future Process Stacks

The stack you're setting up now is the **boot stack** — the kernel's initial stack. It will be used by `kernel_main` and all initialization code. Once you create processes (Chapter 13), each process will have its own kernel stack (for when it traps into the kernel), and the boot stack will become the idle loop's stack or be repurposed.

For now, think of it as "the stack." Later, think of it as "one of many stacks."

An important design point: the boot stack is statically allocated (in `.bss`). Process stacks will be dynamically allocated (from the frame allocator, Chapter 9). Static allocation is fine for a single known stack; it would be terrible for an arbitrary number of process stacks.

---

## Conceptual Exercises

1. **What would happen if you forgot to place `boot.S`'s code in `.text.boot` (or otherwise ensure it comes first), and the linker placed `main.o`'s `.text` at `0x80000000` instead?** What instruction would the CPU execute first? Would it crash immediately, or would something more subtle go wrong?

2. **Why do we zero BSS with `sd` (8-byte stores) instead of `sb` (1-byte stores)?** What assumption does this make about the alignment of `_bss_start` and `_bss_end`? What would happen if BSS started at an address that isn't 8-byte aligned?

3. **Trace what happens if the stack pointer is set to `stack_bottom` instead of `stack_top`.** The first function call pushes a frame — where does it go? Why is this catastrophic?

4. **Why does the `call kernel_main` instruction in `boot.S` need to store the return address?** `kernel_main` never returns (it loops forever). Is the `ra` save wasted? (Think about what the CPU would do if you used `j kernel_main` instead of `call kernel_main`.)

5. **You've been told that QEMU's `-bios none` places a trampoline at `0x1000`. What would happen if you also specified `-bios none` but set your linker script to place `.text` at `0x1000`?** Would the CPU execute your code or the trampoline? (Think about who writes to that address first.)

6. **Why is it important that the hart parking check is the very first thing in `_start`, before setting the stack pointer?** What would go wrong if multiple harts all set `sp` to the same `stack_top` address and then tried to call functions simultaneously?

---

## Build This

This chapter synthesizes everything from Chapters 1–4. Here is exactly what you should have when you're done:

### Files

**`kernel/boot.S`** — The assembly boot stub.

It should:
- Be placed in `.section .text.boot`
- Declare `_start` as `.globl`
- Read `mhartid` and park non-zero harts in a `wfi` loop
- Load `stack_top` into `sp`
- Load `_bss_start` and `_bss_end`, zero the BSS with a loop
- Call `kernel_main`
- After `kernel_main` returns, spin in a `wfi` loop
- Declare a stack region in `.section .bss` with a `stack_bottom` label, appropriate `.space`, and a `stack_top` label

**`kernel/main.c`** — The C entry point.

It should:
- Define `void kernel_main(void)`
- Contain a `while(1)` loop (optionally with a `wfi` instruction via inline assembly)
- Nothing else for now

**`kernel/kernel.ld`** — The linker script.

It should:
- Declare `ENTRY(_start)`
- Set the base address to `0x80000000`
- Lay out sections: `.text` (with `.text.boot` first), `.rodata`, `.data`, `.bss`
- Align each section to `0x1000` (4096 bytes)
- Export `_bss_start`, `_bss_end`, and `_kernel_end` symbols
- Discard `.comment` and `.note*` sections (optional but clean)

**`Makefile`** — The build system.

It should:
- Compile `.S` files with the cross-assembler
- Compile `.c` files with `-ffreestanding -nostdlib -mcmodel=medany -mno-relax -Wall -Wextra -O0 -g`
- Link with `-T kernel/kernel.ld -nostdlib`
- Have `run`, `debug`, and `clean` targets
- Generate and include `.d` dependency files

### Verification Checklist

Run each of these and confirm the expected result:

**1. `make` completes without errors.**

**2. `file build/kernel.elf` shows:**
```
ELF 64-bit LSB executable, UCB RISC-V, ...
```

**3. `riscv64-unknown-elf-readelf -h build/kernel.elf` shows:**
- Entry point: `0x80000000`
- Machine: RISC-V

**4. `riscv64-unknown-elf-objdump -h build/kernel.elf` shows:**
- `.text` at `0x80000000`
- All sections within the DRAM range

**5. `riscv64-unknown-elf-objdump -d build/kernel.elf | head -40` shows:**
- Instructions at `0x80000000` that match your boot stub (hart ID check first)

**6. `riscv64-unknown-elf-objdump -t build/kernel.elf | grep -E "bss_start|bss_end|kernel_end|stack_top"` shows:**
- All symbols with valid addresses in the expected range

**7. GDB verification:**
```
# Terminal 1:
make debug

# Terminal 2:
riscv64-unknown-elf-gdb build/kernel.elf
(gdb) target remote :1234
(gdb) break kernel_main
(gdb) continue
```
- GDB stops at `kernel_main`
- `info registers sp` shows `sp` pointing to your `stack_top` address
- `info registers pc` shows `pc` at `kernel_main`'s address

**If all seven checks pass, you've booted a kernel.** You have a working boot sequence that takes a bare CPU from reset to C code. Everything from here builds on this foundation.

---

## When Things Go Wrong

**GDB never hits the `_start` breakpoint**
QEMU isn't jumping to your code. Verify:
- Your ELF entry point is `0x80000000` (`readelf -h`)
- The file is a valid RISC-V ELF (`file kernel.elf`)
- You're using `-bios none -kernel kernel.elf` in your QEMU command

Try stepping from the reset vector: `stepi` from `0x1000` and watch where the CPU goes. You should see it load `0x80000000` and jump.

**GDB hits `_start` but the BSS loop never terminates**
Set a breakpoint at the BSS loop and check the values of `_bss_start` and `_bss_end`. If both are zero, your linker script isn't exporting them correctly. If `_bss_end` is less than `_bss_start` (which shouldn't happen), your comparison branch is wrong. Use `stepi` to step through the loop and watch the registers.

Common mistake: using `ld` instead of `la` to load the BSS addresses. `ld` loads a value FROM the address (treating the symbol as a memory location), `la` loads the address itself. You want `la`.

**GDB hits `_start` but crashes before `kernel_main`**
Use `stepi` to find exactly which instruction causes the crash. Common culprits:
- Stack pointer set to an invalid address (check `sp` before and after the `la sp, stack_top`)
- The `call kernel_main` instruction computes the wrong target address (check the disassembly to see where it jumps)

**QEMU immediately exits (no hang, just returns to shell)**
Your kernel might be executing a `wfi` with interrupts disabled and nothing to wake it up. Depending on the QEMU version, this may cause QEMU to exit immediately. Try using `j .` (infinite loop) instead of `wfi` for your spin loops and see if the behavior changes. If it does, the issue is `wfi`-related.

Alternatively, QEMU might be failing to load your kernel. Run with `-d guest_errors` to see if QEMU reports any load errors.

**"Cannot insert breakpoint" in GDB**
The address might not be mapped. Verify with `info target` in GDB that the symbol addresses match what you expect. If you're using software breakpoints (the default), GDB needs to write to the address, which requires it to be writable — `.text` loaded from ELF is generally writable from GDB's perspective, but if something is wrong with the memory mapping, GDB can't insert the breakpoint. Try `hbreak kernel_main` (hardware breakpoint) instead.

**The BSS symbols exist but have address 0**
Your linker script defines them outside the `SECTIONS { }` block, or defines them in a way the linker doesn't understand. The symbol assignment (`. = ALIGN(4096); _bss_start = .;`) must be inside the `SECTIONS` block, typically inside or adjacent to the `.bss` output section definition.

---

## Further Reading

- **[xv6 Book, Chapter 2: Operating System Organization](https://pdos.csail.mit.edu/6.828/2024/xv6/book-riscv-rev4.pdf)** — Covers xv6's boot process, which is remarkably similar to ours. The xv6 `entry.S` corresponds to our `boot.S`, and their `main()` corresponds to our `kernel_main()`.
- **[xv6 source: `entry.S` and `start.c`](https://github.com/mit-pdos/xv6-riscv)** — Read these after you've written your own. `entry.S` sets the stack and calls `start()`, which configures M-mode and drops to S-mode before calling `main()`. Clean, readable code.
- **[RISC-V Privileged Specification, Section 3.4: Reset](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf)** — The formal specification of CPU state at reset. Short but authoritative.
- **[OSDev Wiki: "Bare Bones"](https://wiki.osdev.org/)** — The OSDev wiki has a "bare bones" tutorial for x86. The concepts (linker script, boot assembly, reaching C) are the same as ours, just for a different architecture. Useful for cross-referencing.
- **[OSTEP Chapter 6: Mechanism: Limited Direct Execution](https://pages.cs.wisc.edu/~remzi/OSTEP/)** — The conceptual framework of "bootstrapping" the OS from a bare machine. Arpaci-Dusseau explain why the OS needs hardware cooperation to gain control.

---

*Next: [Chapter 6 — UART and Serial Output](ch06-uart-serial-output.md), where we teach the kernel to speak. You'll implement a UART driver, build `kprintf`, and see your first output on the terminal. The blank screen ends here.*
