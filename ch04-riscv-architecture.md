# Chapter 4: The RISC-V Architecture

**[Difficulty: ★★★☆☆]**

---

## Why This Chapter Exists

You're about to write software that runs directly on a CPU. Not software that *asks* an operating system to do things. Not software that a runtime manages. Software that configures the CPU itself — tells it where to jump when something goes wrong, tells it which memory addresses are valid, tells it whether the currently running code is allowed to touch hardware registers. To do this, you need to understand the CPU as a machine, not as an abstraction.

This chapter is your RISC-V reference. It's the chapter you'll flip back to more than any other, because every subsequent chapter depends on something here: the register file, the privilege architecture, the CSR mechanism, the trap model. When Chapter 7 says "write the trap cause to `mcause`," you need to know what `mcause` is, how wide it is, how to read it, and what its bit layout means. When Chapter 10 says "write the root page table address to `satp`," you need to know what `satp` is and what happens when you write to it.

[RISC-V](https://riscv.org/technical/specifications/) was designed at UC Berkeley starting in 2010 by Krste Asanović and David Patterson (yes, the Patterson of "Patterson and Hennessy," the canonical computer architecture textbook). It was designed explicitly to be simple, clean, and free of the historical baggage that makes x86 a nightmare for teaching and ARM a licensing minefield. The name stands for "[Reduced Instruction Set Computer](https://en.wikipedia.org/wiki/Reduced_instruction_set_computer), Version 5" — the fifth iteration of RISC designs from Berkeley. It is open-source: the specification is freely available, anyone can build a RISC-V chip without paying royalties, and the ecosystem is growing rapidly.

This matters to you because RISC-V is *designed to be understood*. The privilege architecture is clean. The CSR layout is logical. The instruction encoding is regular. Where x86 has layers of cruft from 1978, RISC-V has a specification that a motivated person can read in a weekend. You should read it. This chapter will tell you what to look for and what matters for OS development.

---

## The Register File

RISC-V has 32 general-purpose integer registers, each 64 bits wide on RV64 (our target). They're named `x0` through `x31`, but by convention (the RISC-V ABI), they have more meaningful names:

```
Register   ABI Name   Purpose                          Saved By
────────   ────────   ───────                          ────────
x0         zero       Hardwired to zero                 N/A
x1         ra         Return address                    Caller
x2         sp         Stack pointer                     Callee
x3         gp         Global pointer                    N/A (set once)
x4         tp         Thread pointer                    N/A (set once)
x5-x7      t0-t2      Temporaries                       Caller
x8         s0/fp      Saved register / frame pointer    Callee
x9         s1         Saved register                    Callee
x10-x11    a0-a1      Function args / return values     Caller
x12-x17    a2-a7      Function arguments                Caller
x18-x27    s2-s11     Saved registers                   Callee
x28-x31    t3-t6      Temporaries                       Caller
```

Several things to internalize:

**`x0` is [hardwired to zero](https://en.wikipedia.org/wiki/Zero_register).** Writing to it does nothing. Reading it always returns 0. This simplifies the instruction set: "move register A to register B" is just `add B, A, x0` (add A plus zero, store in B). "Load the value 0 into register A" is `add A, x0, x0`. There's no separate `mov` or `li` instruction in the base ISA — they're pseudoinstructions that the assembler translates into arithmetic with `x0`.

**`ra` (return address) is just a register.** On x86, the return address is pushed onto the stack by the `call` instruction. On RISC-V, `jal` (jump and link) stores the return address in `ra` and jumps. The return address is in a register, not on the stack. The callee must save `ra` to the stack if it calls another function (because that function's `jal` would overwrite `ra`). Leaf functions — functions that don't call other functions — never need to touch the stack for the return address. This is cleaner and faster.

**`sp` (stack pointer) is just a register.** The hardware doesn't enforce anything about `sp`. It doesn't automatically push or pop. There's no hardware stack limit check. It's a general-purpose register that the software convention designates for stack use. If you corrupt `sp`, the CPU won't stop you — it'll just start reading and writing garbage memory for all subsequent function calls.

**`gp` (global pointer) and `tp` (thread pointer) are set once and left alone.** `gp` is used for relaxed addressing of global variables (linker relaxation). `tp` is used for thread-local storage. In a multicore OS, each core's `tp` would point to that core's per-CPU data structure. For our single-core OS, we'll set `tp` and use it to identify the current process or hart.

**[Caller-saved vs. callee-saved](https://en.wikipedia.org/wiki/Calling_convention)** is critical for context switching (Chapter 14). When you save a process's state, you need to save the callee-saved registers (`s0`-`s11`, `sp`, `ra`) — these are the registers that the calling convention guarantees are preserved across function calls. The caller-saved registers (`a0`-`a7`, `t0`-`t6`) are, by convention, already saved by the caller if needed.

> **Aside: Why 32 registers?**
>
> [MIPS](https://en.wikipedia.org/wiki/MIPS_architecture) (the ancestor of RISC-V's design philosophy) also has 32 registers. ARM has 16 (ARMv7) or 31 (AArch64). x86-64 has 16 general-purpose registers. More registers mean fewer spills to memory (which means fewer loads/stores, which means faster code), but more registers mean wider instruction encodings (you need more bits to specify which register), which means larger code, which means more instruction cache pressure.
>
> 32 is a sweet spot, validated by decades of RISC architecture research. The 5-bit register field (`log2(32) = 5`) fits neatly into RISC-V's 32-bit instruction format. Patterson and Hennessy's *Computer Organization and Design* (the RISC-V edition) discusses this trade-off in detail.

---

## Instruction Encoding

You won't need to encode instructions by hand — the assembler does that. But understanding the encoding helps you read disassembly and appreciate why RISC-V instructions look the way they do.

RISC-V instructions are 32 bits wide (the base ISA; the "C" extension adds 16-bit compressed instructions). The lowest 7 bits are the **opcode**, which identifies the instruction type. The encoding is extremely regular — register fields are always in the same bit positions:

```
Bits:  31       25 24   20 19   15 14  12 11    7 6      0
       ┌─────────┬───────┬───────┬──────┬───────┬────────┐
R-type │  funct7  │  rs2  │  rs1  │funct3│  rd   │ opcode │
       └─────────┴───────┴───────┴──────┴───────┴────────┘
```

- **`rd`** (bits 11-7): destination register
- **`rs1`** (bits 19-15): first source register
- **`rs2`** (bits 24-20): second source register
- **`funct3`** and **`funct7`**: sub-opcode fields that distinguish variants (e.g., `add` vs. `sub` share an opcode but differ in `funct7`)

There are six instruction formats (R, I, S, B, U, J), but they all keep register fields in the same positions. This regularity is intentional: the hardware can extract the register numbers before it even knows what instruction it's looking at, which simplifies and speeds up the decode stage of the pipeline.

**Immediate encoding** is the one place RISC-V gets a bit tricky. To fit larger immediate values into 32-bit instructions, the bits of the immediate are scattered across the instruction word in non-obvious ways (particularly in the B-type and J-type formats, where the bits are shuffled to keep other fields aligned). You don't need to memorize this — the assembler handles it — but if you ever look at raw hex machine code, know that the immediate fields are rearranged, not sequential.

### Key Instructions You'll Use

**Arithmetic/logic:** `add`, `sub`, `and`, `or`, `xor`, `sll` (shift left logical), `srl` (shift right logical), `sra` (shift right arithmetic). All operate on registers. Immediate variants (`addi`, `andi`, etc.) have a 12-bit signed immediate.

**Memory access:** `ld` (load doubleword, 64-bit), `lw` (load word, 32-bit), `lh` (load halfword), `lb` (load byte), and the unsigned variants `lwu`, `lhu`, `lbu`. Store instructions: `sd`, `sw`, `sh`, `sb`. All use a register base plus a 12-bit signed offset.

**Branches:** `beq`, `bne`, `blt`, `bge`, `bltu`, `bgeu`. Compare two registers and branch to a PC-relative offset. No [condition flags](https://en.wikipedia.org/wiki/Status_register) register — the comparison and branch are in one instruction.

**Jumps:** `jal` (jump and link — stores return address in `rd`, jumps to PC-relative offset) and `jalr` (register-indirect jump — jumps to `rs1 + offset`). `ret` is a pseudoinstruction for `jalr x0, 0(ra)` — jump to the address in `ra`, discard the link.

**Upper immediate:** `lui` (load upper immediate — loads a 20-bit immediate into the upper bits of a register, zeroing the lower 12) and `auipc` (add upper immediate to PC). These two are the mechanism for building 32-bit constants or PC-relative addresses. The assembler's `li` pseudoinstruction may expand to a `lui`/`addi` pair.

**CSR instructions:** `csrrw`, `csrrs`, `csrrc`, `csrrwi`, `csrrsi`, `csrrci`. These read and/or modify CSRs. We'll cover them in detail shortly.

**System instructions:** `ecall` (environment call — triggers a trap, used for syscalls), `ebreak` (breakpoint — triggers a debug trap), `mret`/`sret` (return from M-mode/S-mode trap), `wfi` (wait for interrupt), `sfence.vma` ([TLB](https://en.wikipedia.org/wiki/Translation_lookaside_buffer) flush).

> **Aside: RISC vs. CISC — why this matters for OS development**
>
> RISC-V is a **load-store architecture**: arithmetic operates only on registers, and separate load/store instructions move data between registers and memory. x86 is a **CISC** architecture: a single instruction can load from memory, perform arithmetic, and store the result back. Why does this matter for OS development?
>
> In a load-store architecture, every memory access is explicit and visible in the assembly. When you're writing a context switch or a trap handler — saving and restoring 30+ registers — every `sd` (store doubleword) and `ld` (load doubleword) is visible in the instruction stream. You can count them. You can verify them. On x86, complex instructions with hidden memory operands make this accounting harder.
>
> The simplicity of RISC-V means more instructions for the same task, but each instruction is transparent. For learning OS internals, this transparency is a significant advantage.
>
> Patterson and Hennessy's *Computer Organization and Design: The RISC-V Edition* is the canonical reference for the architecture. Chapters 2 and 3 cover the instruction set and arithmetic.

---

## [Privilege Modes](https://en.wikipedia.org/wiki/Privilege_level): The Hardware Security Model

This is the section that matters most for OS development. The privilege mode architecture is the hardware mechanism that makes operating systems possible. Without it, every program could do anything — modify any memory, talk to any device, disable interrupts, corrupt the kernel. With it, the hardware enforces isolation between trust levels.

### The Three Modes

RISC-V defines three privilege levels:

| Mode | Encoding | Typical Use | Power Level |
|------|----------|-------------|-------------|
| M (Machine) | 3 | Firmware / SBI | Unrestricted |
| S (Supervisor) | 1 | OS kernel | Controlled by M-mode |
| U (User) | 0 | Applications | Restricted by S-mode |

The encoding matters because it appears in CSR fields. When you read the MPP (Machine Previous Privilege) field of `mstatus` and see the value 1, that means S-mode. When you see 0, that means U-mode.

**M-mode** is the highest privilege. M-mode code can:
- Access all memory without restriction
- Read and write all CSRs (M-mode, S-mode, and U-mode CSRs)
- Configure which traps are handled at which level
- Execute any instruction
- Control the interrupt controller directly

**S-mode** is where the OS kernel runs. S-mode code can:
- Access memory subject to the page table (if paging is enabled)
- Read and write S-mode and U-mode CSRs, but NOT M-mode CSRs
- Configure S-mode trap handling
- Manage virtual memory (the `satp` register)
- Execute most instructions, but NOT M-mode instructions like `mret`

**U-mode** is where applications run. U-mode code can:
- Access memory only as permitted by the page table
- NOT access any CSRs (attempting to do so causes a trap)
- NOT execute privileged instructions
- Request services from the kernel only via `ecall` (which triggers a trap)

### Transitions Between Modes

The CPU can only change privilege levels through specific mechanisms:

**Upward transitions (less privileged → more privileged): only via traps.**
- An exception occurs (illegal instruction, page fault, etc.)
- An interrupt fires (timer, external device)
- The `ecall` instruction is executed

When a trap occurs, the CPU atomically:
1. Records the trapped instruction's address in `mepc` (or `sepc` for S-mode traps)
2. Records the cause in `mcause` (or `scause`)
3. Saves the previous privilege mode in `mstatus.MPP` (or `mstatus.SPP`)
4. Disables interrupts (clears `mstatus.MIE` or `mstatus.SIE`)
5. Sets the privilege mode to M (or S, if the trap is delegated)
6. Jumps to the trap handler address in `mtvec` (or `stvec`)

This is all done by hardware, atomically, in a single cycle (conceptually). The software trap handler then runs and decides what to do.

**Downward transitions (more privileged → less privileged): only via trap return instructions.**
- `mret` — returns from M-mode to the mode specified by `mstatus.MPP`
- `sret` — returns from S-mode to the mode specified by `mstatus.SPP`

These instructions restore the previous privilege level and jump to the saved PC (`mepc` or `sepc`). By manipulating MPP/SPP and the saved PC before executing `mret`/`sret`, the OS can transition to any lower privilege level at any address. This is how the kernel launches user programs: set SPP to U-mode, set `sepc` to the program's entry point, and execute `sret`.

**There is no instruction that says "switch to U-mode."** The only way to lower the privilege level is through the trap-return mechanism. This is a security property: the kernel can always ensure that the system is in a known state before dropping to a lower privilege level.

```
      ┌─────────┐
      │ M-mode  │ ◄─── Traps from S-mode or U-mode
      │         │ ───► mret (to S or U, per MPP)
      └────┬────┘
           │
    ┌──────┴──────┐
    │   S-mode    │ ◄─── Traps from U-mode (if delegated)
    │             │ ───► sret (to U, per SPP)
    └──────┬──────┘
           │
    ┌──────┴──────┐
    │   U-mode    │ ◄─── Only entered via sret
    │             │ ───► Traps go up to S (or M)
    └─────────────┘
```

> **Aside: Trap delegation — M-mode doesn't have to handle everything**
>
> On a standard RISC-V system, all traps initially go to M-mode. But the firmware can configure **delegation**: specific exception and interrupt types can be delegated to S-mode, meaning they trap directly into the S-mode handler instead of going through M-mode first. This is done via the `medeleg` (machine exception delegation) and `mideleg` (machine interrupt delegation) CSRs.
>
> For example, the firmware typically delegates page faults, system calls (ecall from U-mode), and timer interrupts to S-mode. This way, the OS kernel handles these common traps directly without the overhead of bouncing through M-mode.
>
> Since we're starting in M-mode without firmware, we'll initially handle all traps in M-mode. Later, when we transition to S-mode, we'll set up delegation ourselves.
>
> See the [RISC-V Privileged Specification](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf), Section 3.1.8 (Machine Trap Delegation Registers) for the exact bit layout and which traps can be delegated.

---

## Control and Status Registers (CSRs)

[CSRs](https://en.wikipedia.org/wiki/Control_register) (Control and Status Registers) are special-purpose registers that control the CPU's behavior and report its status. They are not part of the general-purpose register file — you access them through dedicated CSR instructions (`csrrw`, `csrrs`, etc.), not through normal arithmetic instructions.

There are roughly 4096 possible CSR addresses (the CSR address is 12 bits). Not all are defined; most are reserved. The ones that matter for OS development fall into three groups based on privilege level:

### M-mode CSRs (The Most Important Ones)

**`mstatus` (Machine Status, CSR 0x300)**

This is the most complex and important CSR. It controls global interrupt enables, tracks the previous privilege mode, and holds several other configuration bits. Key fields:

```
Bit(s)  Field    Meaning
─────   ─────    ───────
3       MIE      Machine Interrupt Enable. When 0, all M-mode interrupts
                 are disabled regardless of mie register. When 1,
                 interrupts are enabled per the mie register.

7       MPIE     Machine Previous Interrupt Enable. When a trap is taken
                 into M-mode, the hardware saves the current MIE value
                 here. mret restores MIE from MPIE.

12:11   MPP      Machine Previous Privilege. When a trap is taken into
                 M-mode, the hardware saves the previous privilege mode
                 here (0=U, 1=S, 3=M). mret returns to this mode.

17      MPRV     Modify PRiVilege. When set, loads and stores use the
                 privilege mode in MPP instead of the current mode.
                 (Advanced — used for emulating lower-privilege memory
                 accesses from M-mode.)

18      SUM      Supervisor User Memory access. When set in S-mode,
                 allows S-mode to access pages marked as user-accessible.
                 (Relevant when you implement user/kernel memory access.)

19      MXR      Make eXecutable Readable. When set, allows loads from
                 pages that are execute-only. Some OSes need this for
                 reading code pages.
```

The MIE/MPIE/MPP dance is central to the trap mechanism. When a trap occurs:
1. MPIE ← MIE (save current interrupt state)
2. MIE ← 0 (disable interrupts during trap handling)
3. MPP ← current privilege mode

When `mret` executes:
1. MIE ← MPIE (restore interrupt state)
2. Privilege mode ← MPP
3. MPIE ← 1
4. MPP ← U (or M, depending on implementation)

This ensures that interrupts are disabled while the trap handler sets up its context, and restored when it returns.

**`mtvec` (Machine Trap Vector, CSR 0x305)**

Holds the address of the trap handler. When a trap occurs, the CPU jumps to this address. The low 2 bits specify the mode:
- Mode 0 (**Direct**): All traps jump to the address in `mtvec` (with the low 2 bits masked off).
- Mode 1 (**Vectored**): Exceptions jump to the base address, but interrupts jump to `base + 4 * cause`. This lets you have a jump table for different interrupt types.

For simplicity, we'll use direct mode: one trap handler that examines the cause and dispatches internally. Vectored mode is an optimization for fast interrupt dispatch.

**`mepc` (Machine Exception Program Counter, CSR 0x341)**

When a trap occurs, the hardware stores the address of the instruction that caused the trap (for exceptions) or the instruction that was about to execute when the interrupt arrived (for interrupts). `mret` jumps back to this address. If you handle a system call, you need to advance `mepc` by 4 before returning (because `ecall` is 4 bytes, and you want to return to the *next* instruction, not re-execute `ecall`).

**`mcause` (Machine Cause, CSR 0x342)**

Indicates why the trap occurred. The MSB (bit 63 on RV64) distinguishes interrupts (MSB = 1) from exceptions (MSB = 0). The remaining bits encode the specific cause:

Exceptions (bit 63 = 0):
```
Code  Cause
────  ─────
0     Instruction address misaligned
1     Instruction access fault
2     Illegal instruction
3     Breakpoint
4     Load address misaligned
5     Load access fault
6     Store/AMO address misaligned
7     Store/AMO access fault
8     Environment call from U-mode
9     Environment call from S-mode
11    Environment call from M-mode
12    Instruction page fault
13    Load page fault
15    Store/AMO page fault
```

Interrupts (bit 63 = 1):
```
Code  Cause
────  ─────
1     Supervisor software interrupt
3     Machine software interrupt
5     Supervisor timer interrupt
7     Machine timer interrupt
9     Supervisor external interrupt
11    Machine external interrupt
```

You'll use `mcause` in your trap handler to determine what happened and take the appropriate action. The page fault codes (12, 13, 15) become critically important in Chapter 10 (virtual memory).

**`mtval` (Machine Trap Value, CSR 0x343)**

Provides additional information about the trap. For address faults and page faults, `mtval` contains the faulting address. For illegal instruction faults, it contains the offending instruction encoding. Invaluable for debugging — when your kernel page faults, `mtval` tells you *which* address it tried to access.

**`mie` (Machine Interrupt Enable, CSR 0x304)**

Individual enable bits for each interrupt type. Even if `mstatus.MIE` is set (global enable), a specific interrupt only fires if its bit in `mie` is also set.

```
Bit  Interrupt
───  ─────────
1    Supervisor software interrupt (SSIE)
3    Machine software interrupt (MSIE)
5    Supervisor timer interrupt (STIE)
7    Machine timer interrupt (MTIE)
9    Supervisor external interrupt (SEIE)
11   Machine external interrupt (MEIE)
```

**`mip` (Machine Interrupt Pending, CSR 0x344)**

Shows which interrupts are currently pending (requested but not yet handled). Same bit layout as `mie`. Reading `mip` tells you what's waiting. Some bits are read-only (hardware sets them), others are read-write (software can set/clear them for software interrupts).

**`medeleg` / `mideleg` (Machine Exception/Interrupt Delegation, CSR 0x302 / 0x303)**

Each bit corresponds to a cause code. If the bit is set, that trap type is delegated to S-mode (handled by the S-mode trap handler at `stvec`) instead of M-mode. Bit positions match the cause codes in `mcause`.

For example, to delegate all U-mode ecalls to S-mode, set bit 8 of `medeleg`. To delegate timer interrupts to S-mode, set bit 5 of `mideleg`.

**`mscratch` (Machine Scratch, CSR 0x340)**

A scratch register for M-mode trap handlers. Its purpose is to give the trap handler one register to work with before it has saved any general-purpose registers. The typical pattern:
1. At trap entry, `csrrw t0, mscratch, t0` — atomically swap `t0` and `mscratch`. Now `t0` has whatever was in `mscratch` (typically a pointer to a scratch space), and `mscratch` has the old `t0`.
2. Use `t0` to address a save area and store the remaining registers.
3. After saving all registers, read the old `t0` from `mscratch` and save it too.

This is a clever trick that gives you a free register without corrupting any state. You'll see it again in `sscratch` for S-mode traps.

### S-mode CSRs (For When You Run as an OS)

The S-mode CSRs mirror the M-mode ones, but for S-mode trap handling:

| CSR | Address | M-mode Equivalent | Purpose |
|-----|---------|-------------------|---------|
| `sstatus` | 0x100 | `mstatus` | S-mode subset of machine status (SIE, SPIE, SPP, SUM, MXR) |
| `stvec` | 0x105 | `mtvec` | S-mode trap handler address |
| `sepc` | 0x141 | `mepc` | Exception PC for S-mode traps |
| `scause` | 0x142 | `mcause` | Trap cause for S-mode traps |
| `stval` | 0x143 | `mtval` | Trap value for S-mode traps |
| `sie` | 0x104 | `mie` | S-mode interrupt enables |
| `sip` | 0x144 | `mip` | S-mode interrupt pending |
| `sscratch` | 0x140 | `mscratch` | Scratch register for S-mode trap handlers |
| `satp` | 0x180 | (none) | **Supervisor Address Translation and Protection** — THE virtual memory control register |

The most important one here is **`satp`**. It has no M-mode equivalent because M-mode always uses physical addresses. `satp` controls the MMU:

```
Bits    Field    Meaning
─────   ─────    ───────
63:60   MODE     Address translation mode:
                 0 = Bare (no translation)
                 8 = Sv39 (39-bit virtual address, 3-level page table)
                 9 = Sv48 (48-bit virtual address, 4-level page table)

59:44   ASID     Address Space Identifier (for TLB tagging)

43:0    PPN      Physical Page Number of the root page table
```

Writing to `satp` enables or changes the page table. When you write a non-zero MODE and a PPN, the MMU starts translating every subsequent memory access through the page table rooted at the given physical address. This is one of the most consequential single-register writes in your entire OS. We'll cover it thoroughly in Chapter 10.

`sstatus` is not a separate register — it's a *view* (restricted projection) of `mstatus`. Reading `sstatus` in S-mode returns a subset of `mstatus` fields (SIE, SPIE, SPP, etc.), and writing to `sstatus` modifies those same bits in `mstatus`. The M-mode-only fields are hidden from S-mode.

### CSR Access Instructions

There are six CSR instructions, forming a 2×3 matrix:

|          | Read-Modify-Write | Set Bits | Clear Bits |
|----------|-------------------|----------|------------|
| Register | `csrrw rd, csr, rs1` | `csrrs rd, csr, rs1` | `csrrc rd, csr, rs1` |
| Immediate | `csrrwi rd, csr, imm` | `csrrsi rd, csr, imm` | `csrrci rd, csr, imm` |

- **`csrrw`** (CSR Read/Write): Atomically read the current CSR value into `rd`, then write `rs1` into the CSR.
- **`csrrs`** (CSR Read/Set): Read the CSR into `rd`, then set (OR in) the bits specified by `rs1`.
- **`csrrc`** (CSR Read/Clear): Read the CSR into `rd`, then clear (AND with NOT of) the bits specified by `rs1`.

The immediate variants use a 5-bit zero-extended immediate instead of a register, useful for setting/clearing a single bit.

Common pseudoinstructions built from these:
- `csrr rd, csr` → `csrrs rd, csr, x0` (read CSR, set no bits)
- `csrw csr, rs` → `csrrw x0, csr, rs` (write CSR, discard old value)
- `csrs csr, rs` → `csrrs x0, csr, rs` (set bits, discard old value)
- `csrc csr, rs` → `csrrc x0, csr, rs` (clear bits, discard old value)

The read-modify-write atomicity is important: even though these are conceptually two operations (read then modify), the hardware performs them as a single atomic action. On a multicore system, this prevents race conditions when two cores modify the same CSR simultaneously (though most CSRs are core-local, so this is less of a concern than it might seem).

> **Aside: The CSR address space is structured**
>
> CSR addresses aren't random. The 12-bit address encodes the privilege level and read/write permission:
>
> - Bits 11:10 — Read/write access: `00`, `01`, `10` = read/write; `11` = read-only
> - Bits 9:8 — Minimum privilege level needed to access: `00` = U, `01` = S, `11` = M
>
> So CSR `0x300` (`mstatus`): bits 11:10 = `00` (R/W), bits 9:8 = `11` (M-mode required). CSR `0xC00` (`cycle`): bits 11:10 = `11` (read-only), bits 9:8 = `00` (U-mode accessible).
>
> If S-mode code tries to access an M-mode CSR, the hardware raises an illegal instruction exception. If U-mode code tries to access any CSR that requires S-mode or higher, same thing. The hardware enforces privilege boundaries at the CSR level — no software check needed.
>
> See the RISC-V Privileged Specification, Section 2.1 (CSR Address Mapping Conventions) for the complete encoding scheme.

---

## The Memory Model

RISC-V uses a memory model called **[RVWMO](https://github.com/riscv/riscv-isa-manual/releases/download/Ratified-IMAFDQC/riscv-spec-20191213.pdf)** (RISC-V Weak Memory Ordering). For a single-core system — which is what we're building — you can largely ignore memory ordering, because a single core always sees its own writes in program order. But understanding the basics matters for three reasons:

1. When you eventually go multicore, memory ordering becomes critical.
2. The `fence` instruction exists and you'll see it in some code.
3. The distinction between regular memory and device memory (MMIO) affects how loads and stores behave.

### The Short Version

For a single-core OS targeting QEMU:

- **Regular memory (RAM)**: Loads and stores happen in program order from the perspective of the executing core. You generally don't need `fence` instructions.
- **Device memory (MMIO)**: Accesses to device registers should use `volatile` (to prevent compiler reordering) and are naturally ordered by the hardware for device regions. The QEMU `virt` machine marks device regions as "device, non-cacheable," which ensures strong ordering.
- **The `fence` instruction**: `fence` ensures that memory operations before the fence are visible before operations after the fence. `fence iorw, iorw` is a full barrier. `fence.i` synchronizes the instruction cache with memory (needed if you modify code, e.g., when loading a user program).

For our purposes: use `volatile` for MMIO, use `fence.i` after writing code to memory (before jumping to it), and don't worry about finer-grained memory ordering until multicore.

### Physical Memory Attributes (PMA)

The RISC-V specification defines Physical Memory Attributes that describe the properties of each physical address region: is it cacheable? Is it idempotent (can you read it multiple times and get the same result)? Is it naturally aligned?

For MMIO regions, the PMA specifies that accesses are non-cacheable and non-idempotent (each read of a UART data register might return a different character). The hardware uses these attributes to determine how to handle loads and stores to each region.

You don't configure PMAs in software — they're hardwired or set by the platform. On QEMU, the memory map implicitly defines them. But knowing they exist helps you understand why MMIO behaves differently from RAM.

---

## RISC-V Extensions

The RISC-V ISA is [modular](https://en.wikipedia.org/wiki/Modular_design). The base integer ISA (RV64I) is minimal — just arithmetic, logic, loads, stores, branches, and jumps. Extensions add additional functionality:

| Extension | Name | What It Adds |
|-----------|------|-------------|
| M | Multiply | `mul`, `div`, `rem` instructions |
| A | [Atomic](https://en.wikipedia.org/wiki/Atomic_operation) | Atomic memory operations: `lr`/`sc` (load-reserved/store-conditional), `amoswap`, `amoadd`, etc. |
| F | Single-float | 32-bit floating-point registers and instructions |
| D | Double-float | 64-bit floating-point (extends F) |
| C | Compressed | 16-bit instruction forms for common operations (code density) |
| G | General | Shorthand for IMAFD — the standard general-purpose configuration |

We're targeting **RV64GC** — 64-bit with G (general: I+M+A+F+D) and C (compressed) extensions. This is the standard configuration for application-class RISC-V processors.

For OS development, the most relevant extensions beyond the base ISA are:

- **A (Atomic)**: You'll need `amoswap` for implementing [spinlocks](https://en.wikipedia.org/wiki/Spinlock) (if you ever go multicore). `lr`/`sc` (load-reserved / store-conditional) provide an alternative locking mechanism similar to ARM's `ldrex`/`strex`.
- **C (Compressed)**: Transparent to the programmer. The assembler automatically uses 16-bit encodings when possible. This means instructions can be 2 or 4 bytes, which matters when you're advancing `mepc` after a trap — you need to advance by the actual instruction length, not always 4.

> **Aside: Compressed instructions and `mepc` advancement**
>
> Here's a trap (pun intended) that catches people. When handling an `ecall`, you advance `mepc` by 4 to skip past the `ecall` instruction. But `ecall` is always 4 bytes (it has no compressed form), so this is safe. For other exceptions where you might want to skip the faulting instruction (rare, but possible), you'd need to examine the instruction at `mepc` to determine if it's 2 or 4 bytes. On RV64GC, if the lowest two bits of the instruction are `11`, it's 4 bytes; otherwise, it's 2 bytes (compressed).
>
> The xv6 kernel avoids this issue by always advancing by 4 for `ecall` and never skipping other faulting instructions (it either fixes the fault or kills the process). This is the sane approach.

---

## Putting the Pieces Together: A Hardware Narrative

Let's trace what happens, at the hardware level, when a user program executes a [system call](https://en.wikipedia.org/wiki/System_call). This narrative ties together privilege modes, CSRs, and the trap mechanism — and previews what you'll build in Chapters 7, 17, and 18.

1. **A user program running in U-mode executes `ecall`.** The CPU recognizes `ecall` as a trap-generating instruction.

2. **The hardware takes a trap.** Assuming U-mode ecalls are delegated to S-mode (via `medeleg`), the CPU:
   - Saves the PC of `ecall` in `sepc`
   - Sets `scause` to 8 (environment call from U-mode)
   - Saves the current privilege mode (U) in `sstatus.SPP`
   - Saves `sstatus.SIE` in `sstatus.SPIE`, then clears `SIE` (disables S-mode interrupts)
   - Sets the privilege mode to S
   - Sets `pc` to `stvec` (the S-mode trap handler address)

   This all happens in a single clock cycle, atomically. No software has run — this is pure hardware.

3. **The S-mode trap handler executes.** This is your code. It:
   - Uses `sscratch` to get a working register (the `csrrw` swap trick)
   - Saves all user registers to a save area (the "trap frame")
   - Reads `scause` to determine the trap type (8 = U-mode ecall)
   - Reads the system call number from `a7` (by convention)
   - Dispatches to the appropriate system call handler
   - Places the return value in `a0` of the saved trap frame
   - Advances `sepc` by 4 (to skip past `ecall`)
   - Restores all user registers from the trap frame
   - Executes `sret`

4. **The hardware returns from the trap.** `sret`:
   - Restores `sstatus.SIE` from `sstatus.SPIE`
   - Sets the privilege mode to `sstatus.SPP` (U-mode)
   - Sets `pc` to `sepc` (which was advanced past `ecall`)
   - Sets `SPP` to U (implementation-defined, but commonly)

5. **The user program resumes** at the instruction after `ecall`, with the return value in `a0`.

This entire sequence — hardware trap, kernel handler, hardware return — is the fundamental mechanism of all OS services. Page faults, timer interrupts, device interrupts — they all follow the same structure. The cause differs, the handler logic differs, but the trap-handler-return skeleton is identical.

---

## The Interrupt Landscape

[Interrupts](https://en.wikipedia.org/wiki/Interrupt) are the mechanism by which external events (devices completing I/O, timers firing, other cores signaling) get the CPU's attention. On RISC-V, there are three sources of interrupts at each privilege level:

**Software interrupts** — Triggered by writing to a memory-mapped register in the CLINT. Used for inter-processor communication on multicore systems. One core writes to another core's software interrupt register, causing that core to trap.

**Timer interrupts** — Generated when `mtime` (a continuously incrementing counter in the CLINT) equals or exceeds `mtimecmp` (a comparator register). This is the heartbeat of your OS. Chapter 8 is entirely dedicated to this.

**External interrupts** — Routed through the [PLIC](https://github.com/riscv/riscv-plic-spec/blob/master/riscv-plic.adoc) (Platform-Level Interrupt Controller). These come from devices: the UART has data ready, the virtio block device finished an I/O operation, etc. The PLIC prioritizes and distributes external interrupts to cores.

The enabling logic for interrupts is a two-level gate:

```
Interrupt fires if and only if:
  1. Global interrupt enable is set (mstatus.MIE for M-mode, sstatus.SIE for S-mode)
  AND
  2. The specific interrupt type is enabled in mie/sie
  AND
  3. The interrupt is pending in mip/sip
```

All three conditions must be true. This gives you fine-grained control: you can globally disable all interrupts (clear MIE), or selectively disable specific types (clear bits in `mie`), or check what's pending without handling it (read `mip`).

> **Aside: NMI — Non-Maskable Interrupts**
>
> Most architectures have the concept of an NMI — an interrupt that cannot be disabled. On x86, the NMI is used for critical hardware errors (memory parity errors, watchdog timeouts). On RISC-V, NMI handling is implementation-defined and not covered by the base specification. The QEMU `virt` machine doesn't generate NMIs, so we can ignore them. But know that they exist in the wild.

---

## RISC-V vs. x86 vs. ARM: A Brief Comparison

If you've been exposed to other architectures, here's how RISC-V maps:

| Concept | RISC-V | x86-64 | ARM (AArch64) |
|---------|--------|--------|----------------|
| Privilege levels | M/S/U | Ring 0-3 (only 0 and 3 used) | EL0/EL1/EL2/EL3 |
| System call | `ecall` | `syscall` | `svc` |
| Trap return | `mret`/`sret` | `iret`/`sysret` | `eret` |
| Page table register | `satp` | `CR3` | `TTBR0_EL1`/`TTBR1_EL1` |
| Trap handler register | `mtvec`/`stvec` | IDT (table in memory) | `VBAR_EL1` |
| TLB flush | `sfence.vma` | `invlpg`/`mov cr3` | `tlbi` |
| Interrupt enable | `mstatus.MIE` | `IF` flag in RFLAGS | `DAIF` bits |
| I/O model | Memory-mapped only | MMIO + port I/O | Memory-mapped only |
| Condition flags | None (compare-and-branch) | RFLAGS (ZF, CF, etc.) | NZCV |
| Register count | 32 GPRs | 16 GPRs | 31 GPRs |

The most significant conceptual difference for OS development: x86 uses an **[Interrupt Descriptor Table](https://en.wikipedia.org/wiki/Interrupt_descriptor_table)** (a data structure in memory pointed to by the IDTR register) to map interrupt numbers to handler addresses. RISC-V uses a single trap vector (`mtvec`/`stvec`) that points to one handler; the handler itself dispatches based on the cause. ARM is closer to RISC-V's model.

---

## Conceptual Exercises

1. **Why does `x0` being hardwired to zero simplify the instruction set?** Give three examples of operations that other architectures need separate instructions for, but RISC-V implements using `x0` and existing instructions.

2. **A trap occurs while the CPU is in S-mode. Walk through exactly what the hardware does** — which CSRs are read, which are written, what values go where. Assume the trap is not delegated (goes to M-mode). Now repeat assuming the trap IS delegated to S-mode.

3. **What prevents a U-mode program from modifying `satp` to point to its own page table, giving itself access to all of physical memory?** Trace the hardware mechanism that enforces this restriction.

4. **You want to handle timer interrupts in S-mode. List every CSR you need to configure** and what value you'd set in each. (Hint: you need delegation, enables, and the trap vector.)

5. **`mepc` stores the address of the instruction that caused the trap. For an `ecall`, this is the address of the `ecall` instruction itself. Why must you advance `mepc` by 4 before executing `mret`?** What would happen if you didn't?

6. **Explain the `csrrw t0, mscratch, t0` trick.** Before the instruction executes, `t0` has some value X and `mscratch` has some value Y. After the instruction, what does `t0` contain? What does `mscratch` contain? Why is this useful as the first instruction of a trap handler?

7. **In the `mstatus` register, the MPP field is 2 bits wide, but only three privilege modes exist (M=3, S=1, U=0). What happens if you set MPP to 2 (which corresponds to the deprecated H-mode)?** Does the spec say? Why does this matter for security?

8. **Compare RISC-V's single trap vector (`mtvec`) with x86's IDT (256-entry table of handler addresses). What are the trade-offs?** When would vectored mode (`mtvec` mode 1) recover some of the IDT's advantages?

---

## Build This

This chapter is primarily reference material — there's no new runnable code. But there's important preparatory work:

**1. Write your CSR access header.**

If you started this in Chapter 2, expand it now. Create inline assembly wrappers for reading and writing each CSR you'll need in the next few chapters:
- `mstatus`, `mtvec`, `mepc`, `mcause`, `mtval`, `mie`, `mip`, `mscratch`
- `medeleg`, `mideleg`
- `sstatus`, `stvec`, `sepc`, `scause`, `stval`, `sie`, `sip`, `sscratch`, `satp`

For each CSR, you want at least a "read" function and a "write" function. For CSRs where you'll frequently set or clear individual bits (`mstatus`, `mie`), add "set bits" and "clear bits" wrappers using `csrs` and `csrc`.

**2. Define constants for the CSR field values.**

Create defines or constants for:
- `mstatus` bit positions: MIE, MPIE, MPP, SIE, SPIE, SPP, SUM, MXR
- `mie`/`sie` bit positions: SSIE, MSIE, STIE, MTIE, SEIE, MEIE
- `mcause`/`scause` values for each exception and interrupt type
- MPP field encodings: U=0, S=1, M=3
- `satp` MODE field values: Bare=0, Sv39=8, Sv48=9

**3. Read the RISC-V Privileged Specification, Chapters 3 and 4.**

Chapter 3 covers M-mode (machine-level ISA). Chapter 4 covers S-mode (supervisor-level ISA). You don't need to memorize every CSR, but you should skim the entire chapter and carefully read the sections on `mstatus`, `mtvec`, `mcause`, `mepc`, and the trap-handling description. These sections are the hardware contract that your trap handler must implement.

**Checkpoint:** Your project should compile cleanly with the expanded CSR header. No runtime test yet — the CSR wrappers will be exercised starting in Chapter 7. But verify that the inline assembly compiles without errors by checking `make` output.

As a useful exercise, write a small function that reads `mstatus` and prints its value (in hex). You can't print yet (no UART), but you can set a breakpoint in GDB, call the function, and inspect the return value with `info registers` or `print/x`. Verify that the MIE bit is 0 (interrupts should be disabled at boot).

---

## When Things Go Wrong

**"illegal CSR" or assembler errors on CSR names**
CSR names are case-sensitive in assembly. Use lowercase: `mstatus`, not `MSTATUS`. If your assembler doesn't recognize a CSR name, use its numeric address instead (e.g., `0x300` for `mstatus`). Check your toolchain version — older assemblers may not support all CSR names.

**Inline assembly compiles but the result is wrong**
Double-check your operand constraints. A common mistake: using `"r"` (input) where you meant `"=r"` (output), or vice versa. Also verify that the assembly template has the operands in the right order — `csrr %0, mstatus` reads `mstatus` into `%0`, but `csrw mstatus, %0` writes `%0` into `mstatus`. The operand order in CSR instructions is destination first for reads, source last for writes.

**Accessing an S-mode CSR from M-mode works fine, but the reverse causes a trap**
This is by design. M-mode can access all CSRs. S-mode can only access S-mode and U-mode CSRs. If you get an illegal instruction trap at a CSR access, check which privilege mode you're running in (the MPP field of `mstatus` after the trap tells you where you were).

**GDB shows `mstatus` as 0 at boot, but you expected some bits to be set**
Check that you're reading the right register. On some QEMU versions, the initial `mstatus` value might have some bits set (like the FS field for floating-point state). If you're reading all zeros and your CSR access wrapper uses the correct CSR address, the wrapper is probably working correctly — `mstatus` really is mostly zeros at reset.

---

## Further Reading

- **[RISC-V Privileged Specification](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf)** — The definitive reference. Read Chapter 3 (Machine-Level ISA) and Chapter 4 (Supervisor-Level ISA) carefully. Sections 3.1.6 (mstatus), 3.1.7 (mtvec), and 3.1.15-3.1.17 (trap handling) are essential.
- **[RISC-V Unprivileged Specification](https://github.com/riscv/riscv-isa-manual/releases/download/Ratified-IMAFDQC/riscv-spec-20191213.pdf)** — Instruction encoding, register conventions, the base ISA. Chapter 2 (RV32I/RV64I) and Chapter 25 (RVWMO memory model).
- **Patterson and Hennessy, *Computer Organization and Design: The RISC-V Edition*** — Chapters 2 and 3 cover the instruction set and arithmetic. Appendix A covers the assembly language. This is the textbook for the architecture.
- **CS:APP Chapter 3: Machine-Level Representation of Programs** — While focused on x86, the concepts (calling conventions, stack frames, register usage) apply universally. Read it for the conceptual framework, then map to RISC-V specifics.
- **[xv6 Book](https://pdos.csail.mit.edu/6.828/2024/xv6/book-riscv-rev4.pdf), Chapter 2 (Operating System Organization) and Chapter 4 (Traps)** — The xv6 treatment of RISC-V trap handling is clear and practical. Read it alongside the spec.
- **[RISC-V ELF psABI Specification](https://github.com/riscv-non-isa/riscv-elf-psabi-doc)** — The calling convention, register usage, and data layout. Available in the `riscv-non-isa` GitHub organization.

---

*Next: [Chapter 5 — First Boot](ch05-first-boot.md), where we take everything from these four foundational chapters and produce a kernel that actually boots, reaches C code, and is ready for its first I/O.*
