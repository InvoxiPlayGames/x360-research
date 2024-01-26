**Emma's Xbox 360 Research Notes - System Software**

Updated 26th January 2024.

# Software Updates

This page isn't yet going to have any information on how updates for NXE
content works, just the core update process from getting a box on an older
version to a newer one.

The system updates for the 360 OS are stored in STFS packages, signed with
Microsoft's PIRS private key. When a new USB or DVD is mounted, the 360's OS
checks for a file in the `$SystemUpdate` folder with a format of
`su[base kernel]_00000000`, where base kernel is in a hexadecimal representation
of the full number. E.g. `2.0.1888.0` -> `2_0_0760_00` - remove the underscores.
On retail consoles this filename is always `su20076000_00000000`.

The file `$install_extender.xex` does the main bulk of the heavy lifting for
software updates. It is where the bootloader updates are stored, as well as the
code for signing and flashing the new system update to the NAND.

## NAND Filesystem Contents

The update package contains many files beginning with `$flash_`. These are all
copied to the NAND flash's filesystem as-is, with `.xexp`/`.xttp` files being
appended with a number depending on which CF patch slot the update is being
installed to.

This was originally used as a fallback for if updates failed, however 4532 
was the first to totally replace an XEX file on NAND rather than using a patch.
What this resulted in was the system becoming useless as a fallback, now
solely acting as a way of reducing download size for the smaller files that
didn't get updated often.

The file `systemupdate2pre.xex` is copied to NAND during the update process,
launched by the resulting kernel, and subsequently deleted after it isn't
needed anymore. It is responsible for updating the DVD drive firmware.

The kernel, on initialisation, makes a XEXP loading rule for files loaded from
flash, which decides whether to use the `.xexp1` or `.xexp2` based on the
currently loaded CF patch slot.

## HV/Kernel Updates

The updated HV/kernel is stored in a pair of CF and CG bootloaders. CF is a
patching engine, signed with Microsoft's private key, and CG is an LZX-delta
compressed blob of the updated HV/kernel, applied on top of the base kernel
(always version 1888 on retails). This CF/CG pair is in `xboxupd.bin`.

`$install_extender.xex` reads the xboxupd file from the updater's STFS package,
along with some data from flash that gets merged in with the CF. It then passes
this to the `XeKeysSaveSystemUpdate` kernel function. The kernel is responsible
for detecting the currently running bootloader flags and then deciding which
patch slot to tell $install_extender to write the resulting CF to.

The `HvxKeysSaveSystemUpdate` hypervisor syscall takes the input CF, and
writes the required pairing data to an offset in the CF (taken from the current
CPU fuse count value, and adding 1 to it), before signing it with the console's
CPU key with a HMAC-SHA1 over the pairing data, and re-encrypting it with a new
randomly generated key.

An implementation of the CE->CG delta patching code is available in
[xenon-bltool](https://github.com/InvoxiPlayGames/xenon-bltool).

*(TODO: I could explain this better.)*

## Bootloader Updates

One big TODO. Just notes.

Handled by `$install_extender.xex`. There are 2 XEX resources, one containing
an XeKeysExecute payload for reading the bootloader archive and one containing
a big blob of bootloader data, encrypted with the 1BL key and containing a
lookup table of CPU PVR, Xenos ID and other hardware revisions to a CB and CD
version to install, alongside an LZX compressed blob containing all the CB and CD
stages that can be installed.

## DVD drive updates

One big TODO. Just notes.

Handled by `systemupdate2pre.xex`. Involves XeKeysExecute payloads and has
differing code depending on detected model identifier. `oddupdX.xex` files
look to contain no code and are just encrypted firmware blobs?

The XeKeysExecute payloads work with the DVD key in the key vault for some
purpose or another. They never leave hypervisor mode unencrypted.
