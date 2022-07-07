.. title: Accidentally becoming a bootloader dev
.. slug: accidental-kernel
.. date: 2022-07-07 21:10:56 UTC+02:00
.. tags: programming, asm, armasm, aarch64
.. category: 
.. link: 
.. description: 
.. type: text
Figured I'd start off my first proper blogpost with the problem 
I recently found myself developing around. Over the years I've been
getting into more and more esoteric shit; I started programming
for real using some spinoff dialect of ``BASIC``, then hit it off
seriously with ``QB64`` (another, more mainstream dialect), and then
jumped into the deep end with just raw ``C++`` and then ``C`` and a
smattering of other languages.

And you know how it goes, one thing leads to another and you get
caught up in the most fucked up of shit­ [1]_ known to programming
man just because you weren't careful enough to know better. Over
time (say, over the last 18 months) I settled into a weird cycle
of going back and forth between ``C`` and assembly language, and now
I'm feverishly coding in ``Aarch64``. 

So, when a boy meets the unbridled and toxic power of a programming
'language' that is geared towards pain and bitbanging, it's inevitable
that he fall in love, and that love breeds monsters.

=================
The First Sin
=================

Anyone who's ever programmed in the kernel knows that it's a bit
of a pickle to work with. Unlike in user-space, you're very very
close to the metal, you're working with raw memory addresses,
you're fucking around and abusing the ``alloca()­`` function­ [2]_ and unwinding your
stacks manually to save three cycles on some signal return code
that your compiler foolishly tried to play it safe with. This is
all really cool stuff that you can tell your friends and they'll be
really impressed (if they're absolute geek fucks who have never seen
a titty in their lives), and you'd be the star of the evening.

After three good mouthfuls of whisky you're starting to feel
a bit doozy, and you notice the man in the corner of the room. Who
is he? How did he get here? If you had any bitches, he'd be scaring
them away right this fucking now. You don't have hoes, and the booze
has made sure you definitely have no inhibitions either, so you
come up to him and strike up a conversation.

'Hey, kiddo', he goes, 'do you want to learn about the *dark* dark
arts, or are you just forever gonna be a cuck and live in the
*high-level* lands?'

The way he says it oozes with testosterone and the smell of stale
sweat, and you just know you've sold your soul to the devil.

Calling conventions
-------------------

An operating system is no good if it can run only one process at 
any one time inside only one function. To facilitate the generation of
function intercommunication, compiler vendors figured out that they
should make it so that each individual snippet of code could
be compiled independently after the preprocessor was done with its
thing. All code should have an agreed-upon way of dealing with
the stack and registers the CPU provides, and this means that a form
of convention for calling a function came to be. A calling convention
is basically the agreed-upon way that a caller and its subroutine
deal with memory. On the calling side, the procedure has to deal with 
registers and passing arguments when calling a subroutine, how it
gets them returned from the subroutine, and so on. On the called side,
the subroutine handles receiving the args, managing the registers
that its caller did not save, and then returning the result, and all
this in a very rigid very consistent way.

Apart from there necessarily being a calling convention for every
architecture your code runs on (duh), the funny part is that there's
a calling convention for *each compiler, vendor* and *operating system*
you're compiling for. This is one of the reasons that, in the old
days of the Wild DOS West, calling a function compiled with a different
compiler than yours would mysteriously return garbage or nonsense
values [3]_ even though you did absolutely nothing wrong (as far as
you knew). One hairpull trigger for people working on Linux-Windows
interop is also calling convention mismatch: Linux kernel code uses the 
``SystemV`` calling convention expecting the order of arguments to
be ``RDI, RSI, RDX, R10, R8, R9`` and Windows uses Microsoft's x86-64
calling convention that goes ``RCX, RDX, R8, R9``, and handles 
the stack differently (every function call always wastes 32 bytes on Windows).

But calling conventions are a myth. You don't have to use them, 
you don't have to care about them, you don't have to respect who saves
what.

But how?

Assembly programming within an operating system
-----------------------------------------------

A good and polite compiler like GCC not only looks away when you shoot
yourself in the foot, it politely gives you more ammo to do it when you
run out. One of those footgun bullets is every major compiler letting you
blast raw assembly inline into your code. For example, GCC pretends you're
writing *strings* for your *assembler*, and lets you do the following shitfest: [4]_

.. code-block:: c

	int foo(char *p, int a, int b) {
	    int t1,t2;  // dummy output spill slots
	    int r1,r2;  // dummy output tmp registers
	    int res;

	    asm ("# operands: %0  %1  %2  %3  %4  %5  %6  %7  %8\n\t"
	         "imull  $123, %[b], %[res]\n\t"
	         "mov   %[res], %[spill1]\n\t"
	         "mov   %[a], %%ecx\n\t"
	         "mov   %[b], %[tmp1]\n\t"  // let the compiler allocate tmp regs, unless you need specific regs e.g. for a shift count
	         "mov   %[spill1], %[res]\n\t"
	    : [res] "=&r" (res),
	      [tmp1] "=&r" (r1), [tmp2] "=&r" (r2),  // early-clobber
	      [spill1] "=m" (t1), [spill2] "=&rm" (t2)  // allow spilling to a register if there are spare regs
	      , [p] "+&r" (p)
	      , "+m" (*(char (*)[]) p) // dummy in/output instead of memory clobber
	    : [a] "rmi" (a), [b] "rm" (b)  // a can be an immediate, but b can't
	    : "ecx"
	    );

	    return res;

	    // p unused in the rest of the function
	    // so it's really just an input to the asm,
	    // which the asm is allowed to destroy
	} 

Furthermore, and what's really the funniest jokermode part of this, is that
you can just call (well, ``jmp`` to) functions from within your assembly,
and these other functions can be other assembly, and this lets you bypass the
pesky calling convention, ignore setting up the stack and saving registers
and, the actual Apple of Eden here, gives you a taste of how life is like
when even ``C`` is a *high-level*, restrained language.

This still means you have to abide by the rules the OS has set out 
for you, since the kernel does a lot of scheduling, doesn't let you touch
memory pages that you 'should avoid' and 'are sensitive', and in general
just gets in your way. If you're a Linux bro, you can 'just' go into the
kernel, modify the guards you need or don't need, and rebuild the shit from
source. If you're in Windows land, half the things are locked up tighter
than a nun's cunt, so you're stuck with hacky injections and disassembling
Microsoft's ``.dll`` files.

The real benefits of this approach is when you're really juicing blood from
a stone and trying to optimise the last cycle out of code. Sometimes this 
does pay off [5]_ and sometimes the compiler is really, truly smarter than
you and the best case code is, in fact, the dumb shit it bleeted out. Admit
defeat.

Assembly programming deep in kernel land
----------------------------------------

Nobody does this one, and for a reason. Code this deep down gets really
nasty and convoluted (at least judging from Linux internals), and you're
doing away with hundreds of manyears of tradition and whatnot. You know
that there's no data types in assembly? It's just chains of bytes, in
an orientation you don't actually know (is it big or little endian?),
without anything to actually tell you what's what. You can just pop a float
from the stack into an integer register, and treat it like a really funky
``int``. Nobody will know, nobody will care, and you can do the Quake
square root without any typecasts in mind.

The people who live in this layer are usually the real chads of code dev.
You can tell by how few Rustaceans actually poke their head below the '''systems'''
layer. There really isn't any fun to be had here, and the main places
the Linux kernel uses inline assembly over C code is when doing syscalls,
for checking register states, doing atomic operations and all sorts of
nasty barriers that tell a CPU to stop being smart about its memory or
execution order and to do the things we told it to do in the *exact* order
we tell it to.

====================
The Second Sin
====================

But this wasn't enough. I lived a bit in assembly land but generally
avoided it because I hated recalling the exact sequence of jumping 
through hoops every time I wanted to do anything useful with my assembly.
Did you know that inline assembly from within ``C++`` used what's realistically,
but not theoretically, a different calling convention than ``C`` code, except
in the most trivial of cases? [6]_ Did you also know that the OS will also 
execute your code in a nondeterministic way? [7]_

Calling ``printf()`` was a chore, using BIOS interrupts to write text with
was a different chore (who knew that ``int 10h`` was so messy on ``x86``?),
and don't you even dare think about doing anything graphical with
modern hardware.

No, my second sin was getting a simpler piece of hardware—I got an Arduino
Due and, unfortunately, decided to play with it in assembly. Things got
really fun and I ended up absorbing more knowledge about the board specs
than I really should have (who knew that it was that easy to use clock
mismatches as a RNG that's better than the one provided by the manufacturer?).

The Due is a 'simple' board controlled by a ``ATSAM3X8E`` chip based off
the Arm Cortex-M3 core, running on the ARMv7-M instruction set. The 32-bit ARM
instruction set comes with a whole bunch of goodies that you wouldn't see
the ``x86`` be caught dead with: all instructions work with all registers
(cf. the ``x86`` trying to do ``mul``), instructions are *fixed-width* 
(meaning you can reason about code size even without assembling!), 
and conditional code does not require any jumps since practically
all the instructions have a large number of conditional variants.

But to program the Due you needed practically no fancy magic, you 
can just get the board, write absolutely tiny code (as in, your programs
will rarely pop a couple of kilobytes in ``C`` mode), flash it with
your junk code, and just power it up. Unlike the ``x86`` platform,
the startup sequence quite literally is 'start at ``0x0``, execute
code, interrupt vectors are in the words immediately after the start position'. 
There's no juggling with real, unreal or protected mode, no boot sectors,
no trying to claw your way out of 16-bit space to 32-bit.

The compiler the development suite gives you handles most of the work 
for you admittedly, but if you pilfer the headers for a few magic numbers 
and the correct sequence of memory positions to blast, you can actually 
get the Arduino going doing genuine work off an assembly file.

But getting output going beyond 'turn LED on' and input beyond
'button was pressed' was both going to be an electrical chore to wire,
and a bit expensive (screens and whatnot compatible with the
Arduino didn't come cheap at the time), so in my majestic
foolishness I decided I wanted to do Arm assembly programming
on my ``x86`` desktop. This necessitated downloading, installing,
using QEMU.

=================
The Third Sin
=================

So, QEMU is an emulator suite for a bunch of architectures, and a bunch
of computers based off them. In the Arm family it emulates a couple dozen
boards, all of them imperfectly and partially, and all of them in
a very underdocumented way: to get your bearings you have to read
things like the actual source comments (usually outdated), the RedHat
mailing list (—''—), schematics of the boards (sometimes just missing),
of the devices those boards include (...) etc.

So I decided that the next best thing would be to do raw, bare-metal
programming on the ``virt`` board—a fake QEMU platform whose components
you can pick and choose yourself, and that's guaranteed to have the best
emulation experience because it's not tied to actual hardware demands
(did you know that the Raspi boards are booted through their black-box
GPU that runs a binary blob kernel that we have no idea as to how it works?)
and you can just pick and choose parts and QEMU will try and make them work.
The ``virt`` board comes with some predetermined parts as well so you
don't have to do *all* the picking and choosing, which is cool.

And then you realise, again, that there is absolutely no documentation
for anything you want to do, that nobody's publicly written about what
they did to enable the things you want, and that there is between
'practically no' code and 'no' code out there that does what you want to do.

So the first thing I wanted to do was to write a sort of Hello World 
to see any output being displayed. This meant *any* sort of output
whatsoever from the board to the 'outside world', which for now meant
getting it to write to console.

At this point I was still oblivious.

The way that these kinds of boards communicate with the outside
world is, at its most basic level, just getting a hose of bytes and 
blasting a memory location in a specific way until something you want
to happen happens. To get the ``virt`` to shit out text, I had to
talk to the ``UART`` device which QEMU helpfully semiautomatically
maps to ``stdout`` / the console. So you go and read the documentation,
realise you're out of your depth, the links slowly turn from blue
to purple, and it takes you three days to realise what a mess it all is.

The UART is a memory-mapped input-output device (MMIO) that's mapped
to a special memory address somewhere between the flash space (``0x0``) and
start of RAM (here that's ``0x40000000``) and the MMU pretends that
it's a real memory location and not a fake cop-out redirection. So,
to get the thing going I had to:

1. figure out how the fake MMU maps devices
2. learn about the ``dtb`` (device-tree blob)
3. learn how the UART device works in the real world
4. look at UART set-up code for the Raspi3
5. translate this into the ``virt`` UART specs
6. figure out the location of the ``virt`` UART
7. ???
8. profit?

One of the first iterations of this code looked give-or-take like this
(magic numbers not included):

.. code-block:: asm

		ldr		x0,  =AUX_ENABLE
		ldr		x1,  [x0]
		and		x1,  x1, 0x1
		str		x1,  [x0]
		ldr		x0,  =AUX_MU_CNTL
		str		xzr, [x0]
		ldr		x0,  =AUX_MU_MCR
		str		xzr, [x0]
		ldr		x0,  =AUX_MU_IER 
		str		xzr, [x0]
		mov		x1,  0x3
		ldr		x0,  =AUX_MU_LCR
		str		x1,  [x0]
		mov		x1,  0xc6
		ldr		x0,  =AUX_MU_IIR
		str		x1,  [x0]
		mov		x1,  0x48
		mov		x0,  =AUX_MU_BAUD 
		str		x1,  [x0]

		ldr		x1,  =GPFSEL1
		mov		x2,  0x3F000
		and		x1,  x1, x2
		mov		x2,  #1152
		mov		x3,  #64
		mul		x2,  x2, x3  
		orr		x1,  x1, x2
		ldr		x0,  =GPFSEL1
		str		x1,  [x0]
		ldr		x0,  =GPPUD 
		str		xzr, [x0]

		mov	x2, 0xA0
		_loop_1:
			sub		x2, x2, #1
			nop
			cbnz		x2, _loop_1

		mov	x1,  0xC000
		mov x0,  =GPPUDCLK0
		str	x1,  [x0]

		mov	x2,  0xA0
		_loop_2:
			sub  x2, x2, 0x1
			nop
			cbnz x2, _loop_2

		str	xzr, [x0]
		mov	x1,  0x3
		ldr	x0,  =AUX_MU_CNTL
		str	x1,  [x0]
	
Basically, you 'have' to make sure the device
is enabled and ready to accept data from you, 
you need to map its 'pins' to MMIO addresses
the MMU exposed to you, etc etc etc. This specific
snippet crashed and burned, so I spun the wheels for like a week
and eventually settled on like 3x the amount of code
only to get the ability to blit a *single byte* at a
time at this device, passing its contents to the ``stdout``
in the dumbest and most unsafe possible way since
I think I just stopped caring about things at that point:

.. code-block:: asm

    mov x0, 0x40
    ldr x1, =AUX_MU_IO
    str x0, [x1]
    add x0, x0, #1
    str x0, [x1]
    add x0, x0, #1
    str x0, [x1]
    b.

No checking whether the device is ready to accept another byte, no
checking if it's written or has errored, no gods no kings.

I was still oblivious, yeah?

So having enabled the UART and gotten the proverbial Hello World out
(though ofc I hadn't wrtten a string printer just yet, it would've
been trivial though), I figured might as well figure out the other
devices and see how they get set up.

The first one on the list was the framebuffer. Basically, the dumbest
possible way to blit pixels onto a surface is to set up a framebuffer
device with the appropriate dimensions, pitch (how many bytes per
row of pixels), pixel format (i.e. bits per colour and order of colours)
and give it a chunk of RAM to read from to the screen.

The setup for this was even more hellish than the above, since the
documentation was actually *totally absent*, so I had to resort
to the aforementioned RedHat mailing list, and reading the source
for SeaBIOS and the U-Boot bootloader, and then read some more
QEMU source code (why does the documentation actually suck so much?)
etc etc. In short:

1. to set up the image, the framebuffer device needs to be set up
2. the way to set up the device is using QEMU's wonderfully underdocumented ``fw_cfg``
3. you need to also verify whether your CPU has something called the 'dma' via the magic number ``0x51454d5520434647``
4. it works by going through the configuration zone looking for a honest to God *string* value
5. when you find it, you then write a bunch of shit to the config object
6. specifically, you need the dimensions, format, pitch and address
7. voilà! works magically 

This took like a week I think. But then, I could just write raw bytes to 
the designated RAM space and things ~just worked~, and I got my dumb little
images to display with actually no difficulty at all. Godless magic, and I
still didn't see the error of my ways.

===============
Moment of Truth
===============

The moment where I cracked was when I tried to get
persistent storage set up, and then I realised what a fucking
fool I'd been for genuine weeks then.

The way I wanted to do storage was via a ``virtio-blk-device``
which was a simple, abstract interface device for reading and
writing to an image file for board emulator developers, and the
interface it provides is, allegedly, a very decent scheme that
gives *bootloader* coders another option for storage schemes.
As a wise man once said: 'The intent of virtio devices is to be 
implemented by hypervisors (such as QEMU). They simplify things a bit, 
so that it’s easier and more efficient for hypervisors and guests to 
communicate, without having to emulate any quirks of real hardware devices.' [8]_

It reads and writes in 512-byte sectors, is of course also
memory-mapped in its own specific funny way, and to get the device
set up you need to first *discover its secret!! location o:* by
checking more magic numbers (in this case we want ``0x74726976``)
and then the device type, and then finally you have to
actually *talk to the device* to tell it that it's been
discovered and that you have a driver for it, and then you
have to *negotiate* (yes, that's apparently the term) with it
to see what intersection of features is supported by both
your driver and the device |BroFrustration|

And this is where I broke. After a knee-deep slough through
the code in the SeaBIOS repository, *again*, I ended up
on the fucking *OSDev* bootloader page, which is where
I realised that the monkey business I've been blindly doing has,
all this time, been writing drivers in a primitive retarded
bootloader that I never wanted or needed.

And all I ever fucking wanted was a simple platform to write
some funny ``armasm`` and get haha clown results back, not this
level of convoluted hoopjumping that once again revealed how genuinely
rancid software development is once you get to talk to your devices 
on your own terms. The amount of fucky code I had to write, bin,
rewrite over the past months, and the amount of hoops I had to
jump and *will* have to jump if I want persistent storage, is
making me start to reconsider writing at least some of this
code in ``C``—but then what's the point? 

I started this
to avoid doing that, so I'm at the fucked up crossroads of
damned if I do, damned if I don't. 

----

.. [1] Specifically, here I mean I fucked up
	and got *really really* into Brainfuck
	programming and algorithmics. Yeah.

.. [2] A dear friend introduced plebian me
   to this monster of a function. So,
   malloc() pings the OS to give
   you a void pointer to memory of a
   certain size on the heap expecting you
   to behave really nicely; if malloc()
   is a function for boys, alloca()
   is a function for men: it allocates a 
   block of memory *on the stack*, and you
   can't know if there's enough room for it
   because, unlike malloc() which politely
   tells you it failed in the return value,
   a fail in alloca() just means you
   overflowed the stack and, since we're
   in kernel space, are overwriting operating
   system code or memory as we speak unopposed.
   At least user-space is polite enough to
   segfault you out of this misery.

.. [3] Specifically, this would occur when
	using a program compiled for ``Win95`` and
	calling a ``Win3.x`` function with more than
	two arguments in it; the ``Pascal`` calling
	convention that was set up by Borland
	Pascal was the convention common in ``Win3.x``
	API, and the later ``Win32`` API used
	the ``stdcall`` calling convention that
	was register-compatible and stack-compatible
	with ``Pascal``'s, but *reversed the fucking*
	*order of arguments* for some reason.

.. [4] from https://stackoverflow.com/a/48877683/

.. [5] There's this dude that managed to squeeze
	just *so* much blood out of a stone that he made
	a Fizz-Buzz_ impementation that produced 16 bytes 
	of fizz per CPU cycle, at a rate of 56 GB/s.

.. _Fizz-Buzz: https://tech.marksblogg.com/fastest-fizz-buzz.html

.. [6] Specifically, the ``C++`` compiler passes 
	a hidden pointer to ``this`` in the first available
	register, meaning that now instead of ``RDI`` your
	first argument is passed in through ``RSI`` if you're
	on Linux. 

.. [7] You can use the task scheduler as a funky RNG if
	you're feeling wicked enough.

.. [8] from https://brennan.io/2020/03/22/sos-block-device/, which
	I wish I'd read before spending three days on this in vain.

.. |BroFrustration| image:: ../emoji/brofrustration.png
  :width: 32