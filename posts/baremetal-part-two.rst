.. title: Baremetal Aarch64: Pt 2, Hello World Advanced
.. slug: baremetal-part-two
.. date: 2022-07-25 17:17:01 UTC+02:00
.. tags: programming, asm, armasm, aarch64, hello world
.. category: 
.. link: 
.. description: 
.. type: text

So we set up the UART device and we are going to use it as
the main way to talk to the outside world for now. The UART
is dead simple to set up and use, especially when compared
with writing a driver for more conventional communication
methods. Apart from console output, which we tested last time,
we're also going to use it as our (unbuffered) input method.
I'm not gonna go through the effort of writing a PS/2 or 
USB stack for a Hello World kind of deal, UART is sufficient.

==================
Cleaning up
==================

`Last time`_ I cut some corners in that I hardcoded the
UART memory address but now we're gonna properly load it from a
label that we can conveniently change if, somewhere down the
line, the UART base changes or if something else happens that
we gotta take care of. The start of label 1 will now be:

.. _Last time: baremetal-part-one.html

.. code-block:: asm

		UART_BASE: .word 0x09000000
	//	UART_DATA: .byte 0x00
	//	UART_FLAG: .byte 0x18
	//	UART_CNTL: .byte 0x30
	//	UART_FIFO: .byte 0x34
	//	UART_INTC: .byte 0x44

		.section .text
		1:
			adrp x0, UART_BASE
			add  x0, x0, :lo12:UART_BASE
			ldr  w0, [x0]
			
			add  x0, x0, 0x30		// UART_CNTL 

Not much has changed, except we're now loading the address.
You'll note that it's a federal fucking issue to load from
memory in Aarch64. We're first loading the page address of
where the ``UART_BASE`` label is located into ``x0`` via the
``adrp`` instruction; this loads the higher 56 bits of the 
address into the register, which then forces us to add the 
offset separately using ``add`` and the ``:lo12:(name)``
specifier, where the assembler helpfully extracts the lower 
12 bits (hence the name) of the label for us and generates 
a normal ``add`` instruction with the offset as an immediate.

Two pits I fell into while playing with this:

	#. sometimes the assembler fails to error on unaligned access and will joyfully read instructions one byte off
	#. ``ldr x0, [x0]`` was a big headache since reading from a word into a dword register reads the word *twice*, once in low and once in high bits

The second one I keep forgetting about every now and then, 
I'm 100% sure I'll encounter it again. It's not mentioned in
the official online docs, and I don't even see it mentioned in 
the 5200 (!) page architecture reference manual, though that
might be my own search fuckup.

When you ``make`` and run this, it should do the same thing
as before. We're not gonna load the offsets from memory because
they're tiny (meaning it's much more of a chore to load them and
add them to the base) [1]_ and, more importantly, they're guaranteed
to never change relative to base and each other. We're also going
to set UART up only once and won't change anything in the 
control register in the foreseeable future. 

==================================
UART Safety, Printing and You!
==================================

The PL011 UART is equipped with a flag register that can tell us
what the UART is doing right now and how it's going, so we can
poll it to see whether it's ready to take a byte from us to print
or not. From a quick look at the docs we see that we need to
test for whether ``UARTFR & 0x28`` returns true and wait until
it isn't. 

We'll also want to compartmentalise the print into a procedure
or function. This is going to mean we'll have to think about
how to pass arguments to functions, since our ``putchar()``
lookalike will consume a single byte that its caller will pass
into it. Remember that spiel about calling conventions?

We'll call the routine ``_uputc``, standing for "U(ART)-put-c(har)".

We can write the function out in high-level pseudocode to see 
exactly what's going to have to happen:

.. code-block:: c 

	void _uputc(byte x)
	{
		REGISTER  a;
		REGISTER* b = &UARTDR;

		b += 0x18;

		do
		{
			a = *b;
		} while (a & 0x28);

		b -= 0x18;

		*b = x;

		return;
	}

This immediately tells us we need three registers to store
the UART flags (``a``), the UART address (``b``), and the
argument to send to the UART data register (``x``).

Aarch64 specifics
~~~~~~~~~~~~~~~~~~

Though massively simpler than the clusterfuck that is x86,
Aarch64 has a lot of very unfunny specifics that are 
not documented in any user-friendly way. Just the Arm A-profile
architecture reference manual is (as of right now) 11 530
A4 pages, and going through dozens of thousands of pages
looking for relevant info is not the easiest thing, especially
if you have to repeat it.

We're working with a more sophisticated Arm processor than the
average embedded board has. This means there are significant
memory and exception safety features that can be toggled on
or off, and you can do about the same things in a Cortex-A that
you can with x86 privilege rings. There are four rings in
the standard, and not all devices need to implement all the
levels—a CPU needs to have at least ``EL0`` and ``EL1``. 
The higher the number, the more privilege we have executing
code and shit.  We'll be staying in Exception Level One (``EL1``),
which is the highest QEMU provides normally. [2]_

One of the main funnies here is that Aarch64 really
wants to enforce *stack alignment* to sixteen bytes, and if
you try to access the ``SP`` while it's unaligned you will,
in most cases, brick your CPU for the time being since
you can't recover from exceptions. To make sure this
doesn't happen, you can either always align to 16 bytes, or
disable this. We want to disable memory alignment fault
exceptions in general, and we do this by playing with the
``SCTLR_EL1`` system register.

The instruction that reads from a system reg is ``mrs`` and
the one that writes to a system reg is ``msr``.
Each takes one named register and one numbered
general-purpose register as arguments, in a destination-source
order. 

To disable these fault exceptions, we will use:

.. code-block:: asm

	mrs x0, SCTLR_EL1
	mov x1, 0x1a
	neg x1, x1
	and x0, x0, x1
	msr SCTLR_EL1, x0

This disables unaligned stack access faults, and
register load/store misalignment faults, which makes
working with the stack and with the heap quite a bit less
headache-inducing.

Unaligned stack access faults are especially annoying
because this funny processor feature means that storing
a single register on stack means you unknowingly get to
brick the CPU very very easily. Programmer wisdom and
developer references from the likes of Apple and wise
guys from StackOverflow even tell you that you *must*
keep the stack aligned and that there's no way around
this, which is totally untrue. I don't know who thought
this was a good idea.

----

We can't use the stack out of the box, though. Upon
cold reset, the value of the SP *should* be zero, and
on a warm reset it's 'an architecturally UNKNOWN value'
as per the manual. This means that you just gotta set
the stack up yourself every time you boot—though you
have to do that in x86 as well so it's no big deal.

Configuring the stack is not really that Big of a Deal.
If you recall our linker script, we allocated an extra 
``0x100000`` bytes above our ``.data`` and ``.bss`` sections
and assigned that to ``stack_top``. To set the stack pointer up
we need to load this location's address into memory, then
add four (since we will predecrement on push and postincrement
on pop) to ensure we push to the very top of the space. This
new value we'll then load into the ``sp`` register, and the rest
is handled by the assembler. The code is as simple as it gets:

.. code-block:: asm 

	    ldr x0, =stack_top
	    add x0, x0, 0x4
    	mov sp, x0

We gave the kernel stack 1MB of memory to do what it
wants with it. This is a relatively crude approach, but it gets
the job done for now.

Though for now it doesn't matter, we have to be careful
with memory management way down the line. Because we're not
using the MMU to its full capacity, our memory model is essentially
flat as far as code and data are concerned. Aside from the memory-mapped
devices below the ``0x40000000`` address, where the device
has mapped all the fancy stuff like UARTs and whatever,
the MMU gave us the ``memory`` device (RAM) starting at
address ``0x40000000`` and going on for ``0x10000000`` bytes
(configured through the commandline), and the ``flash`` device
starting at ``0x0`` and ending at ``0x04000000``, which should
store ROM but *we're not using it*. Our program code
and data are all in the same flat memory space, with no
mem management or segmentation, which means our code is
writable, which means pushing too much onto the stack will mean
actually overwriting the kernel.

There's several ways of solving this, such as putting the stack below
the kernel or at the top of RAM growing downwards into the heap, but
neither of those are necessary right now really, because we won't
be getting a megabyte of stack exhausted any time soon anyway.  


Back to ``_uputc(byte x)``
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Fact is that, if we want to do any kind of compartmentalisation
and shunting code off into subroutines, we have to be aware
that we're now responsible for making a generic subroutine that
will work independently of what called it, and that the code
doesn't disrupt its caller's operation in any way.

When you're generating a bunch of assembly automatically, 
this is where calling conventions come in handy: the compiler
has a list of what it must, can and cannot do when getting two
chunks of code to interact. 

Generally, calling conventions prioritise passing data through
the register bank, since that's much faster than using memory
for data transfers. When you run out of registers, you usually
have to *spill* data onto the stack, and getting data to and from
the stack is a lot slower than just working with RAM. Sometimes
you see this in professional code (the Go compiler didn't know 
about register transfer until like 2020 I think), but generally
you spill only when you have to.

This convention starts mattering a little bit less when you're
hand-writing both the callers and the subroutines: you can tune
the data transfer mechanism per pair very accurately to reflect
your needs in a way that compilers (still?) can't. Perhaps
this one function will use registers ``x7`` through ``x11``, while
another will want ``q0`` through ``q3`` because of their larger
size.

The one constant is that the subroutine will always have to be
called in a specific way. If you want to change how it's called,
you have to either write boilerplate that maps this new
call to what the subroutine understands, or write a new
routine wholesale. 

This is a sacrifice that you *can* make very
easily, and the way this works out for you is much like how C
does it: you don't get overloading, name-mangling and resolution
like in C++, and you'll have to give separate names to routines
with the same functionality but different parameter schemes.

On the other hand, we can just pass things in and out of
routines using the stack. This simplifies the register saving
dance at the expense of losing at least a few cycles here and
there. Accessing cache is pretty fast, but still slower than
accessing registers (I feel it's a factor of 4 kind of deal),
so we gotta make it worth it.

Luckily for us, Aarch64 gives us an unusually meaty powerful tool 
here. Despite actually lacking dedicated push and pop instructions
that are around in 32-bit Arm, Aarch64 lets us load and store
registers in pairs in a single instruction. We're thus gonna
be using ``ldp`` and ``stp`` (and their variants where necessary)
for this, in addition to using ``ldr`` and ``str`` for the generic
register storage option.

The load and store pair instructions work on all the registers,
both integers (e.g. ``stp x1, x2, [sp, #-16]!``) and vectors/floats
(so, ``ldp q1, q2, [sp], #32``). This lets us transfer up to 256
bits of memory in one go (assuming your CPU uses 128-bit NEON and
not the bigger SVE registers that can go beyond 1024 bits), and
not lose (m)any cycles doing it if the chunk is in cache. [3]_

So for this subroutine, we'll be saving the trashed registers
to the stack and getting the argument from the stack. 

Let's revisit the pseudocode:

.. code-block:: c 

	void _uputc(byte x)
	{
		REGISTER  a;
		REGISTER* b = &UARTDR;

		b += 0x18;

		do
		{
			a = *b;
		} while (a & 0x28);

		b -= 0x18;

		*b = x;

		return;
	}

For this, we'll obviously need two registers to do
the logic work for us, and one to store our argument.
Since we have to save three registers, stack alignment
has to be off—but we've luckily disabled it.

In our caller code we'll have:

.. code-block:: asm

	mov x17, 0x48        // the character 'H'
	str x17, [sp, -8]!   // our output argument
	bl _uputc

Essentially, it doesn't matter which register we get
the argument from, so that the subroutine will always
see the argument on the stack and the caller won't
have to bend over backwards to get the argument
into a specific register.

The subroutine will then look like this:

.. code-block:: asm

	// our print function
	// takes 1 arg on stack, returns 0
	// trashes 3 registers
	_uputc:
		stp  x0, x1, [sp, -16]!
		str  x2, [sp, -8]!
		
		ldr  x2, [sp, 24]    // this is where we first pushed 
		                     // the argument in the caller
		
		adrp x0, UART_BASE
		add  x0, x0, :lo12:UART_BASE
		ldr  w0, [x0]
				// now x0 has the UART_BASE location
		
		add  x0, x0, 0x18    // UART_FLAG address

		_uputc_loop1:
			ldr  x1, [x0]          // read from UART_FLAG
			and  x1, x1, 0xff      // 0010 1000 = busy & transmit full
			bic  x1, x1, 0xc0      // but we gotta do it the long way
			bic  x1, x1, 0x10
			bic  x1, x1, 0x07      
		cbnz _uputc_loop1

		sub  x0, x0, 0x18      // back to BASE / DATA
		str  x2, [x0]

		ldr  x2, [sp], 8
		ldp  x0, x1, [sp], 16

		ret

You'll note that Aarch64 is especially retarded when it
comes to immediates for bitwise operations, in that they have
to be a mask with a bit pattern of some sort, and not actual
immediates like in arithmetic operations. The block of four
logical ops basically performs the check ``and x1, x1, 0x28``
but that will not fit into the immediate field here. The
``bic`` instruction is basically a form of ``a & ~b`` that's
convenient for toggling some specific bits off.

We can actually do some optimisation here and save
one register, though the stack will still be unaligned:

.. code-block:: asm

	// our print function
	// takes 1 arg on stack, returns 0
	// trashes 2 registers
	_uputc:
		stp  x0, x1, [sp, -16]!
		
		adrp x0, UART_BASE
		add  x0, x0, :lo12:UART_BASE
		ldr  w0, [x0]
				// now x0 has the UART_BASE location
		
		add  x0, x0, 0x18    // UART_FLAG address

		_uputc_loop1:
			ldr  x1, [x0]          // read from UART_FLAG
			and  x1, x1, 0xff      // 0010 1000 = busy & transmit full
			bic  x1, x1, 0xc0      // but we gotta do it the long way
			bic  x1, x1, 0x10
			bic  x1, x1, 0x07
		cbnz _uputc_loop1

		sub  x0, x0, 0x18      // back to BASE / DATA
		ldr  x1, [sp, 16]
		str  x1, [x0]

		ldp  x0, x1, [sp], 16
		add  sp, sp, 8         // pop the argument off the stack

		ret

An optimisation assembly affords us that we can't
intentionally get in C is the ability to destructively
reuse registers when their data's lifetime's up. A half-decent
C compiler would do this optimisation for you automatically,
but this isn't something you can specify through the language
itself.

=========================
Nested function calls
=========================

A peculiarity of the Aarch64 ISA is that the ``bl`` instruction
lets us do branches to subroutines without any memory access,
which is extremely good. The way it works is that it stores
the address of the instruction it's accessing + 4 into the
register ``x30``, then jumping to the label we give it.
Correspondingly, we return from the subroutine we jumped
to using the ``ret`` instruction, which jumps to the address
stored in ``x30`` (i.e. it's an alias).

Predictably, this means that you can only ``bl`` exactly once
before you trash your original return address.

This pair of instructions has its advantages over the x86
``call`` and ``ret`` pair—no stack access is performed and
things are kept fast, but you have to write the nesting code yourself
and that gets kind of nasty kind of quickly.

We can thus absolutely avoid having to use the stack for
subroutine calls that are one-deep, which saves on memory
at the expense of losing a register.

The next function we'll write will have to call another function
and so we'll have to write nesting code.

Extending into ``_uputs(byte* x)``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Printing one byte at a time is definitely something we *can*
but *shouldn't* do. It's inconvenient on its own, really. To
exemplify function call nesting and to clear up this mess, we
will write a ``_uputs`` function—that is, "U(ART)-put-s(tring)".
The function will take one argument, a zero-terminated C-style
string of bytes from memory, and print it out byte by byte until
we reach zero.

We'll try to minimalise memory accesses as much as possible,
which means doing funny shit with registers. For example, the following
pseudocode would be a good scheme to implement:

.. code-block:: c 

	void _uputs(byte* x)
	{
		REGISTER  a = *x;
		REGISTER  b;

		do
		{
			b = a & 0xff;
			_uputc(b);
			a = a >> 8;
			if(a == 0)
			{
				x += 8;
				a = *x;
			}
		} while(b != 0x00);

		return;
	}


We'll be reading memory in 8-byte chunks
at a time, meaning we'll need to do only one
access per 8 characters (remember, the UART is
an 8-bit interface). If we haven't
reached a zero character, we call the ``_uputc``
and let it handle the print on its own, abstracted
away from our eyes.

In assembly, we have some more things to think of,
such as storing the ``x30`` on the stack alongside
the other registers in use. We *don't* have to
think about how long our string is, though.

Reading 64-bit chunks of memory at once could be
"potentially dangerous" i.e. we could be leaking
secret memory or whatever if someone checks
register states when they're not supposed to, 
but since the memory model is flat and unprotected,
there won't be any *hardware* faults to think about
too hard.

Our code should now look like this:

.. code-block:: asm

	// handler for string printing
	// takes 1 arg on stack, returns 0
	// trashes 4 registers
	_uputs:
		stp x0, x1,  [sp, -16]!
		stp x2, x30, [sp, -16]!
		ldr x0, [sp, 32]    // the string address	
		_uputs_loop1:
			ldr x1, [x0]                  // the initial memory read
			cbz x1, _uputs_loop1_end
			and x2, x1, 0xff              // extract byte
			cbz x2, _uputs_loop1_end
			str x2, [sp, -8]!
			bl  _uputc 

			asr x1, x1, 8
			and x2, x1, 0xff
			cbz x2, _uputs_loop1_end
			str x2, [sp, -8]!
			bl  _uputc
			
			asr x1, x1, 8
			and x2, x1, 0xff
			cbz x2, _uputs_loop1_end
			str x2, [sp, -8]!
			bl  _uputc
			
			asr x1, x1, 8
			and x2, x1, 0xff
			cbz x2, _uputs_loop1_end
			str x2, [sp, -8]!
			bl  _uputc
			
			asr x1, x1, 8
			and x2, x1, 0xff
			cbz x2, _uputs_loop1_end
			str x2, [sp, -8]!
			bl  _uputc
			
			asr x1, x1, 8
			and x2, x1, 0xff
			cbz x2, _uputs_loop1_end
			str x2, [sp, -8]!
			bl  _uputc
			
			asr x1, x1, 8
			and x2, x1, 0xff
			cbz x2, _uputs_loop1_end
			str x2, [sp, -8]!
			bl  _uputc
			
			asr x1, x1, 8
			and x2, x1, 0xff
			cbz x2, _uputs_loop1_end
			str x2, [sp, -8]!
			bl  _uputc
			
			add x0, x0, 8             // shift pointer by 8, and loop
			b _uputs_loop1
		_uputs_loop1_end:
		ldp x2, x30, [sp], 16
		ldp x0, x1,  [sp], 16
		add sp, sp, 8
	ret

Instead of reading a byte at a time, we're
reading eight, and then doing a call on each
byte of the register at a time. This reduces
the number of memory accesses by 7 (every eighth
step instead of every step of the loop), which
is helpful when ``_uputc`` already does
at least 6 memory ops (and potentially more) every single call.
I even unrolled a loop manually to save 
a register that would otherwise have
gone to waste as a counter. You don't need
to do this, but it saves us a few instructions
and, despite being longer code, will run in fewer 
cycles because we avoided two more stack accesses.

This function is called more or less the same
as the previous one, except now you need
to get the address instead of just passing the argument
raw on the stack:

.. code-block:: asm

	EXAMPLE_STRING: .asciz "Hello, world! "
		.align 8

		...

	ldr x2, =EXAMPLE_STRING
	str x2, [sp, -8]!
	bl _uputs

----

You can avoid this nested call by writing the
``_uputs()`` function so that it itself
writes to UART without having to call a
subroutine to do that work. This would both
be more efficient and go faster (think of it
like inlining a function manually), but I feel
like it suffers from readability. 

Ultimately, we're not squeezing cycles from a
dry stone, there's much more we could do to
optimise these things that we aren't doing because
the tradeoffs are too severe.

==================
Extra features
==================

There's a few more things we can do to make
the environment a bit more fully-featured. For
example, the floating-point unit is by default disabled
on boot, so we have to enable it by fiddling with 
the built-in system control registers:

.. code-block:: asm

	mov x0, (0x3 << 20)

	// msr cptr_el3, xzr
	// msr cptr_el2, xzr
	msr cpacr_el1, x1

We don't have EL3 and EL2 support in our CPU, it only
goes up to EL1, but I'm including the higher EL code
anyway for your convenience for when you play with
real hardware. 

In QEMU, you can now type ``info registers`` to see
a whole new table of vector regs has just appeared
and is initialised to zero.

One other interesting feature that the Aarch64 platform
provides in Cortex-A CPUs, mostly the newer ones, is the
ability to generate random numbers in the hardware.

To see whether you have RNG capability or not you're
supposed to read from the ``ID_AA64ISAR0_EL1`` register.
If ``mrs x10, ID_AA64ISAR0_EL1; and x10, x10, 0x1000000000000000``
returns true, your CPU has the registers ``RNDR`` and ``RNDRRS``
implemented. The CPU will then provide you with a 
random number when you do ``mrs x10, RNDR``—or at least, it 
theoretically should!

But |BroFrustration| the GNU ``binutils`` don't even fucking work,
they don't recognise some register names and error out for
*no fucking reason*, so you have to actually poll that register
using its internal encoding name, which in this case is going to be
``mrs x10, s3_3_c2_c4_0``. This should provide you with a 
random number in ``x10``, which QEMU in turn sources from the OS.

----

.. [1] To load them you need to use up at least one more register
	and will need three more instructions; you're looking at something
	at least like ``adrp x2, UART_CNTL; add x2, x2, :lo12:UART_CNTL; 
	ldrb x2, [x2]; add x0, x0, x2`` while you could just do
	``add x0, x0, 0x30`` instead.

.. [2] You can check your exception level by doing ``msr x1, CurrentEL``
	to get the value and ``lsr x1, x1, 2`` to extract it since it's not
	in the lowest two bits. There's a whole bunch of these unmapped
	internal registers on Cortex-A CPUs, and you can read more about
	them here: https://developer.arm.com/documentation/ddi0595/2021-12/AArch64-Registers

.. [3] There's a
	further trick with the vector instructions, which is to use
	the ``st1`` and ``ld1`` instructions, to load up to four vector
	registers (512 bits!) from memory at once (``ld1 {v0.2d, v1.2d, v2.2d, v3.2d}, [sp, #-64]!``),
	or even the SVE/SVE2 junk with ``addvl sp, sp, #-4; str z1, [sp, #4, MUL VL]`` which
	can net you 1024 bits of storage at once,
	but that's quite overkill for us right now. According to the manual
	the ``ld1`` of four NEON regs takes 8 cycles, and the ``ldp`` of two NEON regs takes
	6 cycles, which means you can save 4 cycles if you use the other kind of store. This kind of
	microoptimisation is not necessary when working in QEMU, but remember that usually
	the less code you write, the better your code will be.

.. |BroFrustration| image:: ../emoji/brofrustration.png
  :width: 32