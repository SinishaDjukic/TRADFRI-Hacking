# Project fork
Note: please visit the original [TRADFRI-Hacking](https://github.com/basilfx/TRADFRI-Hacking) project for original readme notes and information. Below notes have been revised to include more details on using OpenOCD and Arduno-CMSIS-DAP.

# Hacking the IKEA TRÅDFRI
* [Introduction](#introduction)
* [Components](#components)
* [Pinout](#pinout)
* [Flashing using JTAG](#flashing-using-jtag)
* [Using JLink](#using-jlink)
* [Using OpenOCD](#using-openocd)
* [Custom firmware](#custom-firmware)
* [Isolation](#isolation)
* [Pictures](#pictures)
* [Relevant sources](#relevant-sources)
* [License](#license)
* [Disclaimer](#disclaimer)

## Introduction
The [IKEA TRÅDFRI](http://www.ikea.com/us/en/catalog/categories/departments/lighting/36812/) family of products provide you with several lighting solutions that interconnect using [ZigBee Light Link](http://www.zigbee.org/zigbee-for-developers/applicationstandards/zigbee-light-link/).

If we take a simple GU10 light bulb, it contains:

* Power supply
* LED driver
* IKEA TRÅDFRI module

The tiny IKEA TRÅDFRI module is used in many of their products, and is actually a small piece of circuit board with pins exposed. This board uses the energy-efficient Silicon Labs [EFR32MG1P132F256GM32](https://www.silabs.com/products/wireless/mesh-networking/efr32mg-mighty-gecko-zigbee-thread-soc/device.efr32mg1p132f256gm32) microcontroller (MCU), which is a ARM Cortex M4 with 256 KiB of flash and 32 KiB of memory.

You can take out the board, and hook it up to your own lighting solutions. Or, you can flash it with your [own firmware](#custom-firmware), for other purposes.

To find relevant products, I have compiled a [list of IKEA TRÅDFRI products](PRODUCTS.md) (please help me to update this list).

## Components
I have been able to identify the following parts on a IKEA TRÅDFRI module:

* Microcontroller: [EFR32MG1P132F256GM32](https://www.silabs.com/products/wireless/mesh-networking/efr32mg-mighty-gecko-zigbee-thread-soc/device.efr32mg1p132f256gm32)
* 2 Mbit SPI NOR Flash: [IS25LQ020B](http://www.issi.com/WW/pdf/25LQ025B-512B-010B-020B-040B.pdf)
* Crystal: 38.4 MHz

I am very certain that the SPI NOR Flash component is correct. The original firmware contains strings that refer to the exact part number. However, it also contains references to other SPI flash components, so your module may contain another one. The JEDEC ID it responds with is `9d 40 12`.

### Updated module
In January 2020 I bought the successor of the cheapest Trådfri LED bulb (the LED1837R5) and it contains an updated module (ICC-A-1). It looks like some components have been moved, but all the part numbers look the same. I have included updated pictures in the [Pictures](#pictures) section.

The only difference I have found (so far), is that PF3 is no longer an output pin, but used to enable the SPI NOR Flash.

## Pinout
The pinout of both modules is very similar.

[<img src="images/icc-1.png" alt="Back of IKEA TRÅDFRI module (ICC-1)" width="384">](images/icc-1.png)
[<img src="images/icc-a-1.png" alt="Back of IKEA TRÅDFRI module (ICC-A-1)" width="384">](images/icc-a-1.png)

[Marco van Nieuwenhoven](https://diystuff.nl/tradfri/tradfri-zigbee-light-link-module/) has provided a very detailed teardown of the ICC-1 module. He traced most of the copper traces and created a schematics on his website.

## Flashing using JTAG
To connect to an external JTAG/SWD debugger, connect as follows:

* PF0 -> SWCLK
* PF1 -> SWDIO
* PF2 -> SWO
* RESETn -> RESETn
* GND -> GND
* VCC -> VCC (3V3)

In my case, I could leave the module in the light bulb, but for flashing I provided my own power supply by hooking it up to the VCC line directly.

I'm working on a small PCB that can host a TRÅDFRI module. You can find it in [the pcbs folder](pcbs/devboard).

## Using JLink
You can use software like [JLink](https://www.segger.com/products/debug-probes/j-link/) to flash the target.

If you use JLink, you can use the command below to connect to the board:

```
JLink -If SWD -Speed 5000 -Device EFR32MG1PXXXF256
```

To dump the flash contents, use the command below (0x40000 is 256 KiB):

```
savebin output.bin 0x0 0x40000
```

To load a flash from file, use the following command:

```
loadbin output.bin 0x0
verifybin output.bin 0x0
```

I have confirmed that you can dump the flash, erase the device and load it again, and the light bulb will still work.

An analysis of the firmware encountered in the GU10 light I bougth can be found in [FIRMWARE.md](FIRMWARE.md).

## Using OpenOCD
You can use [OpenOCD](http://www.openocd.org) to flash the target. You also can download the [precompliled binaries](https://sourceforge.net/projects/openocd/files/openocd/).

OpenOCD supports both JTAG and SWD debugger protocols for ARM processors, for which you will need the corresponding hardware. You can look for one of the cheap alternatives or you can also use Arduino Pro Micro (3V3) or Teensy 3.2 (3V3).

Setup your Teensy and OpenOCD as follows:

* Clone Arduino CMSIS-DAP from [original project](https://github.com/myelin/arduino-cmsis-dap) or the [fork](https://github.com/sinishadjukic/arduino-cmsis-dap). The fork saves you a minute in the next step
* On your dev machine overwrite the original `C:\Program Files (x86)\Arduino\hardware\teensy\avr\cores\teensy3\usb_desc.h` file with the one provided by the forked project in `arduino-cmsis-dap/teensy3/usb_desh.h`. If you are using another HW variant please check the `arduino-cmsis-dap/arduino-cmsis-dap.ino` sketch for more instructions. These changes will modify your USB product name to include the `CMSIS-DAP` string and get recognized by OpenOCD
* Flash yout Teensy 3.2 with the sketch
* Download OpenOCD from the links above
* Clone the [TRADFRI-Hacking fork](https://github.com/SinishaDjukic/TRADFRI-Hacking/)
* Add the `TRADFRI-Hacking\openocd\scripts\target\efm32_ikea_icca1.cfg` file to your OpenOCD in `share\openocd\scripts\target\efm32_ikea_icca1.cfg`
* Connect your Teensy 3.2 to the Ikea module as follows (connecting the Reset pin does not seem to work)
  * Teensy GND - Ikea GND
  * Teensy VCC - Ikea VCC
  * Teensy 19 - Ikea PF1 (SWDIO)
  * Teensy 20 - Ikea PF0 (SWCLK)
  * Teensy 21 - Ikea PF2 (SWO)

Run OpenOCD in one terminal:
```
openocd -f interface/cmsis-dap.cfg -f target/efm32_ikea_icca1.cfg -d
```

Connect to OpenOCD in the second one via telnet:
```
telnet localhost 4444
```

You should be able to e.g. download the current firmare from the flash now:
```
> flash read_bank 0 ikea_icca1_gu10.bin                                                                                                 detected part: EFR32MG1P Mighty Gecko, rev 161                                                                                           flash size = 256kbytes                                                                                                                   flash page size = 2048bytes                                                                                                             wrote 262144 bytes to file ikea_icca1_gu10.bin from flash bank 0 at offset 0x00000000 in 11.238334s (22.779 KiB/s)  
```


## Custom firmware
The chip is a normal Cortex M4. You can flash it with anything. As a starting point, you could take a look at [this pull request](https://github.com/RIOT-OS/RIOT/pull/8047) for RIOT-OS. To get started.

I've added some firmwares in the [firmwares](firmwares/) folder.

As a proof of concept, check out [this YouTube video](https://www.youtube.com/watch?v=yi_Z2WtmdDU) I made. In there, I show how I control the LED connected via a serial console.

## Isolation
If you plan to leave the board in-place, and run your own light bulb firmware, never connect external devices (e.g. debugger or serial adapter) to a light bulb that is plugged in. Due to different voltage levels, you could destroy your devices.

If you want to connect an external device, ensure that it is properly isolated (e.g. using a optocoupler).

I have designed a board that you could use to isolate UART signals. You can find it [here](pcbs/isolator).

## Pictures

### Modules
I have extracted modules from the LED1650R5 (ICC-1) and the LED1837R5 (ICC-A-1).

Front of two TRÅDFRI modules:

[<img src="images/front.jpg" alt="Back of IKEA TRÅDFRI module (ICC-1)" width="384">](images/front.jpg)
[<img src="images/front2.jpg" alt="Back of IKEA TRÅDFRI module (ICC-A-1)" width="384">](images/front2.jpg)

Back of two TRÅDFRI modules:

[<img src="images/back.jpg" alt="Back of IKEA TRÅDFRI module (ICC-1)" width="384">](images/back.jpg)
[<img src="images/back2.jpg" alt="Back of IKEA TRÅDFRI module (ICC-A-1)" width="384">](images/back2.jpg)

### Test setup
My setup (the small board is a UART isolator):

[<img src="images/setup.jpg" alt="Test setup" width="384">](images/setup.jpg)

My safer setup, including debugger (LED is connected to same pin as it would in the GU10 light):

[<img src="images/setup2.jpg" alt="Safer test setup" width="384">](images/setup2.jpg)

Two soldered development boards that I use nowadays:

[<img src="images/setup3.jpg" alt="Safer test setup" width="384">](images/setup3.jpg)

## Relevant sources
I have gathered some information from the following sources:

* [IKEA Trådfri hacking](https://tradfri.blogspot.nl)
* [Trådfri: ESP8266-Lampen-Gateway](https://www.heise.de/make/artikel/Ikea-Tradfri-Anleitung-fuer-ein-ESP8266-Lampen-Gateway-3598411.html)
* [Trådfri Zigbee Light Link module](https://diystuff.nl/tradfri/tradfri-zigbee-light-link-module)
* [Trådfri Wall switch](https://wiki.iptables.dk/TR%C3%85DFRI_wall_switch)

## License
Creative Commons BY Attribution 4.0 International

## Disclaimer
This page and its content is not affiliated with IKEA of Sweden AB.

The purpose of this project is to learn and improve using reverse engineering techniques. Use this information on your own risk.
