**Emma's Xbox 360 Research Notes - Homebrew - xeBuild**

Updated 11th March 2025.

Incomplete stub page.

# Freeboot-like Patches

xeBuild comes with a set of patches for the hypervisor and kernel, based on
ikari's original Freeboot and updated by the community.

xeBuild NFO includes more credits and information:
https://www.xbins.org/nfo.php?file=xboxnfo2430.nfo

This is a reverse engineering of these patches, to understand more of how they
work and to allow for expanding upon them, or porting to other HV/kernel
versions, et cetera.

All offsets for this are for the latest released hypervisor/kernel, 17559.

## Hypervisor

### Initialisation Patch

`0x1880` = `4800B513`

Replaces a call to one of the startup functions with a branch to some shellcode
at `0xB510`. (See below for more)

### 0xF0 data clear

`0xF0` = `00000000 00000000 00000000 00000000`

No idea.

### Memory Protection Patch

`0x11BC` = `4800154E`

`0x154C` = `38800007 7C212078 7C35EBA6 480011C2`

This patch effectively disables memory protections when writing to instruction
pages. It injects shellcode at an empty space in memory:

```
memprot_patch:
    # on entry, r1 = value to be written to SPR
    li r4, 7
    andc r1, r1, r4      
    # r1 = r1 & ~7 - masks off the lower 3 bits of the value to be set
    mtspr PPE_TLB_RPN, r1
    ba memprot_continue  # address of the next instruction after 0x11BC
```

then modifies an existing `mtspr PPE_TLB_RPN, r1` instruction within the
instruction storage exception handler at `0x400` to branch to here.

This patch unsets the lower 3 bits of the value to be written to the "PPE
Translation Lookaside Buffer Real-Page Number Register" which removes the flags
indicating the memory should be guarded, no-execute, or protected.

This patch gets modified by shellcode later, to replace the "`li r4, 7`" with
"`li r4, 0`", effectively disabling this patch, when a certain syscall is
issued.

### Remove fuse checks

(TODO: is that what this does?)

`0x3120` = `60000000`

Replaces a call to a function that checks the current fuse values with a nop.

### HvxLoadImageData hash check patch

`0x2A30C` = `60000000 60000000`

Removes a check in HvxLoadImageData after a call to XeCryptMemDiff on a SHA-1
hash of an XEX's memory page(?).

### Unknown HvxResolveImports patches

`0x2AA80` = `60000000`

`0x2AA8C` = `60000000`

Patches two checks in HvxResolveImports. No idea what they do yet.

### Initialisation and syscall 0 shellcode.

TODO, this is pretty big.

This gets pointed to by syscall 0 as well as the initialisation patch mentioned
earlier.

### Syscall 0 Replacement

`0x15FD0` = `0000B564`

Replaces the address of syscall 0 with a pointer to the shellcode mentioned
above.

### HvxSecurity patches

`0x6BB0` = `38600000 4E800020` (HvxSecuritySetDetected)

`0x6C48` = `38600000 4E800020` (HvxSecurityGetDetected)

`0x6C98` = `38600000 4E800020` (HvxSecuritySetActivated)

`0x6D08` = `38600000 4E800020` (HvxSecurityGetActivated)

`0x6D58` = `38600000 4E800020` (HvxSecuritySetStat)

Patches a ton of HvxSecurity functions to return 0 and do nothing else.

### HvxKeysGetKey patch

`0x813C` = `48000030`

Skips a check on both the key's flags and current XeKeys flags to allow
HvxKeysGetKey to return any key in the keyvault.

### Keyvault initialisation patches

`0x70BC` = `38600001`

Replaces a call to XeCryptBnQwBeSigVerify with an immediate "return 1" in one of
the keyvault initialisation functions, to allow an unsigned keyvault to be used.

`0x7268` = `38600000`

Replaces a call to XeCryptMemDiff to always return 0 after checking the
RotSumSha of the keyvault, allowing a keyvault with an invalid hash(?) to be
used.

`0x72B4` = `60000000`

`0x72C4` = `60000000`

`0x72EC` = `60000000 39600001`

Nops out several branches to the machine check vector after various checks on
the keyvault. The latter of these patches forces a value at 0x74 to always be 1.
(TODO: Look into what this actually is doing.)

### Patch Media ID check?

`0x24D58` = `38600001 4E800020`

Replaces a function that is called by HvxImageTransformImageKey and
HvxCreateImageMapping to always return 1. Seems to be related to the DVD auth
media ID.

### Patch FCRT hash check

`0x264F0` = `38600001`

Replaces a branch to a hash checking function (?) within a HvxDvdAuthFcrt
subroutine to always return true.

### XEX key derivation patch shellcode

`0x29B08` = shellcode

TODO. Looks to be to allow devkit XEXs to decrypt.

### HvxImageTransformImageKey protected flag check patch

`0x2B778` = `60000000`

Removes a check on whether the protected flags have bit 2 set before deriving
the image decryption key. No idea

### HvxCreateImageMapping hash check patch

`0x2CAE8` = `38600000`

Removes a hash check during HvxCreateImageMapping.

### HvxCreateImageMapping keys flag check patch

`0x2CDD8` = `60000000`

Removes a keys flags check during HvxCreateImageMapping.

### HvxExpansionInstall signature/encryption patches

`0x3089C` = `409A0008 3BA00000 60000000 60000000`

Inserted after a `cmpwi` instruction after the call to XeCryptBnQwBeSigVerify:

```
hvx_expansion_shellcode:
    bne nopnop
    li r29, 0
nopnop:
    nop
    nop
```

r29 is previously set as a pointer to the AES encryption key to use, and a check
is right after this that checks if that is set to NULL and skips decryption. In
effect, this means that an unsigned HvxExpansion payload will not go through the
AES decryption, whereas a signed one will. Very clever!

### HvxExpansionInstall validation patches

`0x304E8` = `60000000`

`0x304FC` = `60000000`

(TODO: Figure out what these do.)

## Kernel

(TODO: Cleanup.)

```
; XexpConvertError
0007B920
00000002
386000004E800020

; XexpLoadXexHeaders: call to XexpVerifyMediaType
0007C4B8
00000001
38600001

; XexpLoadFile: call to RtlImageXexHeaderString
0007C5E8
00000001
38600000

; XexpLoadFile: flag check in XEX headers after RtlImageXexHeaderString
0007C634
00000001
39600000

; XexpLoadFile: flag check in XEX headers after RtlImageXexHeaderField
0007C684
00000001
39600000

; XexpVerifyMinimumVersion
0007AF08
00000002
386000004E800020

; SfcxInspectLargeDataBlock
00094F78
00000001
3AE00010

; SataCdRomAuthenticationExInitialize: cmplwi rA, 0x1
000998D0
00000001
2B0B00FF

; SataCdRomActivateHCDFRuntimePatch: call to VdDisplayFatalError
000992B4
00000005
3860000060000000600000006000000060000000

; XeKeysVerifyRSASignature
00109C90
00000002
386000014E800020

; XeKeysSecurityConvertError
0010A788
00000002
386000004E800020

; XeKeysDvdAuthExConvertError
0010AA68
00000002
386000004E800020

; XeKeysRevokeIsValid
0010AF30
00000002
386000014E800020

; XeKeysRevokeIsRevoked
0010B138
00000002
386000004E800020

; _XeKeysRevokeIsRevoked
0010B0E8
00000002
386000004E800020

; XeKeysRevokeIsDeviceRevoked
0010B278
00000002
386000004E800020

; XeKeysRevokeConvertError
0010B3F8
00000002
386000004E800020

; XeKeysConsoleSignatureVerification
; shellcode checks if r5 (is current console) is 0, if it isn't, writes 1 to it
0010BF20
00000005
2B05000038600001419A0008906500004E800020

; XeCryptBnQwBeSigVerify
00111990
00000002
386000014E800020

; MassConfigureTransferCable: todo: check
000E1754
00000001
48000054

; MassConfigureTransferCable: todo: check
000E17CC
00000003
835C00986000000060000000

; SataDiskAuthenticateDevice
0015D9D8
00000002
386000014E800020

; shellcode inserted into the "dead" body of XeKeysConsoleSignatureVerification.
; loads dashlaunch, etc. TODO: study
0010BF40
0000002C

dashlaunch_load_shellcode:
  ; only launch dashlaunch if xam succeeded, else exit Phase1Initialization
  bge cr6, load_dashlaunch
  blr

load_dashlaunch:
  ; XexLoadImage("\Device\Flash\launch.xex", 8, 0, 0);
  lis r3, 0x8010
  lis r5, 0
  li r4, 0
  ori r4, r4, 8
  ori r3, r3, 0xbfd0 ; 0x8010BFD0 - path to \Device\Flash\launch.xex
  li r6, 0
  bl XexLoadImage

  ; sets a flag so we know we at least attempted to load dashlaunch
  ; *(uint32_t *)0x8010BFEC = 0
  li r3, 0
  lis r4, 0x8010
  ori r4, r4, 0xbfec
  isync 
  stw r3, 0(r4)
  b after_dashlaunch_load_shellcode_hook ; continue Phase1Initialization

; TODO
  addi r5, r1, 0x54
  lis r7, -0x7ff0
  ori r7, r7, 0xbfec
  lwz r8, 0(r7)
  isync 
  cmplwi cr6, r8, 0
  beq cr6, 0x8010bfa0
  mr r31, r31
  b 0x8010bf88
  blr 
  cmplwi cr6, r3, 0x14
  bne cr6, 0x8010bfcc
  lis r7, -0x7ff0
  ori r7, r7, 0xbfec
  lwz r8, 0(r7)
  isync 
  cmplwi cr6, r8, 0
  beq cr6, 0x8010bfcc
  mr r31, r31
  b 0x8010bfb4
  b 0x80108410

; Phase1Initialisation: replace a check to see if Xam failed with a branch to
; the above shellcode
000613CC
00000001
480AAB75

; XexLoadExecutable: branch probably into the above shellcode
0007D7F8
00000001
4808E785

; Replaces some jump to a syscall with a branch to some "li r3, 0; (stack return)".
; No idea. TODO: What does this do?
00108E70
00000001
48003134
```
