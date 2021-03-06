A BIG thanks to the https://www.facebook.com/groups/SHgruppen on Facebook and a special superduper thanks to the core coder @strigeus!
https://github.com/strigeus/sparsnas_decoder

This is a fork is adapted for output in JSON format to a MQTT broker.

This is a decoder for IKEA SPARSNÄS.
===================================

It uses RTL-SDR running on a Raspberry Pi to demodulate the FSK signal, and decodes the packet.

The packet data is encrypted using your sender's ID. The sender ID is the last 6 digits of the serial number located under the battery.

Prerequisites:
-------------
Tested on a Raspberry Pi 3 with Raspbian Stretch with a NooElec Micro 3 USB SDR dongle. You should be able to use any RTL-SDR compatible dongle.

An IKEA Sparsnäs unit (duuh)

Make sure you have updated your Raspbian installation, then
```
sudo apt-get install libusb-1.0 rtl-sdr g++ git mosquitto-clients
```
Using your favorite text editor, create a new file named /etc/modprobe.d/no-rtl.conf and put the following text in the file. You need to run that text editor as sudo (i.e. 'sudo vi' or 'sudo nano' etc) to write to the modprobe.d directory:
```
sudo nano /etc/modprobe.d/no-rtl.conf
```
```
blacklist dvb_usb_rtl28xxu
blacklist rtl2832
blacklist rtl2830
```
Reboot your pi with the USB dongle installed in one of the USB ports.

Test your dongle
```
rtl_test
```
Outcome should look something like this:
```
pi@raspberrypi:~ $ rtl_test
Found 1 device(s):
  0:  Realtek, RTL2838UHIDIR, SN: 00000001

Using device 0: Generic RTL2832U OEM
Found Rafael Micro R820T tuner
Supported gain values (29): 0.0 0.9 1.4 2.7 3.7 7.7 8.7 12.5 14.4 15.7 16.6 19.7 20.7 22.9 25.4 28.0 29.7 32.8 33.8 36.4 37.2 38.6 40.2 42.1 43.4 43.9 44.5 48.0 49.6
[R82XX] PLL not locked!
Sampling at 2048000 S/s.

Info: This tool will continuously read from the device, and report if
samples get lost. If you observe no further output, everything is fine.

Reading samples in async mode...
lost at least 24 bytes
```

How to build the decoder:
-------------
Take a photo/make a note of your transmitter ID number underneeth the battery lid of the transmitter (not the display)

First edit sparsnas_decode.cpp to setup your sender id.
Then: 
```g++ -o sparsnas_decode -O3 sparsnas_decode.cpp```


How to calibrate to your sender frequency:
------------------------------------------
```rtl_sdr -f 868000000 -s 1024000 -g 40 - > sparsnas.raw```

Wait about 20 seconds. Press Ctrl-C.

```./sparsnas_decode sparsnas.raw --find-frequencies```

Wait about a minute for it to finish. It prints something like:

```#define FREQUENCIES {65050.000000, 105050.000000}```

Edit your sparsnas_decode.cpp with those values and rebuild it according to the How to build step above.

Perform a testrun:
-----------
```rtl_sdr -f 868000000 -s 1024000 -g 40 - | ./sparsnas_decode```

Create a bash script to send IKEA Sparsnäs output to MQTT broker:
-----------
```
pi@rpitest:~ $ nano sparsnas.sh
```
```
#!/bin/bash
rtl_sdr -f 868000000 -s 1024000 -g 40 - | sparsnas_decode 2>&1 | /usr/bin/mosquitto_pub -h 192.168.x.x -u username -P password -i sparsnas -l -t "home/sparsnas" &
```

Make the bash script executable
```chmod +x sparsnas.sh```

Copy the binary file to /usr/sbin
```sudo cp sparsnas_decode /usr/sbin```

Test the script:
```./sparsnas.sh```

Make it run on startup
```sudo nano /etc/rc.local```
```
#!/bin/sh -e
#
# rc.local
#
# This script is executed at the end of each multiuser runlevel.
# Make sure that the script will "exit 0" on success or any other
# value on error.
#
# In order to enable or disable this script just change the execution
# bits.
#
# By default this script does nothing.

# Print the IP address
_IP=$(hostname -I) || true
if [ "$_IP" ]; then
  printf "My IP address is %s\n" "$_IP"
fi

/home/pi/sparsnas.sh

exit 0
```

Check status and traffic on your MQTT broker:
```
mosquitto_sub -v -h 192.168.x.x -u username -P password -t '#'
```
Example of outcome:
```
core-ssh:~# mosquitto_sub -v -h 192.168.x.x -u username -P password -t '#'
home/sparsnas {"Sequence":35851,"Watt": 7372.8,"kWh":18889.219,"battery":100,"FreqErr":0.57}
```

Create a sensor in Home Assistant (example code, configuration.yaml):
----------------------------------
```
mqtt:
  broker: 127.0.0.1
  port: 1883
  client_id: home-assistant-1
  username: username
  password: password
  discovery: true
  discovery_prefix: homeassistant

sensor:
  - platform: mqtt
    state_topic: "home/sparsnas"
    name: "Sparsnäs energy consumption momentary"
    unit_of_measurement: "W"
    value_template: '{{ value_json.Watt | round(1) }}'
  - platform: mqtt
    state_topic: "home/sparsnas"
    name: "Sparsnäs energy consumption over time"
    unit_of_measurement: "kWh"
    value_template: '{{ value_json.kWh | round(1) }}'
  - platform: mqtt
    state_topic: "home/sparsnas"
    name: "Sparsnäs Battery remaining"
    unit_of_measurement: "%"
    value_template: '{{ value_json.battery | round(1) }}'
```



Technical details on the decoder:
------------------
SPARSNÄS uses a CC1101 chip configured for GFSK modulation at 868MHz. The two FSK symbol frequencies once downconverted to baseband are roughly 67kHz and 105kHz. (EDIT: It seems like with a different RTL-SDR the baseband frequencies are 12.5khz and 50khz. Maybe my first RTL-SDR was inaccurate). The packet length is 18 bytes, including the length byte, and a 16-bit CRC is appended to the end. The CC1101 sync word is 0xD201. The packet payload byte 5 to 17 are encrypted using a key derived from the sender device's ID.

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
1: uint8_t sender_id_lo;  // Lowest byte of sender ID
2: uint8_t unknown;       // Not sure
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
