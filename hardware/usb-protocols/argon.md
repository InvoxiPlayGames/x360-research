**Emma's Xbox 360 Research Notes - USB Protocols**

Updated 6th May 2024.

# Argon / Ring of Light / Xbox 360 Controller Wireless Receiver

The Ring of Light board on the Xbox 360 is responsible for connecting wireless
controllers to the Xbox 360, connected via an internal USB port. The behaviour
is also identical with the Xbox 360 Controller Wireless Receiver for Windows/PC.

Most of this information was reversed from xusb22.sys in Windows, and packet
analysis of a third-party wireless dongle. I don't have a legitimate one, sorry.
The Xbox 360 may use different report types not listed here.

(TODO: Check out how the Xbox 360 interfaces with the adapter, and how it
differs from Windows.)

## USB Descriptor

| Field           | Value               |
| --------------- | ------------------- |
| bDeviceClass    | `0xFF`              |
| bDeviceSubClass | `0xFF`              |
| bDeviceProtocol | `0xFF`              |
| idVendor        | `0x045E`            |
| idProduct       | `0x0291` / `0x0719` |

(A full copy of the USB descriptor of a third-party adapter is available in
descriptors/3rdparty-wireless-receiver.txt.)

The adapters support connecting 4 controllers at once, and has 8 interfaces -
for each controller, there's 1 controller data interface and 1 audio data
interface. Each interface has an input endpoint and an output endpoint.

**Controller Input Interface**

| Field              | Value  |
| ------------------ | ------ |
| bInterfaceClass    | `0xFF` |
| bInterfaceSubClass | `0x5D` |
| bInterfaceProtocol | `0x81` |

**Audio Input Interface**

| Field              | Value  |
| ------------------ | ------ |
| bInterfaceClass    | `0xFF` |
| bInterfaceSubClass | `0x5D` |
| bInterfaceProtocol | `0x82` |

### Endpoint List

| Address        | Description                        |
| -------------- | ---------------------------------- |
| `0x81` (IN 1)  | Player 1 Controller Input Reports  |
| `0x01` (OUT 1) | Player 1 Controller Output Reports |
| `0x82` (IN 2)  | Player 1 Audio Input Reports       |
| `0x02` (OUT 2) | Player 1 Audio Output Reports      |
| `0x83` (IN 3)  | Player 2 Controller Input Reports  |
| `0x03` (OUT 3) | Player 2 Controller Output Reports |
| `0x84` (IN 4)  | Player 2 Audio Input Reports       |
| `0x04` (OUT 4) | Player 2 Audio Output Reports      |
| `0x85` (IN 5)  | Player 3 Controller Input Reports  |
| `0x05` (OUT 5) | Player 3 Controller Output Reports |
| `0x86` (IN 6)  | Player 3 Audio Input Reports       |
| `0x06` (OUT 6) | Player 3 Audio Output Reports      |
| `0x87` (IN 7)  | Player 4 Controller Input Reports  |
| `0x07` (OUT 7) | Player 4 Controller Output Reports |
| `0x88` (IN 8)  | Player 4 Audio Input Reports       |
| `0x08` (OUT 8) | Player 4 Audio Output Reports      |

*Note: "Player" here doesn't correlate to the player LED as that is set by
software, it only refers to connection order to the adapter.*

## Controller Input Reports

These input reports are sent by the adapter in response to either controller
input changes or as requested by an output report.

### Report Header

All reports have this 5-byte header, and most are followed by a blob of data,
whos size depends on the type.

| Offset | Type / Size  | Description                           |
| ------ | ------------ | ------------------------------------- |
| `0x0`  | byte         | Packet ID (mostly 0x0, sometimes 0x8) |
| `0x1`  | byte         | Packet Type                           |
| `0x2`  | byte         | Unknown / Unused (always 0x00)        |
| `0x3`  | byte[0x2]    | Packet Header Data (used for State)   |

All packets except Connection Info 

### Connection (Packet ID 0x8)

The "connection" packet is the only packet using ID 0x8, and its data is
a bitfield stored in the "type" value of the packet header. It indicates whether
a controller is connected on this interface.

If the "type" value has the bit `0x80` set, it means a controller is connected.
If the bit `0x40` is set, it means a headset is connected.

This packet is returned in response to Connection Status requests sent on the
output interface.

(TODO: Is this report also sent when the status changes, without a request?)

### State (Type 0x0)

This packet contains the state of the currently connected controller and its
connected peripherals (batteries, headset, chatpad, etc). The packet's data
is stored in the "Header Data" field of the header, rather than being appended.

It is 2 bytes long and very bit-packed. The packet should be ignored if
`header[0] & 0xF0` is not equal to `0x10`.

TODO: prettify this into a table

```c
typedef struct _WirelessState {
    // byte 0
    uint8_t vibrationLevel : 2;
    bool headset : 1;
    bool chatpad : 1;
    uint8_t always_0x1 : 4;
    
    // byte 1
    bool unknown : 1;
    uint8_t batteryType : 2;
    bool onlyMic : 1;
    uint8_t powerState : 2;
    uint8_t batteryLevel : 2;
} WirelessState;
```

### Controller State (Type 0x1 / Type 0x3)

The controller state report contains the current controller input state - which
buttons and sticks are being pressed. Note how this structure includes the extra
6 bytes that are excluded from `XINPUT_GAMEPAD` structures in the regular XInput
headers.

| Offset | Type / Size    | Description                           |
| ------ | -------------- | ------------------------------------- |
| `0x0`  | byte           | Size                                  |
| `0x1`  | unsigned short | Buttons held bitmask                  |
| `0x3`  | byte           | Left trigger                          |
| `0x4`  | byte           | Right trigger                         |
| `0x5`  | unsigned short | Left stick X                          |
| `0x7`  | unsigned short | Left stick Y                          |
| `0x9`  | unsigned short | Right stick X                         |
| `0xB`  | unsigned short | Right stick Y                         |
| `0xD`  | byte[0x6]      | Reserved                              |

```c
typedef struct _WirelessController {
    uint8_t size;
    uint16_t buttons;
    uint8_t leftTrigger;
    uint8_t rightTrigger;
    uint16_t leftStickX;
    uint16_t leftStickY;
    uint16_t rightStickX;
    uint16_t rightStickY;
    uint8_t reserved[6];
} WirelessController;
```

This report is sent very frequently, it is sent every time an input changes on
the controller, and doesn't require any interaction from the driver to get
sent.

### Capabilities (Type 0x5)

The capabilities packet contains the controller's capabilities that get reported
via [XInputGetCapabilities / XINPUT_CAPABILITIES](https://learn.microsoft.com/en-us/windows/win32/api/xinput/ns-xinput-xinput_capabilities).

The first byte of the data section of the packet is always `0x12` - the packet
should be ignored if it isn't.

| Offset | Type / Size    | Description                           |
| ------ | -------------- | ------------------------------------- |
| `0x0`  | byte           | Unknown, always 0x12                  |
| `0x1`  | unsigned short | Buttons held bitmask                  |
| `0x3`  | byte           | Left trigger                          |
| `0x4`  | byte           | Right trigger                         |
| `0x5`  | unsigned short | Left stick X                          |
| `0x7`  | unsigned short | Left stick Y                          |
| `0x9`  | unsigned short | Right stick X                         |
| `0xB`  | unsigned short | Right stick Y                         |
| `0xD`  | byte           | Left motor                            |
| `0xE`  | byte           | Right motor                           |
| `0xF`  | byte[0x9]      | Unknown                               |

```c
typedef struct _WirelessCapabilities {
    uint8_t always_0x12;
    uint16_t buttons;
    uint8_t leftTrigger;
    uint8_t rightTrigger;
    uint16_t leftStickX;
    uint16_t leftStickY;
    uint16_t rightStickX;
    uint16_t rightStickY;
    uint8_t leftMotor;
    uint8_t rightMotor;
    uint8_t unk[9];
} WirelessCapabilities;
```

This report is returned as response to the Capabilities Request packet.

### Device Info / Link Report (Type 0xF)

The first byte of the data section of the packet is always `0xCC` - the packet
should be ignored if it isn't.

```c
typedef struct _WirelessLinkReport {
    uint8_t always_0xCC;
    uint32_t unk1;
    uint32_t deviceID;
    uint8_t type;
    uint8_t revision;
    uint8_t state[2];
    uint16_t protocol;
    uint8_t unk2[2];
    uint8_t vendorIDData[3];
    uint8_t subtype;
    uint8_t unk3[3];
} WirelessLinkReport;
```

To get a cohesive vendor ID, do `((vendorIDData[0] & 0xf) | vendorIDData[2] << 4) << 8 | vendorIDData[1]`.

## Controller Output Requests

TODO: better explanation and formats. Here is just binary data

### Controller Status Request

`08 00 0f c0 00 00 00 00 00 00 00 00`

### Battery Status Request

`00 00 00 40 00 00 00 00 00 00 00 00`

### Capabilities Request

`00 00 02 80 00 00 00 00 00 00 00 00`

### Set LED Request 

`00 00 08 41 00 00 00 00 00 00 00 00`

The 4th byte (`request[3]`) is set to `0x40 | led_type`. LED types 2 - 5 will blink player 1 - 4 LEDs. LED types 6 - 9 will set player 1 - 4 LEDs to solid.

### Set Power State Request 

`00 00 08 C0 00 00 00 00 00 00 00 00`

The 4th byte (`request[3]`) is set to `0xC0 | power_state`. Power state 0 will turn off the controller.
