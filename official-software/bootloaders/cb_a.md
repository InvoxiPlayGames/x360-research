**Emma's Xbox 360 Research Notes - Bootloaders**

Updated 8th April 2024.

Stub page.

# CB_A / 2BL

The "CB_A" (the split variant) is the first stage of the boot process that is
loaded from NAND by the 1BL. It was introduced with the Xbox 360 Slim, and added
back to the Xbox 360 phat models in update 2.0.14717.0 with a bootloader update.

It is the most simple stage of the boot process - its job is solely to decrypt
the next stage, CB_B, using the console's unique CPU key, and verify its
integrity with a hardcoded RotSumSha hash (at offset 0x39C).

A partial decompilation of a version of CB_A is available at
[TEIR1plus2's Xbox-Reversing repo](https://github.com/TEIR1plus2/Xbox-Reversing/blob/master/cba_9188.c).
(Thanks, Teir!)
