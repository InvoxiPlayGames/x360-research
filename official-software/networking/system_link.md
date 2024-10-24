**Emma's Xbox 360 Research Notes - Networking**

Updated 24th October 2024.

Stub page.

# System Link

To protect network traffic on LAN multiplayer games from being tampered with,
the Xbox 360 employs network encryption as well as non-standard networking on
local LAN multiplayer.

This article also applies to Games for Windows - LIVE, in sections discussing
cross-platform system link.

Some information on this page was referenced from Xenia's partial
[XNet Implementation](https://github.com/xenia-project/xenia/blob/systemlink/src/xenia/kernel/xnet.cc).

## Encryption Key Initialisation

When a title initialises WinSock and XNet, `CXnIp::IpInit` initialises several
encryption keys, for HMAC validation, DES3 packet encryption, and a feed value
for using DES3 in CBC mode.

These keys are initialised by taking the LAN key from either the optional XEX
header value 0x40404 (16 byte fixed-sized structure with ID 0x0404) on Xbox 360,
or the `lankey` value from the .cfg XML next to the game executable on GfWL,
and using that to build 3 keys forming a buffer of size 0x3C bytes.

**Key Buffer:**

When building the key buffer, numbers 0 through 2 are prefixed at the start of
the LAN key and then hashed with either HMAC SHA-1 with the roamable obfuscation
key (Xbox 360), or encrypted* and then a regular SHA-1 (GfWL / cross-platform)
performed on it.

*\* The LAN key is encrypted - the prefix byte remains unencrypted.*

| Offset | Size   | Description                            |
| ------ | ------ | -------------------------------------- |
| `0x00` | `0x14` | (HMAC) SHA-1 result for 0x00 + LAN key |
| `0x14` | `0x14` | (HMAC) SHA-1 result for 0x01 + LAN key |
| `0x28` | `0x14` | (HMAC) SHA-1 result for 0x02 + LAN key |

This buffer is then sliced up into 3 final keys for the encryption process.

| Offset | Size   | Description                          |
| ------ | ------ | ------------------------------------ |
| `0x00` | `0x10` | HMAC SHA-1 key for packet validation |
| `0x10` | `0x18` | 3DES encryption key for packet data  |
| `0x28` | `0x8`  | Key for building the CBC feed        |

**Pseudocode:**

(Note that this is pseudocode of just key initialisation - it is not C that can
be compiled nor is it any specific CXnIp function)

```c
void initialise_ip_encryption(CXnIp *this) {
    struct {
        char id;
        uint8_t key[0x10]; 
    } config_buffer; // sizeof(config_buffer) = 0x11

    struct {
        uint8_t key1[0x14];
        uint8_t key2[0x14];
        uint8_t key3[0x14];
    } key_buffer; // sizeof(key_buffer) = 0x3c

    // fetch the LAN key from the executable (360) or config file (GfWL)
    uint8_t *lan_key = get_lan_key();
    if (lan_key == NULL) // no key set = use random key, useless lmao
        XeCryptRandom(config_buffer.key, sizeof(config_buffer.key));
    else
        memcpy(config_buffer.key, lan_key, sizeof(config_buffer.key));

    // only 360 takes this path, GfWL always goes down the cross-platform path
#ifdef XBOX360
    if (use_cross_platform() == false) {
        XeCryptRandom(&key_buffer, sizeof(key_buffer));

        config_buffer.id = 0;
        XeCryptHmacSha(ROAMABLE_KEY, &config_buffer, sizeof(config_buffer), NULL, 0, NULL, 0, key_buffer.key1, 0x14);
        config_buffer.id = 1;
        XeCryptHmacSha(ROAMABLE_KEY, &config_buffer, sizeof(config_buffer), NULL, 0, NULL, 0, key_buffer.key2, 0x14);
        config_buffer.id = 2;
        XeCryptHmacSha(ROAMABLE_KEY, &config_buffer, sizeof(config_buffer), NULL, 0, NULL, 0, key_buffer.key3, 0x14);
    } else
#endif
    {
        // encrypt the title key with the cross-platform system link key,
        // protected by the hypervisor / some mad x86 fuckery
        int r = XeKeysAesCbc(XPLAT_SYSLINK_KEY, config_buffer.key, 0x10, config_buffer.key, &key_buffer /*this is IV, what?*/, ENCRYPT);
        if (!r) // encryption failed, use random key, useless
            XeCryptRandom(config_buffer.key, sizeof(config_buffer.key))
        
        config_buffer.id = 0;
        XeCryptSha(&config_buffer, 0x11, NULL, 0, NULL, 0, key_buffer.key1, 0x14);
        config_buffer.id = 1;
        XeCryptSha(&config_buffer, 0x11, NULL, 0, NULL, 0, key_buffer.key2, 0x14);
        config_buffer.id = 2;
        XeCryptSha(&config_buffer, 0x11, NULL, 0, NULL, 0, key_buffer.key3, 0x14);
    }

    // use first 0x10 bytes of key1 for a 0x10 byte HMAC-SHA1 key
    memcpy(this->lan_hmac_key, key_buffer.key1, sizeof(this->lan_hmac_key)); // 0x10

    // use the next 0x4 bytes of key1 and all of key2 for a 0x18 byte 3DES key
    memcpy(this->lan_3des_key, key_buffer + 0x10, sizeof(this->lan_3des_key)); // 0x18
    XeCryptDesParity(this->lan_3des_key, sizeof(this->lan_3des_key), this->lan_3des_key);

    // use the first 0x8 bytes of key3 for some 0x8 byte key for seeding CBC mode
    memcpy(this->lan_cbc_feed, key_buffer.key3, sizeof(this->lan_cbc_feed)); // 0x8
}
```

## Packet Structure

*(TODO: This is for broadcast on 360 packets, but what about P2P/xplat? Check)*

All values are in little endian. Why?

**Xbox 360:** Broadcast messages are sent as IPv4 UDP packets over port 3074,
with a source IP of 0.0.0.1 and a destination IP address of 255.255.255.255.
The destination MAC address is FF:FF:FF:FF:FF:FF.

**Cross-Platform:** TODO

### General Structure

| Offset   | Type / Size | Description                         |
| -------- | ----------- | ----------------------------------- |
| `0x0`    | uint32      | Header flags *(TODO: Check?)*       |
| `0x4`    | variable    | Encrypted packet data               |
| variable | Footer      | Metadata about the packet           |

Note that parts of the footer will be encrypted depending on the length of the
packet data.

### Footer

| Offset | Type / Size | Description                   |
| ------ | ----------- | ----------------------------- |
| `0x0`  | uint16      | Source port                   |
| `0x2`  | uint16      | Destination port              |
| `0x4`  | uint32      | Title ID                      |
| `0x8`  | uint32      | Title version                 |
| `0xC`  | uint32      | System kernel version         |
| `0x10` | uint8       | Bytes encrypted divided by 8  |
| `0x11` | uint16      | Seed value for the CBC feed   |
| `0x13` | byte[0xA]   | HMAC SHA-1 checksum           |

### Encryption

TODO
