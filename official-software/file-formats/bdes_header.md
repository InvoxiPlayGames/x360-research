**Emma's Xbox 360 Research Notes - File Formats**

Updated 8th April 2024.

# "BDES" signed XZP header

In system updates starting with 16197, there are .xzp files included in the ZIP
file that are prepended with a header beginning with "BDES". This is used as a
header to sign the contents of the file, so it can't be modified while on the
hard drive.

## Structure

Total size: 0x11C

| Offset | Type / Size  | Description                         |
| ------ | ------------ | ----------------------------------- |
| `0x0`  | uint32       | Magic value (ascii 'BDES')          |
| `0x4`  | uint32       | Version (always 1)                  |
| `0x8`  | byte[0x114]  | PKCS#1 signature over the file      |

## Signature

The PKCS#1 signature is generated by creating a SHA-1 hash of the entire file
contents, starting at offset 0x11C (the end of the BDES header), and hashing
it in 0x1000 byte chunks. It is verified against the PIRS public key (XeKeys
value `XEKEY_CONSTANT_PIRS_KEY`).

It is verified in `Xui::_XuiVerifyPirs` in Xam.
