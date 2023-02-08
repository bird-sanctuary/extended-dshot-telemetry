### Extended DSHOT Telemetry (EDT) specification v2 (2023)

I will try to keep this draft updated with all proposals you have. Please, if I am wrong with anything let me know. I will try to update this draft assap.
**_TBD_** - Stands for to be defined

### History
202301 - Updated status frame, to add demag, desync and stall events, and max demag metric. Replaced debug3 frame by demag metric frame.
202206 - Initial version

### Extended telemetry enable / disable

Extended telemetry enable
- DIGITAL_CMD_EXTENDED_TELEMETRY_ENABLE - (13) - Need 6x
- Answers with states / events frame 1110 vvvv vvvv (version -> read below)
- This Draft applies to Extended DSHOT Telemetry version 0 (0000 0000), that is the one described here.
- Last 8 bit are extended telemetry version. Values from 0 to 15 could reserved for betaflight, and from 16 to 254 could be reserved for OEM/customer applications. Depending on the version Betaflight could interpret telemetry data one way or another.

Extended telemetry disable
- DIGITAL_CMD_EXTENDED_TELEMETRY_DISABLE - (14) - Need 6x 
- Answers with states events frame 1110 1111 1111 (see below)

### Coding ranges

Telemetry frame: "_**eeem mmmm mmmm**_" where "**_eRPM coding value = m * 2 ^ e_**" (or eRPM coding value = m << e)

eeem mmmm mmmm
- 000m mmmm mmmm - [0, 1, 2, ... ,511] -> Coding eRPM
- 0010 mmmm mmmm - Temperature frame in celsius [0, 1, ..., 255] in degree Celsius, just like Blheli_32 and KISS
- 0011 mmmm mmmm - [512, 514, 516, ... ,1022] -> Coding eRPM
- 0100 mmmm mmmm - Voltage frame -> 0-63,75V step 0,25V
- 0101 mmmm mmmm - [1024, 1028, 1032, ..., 2044] -> Coding eRPM
- 0110 mmmm mmmm - Current frame -> 0-255A step 1A
- 0111 mmmm mmmm - [2048, 2056, 2064, ..., 4088] -> Coding eRPM
- 1000 mmmm mmmm - Debug frame 1
- 1001 mmmm mmmm - [4096, 4112, 4128, ..., 8176] -> Coding eRPM
- 1010 mmmm mmmm - Debug frame 2
- 1011 mmmm mmmm - [8192, 8224, 8256, ..., 16352] -> Coding eRPM
- 1100 mmmm mmmm - Demag metric frame -> 0-255
- 1101 mmmm mmmm - [16384, 16448, 16512, ..., 32704] -> Coding eRPM
- 1110 mmmm mmmm - Status frame: Bit[7] = demag event, Bit[6] = desync event, Bit[5] = Stall event, Bit[3-1] - Demag metric  max level
- 1111 mmmm mmmm - [32768, 32896, 33024, ..., 65408] -> Coding eRPM


### Feature autodiscovery

The only mandatory frame is State/events frame.
After receiving that Betaflight will autodiscover other frames as soon as it receives them. ej. temperature, current and voltage frames. This way it isn't neccesary to activate individual features.


### Frame scheduling

By default eRPM telemetry is sent back to the FC, and every ESC firmware will decide the best frame scheduling based on their configurations, use cases or preferences. So they will be defined as example, instead close defined.

- eRPM - Send it always there is no other telemetry frame
- Status - 1000 ms for example, at ms cycle 0; as a response of extended telemetry enable/disable commands; next frame when an event happens
- Temperature - 1000 ms for example, at ms cycle 333
- Voltage - 1000 ms for example, at ms cycle 666
