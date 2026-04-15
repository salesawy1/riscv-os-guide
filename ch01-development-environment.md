# Chapter 1: Development Environment

**[Difficulty: ★☆☆☆☆]**

---

## Why This Chapter Exists

You wouldn't walk into a machine shop and start welding without knowing where the torches are, how the ventilation works, and which safety glasses to grab. The same principle applies here. Before you write your first line of kernel code, you need to understand your tools — not just how to install them, but *why each one exists* and *what it does for you*.

This chapter is the least glamorous in the entire guide. There are no page tables, no trap handlers, no scheduling algorithms. There is a cross-compiler, a machine emulator, a build system, and a linker script. These are the four legs of the table on which your entire operating system will sit. If any one of them is wobbly, everything falls off.

Here's the thing that trips up most people starting OS development: **you are no longer writing software that runs on an operating system.** When you write a normal C program, you rely on your OS for everything — loading your binary into memory, setting up the stack, initializing the standard library, managing memory. All of that is gone. You are the operating system. There is nothing below you but silicon and firmware. Your development environment needs to reflect this reality.

---

## The Cross-Compiler Toolchain

### What Problem It Solves

You're sitting at your Mac or Linux machine, which runs on x86-64 or ARM. You want to produce binaries that run on RISC-V 64-bit hardware. Your system's default compiler — the one invoked when you type `gcc` or `clang` — produces binaries for your *host* architecture. If you compile your kernel with your host compiler, you'll get a binary that your host CPU can execute, but a RISC-V CPU will look at it like a novel written in a language it doesn't speak.

A **[cross-compiler](https://wiki.osdev.org/Cross-Compiler)** is a compiler that runs on one architecture (the *host*) but produces code for a different architecture (the *target*). This is not exotic — it's how virtually all embedded and OS development works. The Android NDK is a cross-compiler. When Apple builds iOS apps on a Mac, that's cross-compilation. You need one too.

### What You Need

The GNU toolchain for RISC-V includes several components that work together:

**`riscv64-unknown-elf-gcc`** — The cross-compiler itself. The name follows a convention: `<architecture>-<vendor>-<os>-<tool>`. The `unknown` vendor means "generic," and the `elf` part is critical — it means we're targeting *bare metal* (no operating system), producing [ELF binaries](https://refspecs.linuxfoundation.org/elf/elf.pdf). If you instead used `riscv64-unknown-linux-gnu-gcc`, that compiler would assume a Linux environment exists, link against glibc, and produce binaries that expect Linux syscalls to be available. Your OS doesn't exist yet. You need `elf`, not `linux`.

**`riscv64-unknown-elf-as`** — The assembler. Translates RISC-V assembly (`.S` files) into object files. You'll write assembly for your boot stub, trap handlers, and context switch routine. Not much, but the parts that are in assembly are there because they *have* to be — C can't directly manipulate the stack pointer or save all registers to a specific memory layout.

**`riscv64-unknown-elf-ld`** — The [linker](https://sourceware.org/binutils/docs/ld/). Takes your compiled object files (`.o`) and combines them into a single binary. In OS development, the linker is far more important than in application development, because *you* control where everything goes in memory. The linker script (discussed later) is how you exercise that control.

**`riscv64-unknown-elf-objcopy`** — Converts between binary formats. You'll use this to strip an ELF binary down to a raw binary image when needed, or to inspect sections.

**`riscv64-unknown-elf-objdump`** — The disassembler and binary inspector. When your kernel crashes and you need to know what instruction is at address `0x80000a4c`, this is what you reach for. Learn to love it. You'll use it more than you expect.

**`riscv64-unknown-elf-gdb`** — The debugger, configured for RISC-V targets. QEMU has a built-in GDB server; you'll connect this debugger to it to set breakpoints, examine registers, and step through your kernel instruction by instruction.

### Installing the Toolchain

On macOS, the cleanest path is Homebrew:

```
brew tap riscv-software-src/riscv
brew install riscv-tools
```

On Ubuntu/Debian:

```
sudo apt install gcc-riscv64-unknown-elf binutils-riscv64-unknown-elf
```

If your distribution doesn't package it, or if the packaged version is ancient, you can build it from source via the `riscv-gnu-toolchain` repository on GitHub. Expect the build to take 30–60 minutes. Configure it with `--with-arch=rv64gc --with-abi=lp64d` to target 64-bit RISC-V with the general-purpose and compressed instruction extensions plus double-precision floating-point.

After installation, verify it works:

```
riscv64-unknown-elf-gcc --version
```

If this produces version output without errors, you're in business.

> **Aside: Why not Clang/LLVM?**
>
> Clang has solid RISC-V support and can cross-compile without a separate toolchain installation — you just pass `--target=riscv64-unknown-elf`. For OS development, either GCC or Clang works. This guide assumes the GNU toolchain because the xv6 project uses it, most RISC-V OS tutorials use it, and the debugging story with `riscv64-unknown-elf-gdb` is more mature. If you prefer Clang, everything in this guide still applies — just adjust the compiler invocations and be aware that Clang's inline assembly syntax has minor differences from GCC's.

### The `-ffreestanding` Flag

When you compile your kernel, you'll pass `-ffreestanding` to the compiler. This is one of the most important flags in OS development, and it does two things:

1. **It tells the compiler not to assume the standard library is available.** Normally, GCC knows that `memcpy` exists and may silently generate calls to it as an optimization — for instance, when you assign one struct to another. With `-ffreestanding`, the compiler *still might do this*, but it's telling you: "I know you might not have libc, so you're responsible for providing these functions if I need them." In practice, you'll need to implement `memcpy`, `memset`, and `memmove` early on, because the compiler will generate calls to them whether you like it or not.

2. **It tells the compiler that `main()` is not necessarily the entry point.** In a hosted environment, the C runtime starts at `_start`, sets up the stack, initializes global constructors, and calls `main()`. In a freestanding environment, none of that exists. Your entry point is wherever your linker script says it is.

Other critical compiler flags you'll use:

- **`-nostdlib`** — Don't link against the standard C library or the default startup files. You have no libc. You have no `crt0.o`.
- **`-mcmodel=medany`** — The code model. RISC-V has several; `medany` means "medium any," which allows code and data to be placed anywhere in the address space, as long as they're within 2 GiB of each other. This is the standard choice for kernel code.
- **`-mno-relax`** — Disables linker relaxation, a RISC-V-specific optimization where the linker replaces instruction sequences with shorter ones. This can cause confusion during debugging and interacts badly with some linker script setups. Disable it until you know enough to want it.
- **`-Wall -Wextra`** — Turn on warnings. When you're writing code that runs without an OS, compiler warnings are often the only safety net you have.
- **`-O0` or `-Og`** — Optimization level. Start with no optimization or debug-friendly optimization. Optimized code is harder to debug because the compiler reorders instructions, eliminates variables, and inlines functions. Once your OS works, you can crank it up.

---

## QEMU: Your Virtual Hardware

### What Problem It Solves

You could, in theory, buy a RISC-V development board (the SiFive HiFive Unmatched, the StarFive VisionFive 2) and test your OS on real hardware. This would be a terrible idea at this stage, for several reasons:

1. **Debugging on real hardware is agonizing.** Your only feedback mechanism is an LED or a serial port. There's no `printf` until you build one. There are no breakpoints. When your kernel crashes, the board just... sits there.

2. **Real hardware has real firmware with real quirks.** Every board has its own boot sequence, its own device tree, its own set of peripherals. You'd spend weeks fighting hardware-specific issues instead of learning OS concepts.

3. **Iteration time matters.** On real hardware, you flash an SD card, power cycle the board, wait for it to boot, see it crash, and repeat. With QEMU, the cycle is: save the file, run `make run`, see the result in under a second.

QEMU is a full-system emulator. When you run `qemu-system-riscv64`, it simulates an entire RISC-V computer: CPU, RAM, interrupt controllers, serial ports, block devices, and more. The `virt` machine type is a QEMU-specific board designed specifically for OS development and testing — it has predictable device addresses, a clean device tree, and no firmware quirks to work around.

### Installing QEMU

On macOS:

```
brew install qemu
```

On Ubuntu/Debian:

```
sudo apt install qemu-system-misc
```

Verify:

```
qemu-system-riscv64 --version
```

You want version 7.0 or later. Older versions have bugs in their RISC-V support that will waste your time.

### The QEMU `virt` Machine

When you invoke QEMU with `-machine virt`, you get a simulated board with the following peripherals (among others):

| Device | Address | What It Is |
|--------|---------|------------|
| CLINT | `0x0200_0000` | Core Local Interruptor — timer and software interrupts |
| PLIC | `0x0C00_0000` | Platform-Level Interrupt Controller — external interrupts |
| UART0 | `0x1000_0000` | 16550A-compatible serial port |
| virtio | `0x1000_1000+` | Virtio MMIO devices (block, network, etc.) |
| DRAM | `0x8000_0000` | Start of physical memory |

These addresses are *memory-mapped*. There's no special I/O instruction on RISC-V (unlike x86 with its `in`/`out` instructions). To talk to the UART, you read and write to memory addresses starting at `0x10000000`. The hardware intercepts those reads and writes and routes them to the device. This is called **[memory-mapped I/O (MMIO)](https://wiki.osdev.org/Memory_Mapped_IO)**, and it's how most modern architectures handle device communication. We'll explore this thoroughly in Chapter 6.

The key thing to know now is that your kernel's memory map is not just RAM. Some addresses are RAM, some are devices, and writing to the wrong address will either do nothing or cause a fault. The QEMU `virt` machine's memory map is documented in QEMU's source code (`hw/riscv/virt.c`), and we'll reference specific addresses as we encounter each device.

### The QEMU Invocation You'll Use

Here's the command you'll eventually run (don't worry about understanding every flag yet — we'll add them one at a time):

```
qemu-system-riscv64 \
    -machine virt \
    -cpu rv64 \
    -smp 1 \
    -m 128M \
    -nographic \
    -bios none \
    -kernel your_kernel.elf
```

Let's break this down:

- **`-machine virt`** — Use the `virt` board.
- **`-cpu rv64`** — A generic 64-bit RISC-V CPU. (You can also specify specific CPU models, but the generic one is fine.)
- **`-smp 1`** — One CPU core. We're building a single-core OS first. Multicore is a Chapter 24 topic.
- **`-m 128M`** — 128 megabytes of RAM. Plenty for a teaching OS.
- **`-nographic`** — No graphical window. All output goes to the terminal via the serial port. This is what you want — your OS communicates through the UART, and `-nographic` connects QEMU's serial port to your terminal's stdin/stdout.
- **`-bios none`** — Don't load any firmware. By default, QEMU loads OpenSBI (an open-source RISC-V firmware) as the BIOS. OpenSBI is useful in production, but for learning, it hides the boot process from you. We'll discuss this trade-off in Chapter 3, but for now, `none` means your kernel is the first thing that runs.
- **`-kernel your_kernel.elf`** — Load your kernel ELF binary and jump to its entry point.

> **Aside: `-bios none` vs. using OpenSBI**
>
> In the real RISC-V ecosystem, firmware like OpenSBI runs first in M-mode (machine mode, the highest privilege level), performs hardware initialization, sets up a standard interface called the SBI (Supervisor Binary Interface), and then hands control to your kernel running in S-mode (supervisor mode). This is analogous to how on x86, BIOS/UEFI runs first and then loads the bootloader.
>
> We're going to skip OpenSBI initially and boot directly into M-mode. This means *you* are responsible for everything that firmware normally does: setting up trap vectors, configuring the interrupt controller, transitioning to S-mode. This is more work, but it means you understand every layer of the system. In a later chapter, we'll discuss what SBI provides and how you could use it to simplify things — but by then, you'll understand what it's doing under the hood.
>
> The xv6 teaching OS takes the other approach — it boots on top of OpenSBI and runs entirely in S-mode. That's a valid pedagogical choice. Ours gives you a deeper understanding of the hardware.

### Debugging with QEMU

QEMU has two invaluable debugging features:

**1. The GDB stub.** Add `-s -S` to your QEMU command:
- `-s` starts a GDB server on TCP port 1234
- `-S` pauses the CPU at startup, waiting for GDB to connect

Then in another terminal:
```
riscv64-unknown-elf-gdb your_kernel.elf
(gdb) target remote :1234
(gdb) break kernel_main
(gdb) continue
```

You now have full source-level debugging of your kernel: breakpoints, register inspection, memory examination, single-stepping. This is your primary debugging tool for the rest of this guide.

**2. Trace logging.** The `-d` flag tells QEMU to log internal events:
- `-d int` — Log all interrupts and exceptions. When your kernel takes an unexpected trap, this shows you exactly what happened.
- `-d cpu_reset` — Log CPU resets.
- `-d in_asm` — Log every instruction executed (warning: extremely verbose).
- `-d guest_errors` — Log guest errors like accessing invalid memory.
- `-D logfile.txt` — Send the log to a file instead of stderr.

When your kernel hangs and you don't know why, `-d int` is usually the first thing you reach for. We'll point you to specific flags throughout the guide.

**3. The QEMU monitor.** If you're not using `-nographic`, you can access the QEMU monitor with `Ctrl-A, C`. This lets you inspect the machine state: `info registers` shows CPU registers, `info mtree` shows the memory map, `xp /10x 0x80000000` examines memory. With `-nographic`, press `Ctrl-A, C` to toggle between the serial console and the monitor.

To quit QEMU with `-nographic`, press `Ctrl-A, X`. (Not `Ctrl-C` — that sends an interrupt to the guest, not to QEMU.)

---

## The Build System: Make

### What Problem It Solves

Your OS will consist of assembly files (`.S`), C source files (`.c`), and a linker script (`.ld`). The build process is:

1. Assemble `.S` files into `.o` object files
2. Compile `.c` files into `.o` object files
3. Link all `.o` files into a single ELF binary using the linker script
4. (Optionally) convert the ELF to a raw binary image

You *could* type out these commands by hand every time. You'll stop doing that approximately 90 seconds into development. A Makefile automates this build process.

### A Makefile for OS Development

Let me walk you through what your Makefile needs to express, conceptually. You'll write the actual Makefile yourself, but here are the pieces:

**Variables at the top:**
- The cross-compiler prefix (`riscv64-unknown-elf-`)
- Compiler flags (`-ffreestanding`, `-nostdlib`, `-mcmodel=medany`, warning flags, optimization level, debug info)
- The linker script path
- Source file lists (or patterns to find them)

**Rules:**
- A rule to assemble `.S` files into `.o` files
- A rule to compile `.c` files into `.o` files
- A rule to link all `.o` files into the final kernel ELF
- A `run` target that invokes QEMU with the right flags
- A `debug` target that adds `-s -S` to the QEMU invocation
- A `clean` target that removes build artifacts

**Pattern rules** will save you from writing a separate rule for every source file. A pattern rule like "any `.o` file can be built from a `.c` file with the same name by invoking the compiler" covers all your C sources in one declaration.

**Dependency tracking** is worth setting up early. Pass `-MMD -MP` to the compiler, and it will generate `.d` files that list the headers each source file includes. Include these `.d` files in your Makefile, and Make will automatically recompile a source file if any of its headers change. Without this, you'll change a header and wonder why your kernel doesn't reflect the change (answer: Make didn't know it needed to recompile anything).

> **Aside: Make vs. CMake vs. Meson vs. Ninja**
>
> For a teaching OS, Make is the right choice. It's universal, it's simple, and there's nothing it can't handle at this scale. CMake and Meson are designed for large, portable projects with complex dependency management — they'd be overkill here and would add a layer of abstraction between you and the build process. You *want* to see the raw compiler invocations. You want to understand exactly what's happening when you type `make`.
>
> The xv6 project uses a plain Makefile. So does the Linux kernel (though it's... considerably more complex).

---

## The Linker Script: Placing Your Code in Memory

### What Problem It Solves

When you compile an application for Linux, the compiler and linker produce an ELF binary, and the OS's program loader decides where in memory to place it. The linker uses default scripts that put the `.text` section (code) at some address, `.data` (initialized global variables) at another, `.bss` ([uninitialized globals](https://wiki.osdev.org/Memory_Layout)) at another, and so on. The exact addresses don't matter much because the OS handles it.

You don't have an OS. *You are the OS.* Nobody is going to load your binary and set up its memory layout. The QEMU `-kernel` flag loads your ELF into memory and jumps to its entry point, but *you* must tell the linker exactly where every section should go in the physical address space. This is what a linker script does.

### Why You Can't Skip This

Without a linker script, the linker uses its built-in defaults, which are designed for user-space applications on a hosted system. Those defaults will place your code at addresses that don't correspond to where QEMU's RAM actually is. Your kernel will either fail to load or crash immediately because the entry point is at the wrong address.

The QEMU `virt` machine places RAM starting at physical address `0x80000000`. When you use `-bios none -kernel`, QEMU loads the ELF at the address specified in its [program headers](https://refspecs.linuxfoundation.org/elf/elf.pdf#page=7) and jumps to the entry point. So your linker script must place the `.text` section starting at `0x80000000`, and specifically, it must place your *entry point* — the very first instruction of your boot stub — at that address.

### Anatomy of a Linker Script

A linker script has three key components:

**1. The entry point declaration.**

`ENTRY(_start)` tells the linker that the symbol `_start` (defined in your assembly boot stub) is the first instruction to execute. The linker records this in the ELF header so that QEMU (or any ELF loader) knows where to jump.

**2. The base address.**

The `. = 0x80000000;` directive (or similar) sets the location counter — the address at which the following sections will be placed. This must match where QEMU places RAM for the `virt` machine.

**3. Section layout.**

The linker script defines output sections and maps input sections to them. A simplified conceptual layout:

```
.text   → Executable code (your assembly boot stub first, then compiled C functions)
.rodata → Read-only data (string literals, constants)
.data   → Initialized global/static variables
.bss    → Uninitialized global/static variables (zeroed at startup)
```

The order matters. `.text` comes first because the entry point must be at the base address. Each subsequent section follows the previous one in memory. The linker script uses the `.` (dot) symbol as the location counter — it advances as you place sections, and you can read it to record addresses.

**What makes linker scripts unique in OS development:**

You'll use the linker script to export symbols that your C code reads. For example, you'll define a symbol `_bss_start` at the beginning of the `.bss` section and `_bss_end` at the end. Your boot code will use these symbols to zero out the BSS — because in a freestanding environment, nobody initializes your BSS for you. If you don't zero it, global variables that should be zero will contain whatever garbage was in RAM at boot.

Similarly, you'll export symbols marking the end of the kernel image. These tell your memory allocator where free memory begins — you don't want to hand out memory that your kernel code is sitting in.

You'll also use `ALIGN()` directives to ensure sections start at page-aligned boundaries. When we get to virtual memory (Chapter 10), you'll need code and data to be on page boundaries so you can set different permissions on them (executable but not writable for `.text`, writable but not executable for `.data`). Setting this up in the linker script now saves pain later.

> **Aside: The linker script language**
>
> The GNU [linker script](https://sourceware.org/binutils/docs/ld/Scripts.html) language is its own little DSL, and its documentation (the GNU `ld` manual, specifically Chapter 3: "Linker Scripts") is the authoritative reference. The language looks vaguely like C but has its own semantics. Some things that will trip you up:
>
> - The `.` symbol is both readable and writable. Assigning to it moves the location counter forward. Reading it gives you the current address.
> - `PROVIDE()` defines a symbol only if nothing else defines it. Useful for defaults.
> - `KEEP()` prevents the linker from garbage-collecting a section, even if it appears unused. Important for things like interrupt vector tables.
> - `/DISCARD/` explicitly throws away sections you don't want. The compiler generates some sections (like `.comment` or `.eh_frame`) that are useless in a freestanding kernel.
>
> The xv6 linker script (`kernel.ld`) is about 30 lines and is a good model. Look at it after you've written yours.

### The BSS: A Section That Matters More Than You Think

One thing that catches new OS developers: the [`.bss` section](https://wiki.osdev.org/Memory_Layout). In a normal C program, the OS loader zeroes the BSS before `main()` runs, so uninitialized global variables start at zero as the C standard guarantees. In your kernel, *you* must zero it yourself. Your boot assembly stub, after setting up the stack and before calling `kernel_main()`, should loop from `_bss_start` to `_bss_end` and write zeros. If you skip this, you'll get mysterious bugs where global variables have unpredictable initial values.

This is the kind of thing that "just works" in application programming and silently fails in OS development. You'll encounter many of these. Each one teaches you something that was previously invisible about how programs work.

---

## Project Layout

Before you start writing code, set up a directory structure. A clean layout will save you headaches as the project grows. Here's what you should aim for:

```
my-os/
├── Makefile
├── kernel/
│   ├── kernel.ld          # Linker script
│   ├── boot.S             # Assembly boot stub
│   ├── main.c             # kernel_main() lives here
│   ├── uart.c             # UART driver (Chapter 6)
│   ├── trap.c             # Trap handler (Chapter 7)
│   ├── ...                # More as we go
│   └── include/
│       ├── types.h        # uint64_t, etc. (you define these yourself)
│       └── ...
├── user/                  # Userspace programs (Part V)
└── build/                 # Build artifacts (.o files, the kernel binary)
```

A few things to note:

- Your `types.h` will define `uint8_t`, `uint16_t`, `uint32_t`, `uint64_t`, and their signed variants. Normally these come from `<stdint.h>`, which is part of the C standard library. But wait — you said `-nostdlib`, no standard library. Here's a nuance: `<stdint.h>` is actually a *freestanding* header. The C standard distinguishes between *hosted* and *freestanding* implementations, and freestanding implementations are required to provide a few headers: `<stddef.h>`, `<stdint.h>`, `<stdbool.h>`, `<stdarg.h>`, `<float.h>`, and `<limits.h>`. Your cross-compiler provides these even in `-ffreestanding` mode. So you can `#include <stdint.h>` and get `uint64_t` without implementing it yourself. But you *cannot* use `<stdio.h>`, `<stdlib.h>`, `<string.h>`, or any other hosted header.

- Keep assembly files (`.S`, capital S) separate from C files. The capital S matters — it tells the compiler to run the C preprocessor on the assembly file, letting you use `#include`, `#define`, and `#ifdef` in assembly. This is useful for sharing constants between C and assembly code.

---

## Conceptual Exercises

Before you set anything up, make sure you can answer these questions. If you can't, re-read the relevant section.

1. **Why can't you use your system's `gcc` to compile kernel code for RISC-V?** What specifically would go wrong if you tried?

2. **What does `-ffreestanding` actually change about the compiler's behavior?** Give two concrete examples of things the compiler does differently in freestanding mode.

3. **Why is `-bios none` important for our learning purposes?** What would happen if you left it out? (Hint: QEMU's default behavior loads firmware.) What trade-off are we making?

4. **If you write a global variable `int counter;` (without an initializer) in your kernel, where does it end up in the binary?** What value does the C standard say it should have at program start? Who normally ensures that? Who ensures it in your kernel?

5. **Why does the linker script need to place `.text` at `0x80000000` specifically?** What would happen if it placed `.text` at `0x0`? (Think about the QEMU `virt` machine's memory map.)

6. **Explain the difference between `riscv64-unknown-elf-gcc` and `riscv64-unknown-linux-gnu-gcc`.** When would you use each one? What assumptions does the `linux` variant make that would be problematic for kernel development?

---

## Build This

Your task for this chapter is purely tooling. No kernel code yet — just the workshop.

**Set up:**

1. **Install the RISC-V GNU toolchain.** Verify that `riscv64-unknown-elf-gcc --version` works.

2. **Install QEMU.** Verify that `qemu-system-riscv64 --version` produces version 7.0 or later.

3. **Create the project directory structure** as outlined above.

4. **Write a linker script** (`kernel.ld`) that:
   - Declares `_start` as the entry point
   - Places `.text` at `0x80000000`
   - Lays out `.rodata`, `.data`, and `.bss` in order after `.text`
   - Exports `_bss_start` and `_bss_end` symbols around the `.bss` section
   - Exports a `_kernel_end` symbol after all sections
   - Aligns each section to a 4096-byte (page) boundary

5. **Write a Makefile** that:
   - Uses the RISC-V cross-compiler with the flags discussed above
   - Has pattern rules for `.S` → `.o` and `.c` → `.o`
   - Links all objects into a kernel ELF using your linker script
   - Has a `run` target that invokes QEMU
   - Has a `debug` target that invokes QEMU with `-s -S`
   - Has a `clean` target
   - Generates and includes `.d` dependency files

6. **Write a minimal `boot.S`** that does nothing but enter an infinite loop (`j .` — jump to self). This is the smallest possible "kernel" — it boots and hangs.

7. **Write a minimal `main.c`** with an empty `kernel_main()` function. It won't be called yet (your boot.S just loops), but it should compile.

**Checkpoint:** You should be able to run `make` and produce a kernel ELF binary without errors. Running `make run` should launch QEMU, which will appear to hang (because your kernel is an infinite loop). Verify with `Ctrl-A, X` that you can exit QEMU. Run `riscv64-unknown-elf-objdump -h build/kernel.elf` and verify that the `.text` section starts at `0x80000000`.

If you can do all of that, your workshop is ready. The real work begins in Chapter 5.

---

## When Things Go Wrong

**"command not found: riscv64-unknown-elf-gcc"**
The toolchain isn't installed or isn't on your `$PATH`. On macOS with Homebrew, check `brew list | grep riscv`. On Linux, make sure you installed the right package — some distributions call it `gcc-riscv64-unknown-elf`, others `gcc-riscv64-linux-gnu` (which is NOT what you want for bare-metal development).

**"cannot find -lgcc" or linker errors about missing libraries**
You're probably missing `-nostdlib` in your linker flags. Or the toolchain was built without the bare-metal support library. Ensure your toolchain was configured with `--with-newlib` or is the `elf` target, not the `linux` target.

**QEMU prints nothing and appears to hang**
This is expected with your infinite-loop kernel! It's working correctly. Exit with `Ctrl-A, X`. If even that doesn't work, your terminal might be intercepting `Ctrl-A` — try running QEMU in a plain terminal (not inside tmux or screen, which may steal the `Ctrl-A` prefix).

**"qemu-system-riscv64: Unable to find the kernel binary"**
Check the path you passed to `-kernel`. It must be the ELF file, not an intermediate `.o` file.

**`objdump` shows `.text` at the wrong address**
Your linker script is wrong, or you're not passing it to the linker. Make sure your link command includes `-T kernel.ld` (or whatever you named your linker script).

**"undefined reference to `_start`"**
Your `boot.S` either doesn't define a `.globl _start` label, or it's not being assembled and linked. Check that your Makefile includes `boot.S` in the list of sources and that the pattern rule for `.S` files is correct.

---

## Further Reading

- **[GNU Linker (`ld`) Manual, Chapter 3: Linker Scripts](https://sourceware.org/binutils/docs/ld/Scripts.html)** — The authoritative reference for linker script syntax. Dry but complete.
- **[QEMU RISC-V `virt` machine documentation](https://www.qemu.org/docs/master/system/riscv/virt.html)** — Describes the simulated hardware and memory map. When in doubt about device addresses, refer here.
- **CS:APP Chapter 7: Linking** — Bryant & O'Hallaron's treatment of the linking process, object files, and symbol resolution. Excellent background for understanding what the linker is doing with your `.o` files.
- **[xv6 source code](https://github.com/mit-pdos/xv6-riscv)** — Study the Makefile and `kernel.ld` after you've written yours. They're clean and well-commented.

---

*Next: [Chapter 2 — C for OS Development](ch02-c-for-os-dev.md), where we revisit the parts of C that application programmers rarely touch but OS developers can't live without.*
