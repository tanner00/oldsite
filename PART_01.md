# Part One: Booting

## Prerequisites

To build and run this code you are going to want to download [FASM] and QEMU. Assuming you are on an Ubuntu-based system you can simply type `sudo apt-get install qemu fasm` into your terminal to install them. I will explain how to use them as needed.

## Early days

The Basic Input/Output System (BIOS) is a piece of firmware installed onto your computer to check and initialize the hardware and also to find executable code on the hard drive. The code on the hard drive is called the Master Boot Record [^1] (MBR) and it must be exactly 512 bytes at the very beginning of the hard drive with the word [^2] 0xaa55 at offset 510. The BIOS will load our code at address 0x7c00 and jump to it.

```nasm
use16
org 0x7c00

jmp $

times (510 - ($ - $$)) db 0
dw 0xaa55
```

##### How do I build and run it? (I suggest you automate this by creating a [Makefile](https://www.gnu.org/software/make/manual/html_node/index.html#Top))
```bash
# Don't clutter your code!
mkdir build
# Compile it
fasm boot.asm build/boot.bin
# This will be important later when we start to paste separate binary files together
# bs=512 sets the sector size
dd if=build/boot.bin of=build/kernel.bin bs=512
# We'll add things to this later for configuring how things will be displayed, how to receive output from serial ports for debugging, and how much memory to supply the OS.
# The -drive option is to prevent QEMU from complaining about guessing the file format.
qemu-system-i386 -drive format=raw,file=build/kernel.bin
```

This very simple code will merely boot and loop forever. The first line of code reads `use16` because your x86 CPU starts up in something called 16-bit Real Mode. Staying 16-bit Real Mode is undesirable because using it means we can only use 1 MiB (+ 64 KiB) of memory and can't use any hardware-based memory protection to stop processes from reading from and writing to other processes memory. Through the release of the [80286](https://en.wikipedia.org/wiki/Intel_80286]), Intel provided Protected Mode which was later enhanced by [80386](https://en.wikipedia.org/wiki/Intel_80386) to have 32-bit addresses and hardware-based memory protection. Before we switch to this mode, it's necessary to load the rest of our OS from the hard drive because the MBR is limited to 512 bytes. We must do this before we switch to Protected Mode because the BIOS functions used to load the OS will not be available in 32-bit Protected Mode. There are multiple BIOS functions you could use to load sectors [^3], but I have chosen to use the "Extended Read" function. It is called by configuring registers and memory according to the chart below.

### Int 0x13

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

#### Return Values

| Register         | Meaning      |
|------------------|--------------|
| CF               | Set on error |
| AH               | Error code (values [here](http://www.delorie.com/djgpp/doc/rbinter/it/34/2.html))|

A few caveats to this function are that it can't cross a 64 KiB boundary and some BIOSes will only read up to 127 sectors per int 0x13 call. The call cannot correctly read past a 64 KiB boundary due to the Segmentation in 16-bit Real Mode as explained in [Appendix A](https://todo.com) the offset part of the "segment:offset" address would overflow as it is just 16 bits [^4]. The "127 sectors" restriction is a shortcoming of some [Phoenix](https://en.wikipedia.org/wiki/Phoenix_Technologies) BIOSes.

#### Commented code for loading 63 sectors

```nasm
use16
org 0x7c00

;; Interrupts are not necessary here and must be disabled before entering
;; Protected Mode (to come later).
cli

;; DS is 0 because disk_address_packet's address is somewhere between
;; 0x7c00..0x7e00 which is in segment 0.
xor ax, ax
mov ds, ax
mov si, disk_address_packet

mov ah, 0x42
int 0x13

mov si, int13_error_msg
jc error_and_die

jmp $

error_and_die:
	;; lodsb reads value at address in si to register al
	;; it increments (or decrements if the direction flag is set)
	;; the address by 1 for each call to lodsb
	lodsb
	;; al = 0 signifies the end of the string
	test al, al
	jz @f

	;; int 0x10 is for video services.
	;; ah=0xe selects the function for writing characters to the
	;; screen al is the character to write to the screen which is
	;; loaded by lodsb.
	mov ah, 0xe
	int 0x10
	jmp error_and_die
@@:
	;; Loop forever
	cli
	hlt
	jmp @b

disk_address_packet:
	;; Size of this packet
	db 16
	;; Must be 0
	db 0
	;; # sectors to load.
	;; This value will most likely change as the kernel grows larger, but
	;; it is chosen now because it will not cause the read to pass over a
	;; 64 KiB boundary and the binary of the kernel will be exactly 32 KiB
	;; (63 * 512 (bytes per sector) + 512 (MBR).
	dw 63
	;; segment:offset of where to place the sectors
	dd 0x7e00
	;; LBA of the starting sector to read. The MBR is at LBA 0 and the rest
	;; of the code starts at LBA 1.
	dq 1

int13_error_msg: db 'Extended Read Failure... Halting', 0

times (510 - ($ - $$)) db 0
dw 0xaa55
```

If you run this code with your current commands to build and run the OS, your code will hit problems at instruction "jc error_and_die". This is because QEMU is trying to load the sectors you requested, but there weren't enough sectors to read in the file you supplied (build/kernel.bin). The solution to this I'm using currently is to pad the kernel.bin file with zeros so QEMU has 63 sectors of data to read. A more sophisticated solution would be to write a program which determines the size of the kernel and make modifications to the disk_address_packet sectors field dynamically so as to not read unnecessary sectors.

#### Changes to the build process

```bash
# Write zeros for 64 (+1 for MBR) sectors
dd if=/dev/zero of=build/kernel.bin bs=512 count=64
# conv=notrunc prevents dd from truncating the file to size 0 before it does the write.
dd if=build/boot.bin of=build/kernel.bin bs=512 conv=notrunc
```

### A20 Line

```nasm
;; Set the second bit of port 0x92 to enable the A20 line
in al, 0x92
;; Only write when necessary
test al, 10b
jnz already_enabled
;; Enable the second bit
or al, 10b
;; [Apparently](http://www.win.tue.nl/~aeb/linux/kbd/A20.html) sometimes first
;; bit will cause a reset
and al, 11111110b
;; Solidify enabling the A20 line
out 0x92, al
already_enabled:
```

[^1]: The Master Boot Record should also store information on how the partitions of the hard drive are organized. Our OS's code will ignore this for the time being.
[^2]: A word refers to 16-bits of contiguous memory.
[^3]: Hard drive speak for 512 contiguous bytes.
[^4]: 16 bits can hold a maximum value of 0xffff. This is equivalent to 64 KiB.
