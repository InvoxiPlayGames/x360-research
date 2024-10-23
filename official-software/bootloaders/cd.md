**Emma's Xbox 360 Research Notes - Bootloaders**

Updated 23rd October 2024.

Stub page, for the most part. Needs some work.

# CD / 4BL

The "CD" bootloader stage is the 4th and is responsible for decompressing "CE"
(the compressed hypervisor and kernel) and searching for + loading "CF" patch
slots from the NAND.

## TODO

Everything before launch needs to be documented

## Launching the Hypervisor

Since CD bootloader runs in a 32-bit translated address space, it can't just
jump to the hypervisor's entrypoint/reset vector. When loading into the
hypervisor, it does the following:

* Flushes any bootloader stages from cache and into RAM(?)
* Clears some special purpose registers
* Invalidates the translation lookaside buffer
* Disables instruction and data address translation in the MSR
* Sets the HRMOR to `0x00000100_00000000`
    * This sets all memory to be read encrypted and hashed with key 0x00
* Jumps to the hypervisor's reset vector (at offset `0x100`)

### Disassembly

```
launch_hypervisor:

; store CE/CF/CG into data cache and invalidate instruction cache
; (i think? check)
cache_flush:
    lis r3, 0x28        ; r3 = 0x280000
    li r4, 0x2a00
    mtspr CTR, r4
cache_flush_loop:
    dcbst 0, r3
    icbi 0, r3
    addi r3, r3, 0x80
    bdnz cache_flush_loop
    sync
    isync

; set HSPRG registers to 0, likely so the hypervisor itself
; can use them. the hv checks these during startup
; (HSPRG = "Hypervisor Software Use Special Purpose Register")
set_hsprg:
    li r3, 0
    mtspr HSPRG0, r3
    mtspr HSPRG1, r3

; invalidate the TLB entries
; todo: what does this do exactly?
invalidate_tlb:
    li r3, 0x3FF
    rldicr r3, r3, 0x20, 0x1f     ; r3 = 0x3FF_00000000
    tlbiel r3, r1, 0, 0, 0
    sync

; sets up a few special purpose registers (SPR) in the processor
setup_registers:
    ; prepare the HRMOR in r4, gets set before we jump
    li r4, 0x100
    rldicr r4, r4, 0x20, 0x1f     ; r4 = 0x100_00000000
                                  ; (flag bits = encrypted+hashed memory)
    ; disables instruction and data relocation in the machine state register
    mfmsr r5
    li r6, 0x30
    andc r5, r5, r6               ; r5 = MSR & ~(0x30)
    mtspr SRR1, r5                ; SRR1 gets restored into MSR when we return
    ; set interrupt return address to the reset vector in RAM
    li r6, 0x100
    mtspr SRR0, r6                ; SRR0 gets restored into IAR when we return

; sets the HRMOR, calls return from interrupt to jump to HV
; this code is always padded out to the nearest instruction cache line (aligned
; to 0x10 byte boundary) - since setting the HRMOR will cause all further reads
; to be encrypted
jump_to_hv:
    mtspr HRMOR, r4
    rfid
```
