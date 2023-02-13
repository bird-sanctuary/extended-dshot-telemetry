# Extended DSHOT Telemetry (EDT) Specification

Extended DSHOT Telemery extends - as the name suggests - the telemetry data which is sent from ESC (via signal wire) to the flight-controller. For this to work, bi-directional DSHOT needs to be active.

In addition to the eRPM data which was the initial data being transmitted back to the flight-controller, EDT allows to encode further details in the telemetry frame.

## Frame structure
The frame structure of the telemetry frame is 16 bit long: `eeem mmmm mmmm`

* `eee`: the exponent
* `mmmmmmmmm`: the eRPM value

To calculate the real eRPM value, it is shifted to the left by the exponent:

```
eRPM = m << e
```

or in other words:

```
eRPM = m * 2 ^ e
```

From this, one can easily see, that the same eRPM values could be encoded in multiple different ways, let's for example take the eRPM value of 8, it could be encoded in these variations:

```
000 0 0000 1000
001 0 0000 0100
010 0 0000 0010
011 0 0000 0001
```
This is true for a lot of different values, thus the idea was born to always right shift the rpm values as far as possible, so that no ambiguity is left and each value has only one representation, the other - now free - values can be used to encode different data.

The first 3 bit (the exponent) together with the 4th bit allow to distinguish if it is an eRPM Frame, or an extended DSHOT frame. Those four bits are called the **prefix**.

### eRPM Frames
- `000 0 mmmm mmmm` - [0, 1, 2, ... , 255]
- `000 1 mmmm mmmm` - [256, 257, 258, ..., 511]
- `001 1 mmmm mmmm` - [512, 514, 516, ... , 1022]
- `010 1 mmmm mmmm` - [1024, 1028, 1032, ..., 2044]
- `011 1 mmmm mmmm` - [2048, 2056, 2064, ..., 4088]
- `100 1 mmmm mmmm` - [4096, 4112, 4128, ..., 8176]
- `101 1 mmmm mmmm` - [8192, 8224, 8256, ..., 16352]
- `110 1 mmmm mmmm` - [16384, 16448, 16512, ..., 32704]
- `111 1 mmmm mmmm` - [32768, 32896, 33024, ..., 65408]

> If the first 4 bit are 0 OR the 4th bit is 1, it is a eRPM frame, the other ranges are EDT frames.

### EDT Frames
This is where EDT versions come into play if not explicitly stated, the values are available in all versions

- `001 0 mmmm mmmm` - **Temperature** frame in degree Celsius, just like Blheli_32 and KISS [0, 1, ..., 255]
- `010 0 mmmm mmmm` - **Voltage** frame with a step size of 0,25V [0, 0.25 ..., 63,75]
- `011 0 mmmm mmmm` - **Current** frame with a step size of 1A [0, 1, ..., 255]
- `100 0 mmmm mmmm` - **Debug frame 1** not associated with any specific value, can be used to debug ESC firmware
- `101 0 mmmm mmmm` - **Debug frame 2** not associated with any specific value, can be used to debug ESC firmware
- `110 0 mmmm mmmm` - **Demag metric** frame [0, 1, ..., 255] (since v2.0.0)
- `111 0 mmmm mmmm` - **Status frame**: Bit[7] = demag event, Bit[6] = desync event, Bit[5] = Stall event, Bit[3-1] - Demag metric  max level (since v2.0.0)

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
All available data is sent if EDT has been enabled, it is not necessary to enable specific frames on the receiving end.

## Frame scheduling
By default eRPM telemetry is sent back to the FC, and every ESC firmware has to decide the best frame scheduling based on their configurations, use cases or preferences.

The following are an example schedule for transmitting EDT frames:
- `eRPM`: is sent if there is no other telemetry frame
- `Status`: - 1000 ms for example, at ms cycle 0; as a response of extended telemetry enable/disable commands; next frame when an event happens
- `Temperature`: - 1000 ms for example, at ms cycle 333
- `Voltage`: - 1000 ms for example, at ms cycle 666

## Glossary
* TBD: To be defined
* prefix: The first 4 bit of the telemetry frame
* FC: Flight-controller, or receiver of the telemetry frame to keep it general

## History
* v2.0.0 - Updated status frame, to add demag, desync and stall events, and max demag metric. Replaced _debug3_ frame by demag metric frame.
* v1.0.0 - Initial version
