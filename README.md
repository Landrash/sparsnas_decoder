
This is a decoder for IKEA SPARSNÄS.
===================================

It uses RTL-SDR running on a Raspberry Pi to demodulate the FSK signal, and decodes the packet.

Note: The packet data is encrypted using your sender's ID. I have yet to figure out how to reliably determine the sender ID, so unless you know your sender ID, or know how to perform a known plaintext attack, this project might be of limited use.

How to build:
-------------
```g++ -o sparsnas_decode -O3 sparsnas_decode.cpp```

How to run:
-----------
```rtl_sdr -f 868000000 -s 1024000 -g 40 - | ./sparsnas_decode```


Technical details:
------------------
SPARSNÄS uses a CC1101 chip configured for GFSK modulation at 868MHz. The two FSK symbol frequencies once downconverted to baseband are roughly 67kHz and 105kHz. The packet length is 18 bytes, including the length byte, and a 16-bit CRC is appended to the end. The CC1101 sync word is 0xD201. The packet payload byte 5 to 17 are encrypted using a key derived from the sender device's ID.

The encryption key is a repeating XOR key:
```
      const uint32_t sensor_id_sub = SENSOR_ID - 0x5D38E8CB;
      enc_key[0] = (uint8_t)(sensor_id_sub >> 24);
      enc_key[1] = (uint8_t)(sensor_id_sub);
      enc_key[2] = (uint8_t)(sensor_id_sub >> 8);
      enc_key[3] = 0x47;
      enc_key[4] = (uint8_t)(sensor_id_sub >> 16);
````

Packet format (big endian):
```
0: uint8_t length;        // Always 0x11
1: uint8_t always_0xff;   // Always 0xFF for me, maybe the same as the low byte of the sender_id, maybe not.
2: uint8_t always_0x70;   // Always 0x70
3: uint8_t major_version; // Always 0x07 - the major version number of the sender.
4: uint8_t minor_version; // Always 0x0E - the minor version number of the sender.
5: uint32_t sender_id;    // ID of sender
9: uint16_t time;         // Time in units of 15 seconds.
11:uint16_t effect;       // Current effect usage
13:uint32_t pulses;       // Total number of pulses
17:uint8_t battery;       // Battery level, 0-100.
```
This is how to convert the 'effect' field into Watt:
```
float watt =  (float)((3600000 / PULSES_PER_KWH) * 1024) / (effect);
```

Note: Due to how the encryption works, it's possible for two different SPARSNÄS with unique sender_id to confuse each other. When decrypting a packet from sender B with the A key, it's possible that the resulting packet gets a sender_id equal to A, which means A will display incorrectly decrypted data from the B device.

Example usage
-------------
```
ludde@raspberrypi ~/sparsnas $ sudo rtl_sdr -f 868000000 -s 1024000 -g 40 - | ./sparsnas_decode               
Found 1 device(s):
  0:  Realtek, RTL2838UHIDIR, SN: 00000001

Using device 0: Generic RTL2832U OEM
Found Rafael Micro R820T tuner
[R82XX] PLL not locked!
Sampling at 1024000 S/s.
Tuned to 868000000 Hz.
Tuner gain set to 40.20 dB.
Reading samples in async mode...
[2016-12-05 21:29:35] 12711:   362.3 W. 43.339 kWh. Batt 99%.
[2016-12-05 21:29:50] 12712:   362.6 W. 43.341 kWh. Batt 99%.
```