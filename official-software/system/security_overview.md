**Emma's Xbox 360 Research Notes - System Software**

Updated 17th October 2026.

"Stub" page, not in-depth, just trying to put some notes and very opinionated
thoughts here:

# Software Security Overview

The Xbox 360 is a secure system. It is mostly designed to block out homebrew,
with the attempted prevention of piracy and online cheating being a side-effect.

As of 2024, every console manufactured before 2011 is subject to trivial
piracy, and cheating online in many games is possible with savegame exploits,
patched game files on burned DVDs, or network exploits.

As of 2025, there have been two demonstrated vulnerabilities allowing for
software-only hypervisor mode code execution:
- "King Kong" syscall handler exploit in 2007 (4532/4548), patched in 2007
  - https://free60.org/Hacks/King_Kong_Hack/
  - Patched in kernel version 4552.
  - Hypervisor versions 4532/4548 blacklisted in the bootloader with 8498 (2009)
- "Xbox360BadUpdate" exploit chain in 2025 (??-17559), **unpatched**
  - https://github.com/Grimdoomer/Xbox360BadUpdate
  - https://icode4.coffee/?p=1047 - "System Overview"
  - https://icode4.coffee/?p=1081 - "The Bad Update Exploit"

As of 2026, it is now possible to gain code execution at system bootup on all
motherboard revisions produced between 2005 and ~2011 without hardware
modification:
- "Xbox360PeerPreessuree" exploit chain (??-17559), **unpatched**
  - https://github.com/grimdoomer/Xbox360PeerPressure (closed source. :/)
  - Writeup pending

## Security Features

* Very small and simple boot chain
    * Several stages, each one small and very easy to analyse.
    * All are RSA signature checked by secure code burned into the CPU (1BL).
    * Execution is done from within either SRAM or encrypted and hashed main
      memory, preventing any outside attacks.
    * Vulnerable hypervisor versions are blacklisted, even if an exploit is used
      to attempt a downgrade.
    * E-fuses prevent any and all attempts at downgrading the bootloader,
      permanently.
* Small hypervisor, small attack surface
    * Hypervisor exposes only 120 syscalls to the kernel (as of 17559), each of
      them serve a specific purpose and are easy to audit and analyse. None are
      particularly complex.
    * Security features are in heavy use in the hypervisor, e.g. stack canaries,
      and boundary checks. Nothing passed from userland can ever try to reference
      the hypervisor address space.
    * In the past 21 years, only 2 flaws have been published in the retail
      hypervisor.
    * E-fuses prevent downgrading the hypervisor using previous NAND backups, and
      prevent all bootloader downgrades. The 2 vulnerable hypervisor versions,
      4532 and 4548, are blacklisted in newer bootloaders.
* Hypervisor/kernel boundary
    * Kernel has virtual memory mappings, with enforced W^X. That is to say,
      the hypervisor never allocates executable virtual memory to the kernel
      that has the execute permission *and* write permission.
    * New memory when loading new executables can't be marked as executable
      unless all signature checks have been passed by both the kernel and
      hypervisor.
    * Games for the most part run in the same privilege ring as kernel, for
      performance. Certain software uses a "user-mode" but that doesn't act
      as much of a security barrier, rather a memory management mode.
    * Hypervisor memory is hidden from the kernel (and thus from games), so
      keys and other secrets can't be extracted even with a software exploit.
* Memory encryption and hashing.
    * All executable memory is encrypted, and all hypervisor memory is hashed,
      preventing hardware DMA attacks or kernel-land software exploits from
      tampering with it or its state, even in uncontrollable ways.
    * The encryption is AES-128 with a slightly customised algorithm. Encryption
      is done per every 0x10 bytes (AES block size) with a random key.
    * The hashing is a custom variation of CRC16 by IBM. Hashing is done per
      every 0x80 bytes (cache line size) and is done using a random key generated
      at startup.
    * The encryption and hashing is done transparently to the kernel by the MMU
      and L2 cache.
    * Random AES keys are chosen at startup by 2BL, with help from hardware RNG.
      The 2BL also checks to make sure there's sufficient randomness so the RNG
      can't be rigged or disabled in hardware.
* (2011+) More hardware protection against DMA
    * "KSB" southbridge revision introduced with "Corona" motherboards hardens
      against access to DMA registers, preventing the SMC Hack (2009) and
      Peer Pressure (2026). *(TODO: How?)*
* (2014+) Hardware protection against glitching
    * "Winchester" motherboards have POST output disabled when security bits are
      blown in e-fuses, and the reset line is latched to prevent RGH.

## Security Pitfalls

* The DVD drive firmware, until a hardware revision in 2011-2012, was able to be
  modified and flashed onto the drives, allowing for spoofing of the security
  checks when reading Xbox 360 game discs and as such allowing for piracy.
    * This was attempted to be fixed with extra security checks, but the whole
      system is based around trusting the DVD drive to be telling the truth.
* Xbox 360 game discs (XGD2) have no integrity validation on their contents from
  modification. This role was entirely given to game developers, where most devs
  did not. Aforementioned DVD drive mods could be used to load modified versions
  of games to give unfair competitive advantage in online play.
    * This didn't require re-flashing of drive firmware! You can remove the top
      casing of the DVD drive and hot swap in a burned disc.
    * XGD3 seemed to have partially fixed this problem.
* While various system applications are compiled with stack canaries enabled,
  plenty of games are not, as was standard for the time for performance reasons.
    * Games can be patched when the system is connected to Live, but being
      offline and clearing cache makes the system vulnerable again.
    * The kernel and XAM, both critical system components, are not compiled with
      stack canaries, allowing vulnerabilities in those to be used to exploit
      the system.
        * [ABadAvatar](https://github.com/shutterbug2000/ABadAvatar) (shutterbug2000, 2025)
* While the CPU's JTAG bus is disabled during Power-On-Reset (POR) if the CPU
  is fused as security-enabled, the GPU's JTAG bus is not disabled until the
  CB / 2BL completes "HWINIT", allowing an attack to be launched via the GPU's
  JTAG lines before it gets disabled.
    * [SMC Hack / "JTAG"](https://github.com/Free60Project/tools/blob/master/imgbuild/hack.txt) (tmbinc, Tiros, et al., 2009)
* System management controller / SMC / Southbridge firmware is not signed,
  meaning it can be replaced to modify the behaviour or be used to attack the
  CPU. This has been one of the core enablers of every known user-facing
  hardware-based attack on the Xbox 360:
    * [SMC Hack / "JTAG"](https://github.com/Free60Project/tools/blob/master/imgbuild/hack.txt) (tmbinc, Tiros, et al., 2009)
    * [Reset Glitch Hack](https://github.com/Free60Project/tools/blob/master/reset_glitch_hack/reset_glitch_hack.txt) (GliGli, Tiros, et al., 2011)
    * [RGH3](https://ptswarm.com/blog/xbox-360-security-in-details-the-long-way-to-rgh3/) (15432, 2021)
    * [PeerPressure](https://github.com/grimdoomer/Xbox360PeerPressure) (Grimdoomer, 2026)
* The hypervisor does not check userland page permissions before writing data
  to a user-controlled pointer, allowing for userland exploits to overwrite the
  (unencrypted view of) memory in any write-protected page. Related ...
* ... the memory management unit only hashes the hypervisor's memory space,
  allowing for userland exploits to corrupt kernel, XAM and game ciphertext in
  ways that may result in a favourable plaintext when decrypted.
  * [BrutalBranch](https://github.com/ihatecompvir/BrutalBranch) (ihatecompvir, Emma (IPG), January 2024)
  * [Xbox360BadUpdate](https://icode4.coffee/?p=1081#:~:text=Attacking%20Encrypted%20Memory) (Grimdoomer, September 2024)

## References

* Free60 wiki:
  https://free60.org/
* Memory encryption/hashing information:
  https://github.com/GoobyCorp/Xbox-360-Crypto/blob/master/MemCrypto.py
* Ryan Miceli's "Hacking the Xbox 360 Hypervisor Part 1: System Overview":
  https://icode4.coffee/?p=1047

I must've got some more of this info from other places, but I can't remember.
