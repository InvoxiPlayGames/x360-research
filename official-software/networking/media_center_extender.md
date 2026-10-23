**Emma's Xbox 360 Research Notes - Networking**

Updated 23rd October 2026.

# Windows Media Center Extender

*(TODO: Pairing process, RDP-like stream, RTP-like stream(s).)*

## Image Download

Since there is not enough space on the 360's NAND to store the Media Center
Extender, as well as to support the vastly different Windows Vista Media Center,
the Xbox 360 talks a custom protocol to a service running on the PC that
"finds the right XEX" and then sends that over TCP to the console.

When the Media Center is paired and then attempted to be launched from the
dashboard, the dashboard sends a wake on LAN packet for the paired Media Center
PC's MAC address (repeatedly), `ssdp:byebye` as per \[MS-SSDP\], then the
following SSDP broadcast packet:

```
NOTIFY * HTTP/1.1
HOST:239.255.255.250:1900
NT:urn:schemas-microsoft-com:nhed:presence:1
NTS:ssdp:alive
LOCATION:*
CACHE-CONTROL:max-age=4
AL:<urn:schemas-microsoft-com:nhed:attributes?type=X02&firmwarever=17559.0&udn=uuid:10000000-0000-0000-0200-xxxxxxxxxxxx&modelname=Xbox%20360&DvcFN=Xbox%20360&vendorid=1118&productid=2&hwrev=200&swrev=2&language=EN-US&manufacturer=Microsoft&imageheaderver=1074063928&affcode=153><urn:schemas-microsoft-com:nhed:download-target?address=192.168.50.58&port=1107>
USN:uuid:00000000-0000-0000-0200-xxxxxxxxxxxx::urn:schemas-microsoft-com:nhed:presence:1
SERVER:dashboard/1.0 UpnP/1.0 xbox/2.0
```
*(Note: Console MAC address replaced with `xxxxxxxxxxxx`.)*

The console listens on the TCP port specified in the
`urn:schemas-microsoft-com:nhed:download-target` segment of the `AL` header, and
waits for Mcx2Svc on the Windows PC to wake up and connect to it.

Upon recieving this SSDP request, the Windows PC will start up Mcx2Svc (located
in system32) and attempt to find the XEX file with the highest version number.

The locations XEX files are searched for:
* **Windows XP MCE**: *TODO*
* **Windows Vista and 7**: `%SystemRoot%\Windows\ehome\MCX\`

The subdirectory in the MCX folder is decided by the "type" parameter in the
`urn:schemas-microsoft-com:nhed:attributes` segment in the AL header.

If a valid XEX2 isn't found in that folder, then no connection is made. If a
suitable one is, a TCP connection is made to the IP address and port provided.

An implementation of the protocol is available at:
https://github.com/InvoxiPlayGames/XEXSled

### "Device Info" / Image Download Protocol

#### Packet Header

The packet header is 8 bytes. All fields are big endian.

| Offset | Type / Size | Description                  |
| ------ | ----------- | ---------------------------- |
| `0x0`  | uint32      | Packet magic (`0x51C674C0`)  |
| `0x4`  | uint16      | Packet type                  |
| `0x6`  | uint16      | Packet length (minus header) |

#### Optional Header

Data in this protocol is split into segments of "optional heders", in a
Type-Length-Value structure. The header is 4 bytes.

| Offset | Type / Size | Description                |
| ------ | ----------- | -------------------------- |
| `0x0`  | uint16      | Data type                  |
| `0x2`  | uint16      | Data length (minus header) |

#### `0x0141` Hello Packet (PC -> Xbox)

This is the first packet sent once the PC connects to the Xbox. It consists of
the following optional headers. All values little endian.

| Type     | Type / Size | Description                                        |
| -------- | ----------- | -------------------------------------------------- |
| `0x0118` | uint32 (LE) | Status code, always 0                              |
| `0x0110` | UUID / 0x10 | UUID of the Xbox, from `USN` header                |
| `0x0113` | UUID / 0x10 | UUID of the PC's Media Center Extender certificate |
| `0x0139` | uint32 (LE) | Length of the XEX2 file in bytes                   |
| `0x019D` | uint32 (LE) | Version of the XEX2 file from the Execution ID     |

After receiving this, the Xbox sends an Image Download Response packet.

#### `0x0122` Image Download Response (Xbox -> PC)

| Type     | Type / Size | Description                                         |
| -------- | ----------- | --------------------------------------------------- |
| `0x0118` | uint32 (LE) | Status code: 1 if the XEX2 file is cached, 0 if not |

If the status code is 1, the PC closes the connection after receiving this, and
the Xbox 360 launches the cached executable. This will be if the file size and
XEX version in the Hello packet match the one in the console's cache.

If the status code is 0, the PC will then begin sending the XEX2 file over raw
TCP, with no packet headers or structure.

#### `0x0126` Image Download Stop (Xbox -> PC)

| Type     | Type / Size | Description               |
| -------- | ----------- | ------------------------- |
| `0x0118` | uint32 (LE) | Status code: 0 on success |

(On XEX validation failure, the Xbox drops the connection with the PC and does
not return a response.)

Sent by the Xbox after verifying the XEX file. The PC closes the connection
after this packet is received.

If the Xbox has a properly formatted HDD connected, it will store the XEX file
in a console-signed container for the "Xbox 360 Dashboard" title ID (`FFFE07D1`)
as a system title (`000A0000`), named `MCXFile`. The XEX2 will be saved in this
container as `xboxmcx.xex`.

## Pairing Process

*(TODO: Document more of this, this is a stub section.)*

### Certificate/Key Generation

When the extender application is pairing with the console, the console creates
a self-signed X.509 certificate based off the system's MAC address and the
console-specific public/private keypair in the key vault.

#### Generated Certificate Details

The generated certificate is a version 3 X.509 DER-encoded certificate with some
extensions. It is self-signed with no other links in the chain.

Cleaned up OpenSSL info output:

* **Serial Number:** Last 5 characters of MAC address suffixed with `0001`
    * e.g. `002248ABCDEF` = `0x2248ABCDEF0001`
* **Signature Algorithm:** SHA-1 with RSA
* **Issuer:**
    * **Common Name:** `Microsoft XBOX 360`
* **Subject:**
    * **Common Name:** `Microsoft XBOX 360`
* **Valid After:** Nov 16 18:44:41 2005 (`051116184441Z`)
* **Valid Before:** Nov 16 15:00:03 2025 (`251116150003Z`). ha they're expired!
* **Public Key:** 1024-bit public key from the console certificate, with an
  exponent of 65537
* **X509v3 Extensions:**
    * **Subject Alternative Name:**
      `URI:uuid:20000000-0000-0000-0200-xxxxxxxxxxxx` (MAC address)
    * **Key Usage:** Digital Signature, Non Repudiation, Key Encipherment,
      Data Encipherment
    * **Extended Key Usage:** TLS Web Client Authentication,
      1.3.6.1.4.1.311.10.5.12
    * **1.3.6.1.4.1.311.55.1.1:** The console certificate verbatim, including
      master Microsoft signature (Mcx2Prov.exe verifies this!)
* **Signature:** Self-signed

*(TODO: Make a script that makes this certificate and private key given a KV.)*

## Known Extender XEX files

These are just the Media Center Extender files I've been able to find.
This is an incomplete list.

(Thank you to pivotman319 for helping find most of these!)

**Windows XP MCE filename:** XboxMcx.xex

**Windows Vista and 7 filename:** XboxMCX-V.XEX

### Retail, Windows release

Versions of the extender binary found in release Windows versions or shipped
via Windows Update. These are all retail signed.

| OS Build | XEX2 Version | XEX2 Title ID | PE timestamp (UTC) | SHA256 |
|-|-|-|-|-|
| Windows XP MCE 2005\* | v2.0.12980.9 | `584507D1` (XE-2001) | October 26th 2005, 00:48:45 | 5c3750daa4a797429d4c2795ddc4e15046f15c539201d9793b1e44c8da7efed4 |
| Windows Vista (6.0.6000.16386) | v2.0.22690.0 | `584507D2` (XE-2002) | September 26th 2006, 22:31:02 | 2b97da190acc1e7d34a72afb314b9a0892138aa30e25f9d3be5049df328f818b |
| Windows Vista (KB950125)\*\* | v2.0.22860.0 | `584507D2` (XE-2002) | February 28th 2008, 22:04:16 | c3a7af890f9023c83fb742540d2f4e409084166793e14ee2737ab3f821ea8494 |
| Windows 7 (6.1.7600.16385) | v2.0.25960.4 | `584507D3` (XE-2003) | May 22nd 2009, 15:38:11 | 468e674a29dc36b588e0d5186254e2e3f25efc6f3071f959e02bb64c58061a77 |

*\*: Windows XP MCE 2005 did not ship with the XEX, it was an optional update.*

*\*\*: KB950125 is included in Windows Vista Service Pack 2.*

### Retail, Windows beta

Versions of the extender binary found in beta builds of Windows that are retail
Xbox 360 signed.

Some of these builds may have been also shipped in release versions of Windows
or via Windows Update; I haven't checked extensively.

| OS Build | XEX2 Version | XEX2 Title ID | PE timestamp (UTC) | SHA256 |
|-|-|-|-|-|
| 6.1.6574.1 | v2.0.22720.0 | `584507D2` (XE-2002) | December 18th 2007, 23:51:59 | 74ea7ba022c5491bd2b5143791dca6296956d397f8837dddfb7a10dea568b24a |
| 6.1.6730.1 | v2.0.22920.0 | `584507D2` (XE-2002) | March 5th 2008, 07:10:52 | 14a900dbddbf6c05160a3517ffd7ce7cbd7f89d4e54651d50d9a15bd97f7e9d1 |
| 6.1.6758.0 | v2.0.22920.1 | `584507D2` (XE-2002) | March 5th 2008, 07:10:52 | abc9910ce177942e1a5abf2f0151a788e17bc877e6d0e789d6f59a8e5abeef59 |
| 6.1.6776.0 | v2.0.25570.0 | `584507D2` (XE-2002) | August 1st 2008, 04:31:09 | 1d04ecad51e73cd2ad2028684ae8bfb7a70b035c2f112e7f66200bb8b0fe65c4 |
| 6.1.7070.0 | v2.0.25960.0 | `584507D3` (XE-2003) | November 6th 2008, 23:19:42 | 793b4863031b63b07b62a56c0f48cc4e421b9e3ca5e3a97b16a308fe2bd6259e |
| 6.1.7127.0 | v2.0.25960.2 | `584507D3` (XE-2003) | March 30th 2009, 22:04:56 | 9853d4b5c35880e240643c77b8289adcc08d03aa321c3bdf293d385898eeb74b |

*(Note: All builds of Windows 6.1.7262.0 and up ship with v2.0.25960.4 of the
extender, listed in the "Windows release" section.)*

### Devkit, Windows beta

Versions of the extender binary found in beta builds of Windows that are devkit
Xbox 360 signed.

| OS Build | XEX2 Version | XEX2 Title ID | PE timestamp (UTC) | SHA256 |
|-|-|-|-|-|
| 6.0.5365.8 | v2.0.20860.0 | `584507D2` (XE-2002) | March 28th 2006, 04:35:11 | eaefa698b5d7b7a280663ef6d715d50dc430de1d3705acd2af42819bd624686d |
