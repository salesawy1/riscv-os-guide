# Chapter 6: UART and Serial Output

**[Difficulty: ★★☆☆☆]**

---

## Why This Chapter Exists

You have a kernel that boots. You know this because GDB told you so. But GDB is a crutch — a powerful one, but a crutch. You can't run GDB in production. You can't attach GDB every time you want to check if something worked. You need your kernel to *speak* — to produce visible output on its own, without external tools probing its state.

This chapter gives your kernel a voice.

The [UART](https://wiki.osdev.org/Serial_Ports) (Universal Asynchronous Receiver/Transmitter) is a serial communication device. On the QEMU `virt` machine, it's a simulated [NS16550A](https://en.wikipedia.org/wiki/16550_UART) — the same chip that's been in PCs since the 1980s, and whose programming interface has become a de facto standard. When QEMU is run with `-nographic`, the UART is connected to your terminal: anything your kernel writes to the UART's transmit register appears as text on your screen. Anything you type on your keyboard is readable from the UART's receive register.

By the end of this chapter, you'll have:
1. A UART driver that can send and receive characters
2. A `kprintf` function that formats and prints strings, integers, and hex values
3. The ability to print "Hello from the kernel!" on your terminal

This might sound trivial. It isn't. This is your first interaction with a hardware device. You're going to learn memory-mapped I/O — not as an abstract concept from Chapter 1, but by actually writing bytes to specific physical addresses and watching a device respond. Every device driver you write from here on (the timer in Chapter 8, the block device in Chapter 20) follows the same pattern: find the device's registers in the memory map, read the datasheet to understand what each register does, and write C code that reads and writes those registers in the correct sequence. The UART is the simplest device in the system, which makes it the perfect first driver.

Once you have `kprintf`, debugging changes fundamentally. Instead of stepping through instructions in GDB, you can sprinkle print statements through your code. Yes, this is the `printf` debugging that CS professors look down on. In OS development, it's the most effective debugging technique there is, because the bugs you'll encounter — subtle memory corruption, race conditions, incorrect page table entries — are often easier to trace with print output than with breakpoints. Every production kernel has a `printk` or equivalent, and kernel developers use it constantly.

---

## Memory-Mapped I/O: The Concept Made Real

In Chapter 1, we mentioned that RISC-V uses [memory-mapped I/O](https://en.wikipedia.org/wiki/Memory-mapped_I/O) ([MMIO](https://wiki.osdev.org/Memory_Mapped_IO)): device registers appear as addresses in the physical address space. Let's make this concrete.

The UART on the QEMU `virt` machine has its registers starting at physical address `0x10000000`. This doesn't mean there's RAM at `0x10000000`. The memory controller recognizes that address range and routes reads and writes to the UART hardware (or, on QEMU, to the UART simulation logic) instead of to the DRAM controller.

<details>
<summary>How does the CPU distinguish between stores to RAM and stores to MMIO devices?</summary>
<div>

It doesn't — the CPU just puts an address and data on the bus. The memory interconnect decodes the address and routes it: if it's in the RAM range, it goes to the DRAM controller; if it's in a device range like `0x10000000`, it goes to the UART. From the CPU's perspective, a store to a UART register is identical to a store to RAM. This is the elegance of MMIO — no special instructions, just load/store to magic addresses.

</div>
</details>

<details>
<summary>Why is `volatile` mandatory for MMIO but not for RAM?</summary>
<div>

RAM reads return the same value (until you write new data), so the compiler can cache them. MMIO registers have side effects: reading UART data *consumes* the character, and writing UART data *transmits* it. Without `volatile`, the compiler might optimize away repeated reads/writes or reorder them. `volatile` tells the compiler: this memory has invisible side effects, don't optimize.

</div>
</details>

> **Aside: Port-mapped I/O on x86**
>
> x86 has two I/O models: memory-mapped I/O (same as RISC-V) and **port-mapped I/O**, which uses a separate 64 KiB I/O address space accessed via dedicated `in` and `out` instructions. The legacy PC UART (COM1) is at I/O port `0x3F8`, accessed with `outb 0x3F8, al` (not a memory store). This dual-addressing scheme is a historical artifact — modern x86 devices primarily use MMIO, and the port I/O space is mostly legacy.
>
> RISC-V has only MMIO. ARM has only MMIO. The I/O port model is an x86-ism. One fewer thing to worry about.

---

## The NS16550A: Your First Datasheet

The [NS16550A](https://en.wikipedia.org/wiki/16550_UART) is a UART chip originally designed by National Semiconductor in 1987. It became the standard serial port controller in IBM PC compatibles and its register interface has been emulated by countless devices since. QEMU's `virt` machine emulates an NS16550A-compatible UART at address `0x10000000`.

Understanding a device from its datasheet (or register specification) is a core skill in OS development. Let's walk through the NS16550A's registers.

### Register Map

The NS16550A has 8 registers, each 8 bits wide, at consecutive byte offsets from the base address:

```
Offset  Name (read)        Name (write)       Description
──────  ──────────         ────────────       ───────────
0x00    RBR                THR                Receive Buffer / Transmit Holding
0x01    IER                IER                Interrupt Enable Register
0x02    IIR                FCR                Interrupt ID / FIFO Control
0x03    LCR                LCR                Line Control Register
0x04    MCR                MCR                Modem Control Register
0x05    LSR                LSR                Line Status Register
0x06    MSR                MSR                Modem Status Register
0x07    SCR                SCR                Scratch Register
```

<details>
<summary>How can one address be two different registers?</summary>
<div>

Offset 0x00: reading gives RBR (receive buffer), writing goes to THR (transmit). Same physical address, different behavior based on read vs. write. This saves address space — hardware decodes the bus signal to distinguish load from store. Similarly, 0x02 is IIR when read, FCR when written. This pattern appears throughout hardware.

</div>
</details>

Let's go through the registers we actually need:

### THR — Transmit Holding Register (Write, Offset 0x00)

Write a byte here to transmit it. The UART serializes the byte and sends it out. On QEMU, the character appears in your terminal. Simple as that — write a byte, it shows up on screen.

<details>
<summary>What happens if you write to THR without checking if the UART is ready?</summary>
<div>

The UART can only transmit so fast. Writing a second byte before the first completes might cause the second byte to be dropped or corrupted. You must check the THRE bit in LSR before writing to ensure the transmit holding register is empty.

</div>
</details>

### RBR — Receive Buffer Register (Read, Offset 0x00)

<details>
<summary>What happens if you read RBR when no character is available?</summary>
<div>

On the real chip, you get the last received character (or garbage). On QEMU, you get whatever's in the buffer. You must check the Data Ready bit in LSR to know if new data actually arrived. Reading RBR without checking LSR gives stale or garbage data.

</div>
</details>

### LSR — Line Status Register (Read, Offset 0x05)

This is the status register you check before transmitting or receiving. Key bits:

```
Bit 0: Data Ready (DR)
       1 = A character has been received and is waiting in RBR
       0 = No character available

Bit 5: Transmitter Holding Register Empty (THRE)
       1 = THR is empty; you can write a new character
       0 = THR is full; wait before writing

Bit 6: Transmitter Empty (TEMT)
       1 = Both THR and shift register are empty (transmission complete)
       0 = Transmission in progress
```

The standard pattern for transmitting:
1. Read LSR
2. Check bit 5 (THRE). If it's 0, loop back to step 1.
3. Write your byte to THR.

The standard pattern for receiving:
1. Read LSR
2. Check bit 0 (DR). If it's 0, no character available (return, or loop if blocking).
3. Read the byte from RBR.

### IER — Interrupt Enable Register (Read/Write, Offset 0x01)

Controls which UART events generate interrupts:

```
Bit 0: Received Data Available Interrupt Enable
Bit 1: Transmitter Holding Register Empty Interrupt Enable
Bit 2: Receiver Line Status Interrupt Enable
Bit 3: Modem Status Interrupt Enable
```

<details>
<summary>Why poll instead of using UART interrupts now?</summary>
<div>

Polling means busy-waiting on LSR in a loop. It's wasteful of CPU cycles but simple and doesn't require a trap handler. Interrupts would notify the kernel when data arrives, freeing the CPU to do other work. But interrupts require trap infrastructure (Chapter 7), which we don't have yet. We'll upgrade to interrupts later.

</div>
</details>

### LCR — Line Control Register (Read/Write, Offset 0x03)

Configures the serial line parameters:

```
Bits 1:0  Word Length Select
          00 = 5 bits
          01 = 6 bits
          10 = 7 bits
          11 = 8 bits

Bit 2     Stop Bits
          0 = 1 stop bit
          1 = 1.5 or 2 stop bits

Bit 3     Parity Enable

Bits 5:4  Parity Select (if parity enabled)

Bit 7     Divisor Latch Access Bit (DLAB)
          When set, offsets 0x00 and 0x01 access the baud rate
          divisor instead of THR/RBR and IER
```

For our purposes: 8 data bits, 1 stop bit, no parity. That's `0b00000011` (0x03) written to LCR, with DLAB cleared.

### FCR — FIFO Control Register (Write, Offset 0x02)

Controls the 16-byte FIFOs (the NS16550A has FIFOs; the older 16450 didn't):

```
Bit 0: FIFO Enable
       1 = Enable the transmit and receive FIFOs
       0 = Disable FIFOs (16450 compatibility mode)

Bit 1: Receiver FIFO Reset
       Writing 1 clears the receive FIFO

Bit 2: Transmitter FIFO Reset
       Writing 1 clears the transmit FIFO

Bits 7:6: Receiver FIFO Trigger Level
          How many bytes in the FIFO before an interrupt is generated
          00 = 1 byte
          01 = 4 bytes
          10 = 8 bytes
          11 = 14 bytes
```

Enable FIFOs by writing `0x01` (or `0x07` to enable and reset both FIFOs). FIFOs allow the UART to buffer multiple characters, reducing the chance of data loss at high throughput.

### The Divisor Latch (Baud Rate Configuration)

The UART's transmission speed (baud rate) is configured via a 16-bit divisor:

```
Divisor = Input Clock Frequency / (16 × Desired Baud Rate)
```

On the QEMU `virt` machine, the UART's input clock is 3.6864 MHz (a traditional serial port clock frequency). For 38400 baud:

```
Divisor = 3686400 / (16 × 38400) = 6
```

To set the divisor:
1. Set the DLAB bit (bit 7) of LCR to 1. This remaps offsets 0x00 and 0x01 to the divisor latch low/high bytes.
2. Write the low byte of the divisor to offset 0x00.
3. Write the high byte of the divisor to offset 0x01.
4. Clear the DLAB bit in LCR (and set your desired word length, etc.).

On QEMU, the [baud rate](https://en.wikipedia.org/wiki/Serial_communication) setting is largely cosmetic — the emulated UART transfers data instantly regardless of the configured baud rate. But initializing it correctly is good practice, and the sequence of setting DLAB, writing the divisor, and clearing DLAB teaches you an important pattern: **register overloading controlled by a mode bit.** The same physical offset serves different purposes depending on a bit in another register. This pattern appears in many hardware devices.

> **Aside: Why 8-N-1?**
>
> "8-N-1" means 8 data bits, No parity, 1 stop bit. This is the universal default for modern serial communication. Historically, serial terminals used 7 data bits (enough for ASCII) with various parity settings for error detection. The 8-N-1 convention became standard with the shift to 8-bit character sets and binary data transfer. Every terminal emulator, every serial monitor, and every UART default uses 8-N-1. If your serial output shows garbled characters, a baud rate or framing mismatch is the first thing to check — but on QEMU, this virtually never happens.

---

## UART Initialization

Before you can transmit or receive, you need to initialize the UART. The initialization sequence:

1. **Disable interrupts.** Write 0x00 to IER. We don't have interrupt handling yet, and enabling UART interrupts before setting up the trap handler would cause an unhandled trap.

2. **Set the baud rate.** Set DLAB in LCR, write the divisor low and high bytes, then clear DLAB.

3. **Configure the line.** Write the desired line control settings (8-N-1) to LCR. This happens as part of step 2 — when you clear DLAB, you simultaneously set the word length, stop bits, and parity.

4. **Enable FIFOs.** Write to FCR to enable and reset both FIFOs.

5. **Optionally, enable interrupts later.** We'll come back to this in Chapter 7 when we have a trap handler.

The order matters. You set the baud rate before enabling FIFOs because some hardware implementations behave unpredictably if you enable FIFOs first. On QEMU, the order probably doesn't matter, but following the conventional initialization sequence is good practice.

---

## Writing a Character

<details>
<summary>Why is busy-waiting wasteful, and when is it acceptable?</summary>
<div>

Busy-waiting spins the CPU in a tight loop, burning cycles reading LSR until THRE is set. This wastes power and CPU time. On a real system with many processes, you'd want interrupt-driven I/O. But at this stage — early kernel development, single-threaded — simplicity matters more than efficiency. Once you have a scheduler and meaningful work to do during I/O waits, you'll switch to interrupts.

</div>
</details>

Later, when you have interrupts (Chapter 7), you can switch to interrupt-driven I/O: instead of polling LSR, you enable the THRE interrupt, write a character, and then go do other work. When the UART is ready for the next character, it generates an interrupt, and your interrupt handler sends the next byte. This frees the CPU from busy-waiting. But that requires infrastructure we don't have yet.

<details>
<summary>How does UART transmission speed differ on QEMU vs. real hardware?</summary>
<div>

On QEMU, transmission is instant — characters appear immediately. The busy-wait loop executes zero times. On real hardware at 115200 baud, each character takes about 87 microseconds. A long string means significant CPU wait time. At 9600 baud, it's a millisecond per character. This is why real systems use interrupt-driven I/O — but on QEMU, polling is fine for development.

</div>
</details>

---

## Reading a Character

The complement of writing: get one byte from the keyboard.

1. Read LSR and check bit 0 (Data Ready).
2. If bit 0 is clear, no character is available. Either return immediately (non-blocking read) or loop (blocking read).
3. If bit 0 is set, read RBR (offset 0x00) to get the character.

You'll want both blocking and non-blocking versions. The non-blocking version is useful for checking if input is available without stalling the kernel. The blocking version is useful when the kernel is explicitly waiting for user input (e.g., in a shell).

On QEMU with `-nographic`, your terminal's stdin is connected to the UART. Characters you type go into the UART's receive FIFO. Pressing Enter sends a newline. Ctrl-C sends byte 0x03 (but be careful — QEMU may intercept it). Ctrl-A followed by X quits QEMU. Remember that Ctrl-A is QEMU's escape prefix, so Ctrl-A Ctrl-A sends a literal Ctrl-A.

---

## Building kprintf: Your Kernel's Voice

Raw character output is necessary but tedious. You don't want to write one character at a time for every message. You need formatted output — the kernel equivalent of `printf`.

You can't use the standard library's `printf` — it doesn't exist in your kernel. You'll write your own, typically called `kprintf` or `printk` ([Linux convention](https://en.wikipedia.org/wiki/Printf)). It doesn't need to support the full `printf` specification. You need:

- **`%s`** — String (null-terminated `char *`)
- **`%d`** — Signed decimal integer
- **`%u`** — Unsigned decimal integer
- **`%x`** — Unsigned hexadecimal integer (lowercase)
- **`%p`** — Pointer (just hex with a "0x" prefix)
- **`%c`** — Single character
- **`%%`** — Literal percent sign

That's it. You don't need width specifiers, padding, floating-point, or any of the other `printf` features. If you want them later, add them later. Start minimal.

### The Implementation Strategy

`kprintf` is a variadic function — it takes a format string and a variable number of arguments. C's variadic mechanism uses `<stdarg.h>`, which is a freestanding header (available without libc). The macros you need:

- **`va_list`** — A type that holds the state of the variadic argument list.
- **`va_start(ap, last_fixed_arg)`** — Initialize `ap` to point to the first variadic argument after `last_fixed_arg`.
- **`va_arg(ap, type)`** — Retrieve the next argument as `type` and advance `ap`.
- **`va_end(ap)`** — Clean up (typically a no-op, but call it anyway).

The logic of `kprintf`:

1. Walk through the format string character by character.
2. If the character is not `%`, output it directly (via your UART write function).
3. If the character is `%`, read the next character to determine the format specifier.
4. Based on the specifier, retrieve the appropriate argument with `va_arg` and convert it to a string representation.
5. Output the string representation character by character.

### Integer-to-String Conversion

The trickiest part of `kprintf` is converting integers to strings. For `%d` and `%u`, you need decimal conversion. For `%x`, hexadecimal.

**Decimal conversion:** The standard algorithm divides the number by 10 repeatedly, collecting remainders (which are the digits, in reverse order). You then output the digits in reverse. The classic approach:

1. Handle the sign for `%d`: if negative, output `-` and negate the number. Watch out for the edge case where the number is the minimum value of the type (`INT64_MIN` = `-2^63`), which cannot be negated without overflow. Either handle it as a special case or work with unsigned values internally.
2. Divide by 10 in a loop, storing each remainder as a digit character (`'0' + remainder`). Stop when the quotient is zero.
3. You've accumulated digits in reverse order. Either reverse them, or store them in a buffer starting from the end and work backward.

<details>
<summary>How do you extract hex digits without dividing?</summary>
<div>

Shift right by 4 bits and mask with 0xF to extract each nibble (4 bits). Repeat 16 times for a 64-bit value. Convert each 4-bit value to a character: 0-9 stays as `'0'–'9'`, 10-15 becomes `'a'–'f'`. Much simpler than decimal division.

</div>
</details>

**Pointer (`%p`):** Print `0x` followed by the pointer value in hex. A `uintptr_t` is 64 bits on RV64, so you'll print up to 16 hex digits.

### A Note on Buffer Management

You have two choices for integer-to-string conversion:

1. **Use a small stack buffer.** A 64-bit integer in decimal is at most 20 digits. A 21-byte stack buffer (20 digits + null terminator) is more than enough. Fill it from the end, then output it.

2. **Output digits directly as you compute them.** This avoids the buffer but produces digits in reverse order. You'd need to reverse them, which requires either a buffer or two passes.

The stack buffer approach is simpler and more common. A 21-byte buffer is trivial.

> **Aside: Why not just call `printf`?**
>
> You might wonder if you could link against a minimal libc like `newlib-nano` and get `printf` for free. You could, and some OS tutorials do this. The problem is that `printf` relies on a `write` function (or `_write` stub) that knows how to output characters. You'd need to provide this stub, which calls your UART driver. At that point, you've written the hard part anyway — the UART driver and the I/O plumbing. The `printf` formatting is the easy part.
>
> More importantly, using a canned `printf` means you don't understand what it does. When it breaks (and it will — mismatched format specifiers, incorrect type widths, memory corruption in the format string), you need to debug it. If you wrote it, you can debug it. If you linked against a library, you're debugging someone else's code with no context.
>
> The xv6 kernel has its own `printf` implementation in `printf.c`. It's about 80 lines. Linux's `printk` is considerably more complex (it handles log levels, buffering, per-CPU contexts, and output to multiple consoles), but the core formatting logic is the same algorithm you'll write.

---

## Putting It Together: The UART Driver

Your UART driver should be organized as a few functions:

**`uart_init()`** — Initializes the UART: disables interrupts, sets the baud rate, configures line parameters, enables FIFOs. Called once from `kernel_main`.

**`uart_putc(char c)`** — Sends a single character. Busy-waits on LSR bit 5, then writes to THR. This is the primitive that everything else is built on.

**`uart_getc()`** — Receives a single character (non-blocking). Checks LSR bit 0; if a character is available, reads and returns it. If not, returns -1 (or some sentinel value).

**`uart_puts(const char *s)`** — Sends a null-terminated string. Just calls `uart_putc` in a loop. Convenient but not strictly necessary if you have `kprintf`.

**`kprintf(const char *fmt, ...)`** — Formatted output. The workhorse.

### Handling Newlines

Terminal convention: a "new line" is actually two operations — carriage return (move cursor to column 0) and line feed (move cursor down one row). In Unix, `\n` is just a line feed (LF, byte `0x0A`). On raw serial terminals, if you only send LF, the cursor moves down but stays in the same column, producing staircase output:

```
Hello
     World
          Foo
```

To get proper line breaks on a raw serial connection, you need to send both `\r` (carriage return, CR, byte `0x0D`) and `\n` (line feed, LF). <details>
<summary>Why send `\r\n` instead of just `\n`?</summary>
<div>

Unix uses `\n` alone, but raw serial terminals need both CR (carriage return, move to column 0) and LF (line feed, advance one line). Without `\r`, output looks like a staircase on real hardware. QEMU's terminal emulator tolerates bare `\n`, but it's good practice to send `\r\n` — it works everywhere and avoids surprises on real hardware.

</div>
</details>

> **Aside: The history of CR and LF**
>
> This goes back to mechanical teleprinters and typewriters. The carriage return physically moved the print head back to the left margin (a mechanical operation that took time). The line feed advanced the paper by one line. They were separate operations because the typewriter needed time to move the carriage back — if you sent the next character too quickly after CR, the carriage might not have reached the left margin yet.
>
> Different operating systems adopted different conventions: Unix uses LF only (`\n`), Windows uses CR+LF (`\r\n`), classic Mac OS used CR only (`\r`). This is why text files from different platforms sometimes display incorrectly on other platforms. The convention persists in serial communication: a bare LF might not produce the expected behavior on all terminals. Sending CR+LF is the safest choice.
>
> This is one of those details that seems ridiculous until it costs you an hour of debugging because your serial output looks like a staircase.

---

## The UART as a Struct Overlay

In Chapter 2, we discussed using structs as hardware overlays. The UART is a perfect candidate. You can define a struct with `volatile uint8_t` fields for each register, then cast the UART base address to a pointer to this struct.

The advantage: instead of writing to `*(volatile uint8_t *)(UART_BASE + 0x05)` every time you check LSR, you write something like `uart->lsr`. This is more readable, less error-prone, and self-documenting.

The struct needs to be 8 bytes total (8 registers × 1 byte each), with fields at the correct offsets. Since all fields are `uint8_t` (1 byte), there's no alignment padding to worry about. The struct's natural layout matches the hardware register layout exactly.

<details>
<summary>How do you represent offset 0x00 (THR/RBR) in a struct?</summary>
<div>

Option 1: Use a single field name (`data`). Reads return RBR, writes go to THR. The hardware handles it; the C code just uses one field. Option 2: Use a union with `thr` and `rbr`. More explicit but adds complexity. Option 3: Skip the struct, use constants and helper functions (what xv6 does). Pick based on readability vs. simplicity. All work.

</div>
</details>

---

## Testing Your UART Driver

### The First Print

After implementing `uart_init()` and `uart_putc()`, your `kernel_main` can do this:

```c
void kernel_main(void) {
    uart_init();
    uart_puts("Hello from the kernel!\n");
    while (1) {}
}
```

Run `make run`. If everything works, you'll see:

```
Hello from the kernel!
```

on your terminal. Then QEMU will appear to hang (infinite loop). Exit with Ctrl-A, X.

This is the moment. This is the first time your OS has produced visible output. Every operating system you've ever used started with a moment exactly like this one.

### Testing kprintf

Once `kprintf` is working, test all your format specifiers:

```c
kprintf("String: %s\n", "test");
kprintf("Decimal: %d\n", -42);
kprintf("Unsigned: %u\n", 12345);
kprintf("Hex: %x\n", 0xDEADBEEF);
kprintf("Pointer: %p\n", (void *)0x80000000);
kprintf("Char: %c\n", 'A');
kprintf("Percent: %%\n");
```

Expected output:

```
String: test
Decimal: -42
Unsigned: 12345
Hex: deadbeef
Pointer: 0x80000000
Char: A
Percent: %
```

If the output is garbled (wrong characters, missing characters, extra characters), the most common causes are:

1. **UART not initialized.** Make sure `uart_init()` is called before any output.
2. **Wrong register offsets.** Double-check your offset constants against the register map.
3. **Missing `volatile`.** If the compiler optimized away your LSR reads, the busy-wait loop might not work correctly.
4. **Integer conversion bug.** Off-by-one errors in the digit loop, wrong sign handling, or buffer overflow. Test with edge cases: 0, -1, `INT64_MIN`, `UINT64_MAX`.

### Testing Input

Test `uart_getc()` by adding a simple echo loop:

```c
while (1) {
    int c = uart_getc();
    if (c != -1) {
        uart_putc((char)c);
    }
}
```

This should echo back every character you type. If it doesn't, check LSR bit 0 handling. Remember that on QEMU with `-nographic`, Ctrl-A is QEMU's escape prefix — it won't be passed to your kernel.

---

## kprintf Design Decisions

A few decisions you'll face when implementing `kprintf`:

### Width and Padding

Do you support `%08x` (zero-padded to 8 digits)? For a first implementation, no. But you'll want it eventually, because hexadecimal values are much easier to read with consistent widths. A 64-bit address looks much better as `0x0000000080000000` than as `0x80000000` — when scanning a list of addresses, fixed width helps alignment.

If you do implement width specifiers, the parsing gets more complex: after the `%`, you need to check for a `0` (zero-pad flag), then parse a decimal number (the width), then the format specifier. It's not hard, but it's more code. Add it when you need it, not before.

### Long and Long-Long Modifiers

On RV64, `int` is 32 bits, `long` is 64 bits, and pointers are 64 bits. If your `%d` treats its argument as `int` (32-bit), printing a 64-bit address with `%d` will give wrong results. You have options:

1. **Always treat integer arguments as 64-bit** (`int64_t` for `%d`, `uint64_t` for `%u` and `%x`). This is the simplest approach and avoids surprises. The variadic argument promotion rules in C ensure that `int` arguments are promoted to `int` (or `unsigned int`) — not to `int64_t`. But since we control both the `kprintf` implementation and all call sites, we can use `long` or `uint64_t` consistently.

2. **Support `%ld` and `%lx` modifiers.** More compatible with `printf` conventions but more parsing.

3. **Just use `%d` for 32-bit and `%p` for 64-bit addresses.** Keep it simple.

For a teaching OS, option 1 (always 64-bit) is pragmatic. You'll mostly print addresses and register values, which are 64-bit. If you occasionally print a 32-bit value, it'll work (it just got promoted on the stack).

Actually, wait — this needs a careful note about variadic promotion. When you pass an `int` to a variadic function, C promotes it to `int` (not `long`). If your `kprintf` retrieves it as `long` (via `va_arg(ap, long)`), you get undefined behavior if `int` and `long` are different sizes. On RV64, `int` is 32 bits and `long` is 64 bits, so `va_arg(ap, long)` when an `int` was passed is technically UB (though it often works because the upper 32 bits are either zero or sign-extended).

The safe approach: retrieve `int`-sized arguments with `va_arg(ap, int)` for `%d` and `va_arg(ap, unsigned int)` for `%u`, and add `%ld`/`%lu`/`%lx` for 64-bit arguments. Or, decide that all your `kprintf` arguments will always be 64-bit and enforce this at call sites (by casting if necessary).

This is a fiddly detail, but variadic type mismatches are a classic source of subtle bugs. Decide on a convention and stick to it.

> **Aside: Linux's printk**
>
> Linux's `printk` is far more capable than what you're building. It supports log levels (`KERN_ERR`, `KERN_INFO`, etc.), deferred output via a ring buffer, per-CPU logging, rate limiting, and output to multiple consoles simultaneously (serial, VGA, network). The formatting engine supports every `printf` specifier plus kernel-specific ones like `%pS` (print a symbol name for a kernel address) and `%pM` (print a MAC address).
>
> Despite this complexity, the core algorithm is the same: walk the format string, extract arguments, convert to text, emit characters. The difference is in the infrastructure around it. You'll build a simple version now; if you ever look at Linux's `vsprintf.c`, you'll recognize the algorithm.
>
> See `lib/vsprintf.c` in the Linux kernel source for the formatting engine, and `kernel/printk/printk.c` for the ring buffer and console management.

---

## UART Interrupts (Preview)

We're using polling now, but let's preview how UART interrupts will work when we add trap handling in Chapter 7.

The UART can generate interrupts for four events:
1. Received data available (character waiting to be read)
2. Transmitter holding register empty (ready for next character)
3. Receiver line status change (error conditions)
4. Modem status change (we don't care about this one)

The most useful is "received data available." With polling, you'd need to periodically call `uart_getc()` to check for input. With interrupts, the UART tells you when a character arrives. Your interrupt handler reads the character and puts it in a buffer. Your kernel reads from the buffer when it's ready.

To enable UART interrupts:
1. Write the appropriate bits to IER (offset 0x01). Bit 0 enables the received-data interrupt.
2. Configure the PLIC (Platform-Level Interrupt Controller) to route the UART's interrupt line to the CPU. The UART is connected to PLIC interrupt source 10 on the `virt` machine.
3. Enable external interrupts in `mie` or `sie`.
4. Have a trap handler that can service external interrupts.

We'll do all of this in Chapters 7 and 8. For now, polling works.

---

## Conceptual Exercises

1. **Why must MMIO accesses be `volatile`?** Consider what happens if the compiler caches the result of reading LSR: you read it once, it says "not ready," and the compiler reuses that value for every subsequent check. How does this manifest as a bug? Would you see the bug at `-O0`? At `-O2`?

2. **The NS16550A uses the same address (offset 0x00) for both the receive buffer and the transmit holding register.** How does the hardware know whether you're reading or writing? What signal on the bus distinguishes a load from a store?

3. **Why does the UART have FIFOs?** What problem do they solve? What would happen if the UART had no FIFO and your kernel was slow to read incoming characters? (Think about what happens at high baud rates.)

4. **Your `kprintf("%d", x)` produces digits in reverse order during computation.** Describe two different strategies for outputting them in the correct order. What are the trade-offs between using a buffer and computing the digits twice?

5. **You forget to send `\r` before `\n`.** On QEMU, the output might look fine. On a real serial terminal, you get staircase output. Why does QEMU mask the problem? (Think about what your terminal emulator does when it receives a bare `\n`.)

6. **You have two functions: `uart_putc()` that busy-waits on LSR, and `uart_putc_unsafe()` that writes directly to THR without checking.** When would `uart_putc_unsafe` be safe to use? When would it lose characters? (Think about how fast the UART transmits vs. how fast your CPU runs.)

7. **Your `kprintf` uses `va_arg(ap, int)` for `%x`, but you pass a `uint64_t` at the call site.** On RV64, what specifically goes wrong? Does the upper 32 bits of the value get lost, or does something worse happen?

---

## Build This

### Implement the UART Driver

Create `kernel/uart.c` and `kernel/include/uart.h`:

1. **Define the UART base address** (`0x10000000`) and register offsets.

2. **Implement `uart_init()`:**
   - Disable interrupts (IER = 0)
   - Set DLAB, write divisor, clear DLAB with desired line parameters
   - Enable FIFOs (FCR = 0x07 or similar)
   - Optionally set MCR for auto-flow control (not necessary on QEMU)

3. **Implement `uart_putc(char c)`:**
   - Busy-wait on LSR bit 5
   - Write character to THR
   - If `c == '\n'`, also send `\r` (or do this in `kprintf`)

4. **Implement `uart_getc()`:**
   - Check LSR bit 0
   - If set, read and return RBR
   - If not, return -1

5. **Implement `uart_puts(const char *s)`:**
   - Loop through the string, calling `uart_putc` for each character

### Implement kprintf

Create `kernel/kprintf.c` and declare `kprintf` in an appropriate header:

1. **Support at minimum:** `%s`, `%d`, `%u`, `%x`, `%p`, `%c`, `%%`
2. **Use `<stdarg.h>`** for variadic arguments
3. **Implement integer-to-string conversion** for decimal and hexadecimal
4. **Handle the sign** for `%d` (including the `INT64_MIN` edge case, if you treat arguments as 64-bit)

### Update kernel_main

```c
void kernel_main(void) {
    uart_init();
    kprintf("Hello from the kernel!\n");
    kprintf("We are running on hart %d\n", 0);
    kprintf("Kernel loaded at %p\n", (void *)0x80000000);

    while (1) {}
}
```

### Checkpoint

Run `make run`. You should see:

```
Hello from the kernel!
We are running on hart 0
Kernel loaded at 0x80000000
```

Then QEMU hangs (infinite loop). Exit with Ctrl-A, X.

If you see this output, congratulations — your kernel has a voice. You can now print debug messages, register values, memory addresses, and error information. From this point forward, `kprintf` is your primary debugging tool alongside GDB.

**Bonus checkpoint:** Add the echo loop described earlier. Run the kernel, type characters, and verify they echo back. Then remove the echo loop (or put it behind a flag) so the kernel doesn't block on input.

---

## When Things Go Wrong

**No output at all — blank terminal**
- Is `uart_init()` being called? Put a GDB breakpoint on it.
- Is `uart_putc()` being called? Breakpoint it and check the character value.
- Are you writing to the correct address? `0x10000000` for the QEMU `virt` machine. Check with `info mtree` in the QEMU monitor (Ctrl-A, C to open, type `info mtree`, Ctrl-A, C to close).
- Is QEMU using `-nographic`? Without it, serial output goes to a separate window, not your terminal.

**Garbled output (random characters, wrong characters)**
- Check your register offsets. Off by one byte and you're reading/writing the wrong register.
- Check that you're writing bytes (8-bit), not words (32-bit). `sd` (store doubleword) to the UART would write 8 bytes starting at the UART base, clobbering multiple registers. Use `sb` (store byte) or ensure your C code uses `uint8_t *` pointers.
- Verify the baud rate divisor. On QEMU it shouldn't matter, but a wrong LCR value (wrong word length) could cause framing errors.

**Output stops mid-string**
- Your busy-wait on LSR might be wrong. If you're checking the wrong bit, you might write one character successfully and then get stuck waiting.
- Check that your string is null-terminated. `uart_puts` loops until it hits `\0`. If the string isn't terminated, it'll keep printing garbage until it either finds a `\0` by luck or walks off the end of mapped memory and faults.

**kprintf prints wrong numbers**
- Integer conversion bugs are common. Test with known values: `kprintf("%d", 0)` should print "0", `kprintf("%d", -1)` should print "-1", `kprintf("%x", 255)` should print "ff".
- Check your digit-to-character conversion: `'0' + digit` for 0-9, `'a' + (digit - 10)` for 10-15 (hex).
- Check your division/modulo logic: `n % 10` gives the least significant digit, `n / 10` removes it. Off-by-one in the loop termination condition causes either missing leading digits or an extra zero.

**Staircase output (each line is indented further than the last)**
- You're sending `\n` without `\r`. Add carriage return handling.

**QEMU monitor shows UART at a different address than expected**
- Run QEMU with `-d guest_errors` or check `info mtree` in the QEMU monitor. If the UART is at a different address (unlikely for the `virt` machine, but possible on different machine types), update your constant.

---

## Further Reading

- **NS16550A Datasheet** — Search for "16550 UART datasheet" or "PC16550D datasheet" (the TI-branded version). The register descriptions are the authoritative source for bit layouts and initialization sequences.
- **OSDev Wiki: Serial Ports** — The OSDev wiki has a comprehensive article on programming the 16550 UART, focused on x86 (port I/O) but the register interface is identical.
- **xv6 source: `uart.c`** — The xv6 UART driver is about 100 lines and handles both polling and interrupt-driven I/O. Read it after writing yours. Pay attention to how they handle the console input buffer.
- **xv6 source: `printf.c`** — xv6's printf implementation. Clean and compact. Compare with yours.
- **CS:APP Chapter 10: System-Level I/O** — Covers the Unix I/O model at a higher level. The concepts of buffering, file descriptors, and the relationship between device drivers and the I/O subsystem will become relevant in Chapters 20-22.
- **RISC-V Privileged Specification, Section 3.1.9** — Machine Interrupt Registers (`mie`, `mip`). You'll reference this when enabling UART interrupts in the next chapter.

---

*Next: [Chapter 7 — Interrupts and Trap Handling](ch07-interrupts-and-traps.md), where we teach the CPU what to do when something unexpected happens. This is the chapter that separates toy kernels from real ones.*
