# Airmate

A small ESP-12F (ESP8266)-based DIY IR controller for the Airmate Mystery MSF-2414 fan, hence the name.

Related IRDB PR: https://github.com/probonopd/irdb/pull/30

## Video

[![](http://img.youtube.com/vi/UZf-yPra764/maxresdefault.jpg)](https://youtu.be/UZf-yPra764)

## Hardware

Solder as on the picture (GPIO15 pulled down to GND, EN pulled up to VCC).
IR LED to GPIO4 (doesn't need a resistor).
Hook up UART (RX to TXD0, TX to RXD0).
To enable flash mode, short GPIO0 to GND during startup.
Build and flash [IRSender.ino](https://github.com/joric/airmate/blob/master/IRSender/IRSender.ino)
(use Arduino IDE setup from [ESP8266-HTTP-IR-Blaster](https://github.com/mdhiggins/ESP8266-HTTP-IR-Blaster)).
Check for the IR Sender address in your Wi-Fi router settings.

ESP12-F dies from 5V in about a week, so 3.3V regulator (e.g. AMS1117-based) is a must.
You can also use NodeMCU boards, they are a little bit better (have built in UART and 3.3V regulator), but a few times more expensive.
Mind that cheap fans use 15-20 mA condenser PSU's and Wi-Fi modules need 70-300 mA, so
you probably would also need a separate [power supply](https://www.aliexpress.com/wholesale?catId=0&SearchText=3.3v%20transformer%20power%20supply).

![](https://i.imgur.com/9JLIY99.jpg)

![](https://i.imgur.com/Z3OM5Vm.jpg)

More pictures:

* https://imgur.com/a/qzOVUFb

## Capturing and sending IR codes

Use [IRCapture.ino](https://github.com/joric/airmate/blob/master/IRCapture/IRCapture.ino)
and Arduino Nano with VS1838 IR receiver (from 37 in 1 set) hooked up to pin D2 to capture IR codes.
You can also capture and convert IR codes with [IrScrutinizer](https://github.com/bengtmartensson/IrScrutinizer/releases) and Arduino Nano
with [GirsLite 1.0.2](https://github.com/bengtmartensson/AGirs/releases) firmware (hook up IR receiver module to pins D5, GND and 5V, check "Use receive for capture"
in capture settings, export as Arduino RAW).
After you captured and saved the codes you can send them in a comma-separated RAW format via the IR Sender's HTTP interface.

* https://github.com/bengtmartensson/AGirs/releases (GirsLite 1.0.2 firmwware)
* https://github.com/bengtmartensson/IrScrutinizer/releases (IrScrutinizer)

![](https://i.imgur.com/4w0tTFJ.jpg)

## IR Protocol

There are a lot of formats, autodetection and generation of those formats is pretty complicated.
You could try looking into [IrpTransmogrifier](https://github.com/bengtmartensson/IrpTransmogrifier) code
(used in [IrScrutinizer](https://github.com/bengtmartensson/IrScrutinizer)).
This is probably the best detector and generator that reads IRP format notation from the XML definition file
([IrpProtocols.xml](https://github.com/bengtmartensson/IrpTransmogrifier/blob/master/src/main/resources/IrpProtocols.xml)),
so it's data-driven and not hardcoded as in [decodeir](https://github.com/probonopd/decodeir).

### Raw format

RAW format is pretty simple, it's just a comma-separated sequence of microsecond timings:

`1330,438,1258,440,410,1288,1260,438,1258,440,384,1312,386,1312,412,1286,386,1312,386,1312,1260,440,384,8106`

Sequence starts with light on and alternates. Timings also could be explicitly written with alternating signs, where '+' means light on, '-' means light off (order is always the same):

`+1330 -438 +1258 -440 +410 -1288 +1260 -438 +1258 -440 +384 -1312 +386 -1312 +412 -1286 +386 -1312 +386 -1312 +1260 -440 +384 -8106`

Bits are usually encoded in value pairs, For F12 bit specification (<1,-3|3,-1>) that means [+1x,-3x]=0, [+3x,-1x]=1.

`[+1330,-438],[+1258,-440],[+410,-1288],[+1260,-438],[+1258,-440],[+384,-1312],... =  [1],[1],[0],[1],[1],[0],...`

Examples above encode the same 12 bits, 110110000010. For F12 bitfield (D:3,S:1,F:8) that would mean
D=3, S=1, F=65 (110,1,10000010 = 011b,1b,01000001b in LSB notation).

Carrier frequency (usually 38 Khz) doesn't really affect microsecond timings, every timing consists of many (dozens)
bursts of light at the carrier frequency. It's just a pulse width modulation (PWM) thing, that controls signal brightness and intensity.

Pronto hex format takes carrier frequency into account, so every value is adjusted as 1000000*value/carrierFrequency.

### IRP format notation

Example (F12 format): **{37.9k,422}<1,-3|3,-1>(D:3,S:1,F:8,-80)***

* **{37.9k,422}** General: {carrier frequency, time unit, sequencing rule}
	* Carrier Frequency: Hz; e.g. 38.3k; default is 0k (no modulation)
	* Time Unit: Integer that can represent durations. Suffix u (default) is microseconds, p is pulses of the carrier.
	* Sequencing Rule: lsb|msb; lsb (default) means the least significant bit of a binary form is sent first.


* **<1,-3|3,-1>** BitSpec: Rule for the translating bit sequences to duration <ZeroPulseSeq|OnePulseSeq|TwoPulseSeq....>
	* Most IR protocols use only <ZeroPulseSeq|OnePulseSeq>, and the sequence is simply OnDuration,OffDuration.
	* Example: NEC uses <1,-1|1,-3>. Durations are given in Time Units specified above.


* **(D:3,S:1,F:8,-80)*** Bitfield: B:NumberOfBits:StartingBit (StartingBit is optional. B:6 is equivalent to B:6:0)
	* if B=47=01000111, B:2:5 means x10xxxxx. B:2:5=10b=2. ~ is the bitwise complement operator, ~B=10111000.
	* A trailing + means send one or more times, a trailing * means send zero or more times.
	* No prefix means a flash, a preceeding "-" (minus) means a gap, which trails a signal.
	* D means device code, S is subdevice code, F is function

See http://www.hifi-remote.com/johnsfine/DecodeIR.html for details.

Actual Airmate fan codes in F12 format (see https://github.com/probonopd/irdb/pull/30):

functionname|protocol|device|subdevice|function
--|--|--|--|--
Oscillation | F12|3|1|9
Timer | F12|3|1|17
Fan mode | F12|3|1|33
Speed | F12|3|1|65
Light | F12|3|1|99
Shutdown | F12|3|1|129
Presets | F12|3|1|195

For example, "Speed" (protocol=F12, D=3, S=1, F=65) encodes into bits=110110000010
because D=3 (0x011), S=1 (0x01), F=65 (0x01000001) is `[110][1][10000010]` in MSB notation.

Then we apply pulse specs, `<1,-3|3,-1>` scheme (ZeroPulseSeq=1,-3; OnePulseSeq=3,-1) means 3x on duration followed by 1x off duration is bit 1; 1x on duration followed by 3x off duration is bit 0.

`[+1330,-438],[+1258,-440],[+410,-1288],[+1260,-438],[+1258,-440],[+384,-1312],... =  [1],[1],[0],[1],[1],[0],...`

LED on/off durations start from on and alternate to the end. Durations use carrier frequency units (1/38000s).
So the resulting sequence for the "Speed" command is (24 values):

`1330,438,1258,440,410,1288,1260,438,1258,440,384,1312,386,1312,412,1286,386,1312,386,1312,1260,440,384,8106`


## Roomba

There's also a small project for Roomba 620 (in progress). I tried to utilize IR sender but it
didn't work for some reason, so I just used roomba serial port to send commands (you need it for power anyway).
The serial port is located under the top cover (it can be lifted with bare hands), some kind of
mini-DIN connector (sadly mouse/keyboard PS/2 adapters can't be donors for that, they [won't fit](https://i.imgur.com/9b3O1Kg.jpg)).
I've used 3.3V regulator and the unshielded ESP-12F module, so it fits under the top cover.

* [RoombaSerial.ino](https://github.com/joric/airmate/blob/master/RoombaSerial/RoombaSerial.ino) (RoombaSerial sketch, mostly works fine, occasionally hangs, probably regulator is overheating)
* [roomba_serial.sh](https://github.com/joric/airmate/blob/master/scripts/roomba_serial.sh) (RoombaSerial control script)
* [roomba_ctrl.sh](https://github.com/joric/airmate/blob/master/scripts/roomba_ctrl.sh) (IRSender script, doesn't work yet)

IR timings grabbed with IrScrutinizer from a working [RCoid](http://rcoid.de/remotefiles.html) remote. Don't work yet (maybe need mode change).

Code | Action | Comment / IR sequence
---|---|---
129|baud| (serial only) switch baud rate, 7 for 19200 or 11 for 115200
128|start|(serial only) switch mode, 131 - safe, 132 - full control over actuators
134|spot|3036,981,981,3036,981,3036,981,3036,981,3036,3036,981,981,3036,981
135|clean|3036,981,981,3036,981,3036,981,3036,3036,981,981,3036,981,3036,981
143|dock|2994,1021,1021,2994,1021,2994,1021,2994,2994,1021,2994,1021,2994,1021,2994

As for the serial you just send 3 bytes (with 50 ms delay), e.g. 128,131,135 to clean. See [Roomba OI documentation](https://www.irobot.lv/uploaded_files/File/iRobot_Roomba_500_Open_Interface_Spec.pdf).

![cover removal](https://i.imgur.com/KYULdfd.png)

**WARNING! Measured voltage on Roomba 620 serial adapter is 15V on battery, 20V while charging (!) so don't use AMS1117, it's up to 15V max, it will burn and kill your ESP module. Use DC-DC adapter or MC33269.**

![esp-12f on roomba](https://i.imgur.com/D0KXeZr.jpg)

![esp-01 on roomba](https://i.imgur.com/brmV5nT.jpg)

There's a very useful video about disassembling and adding WiFi to Roomba:

* https://youtu.be/t2NgA8qYcFI (shows how to remove the top cover and send commands for roomba)

Same as a blog post:

* https://www.crc.id.au/hacking-the-roomba-600

While the 400 & 500 series are wakeable on the dock, the 600 series is not less you request [firmware upgrade from iRobot](https://robotics.stackexchange.com/questions/15433/create-2-losing-serial-communication-after-toggling-full-to-passive-while-chargi).
It's possible to wake up the 600 series with a different version of sketch that pulses the BRC pin to prevent sleep, but that seems like a better solution.

* https://github.com/thehookup/MQTT-Roomba-ESP01/blob/master/RoombaESP01_CONFIGURE.ino (500x)
* https://github.com/thehookup/MQTT-Roomba-ESP01/blob/master/Roomba_600_ESP01_CONFIGURE.ino (600x)

To disable sleep, pulse the BRC pin low periodically before these five minutes expire. Each pulse resets this five minute counter.
Careful! BRC pin is high voltage pin (+5V most of the time), direct connection kills your ESP.

![esp pinout](https://i.imgur.com/cO1b62K.jpg)

![din pinout](https://i.imgur.com/erDHbxD.png)

## References

* https://github.com/mdhiggins/ESP8266-HTTP-IR-Blaster (ESP8266-based universal learning remote)
* http://www.hifi-remote.com/johnsfine/DecodeIR.html (IRP format whitepaper)
* https://www.techdesign.be/projects/011/011_waves.htm (different IR waveforms)
* https://clearwater.com.au/code/rc5 (Philips RC5 format description)
* https://github.com/probonopd/irdb (large IR codes database, uses original IR formats)
* https://github.com/bengtmartensson/IrScrutinizer (Desktop IR capturing and decoding tool)
* https://github.com/bengtmartensson/AGirs (IrScrutinizer firmware for microcontrollers)
* https://github.com/probonopd/decodeir (command line IR decoding tool, supports pronto and raw)
* https://github.com/johnboiles/esp-roomba-mqtt (esp roomba firmware, uses Roomba library)
* https://www.reddit.com/r/esp8266/comments/flrqjf/roomba_620_esp/ (a Reddit post about roomba)
