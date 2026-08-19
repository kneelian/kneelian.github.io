.. title: Devblog 0; the "What" and the "Why"
.. slug: devblog-0-the-what-and-the-why
.. date: 2026-08-18 20:54:23 UTC+02:00
.. tags: gamedev, programming, msdos, x86
.. category: 
.. link: 
.. description: 
.. type: text

This post has been a long time coming, and a long time
of me not writing it. Inspired by a recent reddit thread I
participated in on r/gamedev where several users complained
how they'd want to see more actual developer content,
I decided to dust my project off and make a short
write-up. There is no promise of this being a series
of posts or continuing into the future, but I won't mind
writing up some things from time to time.

Most of this is going to be technical babble about the
environment and some design ideas I had. Little real
dev work here!

==========
The Themes
==========

I am not a professional gamedev by any means, so any
gamedev I do is strictly hobbyist pissing around for
fun. This means I can afford myself some more leeway
with what choices I make or what targets I aim for.

I am a big strategy gamer, or at least used to be. 
I grew up playing *WarCraft 3* and *Command & Conquer: Generals*,
was absolutely bedazzled when *Frozen Throne* came
out, and spent years chasing that high. Somewhere around
2011 I fell into the Paradox rabbithole, and currently
have 10,833.6 hours of *Crusader Kings II* logged 
on Steam, and several thousand more outside of it
through hacked or old versions. [1]_

Nowadays, I mostly spin my wheels playing *Dwarf Fortress*,
kind of wishing there was a game that would cater to
my specific narrow interests.

Over the past couple years, I've been toying with the
idea of making a strategy game of some kind that
would do what I wanted it to do. CK2 was *good*, it hit
me like crack, but it still missed the mark somewhat.
I'm really more of a sci-fi guy, and wanted something
in that direction. Space-based or sci-fi-themed
4x and grand strategy games never did exactly what
I wanted them to do: *Stellaris* was too arcadey and
casual about some things, and didn't let you have fine-grained
control over systems or, let alone, planets, and went further
in that direction the more time passed (I still miss the old
*Stellaris* pop systems where each pop would have a physical
location on the planet map); there was *Galactic Civilizations* 
that came close, but it was still too far away; Civ-likes
that were set on single planets just weren't it, either.
The more-than-excellent `Star Dynasties`_ was the closest
I got out of the new stuff out there, with its combination
of a space-based strategy and a dynastic, character-based
system basically directly aping *Crusader Kings* (nothing wrong
with this!), but I didn't like the scale of it: you still
operated in terms of whole systems at once on a scale
that wasn't fine-grained enough.

.. _Star Dynasties: https://store.steampowered.com/app/1194590/Star_Dynasties/

Unfortunately, the actual number of things that really
got what I wanted was roughly next to zero, from what
I could determine. Two games that *did* scratch the
unexplainable itch (or rather one game and one mod)
were the WC3 custom map *Solar Conquest*, where you
could individually control units on several different 
planets and fought for control over each individual planet
with the ability to have partial control over things,
and the now ancient game `Emperor of the Fading Suns`_
which involves dynastic conflict over individual planets
under a vacant interstellar throne.

.. _Emperor of the Fading Suns: https://store.steampowered.com/app/2799350/Emperor_of_the_Fading_Suns_Enhanced/

Given that I now guessed that there would be nothing
to go into that niche (and I doubt that there's a
high enough demand to make someone fill it for me), 
I figured I might want to do some development myself,
knowing full well I'm not likely to release anything
any time soon, or ever.

My 'ideal' game, in this case, would be a space-based
grand strategy game with a dynastic character focus,
primarily or even exclusively dealing with humans and
human factors, with things like sub-system and sub-planet
granularity of control and a feudal system of interdependencies
between characters themselves, and between characters
and titles. Space feudalism, really. How has this never
been done!?

The game will be 2D, since I just like how maps work in
2D (I even turned off the 3D terrain mode in EU5, I
really don't care for the look of it). I will probably
make it turn-based, for one reason or another (mostly
because turn-based games are more intuitive to me and
will be easier to develop) [P]_

It's not really that well-developed in sci-fi literature
either, for one reason or another. Sure, there's *Dune*, but
even then you get less focus on the diplomacy
inherent in feudalism. It's very rare to really stumble
across a piece of media doing what's basically
*A Song of Ice and Fire* in space! But I want to try
and tackle the problem, even if it doesn't result in me
changing that.

==========
The Target
==========

Given that I'm not really limited by commercialisation
of my ideas, I could play around with arbitrary tech
and target whatever I wanted. Given that I picked up
a total shitbox (in today's terms) from a local university
getting rid of its tech, I figured why not target that?

The target for my game was going to be, at least for
the time being, a machine with the following specs:

======== ======== =======================
 ↓ thing  ­←­-­→         spec ↓        
======== ======== =======================
 * CPU             * Pentium III @ 866MHz
 * RAM             * 256MB DDR1
 * GPU             * nVidia GeForce2 MX
 * OS              * Windows 98 SE
======== ======== =======================

We really are cooking with some exotic 
computation gear, aren't we!

Beyond that, I also, for some reason, decided
I wanted to eventually distribute this game on
floppy disks, optimally just one at a time, so
that I could distribute it to my friends and 
go haha guys look how cool this is! You can play
my stupid retro game off a stupid floppy disk!
This means that I'll have to make sure that my
entire game with all its assets fits into
somewhat under 1440KB. [2]_

Given that the system is running Win98, it is
also realistically possible to target MS-DOS
on it, given that Win98 is a DOS-based system.
This simplifies the programming part a lot,
surprisingly, as there are a lot of high-quality
libraries and tools targeting DOS that simply
don't have a match on the Win95/98 side of
things. This also means that I
will not have to do much to test my game, as now
I can run it just fine locally on DOSBox and
expect it to run on hardware as well (and the
inverse direction doesn't need to apply).

The GPU this system has isn't that strong, but
we're not doing 3D graphics anyway, and just need
it to push pixels to the screen. I'm rendering
a 1024x768 screen (how huge!) at 16 bits
per pixel, which the GeForce2 MX takes like
a champ and doesn't stutter a lot. I also don't
want to make a 3D game, and not even just
because we're on MS-DOS: I think a proper
strategy game or wargame can work very well
in a 2D scheme, and gains nothing much from
going 3D. We'll be doing tiles from a tileset,
for this reason.

This set of restrictions feels properly absurd
for today, but I've found it really somewhat
manageable, and limitations do after all breed
innovation.

===========
The Tools
===========

My primary deployment/testing platform is
DOSBox-X, which I use to test my builds.
It's a really decent DOS emulator that aims
for maximal hardware compatibility and tries
to make sure it can run a wide array of
general-purpose DOS applications and not just
games like several other DOSBox forks. It's
stable, and it just so happens to emulate
the exact damn processor my shitbox has,
the Pentium III, down to the 866MHz clock.
In retrospect, this was a very good pick,
though at first I worried whether I could
actually expect matching behaviour. The answer
is a solid and resounding **yes**: everything
I've run so far on DOSBox-X has run identically
on the shitbox, down to the segfault traces
being genuinely byte identical. Isn't emulation
neat! 

Now, I'm a C++ freak. The hammer I wield is C++,
and everything looks like a nail. It's my
go-to for scripting, it's my go-to for
a Discord bot, it's what I use to solve
anything that needs computation done. Thankfully,
doing C++ on MS-DOS is genuinely easy
and not even disgusting to think about.
For this project, I naturally
said *hey, what if I did this in C++*, and
unexpectedly the DOS dev ecosystem has been
pretty clean to work with. I am compiling
code for a 32-bit MS-DOS target using
a toolchain built off DJ Delorie's [3]_
DJGPP, containing GCC 15.2 with C++ support.
This means that, yes, structured bindings
and lambdas Just Work™ on MS-DOS targets,
and that the MS-DOS has a newer compiler
than my actual main machine running Linux Mint 22.3.

DJGPP includes Charles W. Sandmann's DOS extender
``CWSDPMI``, which provides the system with 
virtual memory management and the ability to 
seamlessly run protected mode code on a real
mode OS (such as MS-DOS is, or such as
Win98 pretends to be). Memory protection,
as annoying as it can be, is not a *bad* thing,
and segfaulting on MS-DOS because you touched
a nono array or did a double-free is great
for debugging. [4]_

DJGPP also includes a few more useful treats:

- a port of ``gdb`` for MS-DOS, running
  natively on the system (this is excellent)
- dynamic linker system (``dxegen`` and 
  the associated ``dxe3res``)
- whole bunch of headers for useful
  operations (``file.h`` and even a
  relatively performant ``regex.h``)

It's a surprisingly robust system, and has been
around for longer than I have. Did you know that
DJGPP was what was used to compile ``Quake`` back
in the day!?

Another good thing that DJGPP provides us with
is the ability to have both a natively-running
C/C++ compiler on the MS-DOS itself [5]_ and 
one that works on a modern 64-bit host. Cross-compiling
for the MS-DOS is much, much, *muuuuch* faster than
doing it natively (something on the order of
50x faster is what I've measured), and it
enables me to do a modern type of incremental
build, and also just edit while another thread is
compiling the game. Lots of stuff you can do!

Finally, a very good cherry on top is that
inline assembly and whole-cloth ASM subroutines
can get integrated really easily into the codebase
without much fanfare; I can just point the
compiler at an ``.s`` file and it magically gets
handled just the way I want it :) [6]_

For the rest of the tech, I went with a
trustworthy old friend: the game library
Allegro, with one of the later versions
(but not the very last one) of Allegro 4
still maintaining DOS support. This basically
pushed most of the concerns out of my way,
since Allegro would handle video mode
toggling, input support (mouse and keyboard),
timer interrupts (crucial for building an
animation), and had support for a number
of useful file formats like the ``.pcx``
image format. Allegro also provides good
support for 16-bit colour [7]_ and even has
hand-tuned assembly functions for drawing a
large number of things (lines, circles, pixels)
adjusted for 16-bpp colour mode. Allegro also
has excellent compatibility with DJGPP.

Given that we're linking a large library such
as Allegro into the final game executable,
binary size is understandably a bit awful. 
The ``liballeg.a`` object is by itself 1495KB,
which is already above what we can afford;
aggressive LTO does handle this pretty well,
but the resulting executable still ends up
hovering at around 1.1MB, dangerously close
to the limit, even when compiling with ``-Os``
and stripping as much as possible. 
This isn't a new problem, and developers of the
time also encountered it frequently: the natural
solution is to do what they did, and use
an executable packer. Packers are programs that
take an executable program, compress the majority
of the code with some algorithm, and replace the
entry point with a decompressor stub that would
reconstruct external imports and decompress
the packed executable and jump into it. Packers
of yesteryear weren't quite as sophisticated as
what we have today, but ``upx`` is pretty fresh,
*very* good, and (most importantly) can work
on MS-DOS executables of various formats, including
the .EXEs that DJGPP produces. It claims to
give up to 50% reduction in most cases; 
currently, I'm going from 1127KB to 321KB,
which is ~28.5% of the initial size.

-------------------------------------------
Digression: Microsoft executables and DJGPP
-------------------------------------------

This digression started as a footnote but it grew
way too large to be a footnote (maybe a legnote?).
The DJGPP output format is a bit of a nasty
beast from our modern point of view. 

For a bit of history,
there are actually approximately [8]_ three types of ``.EXE``
files in the history of Microsoft. The first executable
format on the MS-DOS was the ``.COM`` file, which was an
image copy of a single 64K segment of memory with a fixed
(kind of) entry point at ``0x0100`` where DOS would load the
image and jump into it. This meant that ``.COM`` could
be up to ``0xff00`` bytes long (since it must fit into a
single memory segment), which limited it a whole lot,
and had no relocation info. Microsoft's answer to these
limitations was the MZ ``.EXE``, named after Mark
Żbikowśki, one of the devs on MS-DOS and one of the
format's architects. This MZ executable included some
relocation info and allowed executables to span more
than a single segment. In fact, MZ ``.EXE`` is the ancestor
of all its successors, which is why every Windows executable
you can run today will have ``0x4d5a`` (ASCII of "MZ") as
the magic number in the header. 

In any case, the MZ format was decent for MS-DOS, 
but graphical Windowses
ended up needing more cruft :) and so Microsoft produced
the now-ancient *New Executable* which was designed
to run on 16-bit OSes, including later versions of MS-DOS,
but in a 32-bit memory environment. When they made Windows
NT, they also came up with the *Portable Executable*, which
was designed to be started in protected mode on 32-bit OSes
and never dip into real or fakereal, but still include a 
DOS stub that prints the legendary "This program cannot be
run in DOS mode" (and which can be replaced with an actual
DOS program, resulting in a fat binary!). DJGPP, being a
DOS-native toolchain, can't rely on newer executables, so
the programs it produces are, technically, presented as
MZ ``.EXE`` files.

The differences start here.

A now mostly historical object file format of the past
forms the basis of both the DJGPP executable and the 
modern Windows PE. The COFF (common object file format)
object file started on Unix as an attempt at developing
a new format to replace older limited executables (much
like how the MZ ``.EXE`` was made to replace ``.COM`` with
its limits), but was itself quickly replaced by ELF because
it was heavily underspecified (basically every compiler
and linker vendor produced their own microvariants of
COFF files that were mutually incompatible). A variant
of the COFF forms the core of the DJGPP executable. The
toolchain actually generates a MZ ``.EXE`` stub that the
OS loads the proper way, but it includes a COFF payload
that forms the meat of the program.

The way that a DJGPP ``.EXE`` works is that the 16-bit
stub allocates memory for a transfer buffer (section of
memory where the program talks with the DOS host,
located somewhere in 'conventional memory' (the low
640KB of physical RAM)), then tries to set up DPMI
services (first by probing for whether it is already
under DPMI, and if it isn't then trying to load
``CWSDPMI`` either from the current directory or from
the system path), reads the COFF header to see how
much memory it will need, enters protected mode,
loads the executable from the COFF into virtual
memory, allocates selectors for conventional
memory access (for buffers that don't need to touch
the transfer buffer, or for whatever other use that
a DOS program might need conventional memory access
really), sets up ``sbrk``, brings up the stack (giving
you a default of 512K of stack space), hooks signal
and interrupt handlers, sets up the program environment,
passes the commandline arguments, constructs any object
with static storage, and then calls the program's 
``main()`` from the COFF object.

I assume that because COFF is well-known, and MZ
executables aren't that awful to deal with, that ``upx``
has decent results tackling the DJGPP executable.

-----------------------
Space-saving strategies
-----------------------

Whenever you Do Development, you basically just play
many small games of tradeoffs. The main triangle I
had to consider here was the tradeoff between performance,
memory and disk use. 

With the shitbox being the shitbox it was, I expected
that performance would be a pretty big bottleneck. It
wasn't. Other than a few times where I was really stupid
(such as calling a complex function on one pixel, for every
pixel of the framebuffer, for every frame), I haven't really
hit any big performance rough spots. The chip is fairly
good, really; you have access to fast 32-bit floating point
operations with SSE (no hitting the awful slow FPU), to
name one thing. I do expect to eventually also just have to
do some weird optimisations down the line [9]_ when I start
hitting a performance limit, but for now we're doing just
dandy in that regard.

I *didn't* expect to have memory issues. While 256M is
dreadfully small for today's standards, it's positively
monstrous for the MS-DOS. I can basically almost do whatever
I want, though I'll try to keep my memory use within
64MB if I can. I'm so far away from hitting that limit right
now that I can't really imagine how approaching it would even
look like.

That, naturally, leaves disk. Like I mentioned before, aiming
to hit 1.3MB of disk is already making me do things I've never
had to do on a 'modern' system.

One thing that's been fun has been learning about various
file formats, compression and decompression methods etc.
Even though I said the CPU is pretty beefy, I still have
to keep in mind it's a single-core 32-bit chip under a 
gigahertz, so I can't really do some of the sillier stuff
I've done even on a Raspi today. Still, my first default
image format, and the one that Allegro supported out
of the box and was handled just fine by Aseprite, was the
``.pcx``, which is only very slightly compressed (but 
decompresses really easily), and my first big space-saving
operation was to include a ``.png`` decoder (for 17KB of
unpacked executable size, I am saving ~50% on my images).
When I include text-based configs, I will also look
into LZSS since it's not computationally
very expensive, which I know since I'm already doing
the related LZ77 in the ``.png`` decoder and afaik in ``upx``
itself, and since Allegro provides built-in support
for a LZSS unpacker.

Really, spending some time loading and decompressing
or unpacking assets ahead of play is basically free,
and was a beloved tool of many a dev until people
started shipping gigabyte textures |FatHusky| but
that's a topic for a different post, innit.

I still haven't gotten to the audio, but I expect
that I'll ship some small mod playback library
(say, ``libopenmpt`` to play tracker music) to
avoid having to ship actual audio files whenever
I can. Given also the limitations of the FAT12
filesystem, I'll want to avoid making many small files
(which would fill up the root directory if in the
top level directory, or would take up additional space
if in a subdirectory), and would perhaps want to bundle
all my assets in one large file which I could compress
and decompress (hello, reinvention of the tarball),
and which would ease loading. Allegro also already
provides this in the form of the `datafile`_ with
a pretty robust interface and set of manipulation
functions.

.. _datafile: https://liballeg.org/stabledocs/en/alleg031.html

=====================
Development process
=====================

Honestly, I expected development to be much more
painful than it really is.

My actual tech stack is, including some of the
pieces I mentioned above, the following:

- DJGPP toolchain
  - GCC, G++ 15.2 running on a 64-bit Linux host
  - native ``gdb`` running on MS-DOS host
- ``upx`` packer
- ``stb_img`` for format support
- ``make`` to call compiler and packer
- Aseprite for graphics
- VS Code as the IDE (I like having popups telling me what a function wants)
- DOSBox-X for testing

=======================
How does the code look?
=======================

Like honest to God normal C++ code.
If you don't believe me, just take a 
look at this:

.. code-block:: c++
	
	void planet_t::update_minimap()
	{
	    if(minimap == nullptr) { return; }
	    clear_bitmap(minimap);
	    for(uint16_t j = 0; j < PLANET_ROWS; j++)
	        for(uint16_t i = 0; i < PLANET_COLS; i++)
	        {
	            uint16_t colour = 0;
	            switch(map[j][i].tileid)
	            {
	                _CASE(EMTY, EMTYCOL);
	                _CASE(WATR, WATRCOL);
	                _CASE(GRAS, GRASCOL);
	                _CASE(SVNA, SVNACOL);
	                _CASE(DEEP, DEEPCOL);
	                _CASE(SNOW, SNOWCOL);
	                _CASE(MEAT, MEATCOL);
	                _CASE(SAND, SANDCOL);
	                _CASE(VOID, VOIDCOL);
	                _CASE(TOP1, TOP1COL);
	                _CASE(TOP2, TOP2COL);
	                _CASE(ROCK, ROCKCOL);
	                _CASE(MNTN, MNTNCOL);
	                _CASE(SWMP, SWMPCOL);
	                    
	                default:
	                    colour = makecol16(0xff, 0x00, 0x00);
	                    break;
	            }
	            uint8_t x = (i * 3);
	            uint8_t y = (j * 3);
	            if((j % 2) == 0) { x += 1; }
	            rectfill(minimap, x, y, x+2, y+2, colour);
	        }
	    return;
	}

You can just do regular idiomatic C++, there's not a
lot of gotchas. You *don't* have everything you would
want on a modern system, so things like multithreading
aren't in by default (though there exist POSIX thread
library ports to DJGPP if you want them for some
reason), but most things work just fine. Structured
bindings are in, various deductions work fine, 
constructors and destructors are okay, exceptions
work without much of a fuss, etc. With the toolchain
it's less inconvenient to do dev for DOS than it is
for Windows it feels like.

============
What's next?
============

Now, I'm actually a bit along the project, though it's
still in the early stages; I work on it whenever I'm
feeling particularly bored and there's nothing else to
distract me. As I'm loading some configurations from disk,
such as the terrain layouts of planets which are constructed
from a bitmap/tilemap stored in the assets folder,
I've also made some devtools to help me visualise these and
work on them 'in-engine', like a micro level editor. I'll
probably expand these into a bundle of more proper editing
tools that I'll distribute to the end users, to allow them
to make their own planets and maybe even events (if somewhen
in the far future I get to making an event or scripting
system...)

Currently, I have basic screen rendering done, 
tile mouseover detection is there (it's a bit messy
figuring out which tile your mouse is over, actually),
a minimap system is there, as is scrolling the planet map.
Planets are for now cylindrical: they wrap around going east
to west, but are capped off at the poles, and all latitudinal
parallels are of equal length. I don't think I'll actually 
approach making a realistically *spherical* planet; spheres
translate poorly to maps, and I'm a big fan primarily of maps.

If I come back to this devblog series,
I'll be writing first about how I built the animation
system, how I handled hex coordinates and printing
hex tiles, etc. and show some actual screens of the game. 
They're still horrible.

----

.. [1] This adds up to >450 whole days.
	That sounds absolutely devastating, but really
	it's not *that* much spread out over more than
	a decade. I still remember the 2012 patches by
	version number!

.. [2]  A standard 1440K floppy has 2880 sectors of
	512 bytes, and tends to be formatted as a FAT12
	drive. Not all of these are actually usable, as
	the first sector is reserved for drive metadata
	and possible boot code (hence the name boot sector),
	and then there are two file allocation tables (the
	thing that gives FAT12 its name) of 9 sectors each,
	eating up another 18 sectors. The root directory (i.e.
	the directory you see when you open the drive) is treated
	specially and reserves another 14 sectors, which is
	enough to hold 224 files or folders in root (each
	file and folder has 32 bytes of bookkeeping associated
	with it, with 8-character filenames). This leaves 2847 sectors
	for storing data and subfolders, which means
	that ~1420KB of a floppy is realistically usable.
	The driver operates in terms of clusters, and the filesystem
	also references these clusters. These are sectors in the
	"data section" (starting from the first sector after
	the root directory) numbered starting from 0x002.
	A FAT12 drive can maximally index up to 4078 clusters
	as each cluster is identified by three hex digits or
	a 12-bit number, and clusters {0x000, 0x001, 0xff0–0xfff}
	are considered special markers and do not
	refer to physical sectors. Every file on a FAT12 system
	must take up a multiple of 512 bytes on the drive, rounding
	up to the sector boundary: a file that's one byte long
	takes up the same amount of space as a 512-byte file, but
	a 513-byte file now takes up 1KB and has two entries in
	the file table.

.. [3] That's the guy's name! No periods, no abbreviations.
	He genuinely is named DJ.

.. [4] While DJGPP does come with ``CWSDPMI``, 
	other DPMIs can be used just
	fine with it as long as they provide the same
	standardised interface.

.. [5] Since DJGPP GCC can compile C and C++, it can also
	compile itself just fine, and then the resulting compiler
	runs on MS-DOS. I am thinking of maybe using this in the
	future to enable modders to inject code into the game
	directly in some way, by giving them a process that would
	compile custom code into a DXE and loading it at startup.
	Yes, yes, I know, safety, code injection, etc; but, come
	on, it's running on DOS, basically nobody's storing anything 
	sensitive on an MS-DOS anymore, and those that are shouldn't
	be gaming on that machine.

.. [6] There are other projects offering relatively
	modern environments (shout out to DOjs and
	LoveDOS), but I wanted to use my hammer.

.. [7] Once known as "high colour", i.e. 
	65K colour, with 5 bits for red, 6 for
	green and 5 for blue. We have mostly replaced
	656 colour with 888 RGB (sometimes called
	"true"), but this was a period where 656 still
	held good sway, and it's currently still present
	in a lot of embedded devices and low-level graphics
	(like the Raspberry Pi 4 default kernel framebuffer). 

.. [8] There's a few more than the ones mentioned in
	that section. Aside from MZ, NE and PE executables,
	Microsoft also introduced some compatibility for
	executable formats used on the OS/2, of which the
	*linear executable*/LX is the most notable and was
	used for VxD drivers in Windows 3.11, 95 and 98,
	and then some minor LX-derived formats. None of
	these are particularly interesting to us.

.. [9] Specifically, I'm thinking of laying out some
	data in a pseudo-ECS fashion and working it with SIMD
	extensions (be it MMX or SSE) in strides. I suspect
	the fact that the data will be cache-friendly
	will add to the gain extractable from SIMD. I'm not
	at that stage yet, but it gets me giddy when I think
	about manually writing MMX code in 2026.

.. [P] Keep in mind that Paradox games are actually all
	turn-based, just with very fast turns with auto-advancement.
	Each day or hour or whatever is in reality, mechanically, a
	turn, and things resolve at turn boundaries. It's very
	neatly presented in a way that feels like the games are
	real-time, since the intervals the turns represent are
	really just sufficiently small that it doesn't feel like
	each is individually a big deal compared to, say, what
	Civ-like or *Total War* games do with year-like turns.

.. |FatHusky| image:: ../emoji/fat-husky.webp
  :width: 32