---
layout: post 
title: "OpenEEPROM"
exclude: true
---

## An open source universal programmer.

OpenEEPROM is a free alternative to devices like the 
[MiniPRO TL86xx](https://www.amazon.com/Universal-Programmer-TL866II-MiniPro-Adapter/dp/B091TPHW1M). 
OpenEEPROM is itself a protocol. The protocol consists of a set of commands
for programming parallel and SPI chips (and I2C eventually).

The OpenEEPROM project contains two components:

- a programmer running firmware that implements the OpenEEPROM protocol.
- a host that runs on a PC and communicates with the programmer.

Basically, the host implementation must be able to send and receive OpenEEPROM commands, while
the firmware implementation must be able to receive and interpret commands from the host, 
send appropriate digital signals to the memory chip connected to it, and respond to the host.

The project currently implements a Python host. Here are some example uses, assuming
the programmer and host are communicating over a serial port.

Write the contents of `input.txt` to a 25LC320 SPI EEPROM:

```
$ python -m openeeprom write --file input.txt --chip 25LC320 --serial /dev/ttyACM0:115200
```

Or to read the contents of an Atmel 28C256 parallel EEPROM into `output.bin`:

```
$ python -m openeeprom read --file output.bin --chip AT28C256 --serial /dev/ttyACM0:115200
```

You can also import the `openeeprom` package directly:


```
from openeeprom.transport import SerialTransport
from openeeprom.client import OpenEEPROMClient
from openeeprom.chip import AT28C256

client = OpenEEPROMClient(SerialTransport(port='/dev/ttyACM0', baud_rate=115200))
chip = AT28C256()
chip.connect(client)

contents = chip.read(0, chip.size)
```

The OpenEEPROM protocol command set enables you to write your individual chip drivers on the 
host side in a high-level language such as Python, shifting the effort away from the firmware/embedded side.

View the source [here](https://github.com/davidday99/open-eeprom).

