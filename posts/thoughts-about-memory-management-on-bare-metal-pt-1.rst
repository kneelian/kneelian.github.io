.. title: Thoughts about Memory Management on Bare Metal, pt. 1
.. slug: thoughts-about-memory-management-on-bare-metal-pt-1
.. date: 2025-04-07 16:10:30 UTC+02:00
.. tags: programming, baremetal, c, c++, esp32, cyd
.. category: 
.. link: 
.. description: 
.. type: text

========
Prelude
========

Recently I've been toying with some weird, cheap SBCs that offer
a lot for way too little money. You can actually get a device with
a pressure-sensitive 320x240 screen, 4M flash and 512K RAM for
under ten bucks |Moggers| so obviously what I did was that I did
get one of those and decided to see what I could do with it.

Enter the **Cheap Yellow Display** (or CYD for short), name courtesy of 
`Brian Lough <https://github.com/witnessmenow/ESP32-Cheap-Yellow-Display/tree/main>`_,
that's got one of those fancy Espressiv IoT SOCs, with a Bluetooth stack
and WiFi out of the box, Arduino integration, two cores (one of which
will run the RTOS in an Arduino setup by default, but you can hijack it
and run other code on it), easy interrupt registering (with per-core
control to boot), a decent amount of GPIO pins broken out, and a
resistive LCD touchscreen that's trivial to program for.

Its weakest points are its 520K RAM and 4M flash. This is probably
the one thing that can limit its usability in general, and there are other
ESP32 chips and boards with different configs (8M and 16M flash variants exist,
and many boards also integrate an extra 4/8M of PSRAM, absent on the CYD),
but this is already more than enough for a *lot* of things you'd want to do.
If you're crazy enough, `you can even run Linux on it`_
(though the device in the link has 8M of PSRAM, you can probably trim it and make a godawful,
pained, miserable Linux with 512K as well, and it boots just "fine"),
but we're not going down this route because that just feels like hubris:
yes, the old UNIXes did run on hardware with 64-256K RAM, but that was more
than 50 years ago and those machines ate around 9 amps of current at
230V (or at least that's what I'm getting for the PDP-11/20 power
supply); the CYD works at a steady 3.3V/115mA, and is a quintillion times
faster [1]_ with a much saner work environment and tools that don't
make you want to immediately [2]_ kill yourself, and you can replace
it with basically infinite ease if one dies. Try replacing a PDP-11
on a budget.

.. _you can even run Linux on it: https://community.element14.com/challenges-projects/element14-presents/project-videos/w/documents/28339/episode-623-how-to-run-linux-on-an-esp32

Some good things about the lack of PSRAM, though, is that it
|ss| teaches you to use what you have and not be a pussy |se| has
much slower access times and as such you'll want to fall back on fast
internal RAM anyhow; PSRAM would be good for memory you'd access intermittently
rather than hit every cycle. Or maybe this is all just cope. In any case,
520K RAM it is.

It's also a bit more muddied than that: the ESP32 divides its memory space
into two blocks, DRAM for data and IRAM for instructions. These are
asymmetric: DRAM covers some 320K and IRAM covers 200K. `The docs`_ helpfully
state that "[a]ny internal SRAM which is not used for Instruction RAM will 
be made available as DRAM (Data RAM) for static data and dynamic allocation (heap)", 
which means that if we need >328K data we can claw it back from IRAM, which
helps our case a bit. The ability to execute things from RAM can also help with
hot-loading code from external memory (I've looked a lot into this and concluded
that just writing a parser for external code and then interpreting it is much,
*much* less painful than hotloading code into IRAM, and we're currently in the middle
of me musing how to implement that properly), so you can actually just drive a slim
OS in RAM and load programs off an SD card, and `one project <https://tactility.one/>`_ does
just that [3]_ with quite a bit of charm and polish to it.

The thing is, the ESP32 is powerful enough to run a RTOS (in this case it
bundles FreeRTOS [4]_ by default and uses it for a bunch of shit), and this
RTOS reserves some memory for itself, and the WiFi and BT stacks can also
eat into this memory for their own purposes (this eats up a whole 64K of RAM
in case you're wondering). A default setup with Arduino
enabling the use of the ``Serial`` library without any screen code 
has the non-user code reserve 20080 bytes for global variables,
"leaving 307600 bytes for local variables". The astute reader will see
that this sums up to 327680B, i.e. 320K of RAM. A decent amount
of this is going to the RTOS task scheduler and internal states, and
a bit is going on facilitating dynamic memory allocation. `The docs`_ also
tell us that there's a technical limitation to how much we can allocate in
what way: we're limited to 160K static allocation, and the remaining 160K
"can only be allocated at runtime as heap".

Loss to overhead isn't that bad, though, truth be told. A more full-featured
programme with a screen and touch driver and SD card code for both reading
and writing takes up 22560B, just under 2.5K more. This is not that awful
considering how much RAM we have, but it'd be proper deadly on a smaller
device like one of the actual Arduinos that *have* 2.5K RAM in total.

But this all also means that, if we want to use a lot of RAM in our thing,
we have to dynamically allocate more when we start touching the >128K zone.
Dynamic memory allocation on microcontrollers *sucks*, though, and you'll
fragment the memory space in basically no time and won't be able to recover
from it because there's no actual OS to bail you out. People die when an MCU
upredictably returns ``nullptr`` from a ``malloc()`` call.

One of the solutions is to just allocate everything you can statically
ahead of time, and then 'dynamically' allocate during startup and then
*never clean up or call* ``free()`` *again*. The fact that ``malloc()`` can
only give us blocks up to the size of the largest contiguous memory
block and the fact that we're not doing any sophisticated memory mapping
on the fly, this really just means figuring out how big our maximum
memory blocks are and then allocating up to their size and then writing
management code ourselves on top of that. Several tactics for this 
include `memory pools`_ and `slab allocation`_, both of which are
pretty neat ideas, and we'll be tapping into their concepts for our
purposes. Let's dive in.

.. _memory pools: https://en.wikipedia.org/wiki/Memory_pool

.. _slab allocation: https://en.wikipedia.org/wiki/Slab_allocation

===========
Diving In
===========

There's something charming about the CYD. Like I've mentioned above, it's
a dual-core system with 520K RAM and whatnot. The ISA the chips are running
can be called *unusual*: it's called Xtensa and was developed by the IP company
Tensilica. The exact architecture here is the Xtensa LX6 configured to use
32-bit transfers/data buses, and is little-endian. The CYD has a core *without* vector
extensions, but there are Tensilica cores with SIMD (the ESP32-S3 is one
such example, with some `weird SIMD features`_ that are genuine nice-to-haves).
The chip offers some good features, a large number of pins broken out, 
several serials, JTAG, all the good stuff. The CYD cuts into this a decent
bit by integrating an SD card module, multicoloured LED, a screen, a touch
interface for that screen, a micro-USB port, and a buzzer/speaker 
two-wire/mono plug. The result is a still-decent package with 3 clusters of
4 pins exposed, one of which has a 3.3V line, and one of which doubles
as a serial port.

.. _weird SIMD features: https://bitbanksoftware.blogspot.com/2024/01/surprise-esp32-s3-has-few-simd.html

These memory investigations are basically me building up some infrastructure
to support an interpreted programming language running behind the scenes on
the CYD, as a sort of native BASIC-like. There is no practical use to this,
it's just me playing with my toys.

Given that the system has (at least) 320K of SRAM dedicated to data,
this is obviously a good place to start. As no more than 160K can be
statically allocated, and a decent amount of it will be eaten up
by libraries and other code, it becomes "natural" to me to think about
the CYD's RAM in terms of 64K blocks. This might be a problem [5]_ later, 
but currently it seems fine enough. 

The screen itself will also want at least 75K of RAM (320 * 240 = 76800, so
if every pixel is one byte that's a clean 75K; though colour performance
might suffer obviously), and possibly more (the screen has 16-bit colour, so
150K RAM, eating up almost half of DRAM in one go). If I buffer the screen and
expose it as a memory zone, that is; this isn't required in any way, and is anyway not
something I care about right now.

In any case, it would be *prudent* to allocate as much memory as possible for
the application, then manage it manually later, writing an own allocator and deallocator.
It's also perfectly possible to write a kind of "memory mapper" that would provide
a flat contiguous memory space to user code obfuscating the bullshit underneath. One 
pseudocode example of such a a remapper would be:

.. code-block:: cpp

	#include <vector>

	struct memory
	{
		std::array< std::array<uint8_t, 64 * 1024>, 4> contents;

		memory()
		{ 
			for(auto i : contents) 
				i = new std::array<uint8_t, 64 * 1024>;
		}
		~memory() = delete; // must be static

		uint8_t read(uint32_t address)
		{
			uint8_t  page = (address >> 16) & 0xff;
			uint16_t cell =  address & 0xff'ff;

			return contents[page%4]->at(cell);
		}
		void write(uint32_t address, uint8_t payload)
		{
			uint8_t  page = (address >> 16) & 0xff;
			uint16_t cell =  address & 0xff'ff;

			contents[page%4]->at(cell) = payload;
		}
	};

This kind of structure has sub-32 bytes of internal state, manages pages of 64K memory,
and translates accesses in a perceived continuous space to accesses in a discontinuous space.
Of course, this kind of code is unacceptable for embedded because using the STL in the first
place is what is called "rough luck", but it's not the worst thing ever, and the code scaffold
is good enough. This is downright primitive, but it serves the purpose of unifying a disparate
memory space into one field. A lot more can be added on top of this to make it a proper
manager unit that would provide virtual memory with memory safety features. This isn't
currently too important.

This feels like the wrong solution for specific problems, though. Some other things I've toyed
with include:

* Hash-maps
* Slabs and pools (aforementioned)
* Naïve constrained declaration and no dynamicity
* other detritus and miscellanea

The hash-map was probably my favourite idea, though.

---------------------------
Static hashing and hashmaps
---------------------------

A hashmap is a really neat data structure: it's an associative array where input keys
are used to look up locations in memory easily. These keys are put through a hashing
function, transforming them into a hash index or hash code, and this hash will point 
to where the memory is stored in the corresponding target array. Since with fixed-width
input data hashing will 'always' take the same amount of operations, most hashmap designs
will feature amortised constant ``O(1)`` access, though in practice this isn't really the case.

Since a typical hashmap will not run in an environment where each possible key has a unique
location corresponding to it, some kind of hash colision mitigation is unavoidable (thus
the average hashing process is not an injective operation), and since allocating the entire
space is also frequently uneconomic reallocation can end up being necessary at many points (and, 
remember, reallocation and dynamic memory stuff is undesirable in bare metal).

The issue here is that we aren't really allowed to grow and shrink at will, and should aim to
take up as much memory as possible; that is, the initial storage allocation should aim to
give the subsystem the maximum amount of memory it would need during its whole runtime. This
becomes the memory ceiling of the datastructure. The hashmap, then, would either have to be
statically allocated in the flash binary, or will have to be ``malloc()``'d at startup and
never freed. 

This still leaves the question of colisions open.

Let's add more context. As I mentioned before, I started thinking about all of these things
in the context of developing a BASIC-like running on the ESP32, and so some structures and
requirements tend to emerge spontaneously without me having to do much deciding. In a typical
BASIC, variables are stored onto a stack or on a table and then every time they're referenced
the interpreter trawls this structure (``O(n)`` as a function of table size) to find the variable,
and if it's not present it pushes it onto the structure (e.g. in `Microsoft BASIC for the 6502`_ you
can use variables without declaring them, which automatically implicitly declares them there). This
trawl is repeated every time a variable is referenced, absolutely nuking performance. It's not like
it's much of a space-saving thing, either: MS-BASIC used a weird 5-byte format for floating point
data (fine), as well as integers (what) that were treated as 16-bit numbers (what?) with 
24 padding bits (`what!?`), plus two bytes for the name, meaning that each variable took up
seven bytes of space. 

Arrays, as far as I know, need 4+2n bytes of bookkeeping on top of
their actual memory footprint: 1 for the name, 2 for the address, 1 for the number of dimensions, 
and 2n for the length of the dimensions for n dimensions [6]_ give or take.
As you'll probably recognise, arrays go onto the heap of the BASIC, but it's important 
to note that once you ``DIM``/declare an array, it can't be deallocated or resized (for details, see `here`_),
which is a recurring theme in the low-power space. More modern BASICs like the beautiful
QB64 allow `resizing and reallocating arrays`_ as enabled by running on top of a fat OS
that can handle moving a lot of memory around for them [7]_ without having to
think too much about it themselves.

While for arrays this trawl is pretty 'cheap' (it's still ``O(n)`` but with n ≤ 255), for
variables it can get nasty pretty quick (realistically you can input up to 26*36 = 936 variables,
but the interpreter can be reading bytecode that would use 'syntactically inappropriate' variable
names going into thousands of combinations; a theoretical maximum of around 4000 variables based
on memory constraints is an example of such a large number). This is a big performance sink, but
solutions to the problem are tough to come up with. I believe that a statically allocated hashmap
is in fact a solution to this.

.. _Microsoft BASIC for the 6502: https://github.com/mist64/msbasic/blob/master/var.s

.. _here: https://www.c64-wiki.com/wiki/DIM

.. _resizing and reallocating arrays: https://qb64phoenix.com/qb64wiki/index.php/REDIM

Simply using a 16-bit key, it's possible to effortlessly index into a 64K table without any
further adjustments. This would mean that a 2-character name would just point to the address
in the code table without requiring any bookkeeping; this would be an example of a 'perfect
hashing table'. My first musings, thus, involved using 2 CJK characters as possible 
variable names. Encoded as UTF-16 and then somewhat adjusted, this means that both 
are ∈ ``[0x0000, 0x6bff]`` which, when concatenated together, produces a ~30-bit key, which
would somehow have to be mapped onto a 64K table in some way; every single possible key 
would have to compete with ~16K other keys for its slot in the table. 

The naïve way to resolve this would be to add the top and bottom hanzi together. This is
awful, because it'll make a lot of small collisions easy: let's say that we name one variable
女王 ('queen', ``0x5973'738B`` - the sum of the two hanzi is ``0xCCFE``); it will immediately collide
with 女王 ('princess', ``0x738B'0x5973``). This extends to all anagrams, of which there is
a large amount in e.g. Japanese (本日 'today' vs. 日本 'Japan' is another neat example). While
it's tempting to tell the user to Never Ever use anagram variable names, I felt that I needed
to mitigate this in some way. So, addition is not a good hashing function.

What I did was use a `hash prospector`_ to find good 16-bit hashes and incorporate that
into the thing. It spat out quite a few functions; what I settled on was the following:

.. code-block:: cpp

	// bias = 0.0049752954186705221
	uint16_t hash1(uint16_t x)
	{
	    x ^= x >> 11;
	    x *= 0x104bU;
	    x ^= x >> 9;
	    x *= 0xcd1bU;
	    x ^= x >> 7;
	    x *= 0x42a5U;
	    x ^= x >> 11;
	    x *= 0x258dU;
	    x ^= x >> 9;
	    return x;
	}

	// bias = 0.0046809801814303642
	uint16_t hash2(uint16_t x)
	{
	    x ^= x >> 7;
	    x *= 0xb257U;
	    x ^= x >> 9;
	    x *= 0x8829U;
	    x ^= x >> 4;
	    x *= 0x0af7U;
	    x ^= x >> 12;
	    x *= 0x504dU;
	    x ^= x >> 6;
	    return x;
	}

Bias here represents how strongly correlated toggling a bit of input is with specific
bits of output (low hash, meaning less correlation, is good). These two are chained together, operating
on separate halves of the variable name; a simplified code example:

.. code-block:: cpp

	uint16_t hash(uint32_t input)
	{
		uint16_t a, b, c;
		a = hash1(input & 0xff'ff);
		b = hash2(input >> 16);
		c = hash1(a ^ b);

		return c;
	}

This provides surprisingly decent hash performance. Using a collision measurement script I wrote, I
found that for a table of size `n`, there is on average fewer than `n/2` collision events, and that
around half of that is a single collision; iteratively adding mitigations shows that after around 50%
population, the hash table starts getting more and more collisions, and that beyond around 80% population,
the table practically becomes unusable performance-wise; insertion changes from ``O(1)`` in a theoretically
optimal table to ``O(n)`` in a static one. Average-case performance is still pretty good, but worst-case
performance isn't good at all. Still, this beats iterating through a table of variables to extract the
named one. 

.. _hash prospector: https://github.com/skeeto/hash-prospector

To mitigate collisions during insertion, I simply opted for the following simple algorithm:

#. If there is no collision, insert; stop.
#. If there is a collision, rehash ``c`` once. If there is no collision, insert; stop.
#. If there is still a collision, increment ``c = c+1``. If there is no collision, insert; stop.
#. If there is still a collision, go to #3

In pseudocode:

.. code-block:: cpp

	bool hash_insert(uint32_t input)
	{
		uint16_t a, b, c;
		a = hash1(input & 0xff'ff);
		b = hash2(input >> 16);
		c = hash1(a ^ b);

		if(!collides(c))
		{
			insert_at(c);
			return true; 
		}
		else
		{
			c = hash2(c);
			if(!collides(c))
			{
				insert_at(c);
				return true; 
			}
			else
			{
				for(int i = 0; i < SIZE; i++)
				{
					c = c + 1;

					if(!collides(c))
					{
						insert_at(c);
						return true; 
					}
				}
				return false;
			}
		}
	}

Of course, ``c`` should not be allowed to hit the map out of bounds, but otherwise it's a surprisingly
simple principle. 

Unfortunately, because we mitigate collisions in this way, a table of names must be maintained separately
from the data map, so that indexing into the table will tell us if mitigations were applied or not, so we
know whether we've found the variable we want or if it's |ss| in another castle |se| in a different memory
location. This means the table should have a structure like:

.. code-block:: cpp

	struct entry_t
	{
		uint32_t name;		// 4
		uint32_t data;		// +4
	};

	static entry_t name_table[1024];

The astute reader will notice that this is one byte more than the MS-BASIC variable table, `without` the
performance hit of having to loop through an array to find what we want every time a variable is ever used.
Such a table would eat 8K of memory for 1K variables (totalling 4K data) with longer names (four ASCII
characters, or 2 CJK characters). It feels a *bit* like defeat having to still store the name in the
thing, but it's essential to disambiguating between two identically hashed names; basically, we're using
the name as a way to gauge whether the accessor should do a mitigation or not, without wasting too much space and
still preserving some onomastics. As the names are stored as one integer made from concatenated characters,
checking whether it's the right name is a simple single-instruction comparsion, and we're avoiding
complex string routines that would basically be obligatory with a longer variable name. Checking whether anything's been
allocated is also pretty easy: since we know names won't have the highest bit set (remember, we have a ~30-bit key),
simply setting the name to ``0xffff'ffff`` ensures that it's easy to check for whether a 'spot' is taken by simply
doing ``if(int32_t(name) < 0) { /* ... */ }`` without having to worry much. Using a larger table than 1024 is likewise
pretty easy to a point: at 16K entries, the table reaches 128K of memory use, which is near our upper static
allocation limit and should be avoided.

----------------
Slabs and pools
----------------

I haven't really dug deep into slab and pooling because we really don't have too much memory 
to work with in general, so doing stuff in this space would not be quite as efficient 
as it could be if we had more memory to manage. Generally speaking, slab allocation is
most effective when you have an OS that manages memory tightly. Since spinning up and 
destroying pages can be costly, and finding free memory can sometimes be a chore, the kernel
instead allocates a bunch of chunks that can fit a certain object type, and then hands that
memory out when requested and reclaims it when freed, all by adjusting a bookkeeping structure
that shows which parts of slabs are free to use and which occupied. Freeing and allocating memory 
in this case is an illusion: it means just rewriting the appropriate part of the bookkeeping table.
When something is 'destroyed' or reclaimed in a slab allocator, the object structure is kept
cached, so that when another instance is requested there is no set-up involved and the cached
object can be returned basically intact. The objects are usually also constructed ahead of time
and in one go.

Memory pooling is a similar concept, where a large swath of memory is preallocated and then
divided up by a pool allocator. There's bookkeeping and all that jazz, but what separates
it from a slab (though neither is really extremely well-defined) is that there are no
premade objects in pools, and they can store heterogenous data. It's basically a microcosmos
of memory management.

Memory pooling could perhaps work here, but in a way the `idea` of a memory pool is superfluous because
there's no real OS for us to request a lot of memory from and then subdivide it to avoid
the overhead because we're that OS and we have to manage the memory we have in this manner anyway.
That is, all our memory is in memory pools to begin with. Nobody's there to reclaim leaked memory.

The same principles apply, though: we want to keep a small bookkeeping structure that would indicate
which parts of memory are free to use and which are taken. 

As there's very little memory to go around,
a very light bookkeeping structure would be not just optimal, but really necessary. This is balanced
out by the need for space savings and very fine memory granularity. For example, we can
assume that the smallest unit of memory that can be handed out like this would be 8 bytes;
to keep track of every unit, a 64K memory space would need 8K locations keeping storage
status. If we use a bitmap, this means the bookkeeping structure would only have an overhead
of 1K (8 locations per byte for 8 bytes of space per location) plus anything on top (like a
pointer to the structure and whatnot) that would add a handful of extra bytes. Computationally,
this would suck to manage a bit, but if space savings were the main concern this would probably
be the way to go. Less fine-grained memory allocation would in turn allow for more metadata
in the bookkeeping structure while keeping to the same space concerns: if each space manages
16B of data, then 2 bits per location would allow for a total of 4 states to be stored
per item, possibly facilitating things like access control; if each space manages
64B of data, then a full byte is available for this purpose.

Another bookkeeping approach would be to keep a table of what's been allocated, a pointer
to it, and the length. Several other approaches (such as a linked list) also exist, and are,
in my opinion, all better suited for more memory-rich platforms.

------------------------
Naïve without dynamicity
------------------------

Most of the stuff an embedded programme will need will be
statically allocated ahead of time anyway. In this case,
when I say `naïve` I mean that there's no magic behind the
scenes and nothing is playing musical chairs with the memory.
Everything is baked in at compilation time and no deviation
is allowed. This is how most of the "important data" will
live on the chip anyway. A screen buffer, for example, will
be statically allocated as 75K or 150K of RAM with no manager,
and resources like fonts or images or sprites or whatnot
that would be statically burnt to flash are likewise going
to just be this kind of memory: it's basically handling
unmanaged, unscoped static memory with bare code-hands.
There's nothing to talk about in terms of management here,
strictly speaking.

==================
What else?
==================

Really, it's just me staring at the screen blarting out words as they come,
without a coherent idea of where it's going. I'm going to be designing
the BASIC-like over the next few weeks/months (depending on how hard it
holds my attention, really), and will probably be writing some kind of static
hashmap (more sophisticated than the pseudocode above), a stack (possibly
templated for type and depth), and a rudimentary array allocator that would
implement a basic (hehe) heap. I'm not actually sure how useful multi-dimensional
arrays were back in the olden days, though; I can imagine a 2D or 3D array
doing just fine, but do you really need a 37D array when you're not doing
some fancy linear algebra churning vectors (which you shouldn't be doing
on an ESP32 anyway)?

----

.. |ss| raw:: html

   <strike>

.. |se| raw:: html

   </strike>&#8203;

.. |BroFrustration| image:: ../emoji/brofrustration.png
  :width: 32

.. |Moggers| image:: ../emoji/moggers.png
  :width: 32

.. _The docs: https://docs.espressif.com/projects/esp-idf/en/stable/esp32s2/api-guides/memory-types.html 

.. [1] I'm not even exaggerating that much, believe it or not;
	we have a very good idea of the
	`historical performance of the computers of the era <http://www.roylongbottom.org.uk/whetstone.htm>`_,
	with a PDP-11/34 giving 0.204 MWIPS (million Whetstone instructions per second),
	whereas an ESP32 clocked at 240MHz (the default in the CYD) hits
	`around 50 MWIPS <https://github.com/nopnop2002/esp-idf-benchmark>`_, for
	around 250x the floating point performance by this crude benchmark comparsion. Given
	how thoroughly the ESP32's floating point unit can suck, the performance gap
	is probably much, much greater on integer data, which is what
	most stuff on the CYD would be anyway. My personal benchmarks
	give floating point arithmetic around a 12x worse performance
	on the CYD's ESP32 chip (it takes appx. 4.6 ticks to do i32
	addition, compared to appx. 53.5 ticks for f32 addition), if
	we don't account for division (~5.5 / ~ **270** ≈ 50x worse), but
	division is known to be a jagged edge in arithmetic algorithmics.
	It doesn't quite matter what the ticks are, they're the same measure
	for all operations giving a decent baseline to compare these by
	(but if you're wondering what these ticks are, they're a measure of 
	appx. how many ticks, on average, of a GHz clock are needed
	for an operation to finish; a ballpark of 1GHz/4.6 gives us
	the processor's clocked speed at ≈220MHz, which means that
	arithmetic ops have an average throughput of one per cycle,
	while float ops have an average throughput of around one
	per 12 clock cycles).

.. [2] Not immediately, but I do have to say how ESP's development platform,
	called the ESP-IDF (unrelated to the child killer club), is a bit of a
	nuisance to work with and has fairly brittle versioning. It's a shame that
	this is the only way to access more sophisticated tooling for the chips, such
	as partition table editing and (easy) core assignment (you can still do it,
	but it'll be a bit harder in Arduino IDE).

.. [3]  Tactility is cool, although it only serves the capacitative boards and doesn't support
	resistive screen boards. I assume modifying it to work on R-variants would be pretty easy;
	I just haven't bothered, seeing as it wants at least 4MB of PSRAM to make usage not
	miserable; but the project is nonetheless really cool. Building executables for it is also
	not going to be a pleasant experience as it requires a lot of fiddling with ESP-IDF, which
	I've already been burnt twice on. Might be worth looking into further down the line as I explore
	making this into a worthwhile little computer rather than just treating it as a toy as I
	have until now. 

.. [4] I know I said 'bare metal', but it's probably better to think of the ESP32 running a RTOS
	kernel as 'skimpy metal' wearing an inappropriately revealing bikini with no top. You're still
	basically as close to the metal as you can be, and you can certainly just take control over the
	device pretty easily and 'usurp' the kernel and the task manager and do your own thing (and there
	are other kernels available other than RTOS, via ESP-IDF), but the convenience here is that at least
	your *whole* ass isn't out, just both cheeks. It's difficult to overstate how useful it is to
	be handed a codebase that will manage your multitasking on its own! Some drawbacks include the
	watchdog, which will kill any process taking too much time, which may end up killing some code of yours
	running in a loop, so some code pathe engineering won't hurt.

.. [5] The block sizes and types that we can see 
	`in this Github issue <https://github.com/espressif/arduino-esp32/issues/6777#issuecomment-1131658098>`_
	point to a bit more difficult heap situation. In terms of Po2 block sizes, it seems like we can allocate
	4 blocks of 64K, 3 blocks of 32K, and so on, in increasingly smaller chunks. This might be something to
	work around in the future.

.. [6] I didn't really take a look at how MS-BASIC stores the size of each dimension in the array table,
	so this is an estimate, but it's nonetheless documented behaviour that array dimensions can have different
	sizes and that each dimension's length is ∈ [0, 32767], so they obviously must be stored separately.

.. [7] for QB64, this is stuff inherited
	from QuickBASIC, which was a significant step up from MS-BASIC for the 6502 in terms of
	hardware and software capabilities alike. Even though it feels properly rudimentary now,
	MS-DOS (and other DOSes of the time) were proper operating systems with a lot of bells and whistles
	compared to things like the Commodore for which BASIC was the closest it got to an OS at all.
	Even further, QB64 transpiles to C++, which is then piped through GCC, so its dynamic memory
	features depend on code in the STL, predominantly the use of ``std::vector``
