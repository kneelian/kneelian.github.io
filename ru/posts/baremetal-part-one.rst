.. title: Baremetal Aarch64: Pt 1, Hello World
.. slug: baremetal-part-one
.. date: 2022-07-11 02:42:21 UTC+02:00
.. tags: programming, asm, armasm, aarch64, hello world
.. category: 
.. link: 
.. description: 
.. type: text

If you want to start doing *anything* at all in baremetal Aarch64
land, you need a chart and a lot of supplies, because the road is
wild and unkempt. I'll be assuming you know what the 'path' is
and how to install software (duh), but also I'll assume you can
learn the Aarch64 ISA and GNU assembler instructions and whatever
on your own; don't disappoint me. You'll probably have to adapt
these guidelines to your system; I'm working on Windows 10 so
all the filenames will be local, but you'll have no issue with the
needed changes.

=============
The Tools
=============

We'll be programming in Aarch64, the 64-bit instruction set for Arm
CPUs. Specifically, we'll be targetting some variety of Armv8-A
instruction set, the specifics of which we'll hone in on a bit down
the line. We'll be working on *totally* fake 'hardware', on an emulated
board, since this is just about the most convenient thing we can do: no 
flashing, no actual electronics work, no cost, nothing but programming.
We'll be doing this inside QEMU's emulation suite, since it does a
*pretty* good job of emulating most of the things we'll ever need. It
will provide us with peripherals we'll want to use, such as a serial
device we'll get data from on the console.

QEMU_ is easy to install, it comes with a convenient installer for
whatever platform you're on. Make absolutely sure you installed the
Aarch64 [1]_ emulator; once you've added it to your path, it should give
you something like ``qemu-system-aarch64.exe`` and its ``-aarch64w.exe``
equivalent. For now, we want the first one; the difference between
these two is that the second one doesn't open a console window, and
we want one (for now?). You can get it `straight from the website`__
there; I'm using QEMU 7 [2]_ and I suggest you use that too since
things like memory offsets might change between versions and you'll
be stuck pulling your hair out. 

.. _QEMU: https://www.qemu.org/download
__ QEMU_ 

The files you'll want from QEMU include this tiny pair:

* ``qemu-system-aarch64.exe``
* ``qemu-img.exe``

We'll also be using a cross compiler. You'll be all on your own setting
this one up. The *target triplet* [3]_ for our machine will be the fairly
specific ``aarch64-none-elf``. [4]_ You want your compiler to be able 
to emit object code for ``aarch64`` targets, to have no specific vendor 
and no OS it produces this object code for (hence the ``none``), and you
want it to produce ELF executables because ELF is really cool and awesome.

The GCC tools you'll want are:

* ``aarch64-none-elf-ld.exe``
* ``aarch64-none-elf-as.exe``
* ``aarch64-none-elf-objcopy.exe``
* ``aarch64-none-elf-gcc.exe``

You probably also want ``aarch64-none-elf-gdb.exe`` but honestly I'm
not much of a ``gdb`` fan and so far in the project I haven't really
felt a need for it.

You'll also want the following things:

* ``dtc.exe`` (Device tree compiler)
* ``make.exe`` 
* anything capable of running simple scripts (Batch, Powershell, Bash, ...)

==================
QEMU and Platform
==================

QEMU is pretty good at what it does. What it isn't good at is documentation
of pretty much anything inside it. Some parts are amazingly well documented,
others just blatantly suck. You'll have to learn to cope with that.

We'll be developing for the fakest of the emulated boards; our emu target
is going to be the ``virt``, which is a platform 'which does not correspond 
to any real hardware; it is designed for use in virtual machines. 
It is the recommended board type if you simply want to run a guest
such as Linux and do not care about reproducing the idiosyncrasies 
and limitations of a particular bit of real-world hardware', [5]_
which means it's ideal for us.

QEMU emulates just under 50 Arm-based boards which have a more or
less fixed design and component layout, but the ``virt`` lets
you pick and choose components. I appreciate that a lot, since I'm
more or less window shopping for the component that implements
what I need in the least obnoxious way (it's much easier to just
put a framebuffer down into RAM than handle a VGA interface, for
example).

Eventually, we'll fire our system up using the following incantation:
``qemu-system-aarch64.exe -M virt -cpu cortex-a57 -m 256 -kernel kernel.elf -serial mon:stdio -drive id=first,file=disk.img,format=raw,if=none -device virtio-blk-device,drive=first -device ramfb``, 
but we'll get to all of this slowly. 

There is a *lot* to QEMU, so if you want to quickly get 
overwhelmed by it, go ahead and type in ``qemu-system-aarch64.exe --help``
for a massive wall of pain. I have no idea what half of these even
are, but it's good to go through them and read whatever it gives you.

The option ``-M`` specifies which machine/board we want QEMU to emulate
for us, and so we'll tell it ``-M virt`` to give us our desired
target. The default CPU of this board is actually going to be the
Arm Cortex-A15, which is a 32-bit CPU running on Armv7, and not
what we want; so we have to change the default CPU by passing the
option ``-cpu cortex-a57`` which *is* a 64-bit CPU. [6]_

We top off our incantation by telling the emulator to give us
256 megabytes of memory (``-m 256``), that we want
to boot from a specific kernel we provide (``-kernel kernel.elf``),
that we *don't* want graphical output at all (``-nographic``)
and that we want it to redirect the virtual serial port
to standard output (``-serial stdio``). The final thing
won't look almost anything alike what I gave you up there,
but we're working towards it.

To sum up, our first incantation will be:
``qemu-system-aarch64.exe -M virt -cpu cortex-a57 -m 256 -kernel kernel.elf -nographic -serial stdio``

Congrats, you got your first QEMU error! We have no kernel to boot.

======================
Minimum
======================

First, we want to make sure that our toolchain is in order and that
we're assembling our code right. To do this, we'll need to know 
*how* to even assemble and link our code. Tough situation. Before we
even assemble our first program, the linker needs to know where in
memory to put our code, and we need to write a makefile to automate
this process. Since the ``virt`` board with our chosen CPU supports
a MMU and emulates a bunch of memory foolishnesses, we'll actually 
be putting our kernel in RAM, and so we need to know just where this
RAM exactly is.

We'll do this adding the following to our previous invocation: 
``-machine dumpdtb=out.dtb``. This will dump a device tree blob
to disk, and we'll have to convert it to a readable format using
``dtc`` that I mentioned earlier, by invoking it as 
``dtc.exe -I dtb -O dts -o out.txt out.dtb``. My own blob generates
the following device tree:

.. code-block:: 

	/dts-v1/;

	/ {
		interrupt-parent = <0x8002>;
		#size-cells = <0x02>;
		#address-cells = <0x02>;
		compatible = "linux,dummy-virt";

		psci {
			migrate = <0xc4000005>;
			cpu_on = <0xc4000003>;
			cpu_off = <0x84000002>;
			cpu_suspend = <0xc4000001>;
			method = "hvc";
			compatible = "arm,psci-1.0\0arm,psci-0.2\0arm,psci";
		};

		memory@40000000 {
			reg = <0x00 0x40000000 0x00 0x10000000>;
			device_type = "memory";
		};
		.
		.
		.

Of course, I trimmed it since the output is ~9kb long, but we have what
we need right now right at the start of the tree. We see that 
our device's memory starts at the 1GB mark (``0x40000000``), which
means this is where we'll put our kernel.

We are going to need a loader script, and I'm using this stub for
the time being:

.. code-block:: 

	ENTRY(_reset)
	SECTIONS
	{
		. = 0x40000000;
		.startup . : { kernel.o(.text.startup) }
		.text : { *(.text) }
		.data : { *(.data) }
		.bss : { *(.bss COMMON) }
		. = ALIGN(16);
		. = . + 0x100000; /* 1MB of stack atop of BSS */
		stack_top = .;
	}

Did I mention you'll have to wrangle with writing your own
linker scripts? You'll have to wrangle with writing yur own
linker scripts. The above script will suffice for now, though
we might want to expand it later. I've also taken the liberty
of naming our entry point ``_reset``; I particularly like
using underscores for label names and allcaps for constants,
but your mileage may vary. Save it as ``kernel.ld`` for now.

For our makefile, we have the most rudimentary thing known to
man for now:

.. code-block:: make

	CROSS=aarch64-none-elf-

	all: kernel.elf
		
	kernel.o: kernel.s
		$(CROSS)as.exe -g -c $< -o $@

	kernel.elf: kernel.o
		$(CROSS)ld.exe -Tkernel.ld $^ -o $@

This will produce an object file out of the assembly source,
and then use the linker script to produce an ELF executable
with the proper section offsets. 

All we have left is to write our first assembly file and
we're good to go. The most barebones stub we'll be using
for now is:

.. code-block:: asm

	.section .text.startup
	.global _Reset
	_Reset:
	    mov x0, 0x40
	    b .

Save that as ``kernel.s`` and finally run the makefile. This
should produce the file ``kernel.elf``, as we specified in the
makefile, and you almost certainly won't be able to run this
executable natively on your machine. That's alright. Run it
through QEMU (remember the error you got last time?) and it should
freeze instead of error out! This is actually *good*, because
it means the code we wrote is actually doing what we wrote (we
told the CPU to do an infinite loop).

If you want to *see* the CPU's current instructions, you can
pass a further commandline argument ``-d in_asm`` and it'll
blit out the instructions to the console as they're being executed.
For our trashfire kernel up there, it generates the following:

.. code-block::

	IN:
	0x40000000:  d2800800  movz     x0, #0x40
	0x40000004:  14000000  b        #0x40000004

	----------------
	IN:
	0x40000004:  14000000  b        #0x40000004

Which is exactly what we told it to do. If you noticed ``movz`` there
instead of plain ``mov``, that's just because there's no such
``mov`` instruction in Aarch64 to begin with, it's just an
assembler mnemonic, and ``movz`` is one of the instructions
that replaces it.

At this point, I'd make the following files:

run.bat
	``qemu-system-aarch64.exe -M virt -cpu cortex-a57 -m 256 -kernel kernel.elf -nographic -serial stdio``
run-debug.bat
	``qemu-system-aarch64.exe -M virt -cpu cortex-a57 -m 256 -kernel kernel.elf -nographic -serial stdio -d in_asm``
run-dtb.bat
	``qemu-system-aarch64.exe -M virt -cpu cortex-a57 -m 256 -machine dumpdtb=out.dtb``

These three files will make up most of my commandline use (other
than, of course, calling ``make``), and I feel that's what you'll
also need yourself for evaluation, debugs and dumps.

======================
Hello, World?
======================

If you've read the previous_ post, you'll have had it drilled into
your head that nothing comes for free in baremetal land. There is 
no kernel and no bootloader: *you* are the kernel and bootloader
and the kitchen sink, too. There is no BIOS to handle your syscalls
like you'd get in x86 territory. You will not be able to do *anything*
until you write more drivers than you really thought was possible
in a week. Paradoxically, thus, to get anything like the legendary
*Hello, World!*, the simplest of programs, working on bare metal, 
you'll have to first write a driver for whatever device gets us
serial output.

.. _previous: accidental-kernel.html

For the ``virt`` board, your first output will be passed through
the UART device [7]_ and since we redirected serial output
to the console, this means that anything the UART device passes
from the CPU will be passed to the console. This is the way
we'll handle the first print.

The UART is the simplest possible device that we can use for this
purpose. Literally all you have to do is perform a tiny setup
dance and it's ready to go blitting bytes to your console like it
means business. The specific model that the ``virt`` board implements
is the PrimeCell UART module PL011, which is also one of the UARTs
that are present on the Raspi3b. It's a dead simple device that
offers both serial and queued input and output modes, but we
have to ensure it's serial and set up for both transmission
and reception. Like actually everything on the board, it's a
memory mapped device, which means the MMU has conveniently remapped
the UART's internal registers to pretend-RAM locations.

To find out where the device got mapped to in memory, we need
to go revisit the device tree and scroll quite a bit down. Remember
we're looking for the 'PL011', and in my device tree the memory map
is like so:

.. code-block:: 

	pl011@9000000 {
		clock-names = "uartclk\0apb_pclk";
		clocks = <0x8000 0x8000>;
		interrupts = <0x00 0x01 0x04>;
		reg = <0x00 0x9000000 0x00 0x1000>;
		compatible = "arm,pl011\0arm,primecell";
	};

The device tree is telling us that our UART device, called PL011,
starts at memory location ``0x09000000`` and is 4kb (``0x1000``) long. Reads from
and writes to this area get remapped to the device. To figure
out how it works, we need to hit the specs_ and slough through a *lot*
of boring shit.

.. _specs: https://developer.arm.com/documentation/ddi0183/f/preface?lang=en

I'll sum the important bits up for you:

#. PL011 UART has a controllable FIFO buffer (that we want to disable)
#. it can generate maskable and aggregate interrupts
#. there are signal and error bits in its status/flag registers (e.g. busy bit)
#. you must disable the UART when reprogramming control registers [8]_
#. some memory areas are reserved, some are turbo reserved
#. the forbidden areas are at ``0x008``—``0x014``, and at ``0x1c``; writing to them bricks the device
#. the UART *should* be enabled manually, because you can't guarantee it's enabled on reset
#. control registers should also be set manually because their values can be unpredictable

We don't actually care about most of the functionality of
this device and some of it is actively detrimental to our
needs, so we have to deactivate them just in case.

The relevant registers of the PL011 UART:

.. raw:: html

            <div class="bs-example">
              <table style="width:50%" class="table table-striped table-bordered table-hover">
                <thead>
                  <tr>
                    <th>Register</th>
                    <th>Offset</th>
                  </tr>
                </thead>
                <tbody>
                  <tr>
                    <td><b>Data</b></td>
                    <td>+0x0</td>
                  </tr>
                  <tr>
                    <td><b>Flags</b></td>
                    <td>+0x18</td>
                  </tr>
                  <tr>
                    <td><b>Control</b></td>
                    <td>+0x30</td>
                  </tr>
                  <tr>
                    <td><b>FIFO Interrupts</b></td>
                    <td>+0x34</td>
                  </tr>
                  <tr>
                    <td><b>Interrupt clear</b></td>
                    <td>+0x44</td>
                  </tr>
                </tbody>
              </table>

With this in mind, we start by clearing the control register,
then setting the transmit mode, enabling the UART,
clearing all interrupts and disabling FIFO interrupts. Even though
the PL011 schematics say that, on reset, the value in the control
register ix ``0x0300`` (p47 of the schematic PDF), we do this
for extra safety reasons. Never hurts to be safe. The code we'll
be running:

.. code-block:: asm

	.section .text.startup
	.global _Reset
	_Reset:
	    b 1f
	    .skip 8

	//	UART_BASE: .word 0x09000000
	//	UART_DATA: .byte 0x00
	//	UART_FLAG: .byte 0x18
	//	UART_CNTL: .byte 0x30
	//	UART_FIFO: .byte 0x34
	//	UART_INTC: .byte 0x44

	.section .text
	1:
		ldr x0, =0x09000030
		// we're not actually loading these from memory
		// though we probably should; for now we just want output

		mov x1, 0x1
		str x1, [x0]		// set bit 0 = enable
		mov x1, 0x101
		str x1, [x0]

		add x0,	x0, 0x4
		mov x1, 0x03ff
		str x1, [x0]		// disable FIFO interrupts

		add x0, x0, 0x10
		str xzr,[x0]		// clear all interrupts

		sub x0, x0, 0x14
		mov x1, 0x301
		str x1, [x0]		// and finally enable the receive flag

		// now the UART should be barebones functional
		// we can test it by blasting its memory location w bytes

		mov x1, 0x48

		sub x0, x0, 0x44
		2:
		str x1, [x0]		// and we see if it works!
		// you should be seeing a wall of H on your console right about now
		b 2b

And a whole lot of bullshit later, you've got *single character* output
to the console, repeatedly! It's probably still quite buggy, you might 
need to hit a key to get output to display or whatever if it jams, but we'll 
handle that later when we start writing actual printing procedures down the line.

=================
What's next?
=================

With the UART enabled, the next part should be writing actual print
routines, figuring out how we're gonna call procedures, and getting
*input* into our machine (interactivity woo).

----

.. [1] You don't need to install the 32-bit Arm 
	emulators, since the Aarch64 emu also covers the 32-bit instruction
	set, and the Thumb set, if you ever feel like using that. We won't
	be using it, though it's a totally interesting target on its own;
	did you know that the 32-bit Arm ISA includes conditional instructions
	so that you don't even have to do any jumps in your code at all?
	You can just write ``addle`` and your ``add`` will execute only
	if the condition flag is set for less-than-or-equal. How neat! This
	didn't transfer over into Aarch64, sadly, because they doubled
	the amount of registers and did all sorts of other bonus wacky shit,
	but the instructions remained 32 bits wide.

.. [2] Specifically, the exact Win build I'm using is
	``QEMU emulator version 7.0.0 (v7.0.0-11902-g1d935f4a02-dirty)`` if
	that matters to you. You can check your version by running the
	executable with the commandline option ``--version`` fwiw.

.. [3] Obligatory reading: https://wiki.osdev.org/Target_Triplet

.. [4] I'm using ``GCC 11.2`` for this, specifically
	the ``gcc-arm-11.2-2022.02-mingw-w64-i686-aarch64-none-elf`` 
	toolchain. Any reasonably modern GCC should work, I think.

.. [5] From https://qemu-project.gitlab.io/qemu/system/arm/virt.html

.. [6] If you want to read more about the CPUs that the ``virt`` board
	supports, as well as all the intricacies of the board itself, go
	visit https://www.qemu.org/docs/master/system/arm/virt.html; but note
	that either the docs or the QEMU application itself are lying somewhere
	along the line. If you pass the options  ``-M virt -cpu help`` to the
	QEMU application, you'll find that the app is reporting support for
	processors like the old Arm Cortex-A7, the tiny Arm Cortex-M0 and
	Cortex-R5, and actually decades old ``sa1110`` aka Intel's late-90ies
	*StrongARM* microprocessors implementing the motherfucking Armv4 ISA,
	as in the thing powering the fucking *Game Boy Advance*. I don't know
	whether these are *actually* supported, if the docs are outdated,
	or whatever other explanation there is. For 'safety' purposes I'd
	much rather stick to things like the Cortex-A53, which I've been using
	to run all this code. 

.. [7] Meaning the 'Universal asynchronous receiver-transmitter' device

.. [8] Fun fact I've found actual honest to God Google code that violates
	this requirement and reprograms the UART on the fly, going against the
	obligation in the specs. See here: 
	https://github.com/google/early-bringup-tool/blob/main/examples/qemu-armv8A-64/platform/uart/uart.c