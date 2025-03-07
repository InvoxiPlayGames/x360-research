**Emma's Xbox 360 Research Notes - Homebrew - xeBuild**

Updated 7th March 2025.

Stub page.

# Patch Format

| Offset | Type     | Description                         |
| ------ | -------- | ----------------------------------- |
| `0x0`  | uint32   | Address in boot section to patch    |
| `0x4`  | uint32   | Number of 4-byte words to patch     |
| `0x8`  | uint32[] | Array of words to insert at address |

Patch sets are delimited by `0xFFFFFFFF` to signify the end of the current
section.

For SMC hacked images, patches are structured in this order:
* 1BL patches
* CB patches
* CD patches
* Hypervisor/kernel patches

For RGH images, patches are structured in this order:
* CB_B patches
* CD patches
* Hypervisor/kernel patches

## Recommended reading

* `about_patches.S` included in xeBuild.

* DrSchottky's "X360 Reversing" guide about xeBuild's hacked images:
https://www.razielconsole.com/forum/guide-e-tutorial-xbox-360/943-%5Bx360-reversing%5D-intro.html
