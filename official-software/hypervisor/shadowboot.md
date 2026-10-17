**Emma's Xbox 360 Research Notes - Hypervisor**

Updated 17th October 2026.

Not a stub page, but needs a lot of work.

# Shadowbooting

On devkit and testkit development hypervisors, the `HvxShadowBoot` syscall can
be used to reboot from a currently running hypervisor into another one, going
through the complete boot process except "HWINIT" (performed by)

## Kernel-mode Operation

During bootup, the kernel will check KD (kernel debugger), the HDD, or optical
media for a file named `xboxromw2d.bin` (`xboxromtw2d.bin` for testkit-fused
systems). If this exists, it will load the file into the higher areas of RAM
(at least `0x01000000` physically), put all CPU threads except thread 0 to
sleep, then provide the ROM image to the `HvxShadowBoot` syscall.

## Hypervisor-mode Operation

### HvxQuiesceProcessors

This is the syscall that sleeps all processors. When provided with reason "2"
(shadowbooting), it behaves slightly differently. This syscall is *not* run on
thread 0.

*(TODO: document the power steps taken by HvpSaveThread.)*

For all threads, HSPRG0 and HSPRG1 (special-purpose registers used to store
hypervisor state) are set to 0.

For logical threads (thread ID 1, 3, and 5), TE{0,1} are set to 0 in the CTRL
register for the thread, which suspends the thread indefinitely.

For physical threads (thread ID 2 and 4), the HRMOR register is updated to be
the CPU's SOC (`0x00000200_00000000`), the SLB is cleared, and a function is
copied to an area in SRAM (address `0x09E0` offset from the start of SRAM)
then jumped to.

The function in question sets the HID1 register to 0, sets WEXT to 1 in the
TSCR register to enable external interrupts (to allow the core to be awoken)
later, then similarly sets TE{0,1} to 0 to suspend the thread indefinitely.

### HvxShadowBoot

This syscall decrypts 2BL from the provided ROM image in the same way 1BL would,
however it diverges in the target address and in setting flags in the 2BL's
header.

* 1BL->2BL target address: `0x80000200_00010000` (start of CPU SRAM)
* Shadowboot->2BL target address: `0x80000200_00011000` (offset `0x1000` in SRAM)

This is because the start of SRAM is used to store the CRC16 hashes for the
hypervisor and other "protected" pages of memory, and overwriting them would
cause the CPU to checkstop when reading the corresponding cachline(s) while
executing the syscall. This is fine, though, as the first thing 2BL will do is
relocate itself to the correct location.

The 2BL's header is adjusted to include the media flags of the shadowboot in
the "flags" area.
Possible flags:

* `0x200` = shadowbooting (always set)
* `0x400` = shadowbooting from optical media

When launching the 2BL, it sets up a TLB entry to set up `0x02000000` to map to
the CPU's SRAM at `0x80000200_00010000`. It then launches with the following
registers:

* r26 = 2BL load address (`0x80000200_00011000`)
* r27 - r30 = 2BL arguments (2bl_hdr->args[0] - 2bl_hdr->args[3])
* r31 = Next bootloader real address (`0x80000000_00000000` + physical address of shadowboot + start offset of 3BL)

and the entrypoint from the 2BL header + 0x1000 (to account for the SRAM offset)

## Post-Reboot Operation

The boot stages as well as the hypervisor and kernel do deviate slightly from
normal operation if booted with the shadowboot media flag set in the 2BL header
(flags persist through each bootloader stage, with the exception of CB_A).

*(TODO: Clean this up and make this clearer, and actually find all the places
the media flag is checked for.)*

### 2BL / SB

If the 2BL sees the shadowboot flag set, it will skip executing the HWINIT VM
bytecode from 3BL. The decryption will still go through and is required for 4BL
key derivation.

### 3BL / SC

Not executed by 2BL if the shadowboot flag is set.

### 4BL / SD

If the shadowboot flag is set, the 4BL will skip over checking for a 6BL/7BL
patch set and will boot directly into whatever system is provided in 5BL.

### Hypervisor

The hypervisor will skip over setting up some parts of the SOC/CPU if the
shadowboot media flag is set.

### Kernel

The kernel will not attempt to shadowboot during initialisation, nor play the
boot animation, if the shadowboot media flag is set.

## References

* [ExShadowBoot](https://codeberg.org/hax360/ExShadowBoot) (mkukri, 2026)
