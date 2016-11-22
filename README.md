# UF2 Bootloader

This repository contains a bootloader, derived from Atmel's SAM-BA,
which in addition to the USB CDC (serial) protocol, also supports
the USB MSC (mass storage).

## UF2 

UF2 (USB Flashing Format) is a name of a file format, that is particularly 
suitable for flashing devices over MSC devices. The file consists
of 512 byte blocks, each of which is self-contained and independent
of others.

Each 512 byte block consist of (see `uf2format.h` for details):
* magic numbers at the beginning and at the end
* address where the data should be flashed
* size of data
* data (up to 476 bytes; for SAMD it's 256 bytes so it's easy to flash in one go)

Thus, it's really easy for the microcontroller to recognize a block of
a UF2 file is written and immediately write it to flash.

In `uf2conv.c` you can find a small converter from `.bin` to `.uf2`.

## Features

* USB CDC (Serial emulation) monitor mode compatible with Arduino 
  (including XYZ commands) and BOSSA flashing tool
* USB MSC interface for writing UF2 files
* reading of the contests of the flash as an UF2 file via USB MSC
* UART Serial (real serial wire) monitor mode (typically disabled due to space constraints)
* In-memory logging for debugging - use the `logs` target to extract the logs using `openocd`
* double-tap reset to stay in the bootloader mode
* automatic reset after UF2 file is written

## Board identification

Configuration files for board `foo` is in `boards/foo/board_config.h`. You can
build it with `make BOARD=foo`. You can also create `Makefile.user` file with `BOARD=foo`
to change the default.

The board configuration specifies the USB vendor/product name and ID,
as well as the volume label (main thing that the operating systems show).

There is also `BOARD_ID`, which is meant to be machine-readable and specific
to a given version of board hardware. The programming environment might use
this to suggest packages to be imported (i.e., a package for a particular
external flash chip, SD card etc.).

These configuration values can be read from `INFO_UF2.TXT` file.
Presence of this file can be tested to see if the board supports `UF2` flashing,
while contest, particularly `Board-ID` field, can be used for feature detection.

The current flash contents of the board is exposed as `CURRENT.UF2` file.
This file includes the bootloader address space. The last word of bootloader
space points to the string holding the `INFO_UF2.TXT` file, so it can be parsed
by a programming environment to determine which board does the `.UF2` file comes from.

## Handover

When the user space application implements the USB MSC protocol, it's possible to
handover execution to the bootloader in the middle of MSC file transfer,
when the application detects that a UF2 block is written.

Details are being finalized.

## Bootloader update

The bootloader will never write to its own flash area directly.
However, the user code can write there.
Thus, to update the bootloader, one can ship a user-space program,
that contains the new version of the bootloader and copies it to the
appropriate place in flash.

Such a program is generated during build in files `self-uf2-bootloader.bin`
and `self-uf2-bootloader.uf2`.

## Fuses

The SAMD21 supports a BOOTPROT fuse, which write-protects the flash area of
the bootloader. Changes to this fuse only take effect after device reset.

This fuse is currently not utilized by this bootloader. It needs to be investigated.


## Build

### Requirements

* `make` and an Unix environment
* `arm-none-eabi-gcc` in the path (the one coming with Yotta will do just fine)
* `openocd` - you can use the one coming with Arduino (after your install the M0 board support)

Atmel Studio is not supported.

You will need a board with `openocd` support.

Arduino Zero (or M0 Pro) will work just fine as it has an integrated USB EDBG
port. You need to connect both USB ports to your machine to debug - one is for
flashing and getting logs, the other is for the exposed MSC interface.

Otherwise, you can use other SAMD21 board and an external `openocd` compatible
debugger.

`openocd` will flash 16k, meaning the beginning of user program (if any) will
be overwritten with `0xff`. This also means that after fresh flashing of bootloader
no double-tap reset is necessary, as the bootloader will not try to start application
at `0xffffffff`.

### Configuration

There is a number of configuration parameters at the top of `uf2.h` file.
Adjust them to your liking.

By default, you cannot enable all the features, as the bootloader would exceed 
the 8k allocated to it by Arduino etc. It will assert on startup that it's not bigger
than 8k. Also, the linker script will not allow it.

Two typical configurations are:

* USB CDC and MSC, plus flash reading via FAT; UART disabled; logging optional; **recommended**
* USB CDC and MSC, no flash reading via FAT; UART enabled; logging disabled; only this one if you need the UART support in bootloader for whatever reason

The bootloader sits at 0x00000000, and the application starts at 0x00002000.


## License

The original SAM-BA bootloader is licensed under BSD-like license from Atmel.

The new code is licensed under MIT.
