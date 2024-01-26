**Emma's Xbox 360 Research Notes - Kernel**

Updated 26th January 2024.

# Memory Map

The Xbox 360's official kernel uses virtual memory mappings, split up into
several sections based on both the hypervisor and kernel's agreement. The
system itself only has 512MB of memory, however the kernel makes use of the
full 32-bit (4GB) address space using these virtual memory mappings.

This page is solely regarding the userland's high-level view of this memory,
and doesn't go into the details of the MMU, TLB or PTE structure.
The tables below don't include the areas of memory allocated for executables,
and are valid as of the latest retail kernel (17559), taken from a modded
console. 

## Kernel-managed Virtual Memory (4KB) (0x00000000-0x3FFFFFFF)

Used when allocating 4KB pages with NtAllocateVirtualMemory? Needs more research.

## Kernel-managed Virtual Memory (64KB) (0x40000000-0x7FFFFFFF)

This address space is managed entirely by the kernel by a page table at
`0x01F70000` (physical). It is used when allocating 64KB paged memory with
NtAllocateVirtualMemory, and the 360 SDK's malloc/heap implementation allocates
memory from this space by default.

### Memory-Mapped I/O (0x7F000000)

During kernel initialisation, certain hardware peripherals are mapped to this
address space for MMIO access. This allows the kernel to access hardware such as
the SATA controller, SFC or MMC controllers (for internal flash), 

| Address      | Length   | Contents                            | Physical     |
| ------------ | -------- | ----------------------------------- | ------------ |
| `0x7F000000` | 0x800000 | Unknown, TODO                       | `0xC0000000` |
| `0x7FC80000` | 0x20000  | Xenos GPU                           | `0xEC800000` |
| `0x7FD00000` | 0x100000 | Unknown, TODO                       | `0xE1000000` |
| `0x7FEA0000` | 0x10000  | PCI devices                         | `0xEA000000` |
| `0x7FED0000` | 0x10000  | PCI configuration registers         | `0xD0010000` |

## Executable + Encrypted Areas (0x80000000-0x9FFFFFFF)

This address space is managed entirely by the hypervisor, rather than the
kernel, and is the only memory space the kernel mode can execute code from.
This is also the only memory space that the kernel can read encrypted and write
hashed memory from (with the exception of hypervisor memory).

Titles can allocate encrypted memory from this space with
NtAllocateEncryptedMemory.

### 0x80000000

Pages in this area are 64KB aligned.

| Address      | Length   | Contents                            | Physical     |
| ------------ | -------- | ----------------------------------- | ------------ |
| `0x80000000` | 0x40000  | Encrypted hypervisor contents       | `0x00000000` |
| `0x80040000` | Varies   | Xbox 360 kernel (xboxkrnl.exe)      | `0x00040000` |
| `0x8C000000` | 0x20000? | System (XAM) encrypted allocations  | `0x01E70000` |
| `0x8D000000` | TODO     | Title encrypted allocations         | TODO         |
| `0x8E000000` | 0x20000  | Certificate revocation list (CRL)   | `0x01EF0000` |
| `0x8E030000` | 0x10000  | Hypervisor data mirror (flags, etc) | `0x01F10000` |
| `0x8E050000` | 0x10000  | XEX2 header copies(?)               | `0x01F20000` |

XEX2 images loaded into this space should have a base address between
`0x80400000 - 0x8C000000`. (TODO: check hard limits)

### 0x90000000

Pages in this area are 4KB aligned.

| Address      | Length   | Contents                            | Physical     |
| ------------ | -------- | ----------------------------------- | ------------ |
| `0x90001000` | 0x1000   | Installed kernel Hvx/ExExpansions   | `0x01F31000` |
| `0x90002000` | 0x1000   | Unknown, Hvx/ExExpansion related    | `0x01F32000` |

XEX2 images loaded into this space should have a base address between
`0x90003000 - 0x9FFFFFFF`. (TODO: check hard limits)

## Physical Memory (0xA0000000-0xFFFFFFFF)

This area of memory contains 3 different mirrors of physical memory, each with
a different paging set-up. The kernel controls which pages of these mirrors are
enabled or not, with them defaulting to being disabled.

The length of all these should be considered `0x1FFFFFFF` (512MB), however they
are not always enabled except for the 64KB paged mirror.

(TODO: Does MmAllocatePhysicalMemory)

* `0xA0000000` - physical memory, 64KB paged.
    * The kernel enables 9 pages starting at `0xA1F70000` during initialisation,
      for managing the virtual 64KB mappings. (and more?)
* `0xC0000000` - physical memory, 16MB paged.
    * The kernel enables this all as read-only during initialisation.
* `0xE0000000` - physical memory, 4KB paged.
