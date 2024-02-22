**Emma's Xbox 360 Research Notes - Hypervisor**

Updated 14th January 2024.

# Key Derivation

Inside the Xbox 360 hypervisor, a lot of important "keys to the castle" are obfuscated,
so having a plain dump of the hypervisor "in vitro" from the NAND or combined with an
update package won't get you access to any of the keys - the only way to get the keys
required to decrypt executables, game discs, content packages, etc is by dumping them
from a currently running console that you have code execution on, or finding the secret
value and deriving them yourself.

These keys are obfuscated with a value that stems from inside the CPU's security engine.
The code that loads this value is located at `0x2DC8` in the current latest hypervisor
(17559), but can be located by searching for the instruction `std rX, 0x18(0)`, as this
value is always stored at offset `0x18` in memory during hypervisor initialisation.

The MMIO offset this value is read from is `0x80000200_000250B8`. This can be read from
within XeLL and libxenon applications from the 32-bit address `0x200250B8`. The expected
value on a regular Xbox 360 has a CRC32 of `0x4B17A409`. (Begins `81300D...`)

*(Note: If reading from HV or libxenon context, you must make sure this value is fetched
with an "ld" instruction, otherwise the system could lock up.)*

*(TODO: It'd be nice to find out what this value means, and what affects it.)*

The derivation process consists of calls to XeCryptHmacSha, with the aforementioned key 
being used as the HMAC key, and the "root data" for each given key being used as the input
value. The final 4 bytes of the HMAC SHA-1 are cut off. For example:

```c
XeCryptHmacSha(&gCpuSecurityKey, 0x8, gRootKeyXEX2, 0x10, NULL, 0, NULL, 0, gActualKeyXEX2, 0x10);
```

## Encryption Keys

These keys are all different on devkit compared to retail. The CRC32s are for retail.
All of these keys are 16 bytes in length.

* XEX2 decryption key - key used to decrypt all retail .xex executable files.
  * Root key CRC32: `0x1124BB06` (Begins `A26C10...`)
  * Final key CRC32: `0x10802758` (Begins `20B185...`)
  * The "root key" for this is sometimes called the "XEX1 key".
* "DVD authentication" key - key used in HvxDvdAuthRecordXControl.
  * Root key CRC32: `0x01A2298E` (Begins `79AD7F...`)
  * Final key CRC32: `0x535EFCAD` (Begins `D1E3B3...`)
* Cross-platform system link key - key used to encrypt traffic going to and from PCs.
  * Root key CRC32: `0x91F106E5` (Begins `974935...`)
  * Final key CRC32: `0x86741BAC` (Begins `64FA1A...`)
* Roamable obfuscation key - key used to obfuscate profiles, some savegames, etc.
  * Root key CRC32: `0xA6CD129F` (Begins `E489E0...`)
  * Final key CRC32: `0x5997D32D` (Begins `E1BC15...`)
  * This is loaded into the keyvault structure, not at a fixed spot in memory.
  * If the console's keyvault already has a key in that spot, this won't be used.

*(TODO: Are there any more keys?)*

## Title Keys

The syscall HvxKeysSetKey allows for "title keys" to be set, with IDs 0xE0 - 0xE8 (with
the ID 0xF0 being reserved for resets). Titles can use these key slots to perform crypto
operations with the HvxKeys syscalls, while hiding the actual key being used. The input
to this syscall is taken by the hypervisor and used as a HMAC key, and mashed together
against a bunch of different keys, all of which will be different across retail and devkit.
This was likely to ensure that devkits couldn't be used to derive the base keys or used
as decryption oracles, for the few games that used this. (such as the Rock Band series)

Pseudocode for the function responsible is provided below:

```c
// HvxKeysSetKey calls this function with a destination in secure RAM with key IDs 0xE0-0xE8
// Key IDs 0xF0 causes a memset in that secure RAM region, setting it all to zeroes.
// 17559 - 0x3B10
int SetTitleKey(char *key_dest, PHYS_PTR(0x10) char *input_key, int key_len)
{
    if (key_len != 0x10)
        return KEYS_STATUS_INVALID_LENGTH;
    
    XECRYPT_HMACSHA_CTX ctx;
    XeCryptHmacShaInit(&ctx, input_key, 0x10);

    // Values copied from the 1BL during HV init.
    XeCryptHmacShaUpdate(&ctx, gPublicKey1BL, 0x110);
    XeCryptHmacShaUpdate(&ctx, gEncryptionKey1BL, 0x10);
    XeCryptHmacShaUpdate(&ctx, gSalt1BL, 10);

    // The security value copied from the CPU during HV init.
    XeCryptHmacShaUpdate(&ctx, &gCpuSecurityKey, 0x8);

    // A bunch of different keys - this large buffer contains:
    // - 0x110 byte PIRS content verification (XEX, STFS) public key (key 0x39)
    // - 0x330 bytes of NULLs, likely more public key space
    // - 0x10 byte root key for XEX2 key
    // - 0x10 byte root key for roamable obfuscation key
    // - 0x110 byte XeMACS authentication public key (key 0x41)
    XeCryptHmacShaUpdate(&ctx, &gStaticKeyBuffer, 0x590);

    // The 4 keys derived from the CPU security key previously.
    XeCryptHmacShaUpdate(&ctx, gXex2Key, 0x10);
    XeCryptHmacShaUpdate(&ctx, gDvdAuthKey, 0x10);
    XeCryptHmacShaUpdate(&ctx, gKeyVault->roamableObfuscationKey, 0x10);
    XeCryptHmacShaUpdate(&ctx, gXplatSyslinkKey, 0x10);

    // The output key to use in subsequent calls to HvxKeys functions with this key ID.
    XeCryptHmacShaFinal(&ctx, key_dest, 0x10);

    return KEYS_STATUS_OK;
}
```
