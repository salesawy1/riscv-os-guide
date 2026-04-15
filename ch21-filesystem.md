# Chapter 21: A Simple Filesystem

**[Difficulty: ★★★★☆]**

---

## Why This Chapter Exists

You have a block device. You can read and write 512-byte sectors. But sectors are meaningless to a user — nobody wants to say "read bytes 256 through 384 of sector 47." Users want files and directories: "open /home/readme.txt and read its contents."

A **filesystem** is the data structure that maps the file/directory abstraction onto raw disk blocks. It answers: where is "readme.txt" stored on disk? How big is it? Which blocks hold its data? What other files are in the same directory? When was it last modified?

Designing a filesystem means designing an on-disk format — a layout of data structures on the raw blocks that organizes information into a hierarchy of files and directories. It also means implementing the code that reads and writes these structures.

This chapter designs a simple filesystem inspired by the Unix V6 filesystem and xv6's implementation. It's not ext4 or NTFS — those are enormously complex. Ours will handle the essentials: files, directories, reading, writing, creating, and deleting. It's enough to load user programs from disk and to implement basic file I/O system calls.

---

## On-Disk Layout

Our filesystem divides the disk into fixed regions:

```
Block 0     Block 1        Block 2..N     Block N+1..M    Block M+1..end
┌─────────┬──────────────┬──────────────┬───────────────┬──────────────┐
│  Boot   │  Superblock  │  Inode       │  Free Bitmap  │  Data        │
│  Block  │              │  Blocks      │               │  Blocks      │
└─────────┴──────────────┴──────────────┴───────────────┴──────────────┘
```

**Block 0 (Boot Block):** Reserved for a bootloader. We don't use it, but the convention reserves it.

**Block 1 (Superblock):** Contains metadata about the filesystem itself: total number of blocks, number of inode blocks, number of data blocks, the start of each region. The [superblock](https://wiki.osdev.org/Inode) is the filesystem's "table of contents."

```
struct superblock {
    uint32_t magic;         // Filesystem magic number (identifies this FS type)
    uint32_t total_blocks;  // Total blocks on disk
    uint32_t num_inodes;    // Total number of inodes
    uint32_t inode_start;   // First block of the inode region
    uint32_t bitmap_start;  // First block of the free bitmap
    uint32_t data_start;    // First block of the data region
};
```

**Inode Blocks:** Contain the inode table — an array of inodes, each describing one file or directory.

**Free Bitmap:** A bitmap tracking which data blocks are free (0) and which are allocated (1). Each bit represents one data block. For a 16 MiB disk with 512-byte blocks, that's 32,768 blocks, which requires 32,768 bits = 4,096 bytes = 8 blocks for the bitmap.

**Data Blocks:** The actual file content. Each data block is 512 bytes (or you could use 1 KiB or 4 KiB blocks for better performance — 512 matches the disk sector size and is simplest).

---

## Inodes

An [**inode**](https://en.wikipedia.org/wiki/Inode) (index node) describes a file or directory. It contains metadata and pointers to the data blocks that hold the file's content.

```
struct inode {
    uint16_t type;          // 0 = unused, 1 = file, 2 = directory
    uint16_t nlink;         // Number of directory entries pointing to this inode
    uint32_t size;          // File size in bytes
    uint32_t direct[12];    // Direct block pointers (12 blocks × 512 bytes = 6 KiB)
    uint32_t indirect;      // Indirect block pointer (points to a block of pointers)
};
```

### Direct and Indirect Blocks

The `direct` array holds block numbers for the first 12 blocks of the file. For files up to 6 KiB (12 × 512), all data is directly addressable from the inode.

For larger files, the `indirect` pointer points to a **[block full of block pointers](https://wiki.osdev.org/Inode#Indirect_Blocks)**. With 512-byte blocks and 4-byte block numbers, an indirect block holds 128 pointers, adding 128 × 512 = 64 KiB of addressable data. Total maximum file size: 12 × 512 + 128 × 512 = 70 KiB.

<details>
<summary>Why use 12 direct pointers and then an indirect block, rather than just one approach?</summary>
<div>

Small files are the common case. Storing 12 direct pointers in the inode means files up to 6 KiB (the vast majority of files in a real system) can be accessed immediately without an extra disk read to fetch an indirect block. Only for larger files do you incur that extra read. It's a performance tradeoff: optimize for the common case.

</div>
</details>

<details>
<summary>Why not use a double-indirect block to support arbitrarily large files?</summary>
<div>

Simplicity. A double-indirect adds another level of indirection and more code complexity. For a teaching OS handling files up to 70 KiB, it's overkill. Real filesystems do use double and triple indirection (or modern structures like extent trees) because they need to handle files that are gigabytes or larger.

</div>
</details>

### Inode Numbers

Each inode has an implicit number (its index in the inode table). The root directory is conventionally inode 1 (inode 0 is unused/reserved). When a directory entry refers to a file, it stores the file's inode number. When the kernel needs to access the file, it reads the inode from the inode table using the inode number.

<details>
<summary>Why store inode numbers in directory entries instead of full inode structures?</summary>
<div>

Inodes are larger than inode numbers (multiple cache lines vs. a few bytes). Storing only the inode number keeps directory entries small and fast to search. When you need the inode's metadata (size, timestamps, block pointers), you follow the number to the inode table. This separation of concerns — one table for names-to-numbers, another for numbers-to-metadata — makes the data structure simpler and more efficient.

</div>
</details>

---

## Directories

A directory is a file whose content is a list of **[directory entries](https://en.wikipedia.org/wiki/Directory_entry)**:

```
struct dirent {
    uint16_t inum;      // Inode number (0 = unused entry)
    char name[14];      // Filename (null-padded, 14 chars max)
};
```

Each entry maps a name to an inode number. The directory's inode has `type = 2` (directory), and its data blocks contain an array of `dirent` structs.

To look up "/home/readme.txt":
1. Start at the root directory (inode 1)
2. Read the root directory's data blocks
3. Search for an entry named "home" — it gives you inode number N
4. Read inode N's data blocks (it's a directory)
5. Search for an entry named "readme.txt" — it gives you inode number M
6. Read inode M — it's a file. Its `direct` and `indirect` pointers tell you where the data is.

<details>
<summary>Why does pathname resolution require multiple disk reads for a nested path?</summary>
<div>

Each path component requires reading a directory's data blocks to find the next component. For "/home/readme.txt", you read the root directory to find "home", then read "home"'s directory data to find "readme.txt". There's no way around it without caching — the filesystem is hierarchical, and you must traverse the hierarchy. This is why the buffer cache is critical: after the first read, subsequent accesses to the same directory blocks are instant.

</div>
</details>

This path traversal is called **[pathname resolution](https://pages.cs.wisc.edu/~remzi/OSTEP/file-intro.pdf)**, and it's one of the fundamental operations of any filesystem.

### Hard Links and nlink

Multiple directory entries can point to the same inode — these are **[hard links](https://en.wikipedia.org/wiki/Hard_link)**. The `nlink` field in the inode counts how many directory entries refer to it. When `nlink` drops to 0 (and no process has the file open), the inode and its blocks can be freed.

<details>
<summary>Why does `nlink` need to be greater than 0 before freeing an inode?</summary>
<div>

If there are still directory entries pointing to the inode, deleting its blocks would leave those entries dangling — they'd refer to a freed inode. Only when all directory entries (including those created by `ln`) are gone is it safe to reclaim the inode's blocks. And you still need to check that no process has the file open, because an open file descriptor also holds a reference.

</div>
</details>

The `.` entry in a directory points to the directory itself. The `..` entry points to the parent directory. These are hard links that exist in every directory.

---

## Block Allocation

When a file grows (data is written beyond its current size), the filesystem needs to allocate new data blocks. The **free bitmap** tracks which blocks are available:

- Bit N = 0: Block N is free
- Bit N = 1: Block N is allocated

To allocate a block:
1. Scan the bitmap for a 0 bit
2. Set it to 1
3. Return the block number

<details>
<summary>Why use a bitmap to track free blocks instead of a linked list of free blocks?</summary>
<div>

A bitmap is compact (1 bit per block) and fast to scan with bitwise operations. A linked list would require following pointers through the free blocks, much slower. The bitmap is also more cache-friendly — you can check hundreds of blocks with a few memory accesses. The downside is that the bitmap itself must fit in memory or be cached, but that's a small price.

</div>
</details>

To free a block:
1. Set its bitmap bit to 0

Similarly, to allocate an inode, scan the inode table for an entry with `type = 0`.

---

## The Buffer Cache

Reading from disk is slow (even on QEMU, it's relatively slow due to the virtio protocol overhead). The **[buffer cache](https://wiki.osdev.org/Buffer_Cache)** keeps recently read disk blocks in memory so that repeated accesses to the same block don't require disk I/O.

The buffer cache is a pool of in-memory buffers, each holding one disk block:

```
struct buf {
    int valid;          // Has this buffer been read from disk?
    int dirty;          // Has this buffer been modified (needs write-back)?
    uint32_t blockno;   // Which disk block this buffer holds
    uint8_t data[512];  // The block's data
    // ... LRU list pointers, reference count, lock
};
```

The interface:
- **`bread(blockno)`**: Read block from disk (or return cached copy)
- **`bwrite(buf)`**: Write a modified buffer back to disk
- **`brelse(buf)`**: Release a buffer (decrease reference count, make available for reuse)

<details>
<summary>Why does the buffer cache track a `valid` flag separately from the data?</summary>
<div>

A buffer structure is allocated before its data is read from disk. The `valid` flag indicates whether the data has actually been populated. Before reading, `valid=0`. After `bread` completes the I/O, `valid=1`. This lets you allocate and manage buffer structures independently of I/O completion.

</div>
</details>

<details>
<summary>Why does the buffer cache need a reference count and LRU eviction?</summary>
<div>

Many parts of the kernel may hold a pointer to a cached block simultaneously (the inode table read, a directory lookup, a file read). The reference count ensures you don't evict a buffer while someone is still using it. LRU eviction chooses the least recently used buffer because the working set of files typically accesses a locality set — blocks used recently tend to be used again soon.

</div>
</details>

When the cache is full and a new block is needed, evict the least recently used (LRU) buffer. If it's dirty, write it back first.

The buffer cache is a critical performance optimization and also provides a synchronization point: if two processes access the same block, they both get the same cached buffer, preventing inconsistencies.

> **Aside: Journaling and crash consistency**
>
> What happens if the system crashes (power failure) in the middle of a filesystem operation — say, halfway through creating a file (inode allocated, directory entry not written yet)? The on-disk state is inconsistent: an inode is allocated but not referenced by any directory.
>
> Real filesystems solve this with **[journaling](https://pages.cs.wisc.edu/~remzi/OSTEP/file-journaling.pdf)** (ext3, ext4, NTFS) or **copy-on-write** (ZFS, btrfs). A journal records intended operations before performing them. On crash recovery, the journal is replayed to bring the filesystem to a consistent state.
>
> xv6 implements a simple write-ahead log (a form of journaling). For your teaching OS, you can skip journaling — just be aware that an unclean shutdown may leave the filesystem in an inconsistent state. The `fsck` tool exists on real systems to detect and repair such inconsistencies.
>
> [OSTEP Chapter 42](https://pages.cs.wisc.edu/~remzi/OSTEP/file-journaling.pdf) (Crash Consistency: FSCK and Journaling) covers this topic thoroughly.

---

## Building the Filesystem Image

You need a tool that creates a filesystem image — a file containing the on-disk structures (superblock, inodes, bitmap, data blocks) with initial files pre-loaded. This tool runs on your host machine (not in the OS) and produces `disk.img`.

The tool (`mkfs`):
1. Calculate the layout (number of inode blocks, bitmap blocks, data blocks)
2. Write the superblock
3. Create the root directory (inode 1)
4. For each user program: allocate an inode, copy its data into data blocks, add a directory entry in the root directory
5. Write the bitmap

After `mkfs` runs, you have a `disk.img` that QEMU can use as a virtual disk, containing your filesystem with pre-loaded user programs.

---

## Conceptual Exercises

1. **With 12 direct blocks and 1 indirect block (128 pointers), what's the maximum file size?** Now calculate for a hypothetical design with 12 direct, 1 indirect, and 1 double-indirect block.

2. **A directory has 1000 files. How many disk blocks does the directory itself occupy?** (Each dirent is 16 bytes, and a block is 512 bytes.)

3. **You delete a file by removing its directory entry. Why isn't this sufficient to free the file's blocks?** What else must be checked before freeing? (Think about hard links and open file descriptors.)

4. **The free bitmap uses one bit per block. For a 1 GiB disk with 4 KiB blocks, how large is the bitmap?** How many blocks does it occupy?

5. **Why is the buffer cache important for directory lookups?** Consider reading a directory with 100 entries — how many disk reads without the cache vs. with the cache?

---

## Build This

### mkfs Tool

Create `tools/mkfs.c` (runs on host, not in kernel):
1. Takes user program binaries as arguments
2. Produces a `disk.img` with the filesystem layout
3. Creates root directory with entries for each program

### Filesystem Implementation

Create `kernel/fs.c` and `kernel/include/fs.h`:

1. **Define on-disk structures** (superblock, inode, dirent)
2. **Implement the buffer cache** (bread, bwrite, brelse)
3. **Implement `inode_read(inum)`** — read an inode from disk
4. **Implement `inode_read_data(inode, offset, buf, n)`** — read n bytes from a file
5. **Implement `dir_lookup(dir_inode, name)`** — find a name in a directory, return inode number
6. **Implement `namei(path)`** — resolve a pathname to an inode number (traverse directories)

### Checkpoint

```c
fs_init();
int inum = namei("/hello");
kprintf("hello is inode %d\n", inum);
struct inode ip;
inode_read(inum, &ip);
char buf[512];
inode_read_data(&ip, 0, buf, ip.size);
kprintf("First 16 bytes: ");
for (int i = 0; i < 16; i++) kprintf("%x ", buf[i]);
kprintf("\n");
```

This should find the "hello" program in the root directory and read its first bytes (which should be the ELF magic: `7f 45 4c 46`).

---

## When Things Go Wrong

**Superblock magic is wrong**
mkfs didn't write the image correctly, or the block driver is reading the wrong sector.

**namei can't find files**
The root directory entries aren't being read correctly. Check dirent size and alignment. Print all entries to debug.

**File data is garbage**
Inode block pointers are wrong. Verify by manually checking the mkfs output.

---

## Further Reading

- **[OSTEP Chapters 39-42](https://pages.cs.wisc.edu/~remzi/OSTEP/file-intro.pdf)** — Four chapters on filesystems: files and directories, filesystem implementation, locality and FFS, crash consistency.
- **[xv6 Book, Chapter 8: File System](https://pdos.csail.mit.edu/6.828/2024/xv6/book-riscv-rev4.pdf)** — The xv6 filesystem design. Very similar to what you're building.
- **[xv6 source](https://github.com/mit-pdos/xv6-riscv): `fs.c`, `bio.c`, `mkfs.c`** — The filesystem, buffer cache, and mkfs tool.
- **CS:APP Chapter 10: System-Level I/O** — File descriptors, the Unix I/O interface.

---

*Next: [Chapter 22 — File Descriptors and the VFS](ch22-vfs-file-descriptors.md), where we connect the filesystem to the system call interface through file descriptors and the "everything is a file" abstraction.*
