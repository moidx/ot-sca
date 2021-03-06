# Getting Started

## Prerequisites

### Hardware Requirements

To perform side-channel attack (SCA) analysis for [OpenTitan](https://github.com/lowRISC/OpenTitan).
using the infrastructure provided in this repository, the following hardware
equipment is required:
* [ChipWhisperer CW305-A100 FPGA Board](https://rtfm.newae.com/Targets/CW305%20Artix%20FPGA/)
  This is the target board. The Xilinx Artix-7 FPGA is used to implement
  OpenTitan. Note that there are different versions of the board
  available of which the board with the bigger FPGA device (XC7A100T-2FTG256)
  is required.
* [ChipWhisperer-Lite CW1173 Capture Board](https://rtfm.newae.com/Capture/ChipWhisperer-Lite/)
  This is the capture or scope board. It is used to capture power traces of
  OpenTitan implemented on the target board. There are several different
  versions of the board available of which the single board without target
  processor but with SMA and 20-pin cables is required.
* A USB-to-SPI adapter cable. This is required to load the application binary
  into OpenTitan. We have been using the [C232HM-DDHSL-0 USB to MPSSE Cable](https://www.ftdichip.com/Support/Documents/DataSheets/Cables/DS_C232HM_MPSSE_CABLE.pdf)
  for this purpose but other similar adapters should work as well.


### Python Dependencies

This repository has a couple of Python dependencies. You can run
```console
$ pip install --user -r python_requirements.txt
```
to install those dependencies.


### Git Large File Storage (LFS)

This project uses Git LFS for storing binaries like a debian-compatible
pre-built binary of the tool for loading the OpenTitan application binary over
SPI and an example target FPGA bitstream on a remote server. The repository
itself just contains diff-able text pointers to the binaries. It is recommended
to install the `git-lfs` tool for transparently accessing the binaries.
Alternatively, they can be downloaded manually from GitHub.

You can run
```console
$ sudo apt install git-lfs
```
to install the `git-lfs` tool on your Ubuntu machine.

Alternatively, you can rebuild those binaries yourself from the
[OpenTitan](https://github.com/lowRISC/OpenTitan) repository.


### Generating OpenTitan Binaries

Instead of using the example binaries provided by this repository via Git LFS,
you can regenerate them from the [OpenTitan](https://github.com/lowRISC/OpenTitan)
repository.

To this end, follow these steps:

1. Go to the root directory of the OpenTitan repository.
1. Before generating the OpenTitan FPGA bitstream for the CW305 target board,
   you first have to run
```console
$ ./hw/top_earlgrey/util/opentitan_earlgrey_flash_size_reduce.py
```
   in order to reduce the configured flash size. Next, the boot ROM needs to
   be generated by executing
```console
$ ./meson_init.sh
$ ninja -C build-out sw/device/boot_rom/boot_rom_export_fpga_nexysvideo
```
   Finally, the bitstream generation can be started by running
```console
$ fusesoc --cores-root . run --flag=fileset_top --target=synth lowrisc:systems:top_earlgrey_cw305
```
   For more information on the build steps, refer to the
   [OpenTitan FPGA documentation](https://docs.opentitan.org/doc/ug/getting_started_fpga/).

   The generated bitstream can be found in
```
build/lowrisc_systems_top_earlgrey_cw305_0.1/synth-vivado/lowrisc_systems_top_earlgrey_cw305_0.1.bit
```
   and will be loaded to the FPGA using the ChipWhisperer Python API.

1. The OpenTitan SPI Flash tool is used to load the OpenTitan application
   binary. It can be generated by running
```console
$ ./meson_init.sh
$ ninja -C build-out sw/host/spiflash/spiflash_export
```
   For more information on the SPI Flash tool, refer to the corresponding page
   of the [OpenTitan documentation](https://docs.opentitan.org/sw/host/spiflash/README/).

   The generated binary can be found in
```
build-bin/sw/host/spiflash/spiflash
```

1. To generate the OpenTitan application binary, run
```console
$ ./meson_init.sh
$ ninja -C build-out sw/device/sca/aes_serial/aes_serial_export_fpga_nexysvideo
```
   The generated binary can be found in
```
build-bin/sw/device/sca/aes_serial/aes_serial_fpga_nexysvideo.bin
```

## Setup

### Setting up Hardware

To setup the hardware, connect the two boards as well as the adapter to
your PC via USB. Make sure the S1 jumper on the back of the target board
is set to `111` such that the FPGA bitstream can be reconfigured via USB.
The target and the capture board have further to be connected using the
ChipWhisperer 20-pin connector and an SMA cable (X4 output on target
board to MEASURE input on capture board).

The USB-to-SPI adapter cable has to be connected to the JP3 header on the
target board. This will be used to load the OpenTitan application binary. See
the following table for details on how to connect the adapter cable to the
target board.

| FTDI Signal | C232HM-DDHSL-0 Cable Color | CW305 FPGA | OpenTitan IO        |
| ----------- | -------------------------- | ---------- | ------------------- |
| TCK         | Orange                     | JP3.A14    | TCK (IO_DPS0)       |
| TDI         | Yellow                     | JP3.A13    | TDI (IO_DPS1)       |
| TDO         | Green                      | JP3.A15    | TDO (IO_DPS2)       |
| TMS         | Brown                      | JP3.B15    | TMS (IO_DPS3)       |
| GPIOL0      | Grey                       | JP3.C12    | nTRST (IO_DPS4)     |
| GPIOL1      | Purple                     | JP3.C11    | nSRST (IO_DPS5)     |
| GPIOL2      | White                      | JP3.B14    | JTAG_SEL (IO_DPS6)  |
| GPIOL3      | Blue                       | JP3.C14    | BOOTSTRAP (IO_DPS7) |
| GND         | Black                      | GND        | -                   |

In addition you might need to setup the following `udev` rules to gain
access to the three USB devices. To do so, open the file `/etc/udev/rules.d/90-lowrisc.rules`
or create it if does not yet exist and add the following content to it:

```
# NewAE Technology Inc. ChipWhisperer CW305
ACTION=="add|change", SUBSYSTEM=="usb|tty", ATTRS{idVendor}=="2b3e", ATTRS{idProduct}=="c305", MODE="0666"

# NewAE Technology Inc. ChipWhisperer-Lite CW1173
ACTION=="add|change", SUBSYSTEM=="usb|tty", ATTRS{idVendor}=="2b3e", ATTRS{idProduct}=="ace2", MODE="0666"

# Future Technology Devices International, Ltd C232HM USB to MPSSE Cable
ACTION=="add|change", SUBSYSTEM=="usb|tty", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6014", MODE="0666"
```

To activate the rules, type
```console
$ sudo udevadm control --reload
```
and then disconnect and reconnect the devices.

For more details on how to set up `udev` rules, see the corresponding section
in the [OpenTitan documentation](https://docs.opentitan.org/doc/ug/install_instructions/#xilinx-vivado).


### Configuring the Setup

The main configuration of the OpenTitan SCA analysis setup is stored in the file
```
cw/cw305/capture.yaml
```
For example, this file allows to specify the FPGA bitstream to be loaded,
the SPI Flash tool to use, and the OpenTitan application binary to execute.
By default, the prebuilt binaries delivered with this repository are used.
If you want to use custom binaries, open the file and edit the specified
file paths.

Independent of the binaries you are using, you have to adjust the serial number
of the USB-to-SPI adapter cable `dev_sn` to match your adapter. The serial
number of your adapter can be determined using the `lsusb` command.


## Performing Example SCA Attack on AES

SCA attacks are performed in two steps. First, the target device is operated
and power traces are capture. Second, the power traces are analyzed. This is
commonly referred to as the actual SCA attack as performing a different attack
on the same peripheral does not necessarily require the collection of new power
traces.

### Capture Power Traces

Make sure all boards and adapters are powered up and connected to your PC and
that you have adjusted the configuration in `cw/cw305/capture.yaml`
according to your system.

Then run the following commands:

```console
$ cd cw/cw305
$ ./simple_capture_traces.py --num-traces 5000 --plot-traces 5000
```
This script will load the OpenTitan FPGA bitstream to the target board, load
and start the application binary to the target via SPI, and then feed data in
and out of the target while capturing power traces on the capture board. It
should produce console output similar to the following output:

```console
Connecting and loading FPGA
Initializing PLL1
Running SPI flash update.
Image divided into 13 frames.
frame: 0x00000000 to offset: 0x00000000
frame: 0x00000001 to offset: 0x000003d8
frame: 0x00000002 to offset: 0x000007b0
frame: 0x00000003 to offset: 0x00000b88
frame: 0x00000004 to offset: 0x00000f60
frame: 0x00000005 to offset: 0x00001338
frame: 0x00000006 to offset: 0x00001710
frame: 0x00000007 to offset: 0x00001ae8
frame: 0x00000008 to offset: 0x00001ec0
frame: 0x00000009 to offset: 0x00002298
frame: 0x0000000a to offset: 0x00002670
frame: 0x0000000b to offset: 0x00002a48
frame: 0x8000000c to offset: 0x00002e20
Serial baud rate = 38400
Serial baud rate = 115200
Scope setup with sampling rate 100003589.0 S/s
Using key: b'2b7e151628aed2a6abf7158809cf4f3c'
Reading from FPGA using simpleserial protocol.
Checking version: 
Capturing: 100%|████████████████████████████| 5000/5000 [00:57<00:00, 86.78it/s]
```

In case you see console output like
```console
WARNING:root:Your firmware is outdated - latest is 0.20. Suggested to update firmware, as you may experience errors
See https://chipwhisperer.readthedocs.io/en/latest/api.html#firmware-update
```
you should update the firmware of one or both of the ChipWhisperer boards.
This process is straightforward and well documented online. Simply follow
the link printed on the console.

Once the power traces have been collected, a picture similar to the following
should be shown in your browser.

![](sample_traces.png)

Sample analysis run (Failed):

### Perform the Attack

To perform the attack, run the following command:

```console
$ ./simple_cpa_attack.py
```

This should produce console output similar to the output below in case
the attack fails:

```console
Performing Attack: 100%|████████████████████| 5000/5000 [03:41<00:00, 22.77it/s]
known_key: b'2b7e151628aed2a6abf7158809cf4f3c'
key guess: b'97f0e4c3bc14ff64effa5fce0697d9e0'
Subkey KGuess Correlation
 00    0x84    0.05677
 01    0xC1    0.05662
 02    0xF1    0.06032
 03    0x5F    0.05537
 04    0x7E    0.06062
 05    0xBF    0.05854
 06    0x6B    0.06673
 07    0x84    0.06018
 08    0x01    0.05967
 09    0x73    0.06864
 10    0xD7    0.05885
 11    0x92    0.05246
 12    0x18    0.06996
 13    0xA1    0.05271
 14    0x62    0.05984
 15    0xB2    0.05733

FAILED: key_guess != known_key
        0/16 bytes guessed correctly.
Saving results

```

Note that this particular attack is supposed to fail on the OpenTitan AES
implementation as the attack tries to exploit the Hamming distance leakage
of the state register in the last round, whereas the hardware does not write
the output of the last round back into the state register.
