# Chapter 20: Block Devices and virtio

**[Difficulty: ★★★★☆]**

---

## Why This Chapter Exists

Everything your kernel has done so far is ephemeral. Programs live in RAM. Data lives in RAM. When QEMU stops, everything is gone. An operating system needs **persistent storage** — a way to read and write data that survives reboots. This means a disk (or its virtual equivalent), and a disk means a **device driver**.

This chapter is your first real device driver — something significantly more complex than the UART. The UART is a byte-at-a-time device with a trivial register interface. A block device transfers data in chunks (blocks, typically 512 bytes), uses complex data structures for command queues, and requires asynchronous I/O (you submit a request, the device processes it, and an interrupt tells you it's done).

We'll use **[virtio-blk](https://docs.oasis-open.org/virtio/virtio/v1.2/virtio-v1.2.html)**, a virtual block device defined by the [virtio specification](https://docs.oasis-open.org/virtio/virtio/v1.2/virtio-v1.2.html). Virtio is a standard interface for [paravirtualized](https://en.wikipedia.org/wiki/Paravirtualization) devices, designed specifically for VMs and emulators. QEMU implements virtio devices, and they're the standard way to give a guest OS access to disks, networks, and other hardware in a virtualized environment.

---

## What Is virtio?

[Virtio](https://docs.oasis-open.org/virtio/virtio/v1.2/virtio-v1.2.html) (Virtual I/O) is a specification for [paravirtualized](https://en.wikipedia.org/wiki/Paravirtualization) devices. "Paravirtualized" means the device doesn't pretend to be real hardware — both the device and the driver know they're in a virtual environment and use an optimized communication protocol.

The key concept in virtio is the **[virtqueue](https://docs.oasis-open.org/virtio/virtio/v1.2/virtio-v1.2.html)**: a shared-memory ring buffer through which the driver and device exchange requests. The driver puts requests into the virtqueue, notifies the device, and the device processes them and signals completion via an interrupt.

### The Virtqueue

A [virtqueue](https://docs.oasis-open.org/virtio/virtio/v1.2/virtio-v1.2.html) consists of three areas in memory, allocated by the driver:

**1. Descriptor Table:** An array of descriptors. Each descriptor describes a buffer (address, length, flags). A single I/O operation may involve multiple buffers chained together as a [descriptor chain](https://docs.oasis-open.org/virtio/virtio/v1.2/virtio-v1.2.html) (e.g., a "read block" operation has a header buffer, a data buffer, and a status buffer).

```
struct virtq_desc {
    uint64_t addr;    // Physical address of the buffer
    uint32_t len;     // Length of the buffer
    uint16_t flags;   // NEXT (chained), WRITE (device writes to this buffer)
    uint16_t next;    // Index of the next descriptor in the chain
};
```

**2. Available Ring (Driver → Device):** The driver writes descriptor chain heads here to submit requests. The device reads from this ring.

```
struct virtq_avail {
    uint16_t flags;
    uint16_t idx;           // Next index the driver will write to
    uint16_t ring[QUEUE_SIZE];  // Descriptor indices
};
```

**3. Used Ring (Device → Driver):** The device writes completed descriptor chain heads here. The driver reads from this ring to collect results.

```
struct virtq_used {
    uint16_t flags;
    uint16_t idx;           // Next index the device will write to
    struct {
        uint32_t id;        // Descriptor index
        uint32_t len;       // Bytes written by device
    } ring[QUEUE_SIZE];
};
```

### The I/O Cycle

1. **Driver allocates descriptors** for the request: one for the command header, one for the data buffer, one for the status byte. Chain them using the `next` field.
2. **Driver writes** the head descriptor index to the available ring.
3. **Driver increments** `avail->idx`.
4. **Driver notifies** the device by writing to a notification register (MMIO).
5. **Device processes** the request (reads/writes the disk image).
6. **Device writes** the completed descriptor index to the used ring.
7. **Device sends an interrupt.**
8. **Driver reads** the used ring, retrieves the status, and knows the I/O is complete.

This asynchronous model is fundamental to all modern I/O: submit a request, do other work, handle the completion later. It's how [NVMe](https://en.wikipedia.org/wiki/NVMe) drives, [network cards](https://en.wikipedia.org/wiki/Network_interface_controller), and GPU command queues all work. Virtio is a simple, clean version of this pattern.

> **Aside: Why virtio instead of emulating a real disk controller?**
>
> QEMU can emulate real hardware (e.g., an [Intel AHCI SATA](https://en.wikipedia.org/wiki/Advanced_Host_Controller_Interface) controller or an [IDE disk](https://en.wikipedia.org/wiki/Parallel_ATA)). The problem is that real hardware has complex, historical interfaces that require large, complex drivers. An IDE driver alone can be hundreds of lines to handle all the quirks. AHCI is worse.
>
> Virtio is designed to be simple to drive while being efficient. The driver needs to understand the [virtqueue protocol](https://docs.oasis-open.org/virtio/virtio/v1.2/virtio-v1.2.html) and the device-specific header format, but there's no hardware-specific initialization sequence, no register-level timing constraints, and no legacy compatibility modes. It's the ideal device for a teaching OS.
>
> That said, understanding virtio is directly relevant to real-world systems: Linux uses virtio extensively in [KVM](https://en.wikipedia.org/wiki/Kernel-based_Virtual_Machine) virtual machines, cloud VMs (AWS, GCP, Azure all use virtio for block and network devices), and even some bare-metal configurations.

---

## MMIO Discovery

On the QEMU `virt` machine, virtio devices are exposed as [MMIO](https://en.wikipedia.org/wiki/Memory-mapped_I/O) regions starting at `0x10001000`. Each device occupies a 0x1000-byte region. The first virtio device is at `0x10001000`, the second at `0x10002000`, and so on.

To discover virtio devices, the driver reads the **magic number** and **device ID** from each potential [MMIO](https://en.wikipedia.org/wiki/Memory-mapped_I/O) region:

- Offset 0x000: Magic value (must be `0x74726976` — "virt" in little-endian)
- Offset 0x004: Version (we'll use version 2 — the modern virtio MMIO interface)
- Offset 0x008: Device ID (1 = network, 2 = block, etc.)
- Offset 0x00C: Vendor ID

If the magic value is correct and the device ID is 2 (block device), you've found a virtio-blk device.

### Device Initialization

The virtio MMIO initialization sequence:

1. **Reset the device** (write 0 to the Status register at offset 0x070)
2. **Set ACKNOWLEDGE** status bit (write 1)
3. **Set DRIVER** status bit (write 3)
4. **Read and negotiate features** (read from offset 0x010, write accepted features to offset 0x020)
5. **Set FEATURES_OK** status bit (write 11 — 0xB)
6. **Read status back** and verify FEATURES_OK is still set (device may reject)
7. **Configure the virtqueue:**
   - Write queue number (0) to Queue Select (offset 0x030)
   - Read Queue Size Max (offset 0x034) — maximum queue depth
   - Allocate descriptor table, available ring, used ring in physical memory
   - Write their physical addresses to the appropriate registers
8. **Set DRIVER_OK** status bit (write 15 — 0xF)

After step 8, the device is ready to accept I/O requests.

---

## Submitting a Block I/O Request

A virtio-blk request consists of three chained descriptors:

**Descriptor 0: Request Header**
```
struct virtio_blk_req_header {
    uint32_t type;    // 0 = read, 1 = write
    uint32_t reserved;
    uint64_t sector;  // Sector number (512 bytes per sector)
};
```

**Descriptor 1: Data Buffer**
The buffer to read into (for reads) or write from (for writes). Must be `sector_count * 512` bytes. Mark with VIRTQ_DESC_F_WRITE flag if the device should write to it (i.e., for read operations — the device writes data into your buffer via [DMA](https://en.wikipedia.org/wiki/Direct_memory_access)).

**Descriptor 2: Status Byte**
A single byte that the device writes with the completion status: 0 = OK, 1 = I/O error, 2 = unsupported. Mark with VIRTQ_DESC_F_WRITE.

Chain them: descriptor 0 → descriptor 1 → descriptor 2 (using the `next` field and VIRTQ_DESC_F_NEXT flag).

### The Complete Read Sequence

1. Set up the three descriptors in the descriptor table
2. Write the head descriptor index to `avail->ring[avail->idx % QUEUE_SIZE]`
3. Increment `avail->idx`
4. Write to the Queue Notify register (offset 0x050) to kick the device
5. Wait for the interrupt (or poll the used ring)
6. Read the status byte to verify success
7. The data buffer now contains the sector data

---

## Running QEMU with a Disk

To give QEMU a virtual disk:

```
qemu-system-riscv64 ... \
    -drive file=disk.img,if=none,format=raw,id=hd0 \
    -device virtio-blk-device,drive=hd0
```

Create a test disk image:
```
dd if=/dev/zero of=disk.img bs=1M count=16
```

This creates a 16 MiB raw disk image filled with zeros. Your driver can read and write sectors (512-byte blocks) to this image.

---

## Synchronous vs. Asynchronous I/O

For simplicity, your initial driver can be **synchronous**: submit a request, then busy-wait (or sleep) until the device completes it. This is simpler but blocks the calling process until I/O finishes.

An **asynchronous** driver submits the request, returns immediately, and handles completion in an interrupt handler. The calling process can be put to sleep and woken when the I/O completes. This is how production drivers work and is essential for good performance.

For your teaching OS, start synchronous (poll the used ring after each request). Once it works, convert to interrupt-driven.

---

## Conceptual Exercises

1. **Why does virtio use three separate areas (descriptors, available ring, used ring) instead of a single shared buffer?** What would go wrong if the driver and device both read and wrote to the same area?

2. **A read request has three descriptors. Why is the data buffer marked with VIRTQ_DESC_F_WRITE even though it's a "read"?** (Think about who writes and who reads.)

3. **The available ring index (`avail->idx`) is incremented after writing the descriptor index to the ring. Why is order important?** What could the device see if the increment happened first?

4. **Your block driver reads sector 0 successfully but sector 100 returns an error. What might be wrong?** (Think about the disk image size.)

5. **Why is the virtio MMIO initialization sequence so rigid (reset, acknowledge, driver, features, driver_ok)?** What would happen if you just set DRIVER_OK immediately?

---

## Build This

### Virtio Block Driver

Create `kernel/virtio_blk.c` and `kernel/include/virtio_blk.h`:

1. **Define virtio MMIO register offsets** and virtqueue structures.

2. **Implement `virtio_blk_init()`:**
   - Scan MMIO regions for a virtio-blk device
   - Follow the initialization sequence
   - Allocate and configure the virtqueue

3. **Implement `int virtio_blk_read(uint64_t sector, void *buf)`:**
   - Set up the three-descriptor chain
   - Submit to the available ring
   - Notify the device
   - Wait for completion (poll used ring or handle interrupt)
   - Return status

4. **Implement `int virtio_blk_write(uint64_t sector, void *buf)`:**
   - Same as read, but type=1 and the data buffer is not WRITE-flagged

### Checkpoint

Update your QEMU command to include a disk image. In `kernel_main`:

```c
virtio_blk_init();

char buf[512];
virtio_blk_read(0, buf);
kprintf("Sector 0, first 4 bytes: %x %x %x %x\n",
        buf[0], buf[1], buf[2], buf[3]);

memset(buf, 0xAB, 512);
virtio_blk_write(1, buf);
virtio_blk_read(1, buf);
kprintf("Sector 1 after write: %x %x %x %x\n",
        buf[0], buf[1], buf[2], buf[3]);
```

You should see zeros for sector 0 (empty disk), and `0xAB` for sector 1 after the write. This proves your block driver can read and write the virtual disk.

---

## When Things Go Wrong

**Device not found at expected MMIO address**
Check the magic value. Make sure your QEMU command includes `-device virtio-blk-device`.

**Initialization hangs at feature negotiation**
You may need to accept (or not accept) specific features. Start by accepting no optional features.

**Read returns garbage or all zeros**
Descriptor addresses must be physical addresses (the device doesn't use virtual memory). Make sure you're passing physical, not virtual, addresses for the descriptor table, available ring, used ring, and data buffers.

**Write seems to succeed but data doesn't persist**
Check `disk.img` on the host after quitting QEMU. If using `-drive` with `cache=writeback`, data may not be flushed. Try `cache=none` or `cache=writethrough`.

---

## Further Reading

- **[Virtio Specification (v1.2)](https://docs.oasis-open.org/virtio/virtio/v1.2/virtio-v1.2.html)** — The full virtio spec. Section 2 covers the general [virtqueue mechanism](https://docs.oasis-open.org/virtio/virtio/v1.2/virtio-v1.2.html), Section 5.2 covers virtio-blk.
- **[xv6 source: `virtio_disk.c`](https://github.com/mit-pdos/xv6-riscv)** — xv6's virtio-blk driver. About 200 lines, clean and readable.
- **[OSDev Wiki: Virtio](https://wiki.osdev.org/Virtio)** — Community documentation on virtio device programming.
- **[Wikipedia: Paravirtualization](https://en.wikipedia.org/wiki/Paravirtualization)** — Overview of paravirtualized devices.

---

*Next: [Chapter 21 — A Simple Filesystem](ch21-filesystem.md), where we design and implement a filesystem that turns raw disk sectors into files and directories.*
