.. title: Baremetal Aarch64: Pt 3, Hello World Ultimate
.. slug: baremetal-part-three
.. date: 2022-07-26 22:32:54 UTC+02:00
.. tags: programming, asm, armasm, aarch64, hello world
.. category: 
.. link: 
.. description: 
.. type: text

Last time around I said I was gonna do input but then
totally forgot about it |BroFrustration| so today's gonna
be a short one on getting input via UART into the
machine and then making it dance like a little
monkey for us.

===============
UART revisited
===============

The UART interface, like we covered, is dead simple. As
with our output code, our input code is going to be
fully blocking on the CPU until the UART flags change.
Unlike the output, this means that we're going to be
eating CPU time until the user provides with input, which
could take a while. Cheap, nonblocking IO would take
much more of a stack than we have going for us.

Our function will return a single character from input,
and take no arguments. We will need to check if the 
``RXFE`` flag is set (that is, that the receive stack
is empty) on the UART, and wait until it's cleared.

Thankfully, the UART is buffered, so we don't need to
time our reads from the data register to the exact moment
the user writes something.

.. code-block:: asm

	// our input function
	// takes 0 arg on stack, returns 1
	// trashes 2 registers
	_ugetc:
		str  xzr, [sp, -8]!
		stp  x0, x1, [sp, -16]!
		
		adrp x0, UART_BASE
		add  x0, x0, :lo12:UART_BASE
		ldr  w0, [x0]
				// now x0 has the UART_BASE location
		
		add  x0, x0, 0x18    // UART_FLAG address

		_ugetc_loop1:
			ldr  x1, [x0]
			and  x1, x1, 0x10
			cbnz x1, _ugetc_loop1

		sub  x0, x0, 0x18
		ldr  x0, [x0]
		str  x0, [sp, 16]
		ldp  x0, x1, [sp, 16]!
		ret

We reserved a place on the stack for the return argument,
then stored the return there, while popping everything else.

Some fun code to play around with in the main code path:

.. code-block:: asm

	PLEASE_WRITE:   .asciz "Please input a key and I'll do my best to repeat it and tell you if it's odd or even: "

	3:
		ldr x2, =EXAMPLE_STRING
		str x2, [sp, -8]!
		bl _uputs
			// and Hello World, finally!

		ldr x2, =PLEASE_WRITE
		str x2, [sp, -8]!
		bl _uputs

		_parity_loop:
			bl  _ugetc
			ldr x2, [sp]
			and x2, x2, 0xff

			sub x2, x2, 0x40
			cbz x2, 4f
			add x2, x2, 0x40

			bl  _uputc
			
			sub x2, x2, 0x30
			and x2, x2, 0x1
			cbz x2, _is_even
		
				_is_odd:
				mov x2, 0x4f
				b _parity_loop_end

				_is_even:
				mov x2, 0x45

			_parity_loop_end:
			str x2, [sp, -8]!
			bl _uputc
			b  _parity_loop

	4:
		add x2, x2, 0x40
		str x2, [sp, -8]!
		bl _uputc
		b .

This program tests your input whether it's even or odd,
prints E or O depending on the case, and halts when it
encounters ``0x40``—that is, ``@``.

To get the input working correctly, I had to pipe it
to a serial device, and use PuTTY to link to the other side.
If your input isn't working, it's probably due to that.
My own experience here is that the code is somehow extremely
sluggish; there's up to a full second of delay between
my input and the device's response, and I'm not sure
if that's a quirk of using virtual COM ports and PuTTY
to interface with QEMU or a failing of my code. I might
want to write some sort of device driver for input later
on to alleviate that and avoid serial TTY, so we'll see.

.. |BroFrustration| image:: ../emoji/brofrustration.png
  :width: 32