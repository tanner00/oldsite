# Part One: Booting

## Prerequisites

To build and run this code you are going to want to download FASM and QEMU. Assuming you are on an Ubuntu-based system you can simply type `sudo apt-get install qemu fasm` into your terminal to install them. I will explain how to use them as needed.

## Early days

The Basic Input/Output System (BIOS) is a piece of firmware installed onto your computer to check and initialize the hardware and also to find executable code on the hard drive. The code on the hard drive is called the Master Boot Record [^1] (MBR) and it must be exactly 512 bytes at the very beginning of the hard drive with the word [^2] 0xaa55 at offset 510. The BIOS will load our code at address 0x7c00 and jump to it.

```assembly
use16
org 0x7c00

jmp $

times (510 - ($ - $$)) db 0
dw 0xaa55
```

This very simple code will merely boot and loop forever. The first line of code reads `use16` because your x86 CPU starts up in something called 16-bit Real Mode. Staying 16-bit Real Mode is undesirable because using it means we can only use 1 MiB (+ 64 KiB) of memory and can't use any hardware-based memory protection to stop processes from reading from and writing to other processes memory. Through the release of the [80286](https://en.wikipedia.org/wiki/Intel_80286]), Intel provided Protected Mode which was later enhanced by [80386](https://en.wikipedia.org/wiki/Intel_80386) to have 32-bit addresses and hardware-based memory protection. Before we switch to this mode, it's necessary to load the rest of our OS from the hard drive because the MBR is limited to 512 bytes. We must do this before we switch to Protected Mode because the BIOS functions used to load the OS will not be available in 32-bit Protected Mode. There are multiple BIOS functions you could use to load sectors [^3], but I have chosen to use the "Extended Read" function. It is called by configuring registers and memory according to the chart below.

#### Int 0x13

|Register|Value|
|--------|:---:|
|AH      |0x42 |
|DL      |0x80 for the first HDD (set by the BIOS)|
|DS:SI   |segment:offset address of the Disk Address Packet|

#### Disk Address Packet
| Bytes      | Use                                                 |
|:----------:|-----------------------------------------------------|
| 0x00       | Size of this packet                                 |
| 0x01       | Must be set to 0                                    |
| 0x02..0x03 | # of sectors to read                                |
| 0x04..0x07 | segment:offset address of where to read the sectors |
| 0x08..0x0f | [LBA](https://en.wikipedia.org/wiki/Logical_block_addressing) of the starting sector, first sector of the drive is 0|

[^1]: The Master Boot Record should also store information on how the partitions of the hard drive are organized. Our OS's code will ignore this for the time being.
[^2]: A word refers to 16-bits of contigious memory.
[^3]: Hard drive speak for 512 contigious bytes.







