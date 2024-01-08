# Extended DSHOT Telemetry (EDT) Specification

Extended DSHOT Telemetry, as the name suggests, extends the telemetry data which is sent from ESC (via signal wire) to the flight-controller. For this to work, bi-directional DSHOT needs to be active.

In addition to the eRPM data which was the initial data being transmitted back to the flight-controller, EDT allows the encoding of further details in the telemetry frame.

## Prerequisites
* Bits are counted from the least significant bit (LSB) starting at 0

## Frame structure
The frame structure of the telemetry frame is 16 bits long: `eeem mmmm mmmm cccc`

* `eee`: the exponent
* `mmmmmmmmm`: the eRPM value
* `cccc`: the checksum

To calculate the real eRPM value, it is shifted to the left by the exponent:

```
eRPM = m << e
```

or in other words:

```
eRPM = m * 2 ^ e
```

From this, one can easily see that the same eRPM values could be encoded in multiple different ways.

8 can be encoded in this four ways:
```
000 0 0000 1000
001 0 0000 0100
010 0 0000 0010
011 0 0000 0001
```

512 can be encoded in this seven ways:

```
001 1 0000 0000
010 0 1000 0000
011 0 0100 0000
100 0 0010 0000
101 0 0001 0000
110 0 0000 1000
111 0 0000 0100
```

This is true for most values, thus the idea was born to normalize eRPM values, so that no ambiguity is left and each value has only one representation, the other - now free - values can be used to encode different data.

The last three bits (the exponent) together with the 8th bit (the MSB of the value) allow to distinguish between eRPM and extended DSHOT frames. Those four bits are called the **prefix**.

### Normalization
To normalize an eRPM value, the goal is to set Bit 8 by decreasing the exponent and left shifting the value. An implementation of the normalization could look something like so:

- 1. Check exponent
- 1.a. Exponent is 0: **Done**
- 1.b. Exponent > 0
- 2.a. Bit 8 is 1: **Done**
- 2.b. Bit 8 is 0
- 3. Decrease exponent, left shift value by one
- 4. Go to step 1

### eRPM Frames
After applying the normalization, the following ranges are possible for eRPM values:

- `000 0 mmmm mmmm` - [0, 1, 2, ... , 255]
- `000 1 mmmm mmmm` - [256, 257, 258, ..., 511]
- `001 1 mmmm mmmm` - [512, 514, 516, ... , 1022]
- `010 1 mmmm mmmm` - [1024, 1028, 1032, ..., 2044]
- `011 1 mmmm mmmm` - [2048, 2056, 2064, ..., 4088]
- `100 1 mmmm mmmm` - [4096, 4112, 4128, ..., 8176]
- `101 1 mmmm mmmm` - [8192, 8224, 8256, ..., 16352]
- `110 1 mmmm mmmm` - [16384, 16448, 16512, ..., 32704]
- `111 1 mmmm mmmm` - [32768, 32896, 33024, ..., 65408]

> If the last four bits are 0 OR the 8th bit is 1, it is a eRPM frame, the other ranges are EDT frames.

### EDT Frames
This is where EDT versions come into play if not explicitly stated, the values are available in all versions

- `001 0 mmmm mmmm` - **Temperature** frame in degree Celsius, just like Blheli_32 and KISS [0, 1, ..., 255]
- `010 0 mmmm mmmm` - **Voltage** frame with a step size of 0,25V [0, 0.25 ..., 63,75]
- `011 0 mmmm mmmm` - **Current** frame with a step size of 1A [0, 1, ..., 255]
- `100 0 mmmm mmmm` - **Debug frame 1** not associated with any specific value, can be used to debug ESC firmware
- `101 0 mmmm mmmm` - **Debug frame 2** not associated with any specific value, can be used to debug ESC firmware
- `110 0 mmmm mmmm` - **Stress level** frame [0, 1, ..., 255] (since v2.0.0)
- `111 0 mmmm mmmm` - **Status frame**: Bit[7] = alert event, Bit[6] = warning event, Bit[5] = error event, Bit[3-0] - Max. stress level [0-15] (since v2.0.0)

### Checksum
The 4 bits checksum is calculated the same way no matter if eRPM or EDT frame. Value in this example are the 12 data bits.

```
crc = (value ^ (value >> 4) ^ (value >> 8)) & 0x0F;
```

## Enabling EDT
To enable EDT the ESC has to receive the `DIGITAL_CMD_EXTENDED_TELEMETRY_ENABLE` command (DSHOT command 13) at least 6 times in succession. It then answers with a single version frame:

```
111 0 vvvv vvvv
```

How the received data is then interpreted is up to the receiving device - especially how the _DEBUG_ frames are interpreted.

## Disabling EDT
To disable EDT , the ESC has to recieve the `DIGITAL_CMD_EXTENDED_TELEMETRY_DISABLE` command (DSHOT command 14) at least 6 times in succession. It then answers with a single disabled frame:

```
1110 1111 1111
```

## Feature autodiscovery
The only mandatory frame is the status frame.
All available data is sent if EDT has been enabled - it is not necessary to enable specific frames on the receiving end.

## Frame scheduling
By default eRPM telemetry is sent back to the FC, and every ESC firmware has to decide the best frame scheduling based on their configurations, use cases or preferences.

The following is an example schedule for transmitting EDT frames:
- `eRPM`: is sent if there is no other telemetry frame
- `Status`: 1000 ms for example, at ms cycle 0; as a response of extended telemetry enable/disable commands; next frame when an event happens
- `Temperature`: 1000 ms for example, at ms cycle 333
- `Voltage`: 1000 ms for example, at ms cycle 666

## Timing
A bi-directional response frame (EDT frame) is to be sent 30μs after a DSHOT frame has been released. This interval does **not** depend on DSHOT speed.

## Glossary
* TBD: To be defined
* prefix: The first four bits of the telemetry frame
* FC: Flight controller or receiver of the telemetry frame* Stress level: Indicates the effort the ESC has to make to keep commutation going. In Blheli_S/Bluejay for example it is related to desync events. When it reaches the maximum indicates that the ESC cannot make reliable commutation. Zero is the ideal value, when ESC commutation is perfect.
* ESC alert event - Notification made for statistical/debug purposes. In Bluejay, for example this is the demag event. Later all this events can be inspected in the Blackbox Viewer or any other tool.
* ESC warning event - Notifies suspecting or unusual behaviour. In Bluejay, for example this is the desync event. Later all this events can be inspected in the Blackbox Viewer or any other tool.
* ESC error event - Notifies a serious problem. In Bluejay, for example this is the stall event. Later all this events can be inspected in the Blackbox Viewer or any other tool.

## Further Reading
* [DSHOT - The missing Handbook](https://brushlesswhoop.com/dshot-and-bidirectional-dshot/)

## History
* v2.1.1 - Clarified Bit index direction
* v2.1.0 - Added section about timing
* v2.0.2 - Fixed typo in stress level frame
* v2.0.1 - Improved wording, fixed typos
* v2.0.0 - Updated status frame to add demag, desync and stall events, and max demag metric. Replaced _debug3_ frame by stress level frame.
* v1.0.0 - Initial version
