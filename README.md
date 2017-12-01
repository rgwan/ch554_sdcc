# SDCC toolchain for CH554 devices [![Build Status](https://travis-ci.org/Blinkinlabs/ch554_sdcc.svg?branch=master)](https://travis-ci.org/Blinkinlabs/ch554_sdcc)

This is a port of the CH554 SDK, from Keil C51 to SDCC.

# Setup instructions

## Get the toolchain: Windows

You'll need a recent version of SDCC, as well as mingw for make, and likely also git-bash for the bash shell.

[git for windows](https://github.com/git-for-windows/git/releases/download/v2.15.1.windows.2/Git-2.15.1.2-64-bit.exe)
[SDCC 3.6.0](https://sourceforge.net/projects/sdcc/files/latest/download?source=files)
[mingw installer](https://downloads.sourceforge.net/project/mingw/Installer/mingw-get-setup.exe)

TODO: How to set up the enviroment to find these bits automatically

## Get the toolchain: Linux

For Debian-based systems, this should work:

	sudo apt-install build-essential sdcc

The ISP tool is a little more troublesome. It's Windows only, but might work in wine (untested). A better strategy would be to re-implement it.

## Get the toolchain: macOs

You'll need xcode (for make), as well as SDCC

## Build the examples

Then clone this repository, and build the examples:

	git clone https://github.com/Blinkinlabs/ch554_sdcc.git
	cd ch554_sdcc/examples
	make

If everything is set up correctly, all of the examples should now be built.

Use the 'WCHISPTool' to flash an image onto the target device.

## Port a file from Keil C51 syntax to SDCC

The syntax for the two compilers are slightly incompatible. Notable issues from the SDK are:

* SFR and SBIT defines are different
* Absolute addressing format is different
* SDCC doesn't automatically track absolute-addressed variables
* SDCC can use standard types like 'uint8_t', the C51 examples used non-standard defines like 'UINT8'
* SDCC is little endian, while C51 appears to be big endian

This project includes a simplistic python script that can automatically translate some simple grammer changes. It can be used like this:

	python tools/c51_to_sdcc.py [source] [destination]

With any luck it should do 90% of the translation work for you.

## Create a new example

Create a new directory in the examples folder, with the name of the new example:

	cd examples/
	mkdir fastblink
	cd fastblink

Add a Makefile that referes to the master template makefile:

	vi Makefile

With these contents:

	TARGET = fastblink
	
	S_FILES = \
        	main.c
	
	include ../Makefile.include

Change the definition of target to match the new example name.

Next, add a barebones main file:

	vi main.c

With these contents:

	// Blink an LED connected to pin 1.7

	#include <ch554.h>
	#include <debug.h>

	#define LED_PIN 7
	SBIT(LED, 0x90, LED_PIN);

	void main() {

		// Configure pin 1.6 as GPIO output
		P1_DIR_PU &= 0x0C;
		P1_MOD_OC = P1_MOD_OC & ~(1<<LED_PIN);
		P1_DIR_PU = P1_DIR_PU |     (1<<LED_PIN);

		while (1) {
			mDelaymS(10);
			LED = !LED;
		}
	}

And compile:

	make

Oops! There is a problem:

	$ make
	sdcc -c -V -mmcs51 --model-small --xram-size 0x0400 --xram-loc 0x0000 --code-size 0x37FF -I../../include -DFREQ_SYS=12000000 main.c
	+ /usr/bin/sdcpp -nostdinc -Wall -I../../include -DFREQ_SYS=12000000 -obj-ext=.rel -D__SDCC_MODEL_SMALL -D__SDCC_FLOAT_REENT -D__SDCC=3_5_0 -DSDCC=350 -D__SDCC_REVISION=9253 -D__SDCC_mcs51 -D__STDC_NO_COMPLEX__ -D__STDC_NO_THREADS__ -D__STDC_NO_ATOMICS__ -D__STDC_NO_VLA__ -isystem /usr/bin/../share/sdcc/include/mcs51 -isystem /usr/share/sdcc/include/mcs51 -isystem /usr/bin/../share/sdcc/include -isystem /usr/share/sdcc/include  main.c 
	+ /usr/bin/sdas8051 -plosgffw main.rel main.asm
	sdcc main.rel -V -mmcs51 --model-small --xram-size 0x0400 --xram-loc 0x0000 --code-size 0x37FF -I../../include -DFREQ_SYS=12000000 -o blink.ihx
	+ /usr/bin/sdld -nf blink.lk

	?ASlink-Warning-Undefined Global '_CfgFsys' referenced by module 'main'

	?ASlink-Warning-Undefined Global '_mDelaymS' referenced by module 'main'
	+ /usr/bin/sdld -nf blink.lk returned errorcode 512
	../Makefile.include:38: recipe for target 'blink.ihx' failed
	make: *** [blink.ihx] Error 1

Right, we forgot to add the debug.c source file to the Makefile. Update the Makefile so that it looks like this:

	TARGET = fastblink

	C_FILES = \
	        main.c \
        	../../include/debug.c

	include ../Makefile.include

And re-run make. Everything should be fine:

	sdcc -c -V -mmcs51 --model-small --xram-size 0x0400 --xram-loc 0x0000 --code-size 0x37FF -I../../include -DFREQ_SYS=12000000 ../../include/debug.c
	+ /usr/bin/sdcpp -nostdinc -Wall -I../../include -DFREQ_SYS=12000000 -obj-ext=.rel -D__SDCC_MODEL_SMALL -D__SDCC_FLOAT_REENT -D__SDCC=3_5_0 -DSDCC=350 -D__SDCC_REVISION=9253 -D__SDCC_mcs51 -D__STDC_NO_COMPLEX__ -D__STDC_NO_THREADS__ -D__STDC_NO_ATOMICS__ -D__STDC_NO_VLA__ -isystem /usr/bin/../share/sdcc/include/mcs51 -isystem /usr/share/sdcc/include/mcs51 -isystem /usr/bin/../share/sdcc/include -isystem /usr/share/sdcc/include  ../../include/debug.c 
../../include/debug.c:225: warning 158: overflow in implicit constant conversion
	+ /usr/bin/sdas8051 -plosgffw debug.rel debug.asm
	sdcc main.rel debug.rel -V -mmcs51 --model-small --xram-size 0x0400 --xram-loc 0x0000 --code-size 0x37FF -I../../include -DFREQ_SYS=12000000 -o fastblink.ihx
	+ /usr/bin/sdld -nf fastblink.lk
	packihx fastblink.ihx > fastblink.hex
	packihx: read 31 lines, wrote 48: OK.

The .hex file can now be loaded onto the target using WCHISPTOOL.

## Build configuration variables

The build configuration is specified in the master Makefile.include file, however some variables can be overridden by the local Makefile:

| Makefile variable | Description |
| --- | --- |
| TARGET | Example name, used to name the .hex file |
| C_FILES | List of c files to include in the example build |
| FREQ_SYS | System clock frequency. Default is 12000000 (12MHz). See 'include/debug.c' for a list of accepted values |
| XRAM_SIZE | Size of the non-reserved XRAM. Update to reserve a portion of the XRAM for absolute variables, such as for the USB DMA pointer |
| XRAM_LOC | Starting position of the non-reserved XRAM. Update to reserve a portion of the XRAM for absolute variables, such as for the USB DMA pointer |
| STDIO_UART | Set to '0' to use UART0 for STDIO, or '1' to use UART1 for STDIO (not finished) |

# Status

Here is a list of the different peripheral drivers and examples that need to be ported

| Peripheral | Description | Status |
| --- | --- | --- |
| ADC | Analog-to-digital converter | not started |
| DataFlash | DataFlash (EEPROM) peripheral | not started |
| GPIO | I/O peripheral example | not started |
| UART0/stdlib | stdio example using UART0 | not started |
| UART1/stdlib | stdio example using UART1 | not started |
| Watchdog | Watchdog timer configuration | not started |
| IAP | Jump from user program to the bootloader | not started |
| PWM | Pulse Width modulation peripheral | complete |
| SPI | Serial Peripheral Interface | not started |
| Timer | 8051-style Timers 0 and 1 | not started |
| Timer2 | Extended Timer 2 | not started |
| TouchKey | Capacitive touch peripheral | in progress |
| Type-C | USB C power negotiation peripheral | not started |
| USB\Device | USB device peripheral: HID (?) profile | not started |
| S_CDC | USB device peripheral: CDC profile | complete |
| U_DISK | USB device peripheral: USB mass storage device profile | not started |
| Compound_Dev | USB device peripheral: compound device example (?) | not started |
| USB\Host | USB host peripheral: hub (?) example | not started |
| USB\U_DISK | USB host peripheral: read/write USB mass storage device | not started |

# Examples

## blink

Blink an LED using a software timer

Note: the delay() routines run slow in SDCC, and needs to re-tuned.

## pwm

Use the PWM hardware to generate PWM output

## pwm_interrupt

Use the PWM hardware as a periodic interrupt source

## touchkey

Exercise the capacitive touch hardware

## usb_device_cdc

Emulate a USB-to-Serial converter, as a USB-CDC device.

## ws2812

Use GPIO to bitbang a WS2812 LED pattern

