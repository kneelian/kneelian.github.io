.. title: Kernel Configuration, Part One
.. slug: kernel-configuration-pt-one
.. date: 2022-07-29 15:38:39 UTC+02:00
.. tags: programming, asm, armasm, aarch64, configuration
.. category: 
.. link: 
.. description: 
.. type: text

It's time to formally come out of the Hello World sphere of things. We've
set up the UART, talked to the user in several different ways, and even
cobbled together an input system for the user to communicate with the
device. We'll be expanding on I/O later, but for now I'm content with what
we've got.

Now is the time to start thinking about whether the code we're writing
can run on "all" devices in the family, and not just the exact configuration
we're running at one specific moment in time. This means that we need to check
for processor and device features, and integrate fallbacks wherever possible,
and gracefully exit if not.

Fundamentally, this means we can write code for a number of different processors
that QEMU will accept for the ``virt`` board, and expect more-or-less the same
functionality.

===============
Target Analysis
===============

Just because we're bitchin programmers doesn't mean we can just write our code
for all the processors out there and expect it to work: we're
working strictly in Aarch64 land, and this means we have actually
a very limited set of platforms we can ever dare to support without
spinning up a whole new codebase and toolkit. Apart from that, we're
coding for the ``virt`` board (or, well, "board", you know) that has
its own quirks and peculiarities that other boards don't have, and in
turn the other boards have their kinks to be ironed out (like the Pi
actually booting from the GPU and not CPU). 

Since we're writing and targetting Aarch64, the ARM CPU core designs
we can think of are:

* ARMv8.1-A: A34, A35, A53, A57, A72, A73
* ARMv8.2-A: A55, A65, A75, A76, A77, A78, X1, Neoverse N1
* ARMv8.4-A: A65AE (partial), V1
* ARMv9.0-A: A510, A710, X2, Neoverse N2

QEMU gives us a number of other options, of which only the Fujitsu ``A64FX``
core has any merit—it's an ARMv8.2-A core with the 512-bit SVE extension.

There's also the option of using the QEMU ``max`` CPU—a pseudo-core
that has the maximum featureset possible implemented in QEMU's codebase.
This does not correspond to any core, and has a number of extensions
that aren't in any of the currently emulated cores. For example, while
the Fujitsu core is emulated with 512-bit SVE, you can set your ``max``
core to work with 1024-bit SVE registers 'out of the box'.

The QEMU codebase is, of course, incomplete. Some things are just 
not emulated yet, but you can get close enough. For example, there's
no support for Apple chips, but they're running ARMv8.5-A; there's
also no support for ARMv9.0-A cores but their features are more-less
completely implemented and we're just waiting for an actual core
such as the A510 to be written.

If we are going to write truly board-independent Aarch64 code, that
means we're going to have to write peripheral detection routines
and whatnot, and that's also a pain in the ass to do compared
to just reading the device tree and pointing the CPU in the right
direction, so we're avoiding that for now as well. The UART on
a ``virt`` is in a totally different place compared to the 
*two* UARTs on a Pi3, for example, though the setup code is
much the same.

But even within our ``virt`` garden there's a number of things
that can change based on what core we're using. Today we're gonna
take a look at two such features, see how they're implemented 
or left unimplemented, and handle them in two different ways.

==============
Random Numbers
==============

The first feature I wanna take a look at is the hardware RNG
that some Arm cores have and some don't. There's a processor
feature that determines whether there's a RNG accessible
through ``mrs`` instructions, and that's ``FEAT_RNG`` that's
expressed in the top four bits of the ``ID_AA64ISAR0_EL1``
register.

Since we're gonna write a function that returns random
numbers on the stack, we're gonna want to set up a check in
the bootloader to see whether we have this feature enabled, and
if not to rewrite the function address to the alternative code
path.

The subroutine which we'll rely on to get random numbers from
is going to be called ``_rng_64``, and we'll use this same alias
for both the hardware RNG and our fallback implementation.

Somewhere in our code, we're gonna add the following stub:

.. code-block:: asm

	_rng_64_branch: .quad _rng_64_fallback
	_rng_64:
		sub sp, sp, 8
		str x0, [sp, -8]!
		adr x0, =_rng_64_branch
		ldr x0, [x0]
		br  x0
	_rng_64_hardware:
		ldr x0, [sp], 8
		ret
 	_rng_64_fallback:
		ldr x0, [sp], 8
		ret

We're going to use this stub as our jump location, and
use the boot code to change the fallback branch to the
hardware RNG branch if we have the RNG enabled.

In the boot sequence we'll then add this:

.. code-block:: asm 

	mrs x0, ID_AA64ISAR0_EL1
	lsr x0, x0, 60
	and x0, x0, 1

	cbz x0, _skip_rng_fallback

	adr x0, _rng_64_branch
	adr x1, _rng_64_hardware
	str x1, [x0]

	_skip_rng_fallback:

We're overwriting the pointer if the hardwarve feature is
enabled, since its default value is ``_rng_64_fallback``.
The hardware RNG is then used like this:

 .. code-block:: asm
	
	_rng_64_hardware:
	 mrs x0, s3_3_c2_c4_0		// rndr
	 str x0, [sp, 8]
	 ldr x0, [sp], 8
	 ret

Since we're using the same convention as with other
functions, branching into the subroutine using the ``bl``
instruction, returning arguments on the stack, here we read
the hardware register into ``x0``, store it on the
preallocated stack space, restore the ``x0`` register
and ``ret``.

----------------------

framebuffer

linker edits

kernel time

and we are gonna set up the framebuffer
here, allocating it in ram and doing all
sorts of funny shit

..code-block::
	the memory map is as follows:
		0x40000000        -- QEMU virt ram and kernel start here
		STK               -- depending on the size of the kernel
		                     the bottom of the stack can move around
		STK + 0x00100000  -- top of stack (1mb space), bottom of vectors
		STK + 0x00101000  -- top of vectors, bottom of ramfb config
		STK + 0x00101080  -- top of ramfb config (128b), bottom of ramfb itself
		STK + 0x00501080  -- top of ramfb (4mb space), bottom of heap
		STK + 0x10501080  -- top of heap
		
Keep in mind that the entire kernel
and all memory above it are all rewritable
code, and thus we must handle everything with
care in order to not rewrite ourselves.
We are still working with a flat memory model,
and there are no MMU-based safety features; the
code is fully privileged since we're still operating
in kernel mode.
The framebuffer device is basically just
a chunk of RAM we dedicate that QEMU will read
out of and blit to the screen. Framebuffers
are the simplest model of graphical output
you can get on any modern hardware, and they're
also the least powerful and tend to be performance suckers.
To configure the framebuffer, we must first communicate
with QEMU itself, by trawling the QEMU config tree
in the device's memory, figuring out if we can use the DMA
interface, then writing the configuration through
DMA to QEMU's config in big-endian order (!!), and
only then can we use the framebuffer.
Note that the framebuffer *requires* fw_cfg
to have an enabled DMA, so our first step would be
to check if DMA is enabled and, if not, to skip
framebuffer setup completely and only work with 
UART down the line.
see: https://github.com/qemu/qemu/blob/e93ded1bf6c94ab95015b33e188bc8b0b0c32670/hw/display/ramfb.c#L124
We'll do this by setting a flag if the DMA is not available,
and later if we write applications that require
the framebuffer we'll check whether it's enabled
and gracefully exit informing the user that the
functionality isn't available instead of writing to
memory that doesn't have any visible effect.

